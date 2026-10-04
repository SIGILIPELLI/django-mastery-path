---
description: "Test Django apps with TestCase and the test client: model, form and view tests, setUpTestData, assertion helpers, query budgets and test isolation."
---

# 09 · Testing Django Apps

Django's test framework is one of its best features and one of the least used by
beginners. It creates a throwaway database, wraps each test in a transaction, gives you
a fake browser (the test client), and adds assertions that understand HTTP responses,
templates, forms and query counts. This lesson writes a real test suite for the Level 1
reading-list app. Every test below passed on Django 6.1.1; one was deliberately broken to
show what a failure looks like.

## Layout and running

`startapp` creates a single `tests.py`. Once you have more than a handful of tests,
replace it with a package:

```text
books/tests/
├── __init__.py
├── test_models.py
├── test_forms.py
└── test_views.py
```

The runner discovers any file matching `test*.py`. Run everything, one app, one module,
one class or one method:

```bash
python manage.py test
python manage.py test books
python manage.py test books.tests.test_forms
python manage.py test books.tests.test_views.EntryViewTests.test_finish_requires_post
```

Our full run, with `-v 2` to list each test:

```text
Creating test database for alias 'default' ('file:memorydb_default?mode=memory&cache=shared')...
Found 10 test(s).
test_str_includes_title_and_author (books.tests.test_models.EntryModelTests...) ... ok
test_create_redirects_to_detail (books.tests.test_views.EntryViewTests...) ... ok
test_finish_requires_post (books.tests.test_views.EntryViewTests...) ... ok
...
----------------------------------------------------------------------
Ran 10 tests in 0.019s

OK
```

With SQLite, the test database lives in memory, which is why it's so fast. With
PostgreSQL, Django creates a `test_<name>` database and drops it afterwards. Your real
database is never touched.

## Model tests

Test the behaviour you wrote, not Django's:

```python title="books/tests/test_models.py"
from django.test import TestCase
from django.utils import timezone

from books.models import Entry


class EntryModelTests(TestCase):
    def test_str_includes_title_and_author(self):
        entry = Entry(title="Piranesi", author="Susanna Clarke")
        self.assertEqual(str(entry), "Piranesi — Susanna Clarke")

    def test_mark_finished_sets_status_and_date(self):
        entry = Entry.objects.create(title="Piranesi", author="Susanna Clarke")
        entry.mark_finished(rating=5)
        entry.refresh_from_db()
        self.assertEqual(entry.status, Entry.Status.DONE)
        self.assertEqual(entry.finished_on, timezone.localdate())
        self.assertEqual(entry.rating, 5)
```

`refresh_from_db()` matters: it proves the change reached the database, not just the
Python object. (If `mark_finished()` forgot a field in `update_fields`, the in-memory
object would still look right.)

## Form tests

Forms that don't touch the database can use `SimpleTestCase`, which **blocks** database
queries entirely and is faster:

```python title="books/tests/test_forms.py"
from django.test import SimpleTestCase
from books.forms import EntryForm


class EntryFormTests(SimpleTestCase):
    def test_rating_only_allowed_when_finished(self):
        form = EntryForm(data={"title": "T", "author": "A", "status": "reading", "rating": 4})
        self.assertFalse(form.is_valid())
        self.assertEqual(form.errors["rating"], ["Only finished books can be rated."])

    def test_rating_range(self):
        form = EntryForm(data={"title": "T", "author": "A", "status": "done", "rating": 9})
        self.assertFormError(form, "rating", "Rating must be between 1 and 5.")
```

## View tests with the test client

`self.client` is a test client: it builds requests, runs them through the full stack
(middleware, URL resolver, view, templates) and returns the response with extras
attached: `response.context`, `response.templates`, `response.redirect_chain`.

```python title="books/tests/test_views.py"
from django.test import TestCase
from django.urls import reverse

from books.models import Entry


class EntryViewTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.reading = Entry.objects.create(title="Piranesi", author="Susanna Clarke", status="reading")
        cls.want = Entry.objects.create(title="Middlemarch", author="George Eliot")

    def test_list_shows_entries_and_counts(self):
        response = self.client.get(reverse("books:list"))
        self.assertContains(response, "Piranesi")
        self.assertContains(response, "Want to read (1)")

    def test_list_filters_by_status(self):
        response = self.client.get(reverse("books:list"), {"status": "reading"})
        self.assertContains(response, "Piranesi")
        self.assertNotContains(response, "Middlemarch")

    def test_create_redirects_to_detail(self):
        response = self.client.post(reverse("books:create"),
                                    {"title": "Beloved", "author": "Toni Morrison", "status": "want"})
        entry = Entry.objects.get(title="Beloved")
        self.assertRedirects(response, entry.get_absolute_url())

    def test_finish_requires_post(self):
        url = reverse("books:finish", args=[self.reading.pk])
        self.assertEqual(self.client.get(url).status_code, 405)
        self.client.post(url)
        self.reading.refresh_from_db()
        self.assertEqual(self.reading.status, Entry.Status.DONE)

    def test_list_query_count(self):
        for i in range(10):
            Entry.objects.create(title=f"Book {i}", author="X")
        with self.assertNumQueries(2):
            self.client.get(reverse("books:list"))

    def test_missing_entry_is_404(self):
        self.assertEqual(self.client.get(reverse("books:detail", args=[999])).status_code, 404)
```

Notes on what's being used:

- **`setUpTestData`** runs once per class; each test still sees a pristine copy of that
  data, because each test runs inside a transaction that's rolled back. It's much faster
  than `setUp()` for data every test needs. (Django also deep-copies the class attributes
  per test, so `self.reading.refresh_from_db()` in one test can't leak into another.)
- **`assertContains`** checks the status code (200 by default) *and* the text.
- **`assertRedirects`** checks the redirect status and target, and by default fetches
  the target and checks it returns 200.
- **`reverse()`** instead of hard-coded paths, so tests survive URL changes.

### What a failure looks like

We changed the budget in `test_list_query_count` to 1:

```text
AssertionError: 2 != 1 : 2 queries executed, 1 expected
1. SELECT "books_entry"."status" AS "status", COUNT("books_entry"."id") AS "n" FROM "books_entry" GROUP BY 1
2. SELECT "books_entry"."id", "books_entry"."title", "books_entry"."author", ...
```

The failure lists the actual SQL, which is usually enough to see what changed. The
correct budget is 2 (the counts query and the entries query), and it stays 2 with 12
entries, which is the point of creating ten extra rows in that test.

## Logged-in users and permissions

```python
from django.contrib.auth import get_user_model

class OwnerTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        User = get_user_model()
        cls.owner = User.objects.create_user("owner", password="unused-in-tests")
        cls.other = User.objects.create_user("other", password="unused-in-tests")

    def test_other_user_gets_404(self):
        self.client.force_login(self.other)
        ...
```

`force_login()` skips password hashing (which is deliberately slow) and logs the user in
directly. Use `client.login(username=..., password=...)` only when the login process
itself is under test. For speed, many projects also set a fast hasher in test settings:
`PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]`. Never use that
outside tests.

## Choosing a test class

| Class | Database | Use for |
|-------|----------|---------|
| `SimpleTestCase` | blocked | forms without DB lookups, utilities, templates |
| `TestCase` | yes; each test in a rolled-back transaction | almost everything |
| `TransactionTestCase` | yes; tables truncated after each test (slow) | code that depends on real commits: `on_commit` callbacks, `select_for_update` across connections |
| `LiveServerTestCase` | yes, plus a real server thread | browser tests with Playwright or Selenium |

`TestCase` also offers `captureOnCommitCallbacks()` so you can test `transaction.on_commit`
hooks without the slow class (Level 3 · 05).

## Test data: factories over fixtures

JSON fixtures (`fixtures = ["books.json"]`) are brittle: a new required field breaks
every fixture file. Prefer small helper functions or the third-party `factory_boy`
package:

```python
def make_entry(**overrides):
    defaults = {"title": "Untitled", "author": "Anon", "status": Entry.Status.WANT}
    return Entry.objects.create(**(defaults | overrides))
```

Each test then states only the fields it cares about.

## Speed

`python manage.py test --parallel` runs test classes in several processes, each with its
own copy of the test database. `--parallel auto` uses one process per CPU core. Our
10-test suite passed with `--parallel 2`; for a suite this small the start-up cost
outweighs the gain, but on hundreds of tests it can cut minutes. `--keepdb` reuses the
test database between runs (helpful with PostgreSQL and many migrations). `--shuffle`
randomises order, which exposes tests that accidentally depend on each other.

## How It Actually Works

The test runner (`DiscoverRunner`) runs the system checks, then creates test databases
by running **all your migrations** against a fresh database. That's a free check that
your migrations apply from scratch; a broken migration fails the test run before any
test executes.

`TestCase` wraps each **class** in a transaction (for `setUpTestData`) and each **test**
in a nested savepoint. After each test, the savepoint is rolled back, so the next test
sees exactly the class-level data. Rolling back is far cheaper than deleting and
re-inserting rows, which is why `TestCase` is fast and `TransactionTestCase` (which must
truncate tables because the code under test commits for real) is slow. It also means
code inside a `TestCase` never truly commits, so `on_commit` hooks don't fire unless you
capture them.

The test client doesn't open a socket. It builds a WSGI environ dictionary, calls
Django's handler directly and attaches instrumentation: it listens to the
`template_rendered` signal to record which templates rendered with which context. That's
how `response.context` works without any special code in your views.

## Common mistakes

- **Testing Django instead of your code**, e.g. asserting that `CharField` enforces
  `max_length`.
- **Only testing the happy path.** The bugs are in invalid input, missing objects and
  other users' data.
- **Query-budget tests with one row**, which can't catch N+1.
- **Using `setUp` for shared data** and making the suite slow.
- **Hard-coding URLs and IDs** (`/books/1/`); IDs aren't guaranteed between runs.
- **Tests that depend on order**; run with `--shuffle` occasionally.

## Exercise

1. Add the three test modules above to your reading-list project and run them.
2. Add tests for: deleting via GET shows a confirmation and deletes nothing; deleting via
   POST removes the entry; an unknown `?status=` shows all entries.
3. Break `mark_finished()` by removing `"finished_on"` from `update_fields`. Which test
   catches it? Would it be caught without `refresh_from_db()`?
4. Add an owner field to `Entry` (in a branch), then write a test proving one user can't
   see another's entries.
5. Run the suite with `--shuffle` and `--parallel auto`.
