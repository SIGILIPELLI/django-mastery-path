---
description: "Django signals in practice: which operations fire them and which silently don't, registering receivers in ready(), on_commit, and better alternatives."
---

# 04 · Signals — and When Not to Use Them

Signals let one part of a project react to events in another without the two importing
each other: "whenever a `User` is created, create a `Profile`", "whenever a book is
deleted, clear the cache". They're genuinely useful for decoupling reusable apps. They're
also one of the most common sources of hidden, hard-to-debug behaviour in Django
projects, mostly because people assume they fire in situations where they don't. This
lesson shows, by running it, exactly when they fire.

## The built-in signals

| Signal | Sent |
|--------|------|
| `pre_save` / `post_save` | around `Model.save()` |
| `pre_delete` / `post_delete` | around deleting each object |
| `m2m_changed` | when a many-to-many relation changes (`add`, `remove`, `clear`, `set`) |
| `pre_migrate` / `post_migrate` | around `migrate` (auth uses this to create permissions) |
| `request_started` / `request_finished` | around each request |
| `user_logged_in` / `user_logged_out` / `user_login_failed` | in `django.contrib.auth` |

## Connecting a receiver

```python title="catalog/signals.py"
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Book


@receiver(post_save, sender=Book)
def book_saved(sender, instance, created, update_fields, **kwargs):
    ...
```

Receivers must accept `**kwargs`: Django may add arguments in future versions. Always
pass `sender=` so you only hear about the model you care about.

The module must be **imported** for the decorator to run. The standard place is your
app's `AppConfig.ready()`:

```python title="catalog/apps.py"
from django.apps import AppConfig


class CatalogConfig(AppConfig):
    name = "catalog"

    def ready(self):
        from . import signals  # noqa: F401  (connects receivers)
```

## What fires, and what doesn't

We connected receivers to `post_save` and `m2m_changed` for `Book`, then performed a
series of operations. The complete log:

```python
b = Book.objects.create(title="Signal test", author=a, pages=1)
b.pages = 2; b.save(update_fields=["pages"])
Book.objects.filter(pk=b.pk).update(pages=3)
Book.objects.bulk_create([Book(title="Bulk A", ...), Book(title="Bulk B", ...)])
b.tags.add(classic)
```

```text
('post_save', 'Signal test', True, None)
('post_save', 'Signal test', False, frozenset({'pages'}))
('m2m', 'pre_add', [2])
('m2m', 'post_add', [2])
```

Four events from five operations, and the gaps are the important part:

- **`QuerySet.update()` sent nothing.** It's a single SQL `UPDATE`; no instances exist.
- **`bulk_create()` sent nothing**, for the same reason.
- **The tag change sent `m2m_changed`, not `post_save`.** Many-to-many writes go to the
  join table; the book wasn't saved.
- `update_fields` is passed through, so a receiver can ignore saves that didn't touch
  fields it cares about.

Deletion is different: we bulk-created two tags and deleted them with
`Tag.objects.filter(...).delete()`, and `post_delete` fired **once per object**. A
QuerySet delete has to collect objects anyway (for cascades, lesson 01 of Level 2), so it
sends delete signals. (Django 6.1's database-level `DB_CASCADE` skips that collection, so
rows it removes send no signals.)

So "a signal keeps X in sync with the model" is only true if *every* write path goes
through `save()` or `delete()`. A single `update()` in a management command, an admin
action or a data migration quietly breaks the invariant.

Connecting the same function twice is harmless: we did, and the receiver ran once.
Django de-duplicates by function identity (or by an explicit `dispatch_uid`).

## Signals and transactions

Receivers run **synchronously, inside the same transaction**, at the moment the signal is
sent. We simulated a signal that sends an email when a book is created, inside an
`atomic()` block that then failed:

```python
with transaction.atomic():
    Book.objects.create(title="Rolled back", author=a, pages=1)
    transaction.on_commit(lambda: events.append("email sent (on_commit)"))
    events.append("email sent (immediately in signal)")
    raise ValueError("payment failed")
```

```text
['email sent (immediately in signal)']   # and the book does not exist
```

The immediate side effect happened for a book that was rolled back: a customer would
get a confirmation for an order that doesn't exist. The `on_commit` callback correctly
never ran. **Anything with external effects (emails, webhooks, queueing a background task,
clearing a remote cache) belongs in `transaction.on_commit()`**, inside or outside a
signal. In tests, use `TestCase.captureOnCommitCallbacks(execute=True)` to run them.

Receivers also run in the request: a slow receiver makes every save slow, and an
exception in a receiver propagates out of `save()` (use `send_robust()` only for
signals you send yourself, when one failing receiver mustn't stop the others).

## When signals are the right tool

- **Reacting to events from code you don't own.** `user_logged_in` to record login
  history, `post_migrate` to create default data, reacting to a third-party app's models.
- **Reusable apps** that must not import the project's code.
- **Cross-cutting concerns** like cache invalidation, where you accept and test the
  `update()` caveat.

## When they're the wrong tool

When the sender and the receiver are both your code in the same project, an explicit call
is almost always clearer:

```python
# Instead of a post_save receiver that creates a profile somewhere far away...
class UserManager(BaseUserManager):
    def create_user(self, username, email=None, password=None, **extra):
        with transaction.atomic():
            user = super().create_user(username, email, password, **extra)
            Profile.objects.create(user=user)
        return user
```

or a service function (`register_user(...)`) that does the related work in order, inside
a transaction, where a reader can see it. Signals invert control: reading
`Book.objects.create()` gives no hint that six receivers in four apps will run. Debugging
then means searching the codebase for `@receiver`.

## How It Actually Works

A `Signal` object holds a list of receivers, each stored with a lookup key built from
the receiver's identity (or `dispatch_uid`) and the sender's identity. That key is why
connecting the same function twice registers it once. Receivers are held by **weak
reference** by default (`weak=True`), so a receiver defined inside a function can be
garbage-collected and silently stop running; define receivers at module level, or
connect with `weak=False`.

`send(sender, **kwargs)` looks up receivers for that sender (with a cache), then calls
each one in registration order, synchronously, collecting return values. There's no
queue, no thread and no transaction awareness. `Model.save()` sends `pre_save` before
building the SQL and `post_save` after it executes, still inside whatever transaction is
open. `QuerySet.update()` and `bulk_create()` are implemented directly on the SQL
compiler and never create instances, so there's nothing to send.

Since Django 5.0, `asend()` and `asend_robust()` exist for async code, and both sync and
async receivers can be connected; Django adapts between them.

## Common mistakes

- **Assuming `update()`/`bulk_create()` trigger `post_save`.**
- **Sending emails or queueing tasks directly in receivers** instead of `on_commit`.
- **Registering receivers in `models.py` or `__init__.py`** and getting import-order
  problems or duplicate registration. Use `AppConfig.ready()`.
- **Using signals for logic inside one app**, making flow invisible.
- **Receivers without `**kwargs`**, which break when Django adds an argument.
- **Infinite loops:** a `post_save` receiver that calls `instance.save()`.

## Exercise

1. Write a `post_save` receiver that logs book changes, register it in `ready()`, and
   confirm which of `create()`, `save()`, `update()`, `bulk_create()` and `tags.add()`
   trigger it.
2. Make the receiver send an email with the console backend inside `on_commit`, and
   write a test using `captureOnCommitCallbacks`.
3. Replace a "create profile on user creation" signal with an explicit manager method,
   and compare how easy each is to test.
4. Find every `update()` call in a project you work on and check whether any signal
   receiver assumes it will run.
