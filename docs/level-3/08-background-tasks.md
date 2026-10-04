---
description: "Move slow work out of the request: Django 6's tasks framework, what its default backends really do, enqueueing on commit, and when to use Celery."
---

# 08 · Background Work: The Tasks Framework & Celery

Some work doesn't belong in a request: sending email through a slow SMTP server,
generating a PDF, calling a third-party API, resizing images, recalculating statistics.
The user shouldn't wait for it, and a timeout or crash shouldn't lose it. The answer is a
**task queue**: the request records "please do X", returns immediately, and a separate
**worker** process does X.

For most of Django's history that meant a third-party system, usually Celery. Django 6.0
added a built-in **tasks framework**: a standard API for defining and enqueueing tasks,
with pluggable backends. This lesson uses it on Django 6.1.1, shows precisely what the
built-in backends do (one of them runs your task immediately, in the request), and
explains where Celery still fits.

## Defining and enqueueing a task

```python title="catalog/tasks.py"
from django.core.mail import send_mail
from django.tasks import task

from .models import Book


@task
def send_book_added_email(book_id):
    book = Book.objects.get(pk=book_id)
    send_mail(f"New book: {book.title}", f"{book.title} was added.", None, ["team@example.com"])
    return book.title


@task(priority=10, queue_name="default")
def recount(author_id):
    return Book.objects.filter(author_id=author_id).count()


@task(takes_context=True)
def flaky(context, x):
    return {"attempt": context.attempt, "x": x}
```

`@task` turns the function into a `Task` object:

```python
>>> type(send_book_added_email).__name__, send_book_added_email.name, send_book_added_email.priority
('Task', 'send_book_added_email', 0)
>>> result = send_book_added_email.enqueue(3)
>>> type(result).__name__, result.status, result.return_value
('TaskResult', 'SUCCESSFUL', 'Exhalation')
```

`enqueue()` returns a `TaskResult` with an `id`, a `status` (`READY`, `RUNNING`,
`FAILED`, `SUCCESSFUL`), and, once finished, `return_value` and `errors`. You can call
the underlying function directly with `recount.call(1)` (we got `3`, the same as the
enqueued version), which is handy in tests and shells. Tasks with `takes_context=True`
receive a `TaskContext` first; ours reported `{'attempt': 1, 'x': 5}`.

## What actually ran: the default backend

With no `TASKS` setting, Django's default is:

```python
TASKS = {"default": {"BACKEND": "django.tasks.backends.immediate.ImmediateBackend"}}
```

`ImmediateBackend` executes the task **synchronously, inside `enqueue()`**. When we
enqueued the email task, the console email was printed before `enqueue()` returned, and
the result was already `SUCCESSFUL`. That's useful in development and tests, and it means
your code is already written against the task API. But it moves nothing out of the
request: in production with this backend, a slow SMTP server still makes the user wait.

The other built-in backend is `DummyBackend`, which stores enqueued tasks without running
them (useful for asserting "a task was enqueued" in tests). **Django itself ships no
backend that runs tasks in a separate worker.** For real background execution you install
a backend from a third-party package; the `django-tasks` package, where this API was
developed, provides a database-backed queue with a worker command, and other packages
adapt existing queues. Check the package's documentation for which Django versions it
supports before adopting it. We didn't run an external worker for this lesson, so there
is no worker output here.

## Arguments must be serialisable

Tasks are meant to run in another process, possibly much later, so arguments are
serialised (JSON-compatible). Passing a model instance failed immediately:

```python
>>> send_book_added_email.enqueue(Book.objects.get(pk=3))
TypeError: Unsupported type: <class 'catalog.models.Book'>
```

Pass IDs and re-fetch inside the task. That's better anyway: the task sees the data as it
is when it runs, not a stale copy, and has to handle the row having been deleted.

## Failures

We enqueued the email task for a book that doesn't exist:

```python
>>> r = send_book_added_email.enqueue(999)
>>> r.status, r.errors[0].exception_class_path
('FAILED', 'catalog.models.Book.DoesNotExist')
```

`enqueue()` **didn't raise**: the exception was captured in the result and its traceback
logged. That's the right model for background work (the caller has moved on), but it
means failures are silent unless you log, monitor or check results. Decide per task: should
a missing book be an error, or a no-op (`Book.objects.filter(pk=book_id).first()` and
return)?

## Enqueue after commit

The classic bug: a view creates a book in a transaction and enqueues the email; a real
worker picks the task up before the transaction commits, `Book.objects.get()` fails, and
the email never goes out. (Or the transaction rolls back and the email describes a book
that doesn't exist.) Enqueue from `on_commit`:

```python
from functools import partial
from django.db import transaction

def create_book(request):
    ...
    with transaction.atomic():
        book = form.save()
        transaction.on_commit(partial(send_book_added_email.enqueue, book.pk))
```

This is the same rule as Level 3 · 04 and · 05: external effects happen after commit.

## Designing tasks that survive the real world

Workers crash, deploys restart them, networks fail, and most queues deliver
**at least once**. So:

- **Make tasks idempotent.** Running twice must be harmless: check "already sent" before
  sending, use `update_or_create`, record an idempotency key.
- **Keep them small.** One email per task, not one task emailing 10,000 people; fan out.
- **Set timeouts** on network calls inside tasks.
- **Retry deliberately**, with backoff, only for errors that can succeed later
  (timeouts, 503s), not for bugs.
- **Monitor queue length and failure rate.** A growing queue is an outage in slow motion.

## Celery

Celery is the long-established Python task queue: workers, retries with backoff,
scheduled and periodic tasks (`celery beat`), routing, rate limits, chords and chains, and
a large ecosystem. It needs a **broker** (commonly Redis or RabbitMQ) and its own worker
processes. A Celery task looks similar:

```python title="catalog/celery_tasks.py"
from celery import shared_task

@shared_task(bind=True, autoretry_for=(ConnectionError,), retry_backoff=True, max_retries=5)
def send_book_added_email(self, book_id):
    ...
```

and is enqueued with `send_book_added_email.delay(book.pk)`, again from `on_commit`. We
didn't run Celery for this course (it needs a broker service), so no output is shown.

How to choose:

| Situation | Reasonable choice |
|-----------|-------------------|
| New project, simple "do this later" jobs | the built-in tasks API with a production backend package |
| Existing Celery deployment | keep Celery |
| Complex workflows, periodic schedules, heavy routing | Celery (or another mature queue) |
| A tiny site with one nightly job | a management command run by cron (Level 4 · 05) |

Writing new code against `django.tasks` keeps your options open: the backend is a
setting.

## How It Actually Works

`@task` builds a frozen dataclass, `Task`, holding the function, priority, queue name and
backend alias; creating it calls the backend's `validate_task()`. `enqueue()` looks up
the backend through `task_backends` (a connection-handler like `caches` and
`connections`) and calls its `enqueue()`. The task is identified by its **import path**
(`catalog.tasks.send_book_added_email`), which is how a worker in another process finds
the function: tasks must be module-level functions, not closures.

`ImmediateBackend.enqueue()` builds a `TaskResult` in `READY`, sets it to `RUNNING`, calls
the function in the current thread, and records `SUCCESSFUL` with the return value, or
`FAILED` with a `TaskError` holding the exception's class path and traceback; it sends
`task_enqueued`, `task_started` and `task_finished` signals along the way. A
worker-based backend instead serialises the task path and arguments into storage (a
table, a Redis list), and a separate worker process polls, claims a task (for database
queues, typically with `SELECT ... FOR UPDATE SKIP LOCKED`, Level 3 · 05), runs it and
writes the result.

## Common mistakes

- **Shipping `ImmediateBackend` to production** and believing work runs in the background.
- **Passing model instances** instead of IDs.
- **Enqueueing inside a transaction** without `on_commit`.
- **Non-idempotent tasks** under at-least-once delivery: duplicate emails, double charges.
- **Ignoring failed results** because `enqueue()` didn't raise.
- **Giant tasks** that take an hour and lose all progress when a worker restarts.

## Exercise

1. Write `send_book_added_email` as a task and enqueue it from your book-create view via
   `on_commit`. With the console email backend, confirm when the email appears.
2. Switch `TASKS` to `DummyBackend` in a test and assert the task was enqueued with the
   right argument, without sending anything.
3. Make the task idempotent with an `email_sent_at` field on `Book`.
4. Enqueue it for a deleted book. Decide whether that's an error or a no-op, and change
   the code to match.
5. Read the documentation of one worker-backed backend package and list what you'd need
   to deploy it (processes, database tables, settings).
