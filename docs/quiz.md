---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Django Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Which Django architecture description is correct?",
    "options": [
      "URLconf routes to a view; the view uses models and returns a response, often via a template",
      "Controller, view, model",
      "Model renders HTML",
      "Template calls the database"
    ],
    "correctIndex": 0,
    "explanation": "Django calls it MTV."
  },
  {
    "question": "You change a model field. What two commands bring the database up to date?",
    "options": [
      "shell",
      "makemigrations to create the migration, then migrate to apply it",
      "runserver",
      "collectstatic"
    ],
    "correctIndex": 1,
    "explanation": "Commit the migration files."
  },
  {
    "question": "What does this print?\n```python\nclass Author(models.Model):\n    name = models.CharField(max_length=50)\n\nclass Book(models.Model):\n    title = models.CharField(max_length=50)\n    pages = models.IntegerField(default=100)\n    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name=\"books\")\n\n# ... tables created in a throwaway in-memory SQLite database ...\n\na1 = Author.objects.create(name=\"Ada\")\na2 = Author.objects.create(name=\"Bo\")\nBook.objects.create(title=\"X\", pages=100, author=a1)\nBook.objects.create(title=\"Y\", pages=300, author=a1)\nBook.objects.create(title=\"Z\", pages=200, author=a2)\nprint(list(Book.objects.filter(pages__gte=200).order_by('-pages').values_list('title', flat=True)))\n```\n",
    "options": [
      "['Z', 'Y']",
      "['Y', 'Z']",
      "['X', 'Y', 'Z']",
      "['Y']"
    ],
    "correctIndex": 1,
    "explanation": "Descending by pages: Y (300), Z (200). Verified by running it with Django."
  },
  {
    "question": "How many queries does listing books with their authors take, with and without select_related?\n```python\nclass Author(models.Model):\n    name = models.CharField(max_length=50)\n\nclass Book(models.Model):\n    title = models.CharField(max_length=50)\n    pages = models.IntegerField(default=100)\n    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name=\"books\")\n\n# ... tables created in a throwaway in-memory SQLite database ...\n\na1 = Author.objects.create(name=\"Ada\")\na2 = Author.objects.create(name=\"Bo\")\nBook.objects.create(title=\"X\", pages=100, author=a1)\nBook.objects.create(title=\"Y\", pages=300, author=a1)\nBook.objects.create(title=\"Z\", pages=200, author=a2)\nfrom django.test.utils import CaptureQueriesContext\nwith CaptureQueriesContext(connection) as a:\n    [b.author.name for b in Book.objects.all()]\nwith CaptureQueriesContext(connection) as b_:\n    [b.author.name for b in Book.objects.select_related('author')]\nprint(len(a), len(b_))\n```\n",
    "options": [
      "1 1",
      "4 4",
      "3 3",
      "4 1"
    ],
    "correctIndex": 3,
    "explanation": "The N+1 pattern: 1 + one query per book. Verified by running it with Django."
  },
  {
    "question": "What does this print?\n```python\nclass Author(models.Model):\n    name = models.CharField(max_length=50)\n\nclass Book(models.Model):\n    title = models.CharField(max_length=50)\n    pages = models.IntegerField(default=100)\n    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name=\"books\")\n\n# ... tables created in a throwaway in-memory SQLite database ...\n\na1 = Author.objects.create(name=\"Ada\")\na2 = Author.objects.create(name=\"Bo\")\nBook.objects.create(title=\"X\", pages=100, author=a1)\nBook.objects.create(title=\"Y\", pages=300, author=a1)\nBook.objects.create(title=\"Z\", pages=200, author=a2)\nfrom django.db.models import Count\nprint(list(Author.objects.annotate(n=Count('books')).filter(n__gt=1).values_list('name', flat=True)))\n```\n",
    "options": [
      "['Ada', 'Bo']",
      "['Bo']",
      "[]",
      "['Ada']"
    ],
    "correctIndex": 3,
    "explanation": "Annotate with Count then filter on it. Verified by running it with Django."
  },
  {
    "question": "What does this print after the failed block?\n```python\nclass Author(models.Model):\n    name = models.CharField(max_length=50)\n\nclass Book(models.Model):\n    title = models.CharField(max_length=50)\n    pages = models.IntegerField(default=100)\n    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name=\"books\")\n\n# ... tables created in a throwaway in-memory SQLite database ...\n\na1 = Author.objects.create(name=\"Ada\")\na2 = Author.objects.create(name=\"Bo\")\nBook.objects.create(title=\"X\", pages=100, author=a1)\nBook.objects.create(title=\"Y\", pages=300, author=a1)\nBook.objects.create(title=\"Z\", pages=200, author=a2)\nfrom django.db import transaction\ntry:\n    with transaction.atomic():\n        Book.objects.create(title='Tmp', author=a1)\n        raise RuntimeError('x')\nexcept RuntimeError:\n    pass\nprint(Book.objects.count())\n```\n",
    "options": [
      "3",
      "4",
      "Error",
      "0"
    ],
    "correctIndex": 0,
    "explanation": "atomic rolls back the whole block. Verified by running it with Django."
  },
  {
    "question": "What does this print?\n```python\nclass Author(models.Model):\n    name = models.CharField(max_length=50)\n\nclass Book(models.Model):\n    title = models.CharField(max_length=50)\n    pages = models.IntegerField(default=100)\n    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name=\"books\")\n\n# ... tables created in a throwaway in-memory SQLite database ...\n\na1 = Author.objects.create(name=\"Ada\")\na2 = Author.objects.create(name=\"Bo\")\nBook.objects.create(title=\"X\", pages=100, author=a1)\nBook.objects.create(title=\"Y\", pages=300, author=a1)\nBook.objects.create(title=\"Z\", pages=200, author=a2)\nfrom django.db.models import F\nBook.objects.filter(author=a1).update(pages=F('pages')*2)\nprint(sorted(Book.objects.values_list('pages', flat=True)))\n```\n",
    "options": [
      "[200, 400, 600]",
      "[200, 200, 600]",
      "[100, 200, 300]",
      "[100, 200, 600]"
    ],
    "correctIndex": 1,
    "explanation": "F() does the arithmetic in the database for Ada's two books. Verified by running it with Django."
  },
  {
    "question": "A user edits /orders/42/ for an order owned by someone else and it works. What's missing?",
    "options": [
      "CSRF",
      "HTTPS",
      "Object-level authorization: filter the queryset (or check ownership) by request.user",
      "A template"
    ],
    "correctIndex": 2,
    "explanation": "Login isn't authorization."
  },
  {
    "question": "A form POST is submitted twice when the user refreshes. What pattern prevents it?",
    "options": [
      "Caching",
      "Sessions",
      "Post/Redirect/Get: redirect after a successful POST",
      "More validation"
    ],
    "correctIndex": 2,
    "explanation": "Use redirect()."
  },
  {
    "question": "Why must AUTH_USER_MODEL be chosen at project start?",
    "options": [
      "Speed",
      "Swapping the user model later is painful because migrations and foreign keys already point at the default",
      "Style",
      "Law"
    ],
    "correctIndex": 1,
    "explanation": "Start with a custom user model."
  },
  {
    "question": "What's the right way to send an email after saving an order?",
    "options": [
      "Use a signal with sleep",
      "Send before saving",
      "Send inline in the view",
      "Enqueue a background task via transaction.on_commit so it runs only after the commit"
    ],
    "correctIndex": 3,
    "explanation": "Keep requests fast and consistent."
  },
  {
    "question": "A signal handler hides a bug because nobody knows it runs. What's the guidance?",
    "options": [
      "Use more signals",
      "Use signals sparingly; prefer explicit calls for core business logic",
      "Disable signals",
      "Put logic in templates"
    ],
    "correctIndex": 1,
    "explanation": "Hidden control flow."
  },
  {
    "question": "DRF serializer validation passes but a business rule across fields is violated. Where do you enforce it?",
    "options": [
      "In the serializer's validate() method (and in the model/service layer)",
      "In the URL",
      "In the template",
      "Nowhere"
    ],
    "correctIndex": 0,
    "explanation": "Raise ValidationError."
  },
  {
    "question": "An API list endpoint is slow with nested authors. What do you check?",
    "options": [
      "Query count: use select_related/prefetch_related in the viewset's queryset",
      "Bigger server",
      "More workers",
      "Cache everything"
    ],
    "correctIndex": 0,
    "explanation": "N+1 hides in serializers."
  },
  {
    "question": "Which setting must be False in production and why?",
    "options": [
      "STATIC_URL",
      "DEBUG, because it exposes settings, queries and tracebacks",
      "USE_TZ",
      "ROOT_URLCONF"
    ],
    "correctIndex": 1,
    "explanation": "Run manage.py check --deploy."
  },
  {
    "question": "How should the SECRET_KEY be supplied?",
    "options": [
      "Git history",
      "settings.py",
      "README",
      "From an environment variable or secret manager, never committed"
    ],
    "correctIndex": 3,
    "explanation": "Rotate if leaked."
  },
  {
    "question": "A migration renames a column used by running code during a rolling deploy. What's safe?",
    "options": [
      "Lock the table",
      "Expand/contract: add the new column, backfill, switch code, then drop the old one in a later release",
      "Rename in one step",
      "Drop and recreate"
    ],
    "correctIndex": 1,
    "explanation": "Never break running versions."
  },
  {
    "question": "What belongs behind Nginx or a CDN in production?",
    "options": [
      "Migrations",
      "Tests",
      "Django runserver",
      "Static and media files, with Gunicorn serving the Django app"
    ],
    "correctIndex": 3,
    "explanation": "runserver is for development."
  },
  {
    "question": "Caching a view per URL leaks one user's data to another. Why?",
    "options": [
      "The response varies by user but the cache key doesn't; key on user or avoid caching private pages",
      "Redis",
      "Cache bug",
      "DNS"
    ],
    "correctIndex": 0,
    "explanation": "Vary correctly."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Eliminate N+1 queries

report_slow() issues one query per book. Implement report_fast() to return the same strings in a single query with select_related, and implement counts() to return (author, number of books) using annotate and Count. The script prints query counts for both.

**Starter code**

```python
import sys, types
shop = types.ModuleType("shop"); shop.__path__ = ["."]; sys.modules["shop"] = shop
import django
from django.conf import settings
settings.configure(DATABASES={"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": ":memory:"}},
                   INSTALLED_APPS=["shop"], DEFAULT_AUTO_FIELD="django.db.models.BigAutoField")
django.setup()
from django.db import models, connection, transaction
from django.db.models import F, Count
from django.test.utils import CaptureQueriesContext

class Author(models.Model):
    name = models.CharField(max_length=50)
    class Meta: app_label = "shop"

class Book(models.Model):
    title = models.CharField(max_length=50)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")
    class Meta: app_label = "shop"

with connection.schema_editor() as se:
    se.create_model(Author); se.create_model(Book)
for n in ["Ada", "Bo", "Cy"]:
    a = Author.objects.create(name=n)
    for i in range(2):
        Book.objects.create(title=f"{n}-{i}", author=a)

def report_slow():
    return [f"{b.title} by {b.author.name}" for b in Book.objects.all()]

def report_fast():
    # same output as report_slow but in ONE query (avoid N+1)
    return []

def counts():
    # [(author name, number of books)] ordered by name, computed in the database
    return []

with CaptureQueriesContext(connection) as slow:
    a = report_slow()
with CaptureQueriesContext(connection) as fast:
    b = report_fast()
print("slow queries:", len(slow), "fast queries:", len(fast), "same output:", a == b)
print(counts())
```

**Expected output**

```text
slow queries: 7 fast queries: 1 same output: True
[('Ada', 2), ('Bo', 2), ('Cy', 2)]
```

<details><summary>Show solution and explanation</summary>

<pre><code>import sys, types
shop = types.ModuleType(&quot;shop&quot;); shop.__path__ = [&quot;.&quot;]; sys.modules[&quot;shop&quot;] = shop
import django
from django.conf import settings
settings.configure(DATABASES={&quot;default&quot;: {&quot;ENGINE&quot;: &quot;django.db.backends.sqlite3&quot;, &quot;NAME&quot;: &quot;:memory:&quot;}},
                   INSTALLED_APPS=[&quot;shop&quot;], DEFAULT_AUTO_FIELD=&quot;django.db.models.BigAutoField&quot;)
django.setup()
from django.db import models, connection, transaction
from django.db.models import F, Count
from django.test.utils import CaptureQueriesContext

class Author(models.Model):
    name = models.CharField(max_length=50)
    class Meta: app_label = &quot;shop&quot;

class Book(models.Model):
    title = models.CharField(max_length=50)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name=&quot;books&quot;)
    class Meta: app_label = &quot;shop&quot;

with connection.schema_editor() as se:
    se.create_model(Author); se.create_model(Book)
for n in [&quot;Ada&quot;, &quot;Bo&quot;, &quot;Cy&quot;]:
    a = Author.objects.create(name=n)
    for i in range(2):
        Book.objects.create(title=f&quot;{n}-{i}&quot;, author=a)

def report_slow():
    return [f&quot;{b.title} by {b.author.name}&quot; for b in Book.objects.all()]

def report_fast():
    return [f&quot;{b.title} by {b.author.name}&quot; for b in Book.objects.select_related(&quot;author&quot;)]

def counts():
    return list(Author.objects.annotate(n=Count(&quot;books&quot;)).order_by(&quot;name&quot;).values_list(&quot;name&quot;, &quot;n&quot;))

with CaptureQueriesContext(connection) as slow:
    a = report_slow()
with CaptureQueriesContext(connection) as fast:
    b = report_fast()
print(&quot;slow queries:&quot;, len(slow), &quot;fast queries:&quot;, len(fast), &quot;same output:&quot;, a == b)
print(counts())
</code></pre>

<p>Accessing b.author for each book triggers an extra query unless the relation is joined up front. annotate(Count) pushes per-author counting into the database. Verified by running the solution with Django.</p>
</details>

### An atomic money transfer

Implement transfer(src, dst, amount): in one transaction subtract with F() from the source, raise InsufficientFunds if it would go negative (rolling everything back), otherwise add to the destination with F().

**Starter code**

```python
import sys, types
shop = types.ModuleType("shop"); shop.__path__ = ["."]; sys.modules["shop"] = shop
import django
from django.conf import settings
settings.configure(DATABASES={"default": {"ENGINE": "django.db.backends.sqlite3", "NAME": ":memory:"}},
                   INSTALLED_APPS=["shop"], DEFAULT_AUTO_FIELD="django.db.models.BigAutoField")
django.setup()
from django.db import models, connection, transaction
from django.db.models import F, Count
from django.test.utils import CaptureQueriesContext

class Account(models.Model):
    name = models.CharField(max_length=20)
    balance = models.IntegerField()
    class Meta: app_label = "shop"

with connection.schema_editor() as se:
    se.create_model(Account)
Account.objects.create(name="a", balance=100)
Account.objects.create(name="b", balance=50)

class InsufficientFunds(Exception):
    pass

def transfer(src, dst, amount):
    # in ONE transaction: subtract from src and add to dst using F() expressions;
    # if src would go negative raise InsufficientFunds so everything rolls back
    pass

def balances():
    return dict(Account.objects.order_by("name").values_list("name", "balance"))

transfer("a", "b", 30)
print(balances())
try:
    transfer("a", "b", 500)
except InsufficientFunds as e:
    print("rejected:", e)
print(balances())
```

**Expected output**

```text
{'a': 70, 'b': 80}
rejected: a
{'a': 70, 'b': 80}
```

<details><summary>Show solution and explanation</summary>

<pre><code>import sys, types
shop = types.ModuleType(&quot;shop&quot;); shop.__path__ = [&quot;.&quot;]; sys.modules[&quot;shop&quot;] = shop
import django
from django.conf import settings
settings.configure(DATABASES={&quot;default&quot;: {&quot;ENGINE&quot;: &quot;django.db.backends.sqlite3&quot;, &quot;NAME&quot;: &quot;:memory:&quot;}},
                   INSTALLED_APPS=[&quot;shop&quot;], DEFAULT_AUTO_FIELD=&quot;django.db.models.BigAutoField&quot;)
django.setup()
from django.db import models, connection, transaction
from django.db.models import F, Count
from django.test.utils import CaptureQueriesContext

class Account(models.Model):
    name = models.CharField(max_length=20)
    balance = models.IntegerField()
    class Meta: app_label = &quot;shop&quot;

with connection.schema_editor() as se:
    se.create_model(Account)
Account.objects.create(name=&quot;a&quot;, balance=100)
Account.objects.create(name=&quot;b&quot;, balance=50)

class InsufficientFunds(Exception):
    pass

def transfer(src, dst, amount):
    with transaction.atomic():
        Account.objects.filter(name=src).update(balance=F(&quot;balance&quot;) - amount)
        if Account.objects.get(name=src).balance &lt; 0:
            raise InsufficientFunds(src)
        Account.objects.filter(name=dst).update(balance=F(&quot;balance&quot;) + amount)

def balances():
    return dict(Account.objects.order_by(&quot;name&quot;).values_list(&quot;name&quot;, &quot;balance&quot;))

transfer(&quot;a&quot;, &quot;b&quot;, 30)
print(balances())
try:
    transfer(&quot;a&quot;, &quot;b&quot;, 500)
except InsufficientFunds as e:
    print(&quot;rejected:&quot;, e)
print(balances())
</code></pre>

<p>transaction.atomic rolls back the debit when InsufficientFunds is raised, and F() expressions avoid read-modify-write races. On a production database you would also lock the rows with select_for_update(). Verified by running the solution with Django (SQLite).</p>
</details>
