---
title: "Django Interview Questions and Answers (179 Questions)"
description: "179 Django interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Django Mastery Path."
---

# Django Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Django Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Django Is: The Request/Response Cycle](level-1/01-what-django-is.md)

**Q1.** What does 'batteries included' mean for Django?

**Answer:** It ships with ORM, admin, auth, forms, templates, migrations and security features out of the box

A full-stack framework.

**Q2.** Which architecture pattern does Django follow?

**Answer:** MTV: Model, Template, View

Django's view is the controller-like piece.

**Q3.** What does the request/response cycle look like?

**Answer:** URLconf routes to a view, which returns an HttpResponse (often rendering a template)

urls.py maps paths to views.

**Q4.** What is an app in Django?

**Answer:** A reusable Python package with models, views and templates inside a project

A project has many apps.

### [02 · Installing Django &amp; Your First Project](level-1/02-project-setup.md)

**Q5.** What does `python manage.py runserver` do?

**Answer:** Starts the development server

Not for production.

**Q6.** What does `startapp` create?

**Answer:** A new app package with models.py, views.py and more

Add it to INSTALLED_APPS.

**Q7.** Why use a virtual environment?

**Answer:** Isolate project dependencies

Pin with requirements.

**Q8.** Which file holds project settings?

**Answer:** settings.py

Use environment variables for secrets.

### [03 · URLs and Views](level-1/03-urls-views.md)

**Q9.** What does `path('books/<int:pk>/', views.detail)` do?

**Answer:** Captures an integer from the URL and passes it to the view as pk

Path converters.

**Q10.** What must a view return?

**Answer:** An HttpResponse (or subclass)

render() returns one.

**Q11.** Why name URL patterns (name='detail')?

**Answer:** So you can reverse them instead of hard-coding paths

reverse('detail', args=[1]).

**Q12.** What does `include('books.urls')` do?

**Answer:** Delegates a URL prefix to another URLconf

Keeps apps modular.

### [04 · Templates: Inheritance, Tags, Filters &amp; Autoescaping](level-1/04-templates.md)

**Q13.** How does Django's template language escape variables?

**Answer:** Automatically HTML-escapes output by default

Use |safe sparingly.

**Q14.** What does `{% extends 'base.html' %}` do?

**Answer:** Inherits a base template and overrides its blocks

Template inheritance.

**Q15.** Where should logic live?

**Answer:** In views/models, with templates kept to presentation

Keep templates simple.

**Q16.** What does `{% for b in books %}` do?

**Answer:** Loops over a list in the template

Use {% empty %}.

### [05 · Models and Migrations](level-1/05-models-migrations.md)

**Q17.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
print(Book.objects.count(), Author.objects.get(name='Ada').books.count())
```

**Answer:** 3 2

related_name gives the reverse accessor. Verified by running it with Django.

**Q18.** What does `makemigrations` do?

**Answer:** Generates migration files from model changes

migrate applies them.

**Q19.** What does `migrate` do?

**Answer:** Applies migrations to the database

Tracks applied ones in django_migrations.

**Q20.** Why commit migrations to version control?

**Answer:** They're the shared history of the schema

Teammates apply the same changes.

### [06 · Querying with the ORM](level-1/06-orm-basics.md)

**Q21.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
print(list(Book.objects.filter(pages__gt=150).order_by('title').values_list('title', flat=True)))
```

**Answer:** ['Y', 'Z']

Field lookups like __gt. Verified by running it with Django.

**Q22.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
b,created=Book.objects.get_or_create(title='X', defaults={'author':a1})
b2,c2=Book.objects.get_or_create(title='New', defaults={'author':a1})
print(created, c2, Book.objects.count())
```

**Answer:** False True 4

get_or_create returns (obj, created). Verified by running it with Django.

**Q23.** When does a QuerySet hit the database?

**Answer:** When it's evaluated (iterated, sliced with step, list(), count(), etc.)

QuerySets are lazy.

**Q24.** What does `Model.objects.get()` raise if no row matches?

**Answer:** Model.DoesNotExist

MultipleObjectsReturned if many.

### [07 · The Django Admin](level-1/07-admin.md)

**Q25.** What do you need to expose a model in the admin?

**Answer:** Register it with admin.site.register or @admin.register

ModelAdmin customizes.

**Q26.** What does `createsuperuser` do?

**Answer:** Creates an admin user with all permissions

Use strong passwords.

**Q27.** What does `list_display` control?

**Answer:** Columns shown in the admin changelist

Also list_filter, search_fields.

**Q28.** Is the admin meant for end-user UIs?

**Answer:** No, it's an internal back-office tool

Build real views for users.

### [08 · Forms, ModelForms &amp; CSRF](level-1/08-forms.md)

**Q29.** What does `form.is_valid()` do?

**Answer:** Runs validation and populates cleaned_data and errors

Then save().

**Q30.** What's a ModelForm?

**Answer:** A form generated from a model's fields

Also saves the instance.

**Q31.** What does {% csrf_token %} protect against?

**Answer:** Cross-site request forgery on POST forms

Required for POST.

**Q32.** What should a successful POST do?

**Answer:** Redirect (Post/Redirect/Get) to avoid resubmission

Use redirect().

### [09 · Static Files &amp; User Uploads](level-1/09-static-media.md)

**Q33.** What's the difference between static and media files?

**Answer:** Static are your assets (CSS/JS); media are user-uploaded files

STATIC_URL vs MEDIA_URL.

**Q34.** What does `collectstatic` do?

**Answer:** Gathers static files into STATIC_ROOT for production serving

Serve via a web server/CDN.

**Q35.** Should Django serve media in production?

**Answer:** No: use a web server, object storage or CDN

Dev-only helpers.

**Q36.** What does `{% load static %}` enable?

**Answer:** The {% static %} tag for asset URLs

Needed in each template.

### [10 · Project — A Reading-List App](level-1/10-project-reading-list.md)

**Q37.** What does the reading list project combine?

**Answer:** Models, views, templates, forms and admin in one CRUD app

End to end.

**Q38.** How should you test the app?

**Answer:** Django's test client against views and models

Automate.

**Q39.** What's a good structure for CRUD views?

**Answer:** List, detail, create, update and delete with named URLs

RESTful-ish.

**Q40.** What should you validate?

**Answer:** User input via forms and model constraints

Server-side validation.

## Level 2 · Intermediate

### [01 · Model Relationships: ForeignKey, ManyToMany &amp; OneToOne](level-2/01-relationships.md)

**Q41.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
print(sorted(a1.books.values_list('title', flat=True)), Book.objects.get(title='Z').author.name)
```

**Answer:** ['X', 'Y'] Bo

Forward and reverse relations. Verified by running it with Django.

**Q42.** What does on_delete=CASCADE do?

**Answer:** Deletes related objects when the referenced object is deleted

PROTECT blocks; SET_NULL nulls.

**Q43.** When use ManyToManyField?

**Answer:** When each side can relate to many of the other (e.g. tags)

Creates a join table.

**Q44.** What is a OneToOneField for?

**Answer:** Extending a model with exactly one related record (e.g. profile)

Unique ForeignKey.

### [02 · Advanced QuerySets: Q, F, annotate &amp; aggregate](level-2/02-advanced-querysets.md)

**Q45.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.db.models import Count, Avg, F, Q
r=Author.objects.annotate(n=Count('books'), avg=Avg('books__pages')).order_by('name').values_list('name','n','avg')
print(list(r))
```

**Answer:** [('Ada', 2, 200.0), ('Bo', 1, 200.0)]

annotate aggregates per row. Verified by running it with Django.

**Q46.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.db.models import F, Q
Book.objects.update(pages=F('pages')+10)
print(sorted(Book.objects.values_list('pages', flat=True)), Book.objects.filter(Q(pages__lt=120)|Q(title='Z')).count())
```

**Answer:** [110, 210, 310] 2

F updates in the database; Q combines with OR. Verified by running it with Django.

**Q47.** Why use F() for increments?

**Answer:** It avoids race conditions by updating in the database

read-modify-write is unsafe.

**Q48.** What does `exclude()` do?

**Answer:** Returns rows that do NOT match the condition

Chain with filter.

### [03 · Finding and Fixing N+1 Queries](level-2/03-n-plus-one.md)

**Q49.** How many queries do the two loops run?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.test.utils import CaptureQueriesContext
with CaptureQueriesContext(connection) as c1:
    for b in Book.objects.all(): b.author.name
with CaptureQueriesContext(connection) as c2:
    for b in Book.objects.select_related('author'): b.author.name
print(len(c1), len(c2))
```

**Answer:** 4 1

N+1: one query for books plus one per author lookup; select_related joins. Verified by running it with Django.

**Q50.** When use prefetch_related instead of select_related?

**Answer:** For many-to-many and reverse foreign keys (separate query joined in Python)

select_related is for single-valued relations.

**Q51.** How do you detect N+1 queries?

**Answer:** Query counts in tests, django-debug-toolbar or logging SQL

assertNumQueries.

**Q52.** What does `only()`/`defer()` do?

**Answer:** Load fewer columns

Reduce data transferred.

### [04 · Class-Based &amp; Generic Views](level-2/04-class-based-views.md)

**Q53.** What do class-based views offer?

**Answer:** Reusable behavior via inheritance and mixins (ListView, DetailView...)

Pick by clarity.

**Q54.** What does `get_queryset()` customize?

**Answer:** The rows a view operates on

Filter by user, for example.

**Q55.** What does `get_context_data()` do?

**Answer:** Adds extra data to the template context

Call super().

**Q56.** Which mixin requires login for a CBV?

**Answer:** LoginRequiredMixin

Put it first in MRO.

### [05 · Authentication: Login, Logout &amp; Sign-up](level-2/05-authentication.md)

**Q57.** What does `authenticate()` do?

**Answer:** Checks credentials and returns a user or None

Then login(request, user).

**Q58.** How are passwords stored?

**Answer:** Salted hashes (PBKDF2 by default)

Never stored raw.

**Q59.** What does `@login_required` do?

**Answer:** Redirects anonymous users to the login page

LOGIN_URL.

**Q60.** What does `request.user.is_authenticated` tell you?

**Answer:** Whether the request comes from a logged-in user

Anonymous otherwise.

### [06 · Permissions, Groups &amp; Object-Level Access](level-2/06-permissions.md)

**Q61.** What's the difference between a permission and a group?

**Answer:** A permission allows an action on a model; groups bundle permissions for users

user.has_perm().

**Q62.** Why check object-level access in views?

**Answer:** Model permissions don't restrict which rows a user can see

Filter queryset by owner.

**Q63.** What does PermissionRequiredMixin require?

**Answer:** A permission string like 'app.change_model'

Raises 403.

**Q64.** What's least privilege here?

**Answer:** Give users only the permissions they need

Review regularly.

### [07 · The Custom User Model](level-2/07-custom-user-model.md)

**Q65.** Why set AUTH_USER_MODEL early?

**Answer:** Swapping the user model mid-project is hard because of migrations and foreign keys

Do it at the start.

**Q66.** What should a custom user subclass?

**Answer:** AbstractUser (or AbstractBaseUser for full control)

Keep admin compatibility.

**Q67.** Why reference users via settings.AUTH_USER_MODEL?

**Answer:** So foreign keys work with a custom user model

Use get_user_model() in code.

**Q68.** What does USERNAME_FIELD define?

**Answer:** The field used as the unique identifier at login

e.g. email.

### [08 · Sessions, Messages &amp; Middleware](level-2/08-sessions-middleware.md)

**Q69.** What is middleware?

**Answer:** Hooks that process requests and responses globally

Order matters.

**Q70.** Where are sessions stored by default?

**Answer:** In the database (django_session), with a session id cookie

Configurable backends.

**Q71.** Why is SESSION_COOKIE_SECURE useful?

**Answer:** Sends the cookie only over HTTPS

Plus HttpOnly.

**Q72.** What does the middleware order imply?

**Answer:** Request goes top to bottom, response bottom to top

Position matters for auth/session.

### [09 · Testing Django Apps](level-2/09-testing.md)

**Q73.** What does Django's TestCase provide?

**Answer:** A test database with each test wrapped in a transaction that's rolled back

Isolation.

**Q74.** How do you test a view?

**Answer:** self.client.get('/url/') and assert status and content

Test client.

**Q75.** What should you test?

**Answer:** Behavior including permissions, validation and edge cases

Meaningful tests.

**Q76.** What does assertNumQueries check?

**Answer:** The number of database queries a block executes

Catches N+1.

### [10 · Project — A Multi-User Task Board](level-2/10-project-task-board.md)

**Q77.** What does the task board project show?

**Answer:** Relationships, permissions, CBVs and tests together

A fuller app.

**Q78.** How do you restrict tasks to their owner?

**Answer:** Filter the queryset by request.user in get_queryset

Server-side.

**Q79.** Why add tests for permissions?

**Answer:** Authorization bugs are security bugs

Test denial cases.

**Q80.** What belongs in model methods vs views?

**Answer:** Business rules in models/services; HTTP concerns in views

Fat models, thin views.

## Level 3 · Advanced

### [01 · Django REST Framework: Serializers](level-3/01-drf-serializers.md)

**Q81.** What does a DRF serializer do?

**Answer:** Converts model instances to JSON and validates incoming data

Like forms for APIs.

**Q82.** Why use ModelSerializer?

**Answer:** It generates fields and validators from a model

Customize as needed.

**Q83.** Where should extra validation go?

**Answer:** validate_&lt;field&gt; or validate() in the serializer

Raise ValidationError.

**Q84.** What is a nested serializer?

**Answer:** A serializer embedded for related objects

Mind query counts.

### [02 · DRF Views, ViewSets &amp; Routers](level-3/02-drf-viewsets.md)

**Q85.** What does a ViewSet with a Router provide?

**Answer:** CRUD endpoints and URLs from one class

DefaultRouter.

**Q86.** What does ModelViewSet include?

**Answer:** list, create, retrieve, update, partial_update and destroy

Customize with mixins.

**Q87.** What does @action do?

**Answer:** Adds extra custom routes to a ViewSet

detail=True/False.

**Q88.** How do you paginate?

**Answer:** Set a pagination class in settings or per view

Avoid huge payloads.

### [03 · API Auth, Permissions, Throttling &amp; Pagination](level-3/03-drf-auth-permissions.md)

**Q89.** What does TokenAuthentication/JWT provide for APIs?

**Answer:** Stateless credential per request instead of session cookies

Choose per client.

**Q90.** What's the DRF default permission to tighten?

**Answer:** Set DEFAULT_PERMISSION_CLASSES (e.g. IsAuthenticated)

Deny by default.

**Q91.** What does IsAuthenticatedOrReadOnly do?

**Answer:** Allows reads to anyone, writes to authenticated users

Useful for public APIs.

**Q92.** Why check object permissions?

**Answer:** Authenticated users shouldn't edit others' objects

has_object_permission.

### [04 · Signals — and When Not to Use Them](level-3/04-signals.md)

**Q93.** What do signals let you do?

**Answer:** React to events like post_save in decoupled code

Use sparingly.

**Q94.** What's a downside of heavy signal use?

**Answer:** Hidden control flow that's hard to trace and test

Prefer explicit calls.

**Q95.** When does post_save fire?

**Answer:** After a model instance's save()

created flag.

**Q96.** Do bulk_create and queryset.update fire post_save?

**Answer:** No, they bypass save() and its signals

Be aware.

### [05 · Transactions, Constraints &amp; Race Conditions](level-3/05-transactions-integrity.md)

**Q97.** What does this print after the failed block?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.db import transaction
try:
    with transaction.atomic():
        Book.objects.create(title='Tmp', author=a1)
        raise ValueError('boom')
except ValueError:
    pass
print(Book.objects.filter(title='Tmp').exists())
```

**Answer:** False

atomic rolls back on exceptions. Verified by running it with Django.

**Q98.** When do you use select_for_update()?

**Answer:** To lock rows inside a transaction and avoid lost updates

Requires atomic.

**Q99.** What does ATOMIC_REQUESTS do?

**Answer:** Wraps each request's view in a transaction

Trade-offs.

**Q100.** What does transaction.on_commit do?

**Answer:** Runs a function only after the transaction commits (e.g. enqueue a task)

Avoid tasks seeing uncommitted data.

### [06 · Migrations in Depth: Data Migrations &amp; Zero-Downtime Changes](level-3/06-migrations-in-depth.md)

**Q101.** Why never edit an applied migration in shared repos?

**Answer:** Others have already applied it; add a new migration instead

History is shared.

**Q102.** What's a data migration (RunPython) for?

**Answer:** Transforming data alongside schema changes

Provide reverse functions.

**Q103.** How do you avoid downtime on a column rename?

**Answer:** Multi-step: add new, backfill, switch code, drop old

Expand/contract.

**Q104.** What does `squashmigrations` do?

**Answer:** Combines many migrations into one

Keeps history manageable.

### [07 · Caching](level-3/07-caching.md)

**Q105.** What does Django's cache framework cache?

**Answer:** Arbitrary data, view output or template fragments, via pluggable backends

Redis/Memcached.

**Q106.** What's the hard part of caching?

**Answer:** Invalidation

Keys and timeouts.

**Q107.** Why not use LocMemCache in multi-process production?

**Answer:** Each process has its own cache and they won't share

Use a shared backend.

**Q108.** When is per-user data unsafe to cache per URL?

**Answer:** When responses vary by user: key on user or use Vary headers

Avoid leaks.

### [08 · Background Work: The Tasks Framework &amp; Celery](level-3/08-background-tasks.md)

**Q109.** Why run work in a background task queue (Celery/RQ)?

**Answer:** Keep requests fast and handle retries for slow jobs like email

Don't block the request.

**Q110.** What should task arguments be?

**Answer:** Simple serializable values like ids, not model instances

Reload inside the task.

**Q111.** Why make tasks idempotent?

**Answer:** Retries may run them more than once

At-least-once delivery.

**Q112.** What's a broker?

**Answer:** Message transport between the app and workers (e.g. Redis, RabbitMQ)

Plus result backend.

### [09 · Async Django: ASGI, Async Views &amp; the Async ORM](level-3/09-async-django.md)

**Q113.** When is async Django useful?

**Answer:** I/O-bound work like calling external APIs concurrently

ASGI server needed.

**Q114.** What happens when sync ORM code is called in an async view?

**Answer:** It must be wrapped (sync_to_async) or use async ORM methods

Avoid blocking the loop.

**Q115.** What server runs async Django?

**Answer:** An ASGI server like Uvicorn or Daphne

WSGI is sync.

**Q116.** Is async always faster?

**Answer:** No: CPU-bound or DB-heavy code doesn't benefit

Measure.

### [10 · Project — A REST API for the Task Board](level-3/10-project-rest-api.md)

**Q117.** What does the REST API project combine?

**Answer:** DRF serializers, viewsets, auth, tests and pagination

Complete API.

**Q118.** How should errors be returned?

**Answer:** Consistent status codes with a clear JSON body

DRF exceptions.

**Q119.** How do you document the API?

**Answer:** OpenAPI schema (e.g. drf-spectacular)

Keep in sync.

**Q120.** How do you version an API?

**Answer:** URL or header versioning

Avoid breaking clients.

## Level 4 · Master

### [01 · Security: What Django Protects and What It Can't](level-4/01-security.md)

**Q121.** Which Django feature mitigates XSS by default?

**Answer:** Template auto-escaping

Avoid |safe on user input.

**Q122.** What prevents SQL injection in the ORM?

**Answer:** Parameterized queries (never build raw SQL with string formatting)

Use params in raw().

**Q123.** What does `manage.py check --deploy` do?

**Answer:** Flags insecure production settings

HTTPS, HSTS, DEBUG.

**Q124.** What does clickjacking protection use?

**Answer:** X-Frame-Options via XFrameOptionsMiddleware

Default on.

### [02 · Production Settings &amp; Configuration](level-4/02-production-settings.md)

**Q125.** What must be False in production?

**Answer:** DEBUG

DEBUG leaks details.

**Q126.** Where should SECRET_KEY come from?

**Answer:** An environment variable or secret manager

Rotate if leaked.

**Q127.** Why set ALLOWED_HOSTS?

**Answer:** Prevents Host header attacks

List your domains.

**Q128.** How should settings be split?

**Answer:** Per-environment overrides with shared base and env vars

12-factor.

### [03 · Deploying Django: Gunicorn, Uvicorn, Static Files &amp; PostgreSQL](level-4/03-deployment.md)

**Q129.** What does gunicorn do in deployment?

**Answer:** Runs the WSGI app with worker processes behind a reverse proxy

Nginx in front.

**Q130.** Why put Nginx (or a CDN) in front?

**Answer:** TLS, buffering and static/media serving

Offload.

**Q131.** What does a health check endpoint do?

**Answer:** Lets orchestrators know the app is alive/ready

Used by load balancers.

**Q132.** What's needed before serving new code?

**Answer:** Run migrations and collectstatic in the release process

Automate.

### [04 · Database Performance: Indexes, EXPLAIN &amp; Query Budgets](level-4/04-database-performance.md)

**Q133.** What does `EXPLAIN` help with?

**Answer:** Seeing how the database executes a query (indexes, scans)

queryset.explain().

**Q134.** When add db_index=True?

**Answer:** On columns used often in filters/joins/order_by

Indexes cost writes.

**Q135.** What does `.iterator()` help with?

**Answer:** Streaming large result sets without caching all rows

Memory.

**Q136.** Why use bulk_create/bulk_update?

**Answer:** Fewer queries for many rows

Beware signals/save().

### [05 · Custom Management Commands](level-4/05-management-commands.md)

**Q137.** What are management commands for?

**Answer:** Custom CLI tasks like imports and maintenance via manage.py

BaseCommand.handle().

**Q138.** Where do they live?

**Answer:** app/management/commands/&lt;name&gt;.py

Discovered automatically.

**Q139.** What should long commands be?

**Answer:** Idempotent and resumable with logging

Operational.

**Q140.** How do you accept arguments?

**Answer:** add_arguments with argparse

self.stdout.write for output.

### [06 · Custom Template Tags, Filters &amp; Template Partials](level-4/06-template-tags-partials.md)

**Q141.** What do custom template tags/filters offer?

**Answer:** Reusable presentation logic in templates

Register in templatetags/.

**Q142.** What are template partials/includes for?

**Answer:** Reusing fragments across templates

{% include %}.

**Q143.** What belongs in a template tag?

**Answer:** Presentation helpers, not business logic

Keep simple.

**Q144.** What does `simple_tag` do?

**Answer:** Defines a tag returning a value or string

inclusion_tag renders a template.

### [07 · Logging, Error Reporting &amp; Health Checks](level-4/07-logging-errors.md)

**Q145.** How should you log errors in production?

**Answer:** Use the logging framework with handlers and structured context, plus error tracking

Sentry etc.

**Q146.** What does the 'django.request' logger capture?

**Answer:** 4xx/5xx request errors

Configure LOGGING.

**Q147.** Why avoid logging secrets and PII?

**Answer:** Logs are widely accessible and persistent

Redact.

**Q148.** What does ADMINS/mail_admins do?

**Answer:** Emails errors to admins (legacy; prefer trackers)

Limited.

### [08 · Time Zones &amp; Internationalization](level-4/08-i18n-timezones.md)

**Q149.** Why store datetimes in UTC (USE_TZ=True)?

**Answer:** Avoid ambiguity across time zones and DST

Convert at display.

**Q150.** What does `gettext`/_() do?

**Answer:** Marks strings for translation

makemessages/compilemessages.

**Q151.** What's the difference between TIME_ZONE and a user's zone?

**Answer:** TIME_ZONE is the default; activate per user for display

timezone.activate().

**Q152.** What does LocaleMiddleware do?

**Answer:** Picks the language per request

Order after Session.

### [09 · Upgrading Django &amp; Handling Deprecations](level-4/09-upgrading.md)

**Q153.** What's a safe upgrade approach?

**Answer:** Read release notes, move through supported versions, fix deprecation warnings, run tests

Run with -W error.

**Q154.** Why prefer LTS versions in production?

**Answer:** Longer security support

Plan upgrades.

**Q155.** What do deprecation warnings tell you?

**Answer:** What will break in a future version

Fix early.

**Q156.** What protects upgrades?

**Answer:** A good test suite and CI

Confidence.

### [10 · Capstone — A Production-Ready Book Club App](level-4/10-capstone.md)

**Q157.** What does the capstone integrate?

**Answer:** Models, views, API, auth, tests, caching/tasks and deployment practices

Production app.

**Q158.** How do you prove readiness?

**Answer:** Tests, deploy checks, monitoring and rollback

Operate.

**Q159.** What documentation helps?

**Answer:** README, architecture, runbooks and API docs

Onboarding.

**Q160.** What guards against regressions?

**Answer:** CI running tests and linters on every change

Automation.

## Mixed questions across all levels

**Q161.** Which Django architecture description is correct?

**Answer:** URLconf routes to a view; the view uses models and returns a response, often via a template

Django calls it MTV.

**Q162.** You change a model field. What two commands bring the database up to date?

**Answer:** makemigrations to create the migration, then migrate to apply it

Commit the migration files.

**Q163.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
print(list(Book.objects.filter(pages__gte=200).order_by('-pages').values_list('title', flat=True)))
```

**Answer:** ['Y', 'Z']

Descending by pages: Y (300), Z (200). Verified by running it with Django.

**Q164.** How many queries does listing books with their authors take, with and without select_related?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.test.utils import CaptureQueriesContext
with CaptureQueriesContext(connection) as a:
    [b.author.name for b in Book.objects.all()]
with CaptureQueriesContext(connection) as b_:
    [b.author.name for b in Book.objects.select_related('author')]
print(len(a), len(b_))
```

**Answer:** 4 1

The N+1 pattern: 1 + one query per book. Verified by running it with Django.

**Q165.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.db.models import Count
print(list(Author.objects.annotate(n=Count('books')).filter(n__gt=1).values_list('name', flat=True)))
```

**Answer:** ['Ada']

Annotate with Count then filter on it. Verified by running it with Django.

**Q166.** What does this print after the failed block?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.db import transaction
try:
    with transaction.atomic():
        Book.objects.create(title='Tmp', author=a1)
        raise RuntimeError('x')
except RuntimeError:
    pass
print(Book.objects.count())
```

**Answer:** 3

atomic rolls back the whole block. Verified by running it with Django.

**Q167.** What does this print?

```python
class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    title = models.CharField(max_length=50)
    pages = models.IntegerField(default=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")

# ... tables created in a throwaway in-memory SQLite database ...

a1 = Author.objects.create(name="Ada")
a2 = Author.objects.create(name="Bo")
Book.objects.create(title="X", pages=100, author=a1)
Book.objects.create(title="Y", pages=300, author=a1)
Book.objects.create(title="Z", pages=200, author=a2)
from django.db.models import F
Book.objects.filter(author=a1).update(pages=F('pages')*2)
print(sorted(Book.objects.values_list('pages', flat=True)))
```

**Answer:** [200, 200, 600]

F() does the arithmetic in the database for Ada's two books. Verified by running it with Django.

**Q168.** A user edits /orders/42/ for an order owned by someone else and it works. What's missing?

**Answer:** Object-level authorization: filter the queryset (or check ownership) by request.user

Login isn't authorization.

**Q169.** A form POST is submitted twice when the user refreshes. What pattern prevents it?

**Answer:** Post/Redirect/Get: redirect after a successful POST

Use redirect().

**Q170.** Why must AUTH_USER_MODEL be chosen at project start?

**Answer:** Swapping the user model later is painful because migrations and foreign keys already point at the default

Start with a custom user model.

**Q171.** What's the right way to send an email after saving an order?

**Answer:** Enqueue a background task via transaction.on_commit so it runs only after the commit

Keep requests fast and consistent.

**Q172.** A signal handler hides a bug because nobody knows it runs. What's the guidance?

**Answer:** Use signals sparingly; prefer explicit calls for core business logic

Hidden control flow.

**Q173.** DRF serializer validation passes but a business rule across fields is violated. Where do you enforce it?

**Answer:** In the serializer's validate() method (and in the model/service layer)

Raise ValidationError.

**Q174.** An API list endpoint is slow with nested authors. What do you check?

**Answer:** Query count: use select_related/prefetch_related in the viewset's queryset

N+1 hides in serializers.

**Q175.** Which setting must be False in production and why?

**Answer:** DEBUG, because it exposes settings, queries and tracebacks

Run manage.py check --deploy.

**Q176.** How should the SECRET_KEY be supplied?

**Answer:** From an environment variable or secret manager, never committed

Rotate if leaked.

**Q177.** A migration renames a column used by running code during a rolling deploy. What's safe?

**Answer:** Expand/contract: add the new column, backfill, switch code, then drop the old one in a later release

Never break running versions.

**Q178.** What belongs behind Nginx or a CDN in production?

**Answer:** Static and media files, with Gunicorn serving the Django app

runserver is for development.

**Q179.** Caching a view per URL leaks one user's data to another. Why?

**Answer:** The response varies by user but the cache key doesn't; key on user or avoid caching private pages

Vary correctly.

## Practical coding questions

### Eliminate N+1 queries

report_slow() issues one query per book. Implement report_fast() to return the same strings in a single query with select_related, and implement counts() to return (author, number of books) using annotate and Count. The script prints query counts for both.

Starter code:

```
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

Expected output:

```text
slow queries: 7 fast queries: 1 same output: True
[('Ada', 2), ('Bo', 2), ('Cy', 2)]
```

??? success "Show a solution"

    ```
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
        return [f"{b.title} by {b.author.name}" for b in Book.objects.select_related("author")]

    def counts():
        return list(Author.objects.annotate(n=Count("books")).order_by("name").values_list("name", "n"))

    with CaptureQueriesContext(connection) as slow:
        a = report_slow()
    with CaptureQueriesContext(connection) as fast:
        b = report_fast()
    print("slow queries:", len(slow), "fast queries:", len(fast), "same output:", a == b)
    print(counts())
    ```

    Accessing b.author for each book triggers an extra query unless the relation is joined up front. annotate(Count) pushes per-author counting into the database. Verified by running the solution with Django.

### An atomic money transfer

Implement transfer(src, dst, amount): in one transaction subtract with F() from the source, raise InsufficientFunds if it would go negative (rolling everything back), otherwise add to the destination with F().

Starter code:

```
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

Expected output:

```text
{'a': 70, 'b': 80}
rejected: a
{'a': 70, 'b': 80}
```

??? success "Show a solution"

    ```
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
        with transaction.atomic():
            Account.objects.filter(name=src).update(balance=F("balance") - amount)
            if Account.objects.get(name=src).balance < 0:
                raise InsufficientFunds(src)
            Account.objects.filter(name=dst).update(balance=F("balance") + amount)

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

    transaction.atomic rolls back the debit when InsufficientFunds is raised, and F() expressions avoid read-modify-write races. On a production database you would also lock the rows with select_for_update(). Verified by running the solution with Django (SQLite).

---

More free practice: the [Django Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
