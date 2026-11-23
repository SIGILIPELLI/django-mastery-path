---
title: "Django Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Django cheat sheet: 24 quick-reference tables from every level of the Django Mastery Path, each linked to its full lesson."
---

# Django Cheat Sheet

Quick-reference tables from **24 lessons** of the free Django Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Django Is: The Request/Response Cycle](level-1/01-what-django-is.md)

*The parts you get*

| Part | What it does | Where you'll meet it |
|------|--------------|----------------------|
| URL dispatcher | Maps a path like `/books/3/` to a Python function | Lesson 03 |
| Views | Functions or classes that take a request and return a response | Lesson 03, Level 2 · 04 |
| Template engine | Renders HTML with inheritance and auto-escaping | Lesson 04 |
| ORM + migrations | Python classes for tables; generated, versioned schema changes | Lessons 05–06 |
| Admin | A full CRUD back office generated from your models | Lesson 07 |
| Forms | Parse, validate and re-display user input | Lesson 08 |
| Auth | Users, password hashing, login, permissions, groups | Level 2 · 05–07 |
| Middleware | Hooks that wrap every request (sessions, CSRF, security headers) | Level 2 · 08 |

### [02 · Installing Django &amp; Your First Project](level-1/02-project-setup.md)

*The commands you'll use most*

| Command | What it does |
|---------|--------------|
| `runserver` | Development server with auto-reload |
| `startapp NAME` | Create an app skeleton |
| `makemigrations` | Generate migration files from model changes |
| `migrate` | Apply migrations to the database |
| `shell` | Python REPL with Django configured |
| `createsuperuser` | Create an admin user |
| `check` | Run the system checks (config errors, admin errors, deploy checks) |
| `test` | Run the test suite |

### [03 · URLs and Views](level-1/03-urls-views.md)

*Capturing values with path converters*

| Converter | Matches | Python type |
|-----------|---------|-------------|
| `str` (default) | anything except `/` | `str` |
| `int` | `0` or a positive integer | `int` |
| `slug` | letters, digits, hyphens, underscores | `str` |
| `uuid` | a formatted UUID | `uuid.UUID` |
| `path` | anything, including `/` | `str` |

### [04 · Templates: Inheritance, Tags, Filters &amp; Autoescaping](level-1/04-templates.md)

*Tags and filters you'll use daily*

| Tags | Purpose |
|------|---------|
| `{% if %} {% elif %} {% else %}` | branching; supports `and`, `or`, `not`, `in`, comparisons |
| `{% for x in items %} ... {% empty %}` | loops; `forloop.counter`, `forloop.first`, `forloop.last` |
| `{% url 'name' arg %}` | reverse a URL |
| `{% csrf_token %}` | hidden CSRF field in POST forms (lesson 08) |
| `{% with total=items|length %}` | name an intermediate value |
| `{% load static %}` then `{% static 'path' %}` | static file URLs (lesson 09) |
| `{% comment %}` / `{# ... #}` | comments that never reach the browser |

### [05 · Models and Migrations](level-1/05-models-migrations.md)

*Choosing field types*

| Data | Field | Notes |
|------|-------|-------|
| Short text | `CharField(max_length=...)` | `max_length` is required |
| Long text | `TextField()` | no length limit in the database |
| Whole numbers | `IntegerField`, `PositiveIntegerField`, `BigIntegerField` | positive fields add a `CHECK (>= 0)` |
| Money | `DecimalField(max_digits, decimal_places)` | never `FloatField` for money |
| Dates/times | `DateField`, `DateTimeField` | `auto_now_add` sets on create, `auto_now` on every save |
| True/false | `BooleanField(default=False)` | |
| Fixed set of values | `CharField(choices=...)` with `TextChoices` | `get_status_display()` returns the label |
| Email / URL | `EmailField`, `URLField` | `CharField` plus form validation |
| Structured data | `JSONField` | supported on all built-in backends |
| Files | `FileField`, `ImageField` | lesson 09 |

### [06 · Querying with the ORM](level-1/06-orm-basics.md)

*Managers and QuerySets*

| Method | Returns | SQL idea |
|--------|---------|----------|
| `all()` | QuerySet | `SELECT *` |
| `filter(**lookups)` | QuerySet | `WHERE ...` |
| `exclude(**lookups)` | QuerySet | `WHERE NOT (...)` |
| `order_by("-pages")` | QuerySet | `ORDER BY pages DESC` |
| `values("title")` / `values_list("title", flat=True)` | QuerySet of dicts / tuples / values | select specific columns |
| `get(**lookups)` | one object | exactly one row or an exception |
| `first()` / `last()` | object or `None` | `LIMIT 1` |
| `count()` | int | `SELECT COUNT(*)` |
| `exists()` | bool | `SELECT 1 ... LIMIT 1` |

### [07 · The Django Admin](level-1/07-admin.md)

*Configuring a `ModelAdmin`*

| Option | Effect |
|--------|--------|
| `list_display` | columns on the change list (fields, methods, or callables) |
| `list_filter` | the filter sidebar; works on choices, booleans, dates and relations |
| `search_fields` | the search box; `author__name` searches across the join |
| `list_select_related` | controls which relations are joined into the list query |
| `autocomplete_fields` | replaces a `<select>` of every author with an AJAX search box |
| `filter_horizontal` | a two-pane picker for many-to-many fields |
| `inlines` | edit related rows (reviews) on the parent's page |
| `actions` | bulk operations on selected rows |

### [10 · Project — A Reading-List App](level-1/10-project-reading-list.md)

*What we observed*

| Action | Result |
|--------|--------|
| `GET /` on an empty database | 200, shows "Add your first book" |
| POST a *reading* book with rating 4 | 200, form re-shown: "Only finished books can be rated." |
| POST the same book without a rating | 302 → `/1/` |
| `GET /1/finish/` | **405** (the view is POST-only) |
| `POST /1/finish/` | 302; status became `done`, `finished_on` set |
| Edit with rating 9 | "Rating must be between 1 and 5." |
| `GET /` after adding a second, unread book | tabs read `Want to read (1)`, `Reading (0)`, `Finished (1)` |
| `GET /?status=bogus` | 200, all entries listed, no tab highlighted |
| `GET /2/delete/` then `POST /2/delete/` | confirmation page, then 302 → `/`; one entry left |
| `GET /999/` | 404 |

## Level 2 · Intermediate

### [01 · Model Relationships: ForeignKey, ManyToMany &amp; OneToOne](level-2/01-relationships.md)

*`on_delete` — decide what deletion means*

| Option | When the author is deleted... | Use for |
|--------|-------------------------------|---------|
| `CASCADE` | their books are deleted too | parts that can't exist alone (a review without its book) |
| `PROTECT` | deletion raises `ProtectedError` | important records that must be dealt with first |
| `RESTRICT` | like `PROTECT`, but allows deletion if the object is also being deleted via another `CASCADE` path | complex graphs |
| `SET_NULL` | `author` becomes `NULL` (needs `null=True`) | optional links, keep history |
| `SET_DEFAULT` / `SET(...)` | set to a default or computed value | reassign to a placeholder |
| `DO_NOTHING` | nothing in Django; the database decides | only with explicit DB-level handling |

### [04 · Class-Based &amp; Generic Views](level-2/04-class-based-views.md)

*The core generic views*

| View | Does | Default template | Context |
|------|------|------------------|---------|
| `ListView` | list a QuerySet, optional pagination | `<app>/<model>_list.html` | `object_list`, `<model>_list`, `page_obj`, `paginator`, `is_paginated` |
| `DetailView` | one object by `pk` or `slug` | `<app>/<model>_detail.html` | `object`, `<model>` |
| `CreateView` | ModelForm; save; redirect | `<app>/<model>_form.html` | `form` |
| `UpdateView` | ModelForm bound to an object | `<app>/<model>_form.html` | `form`, `object` |
| `DeleteView` | confirm on GET, delete on POST | `<app>/<model>_confirm_delete.html` | `object` |
| `TemplateView` | render a template | `template_name` (required) | URL kwargs |
| `RedirectView` | redirect | — | — |
| `FormView` | a non-model form | `template_name` | `form` |

### [05 · Authentication: Login, Logout &amp; Sign-up](level-2/05-authentication.md)

*What the login view does (observed)*

| Request | Result |
|---------|--------|
| POST wrong password | 200, form re-rendered with "Please enter a correct username and password. Note that both fields may be case-sensitive." |
| POST correct password, `next=/cbv/new/` | 302 → `/cbv/new/` |
| POST correct password, `next=https://evil.example/` | 302 → `/accounts/profile/` (the default `LOGIN_REDIRECT_URL`) |

### [07 · The Custom User Model](level-2/07-custom-user-model.md)

*Referencing the user model*

| Where | Use |
|-------|-----|
| Model fields | `models.ForeignKey(settings.AUTH_USER_MODEL, ...)` (a string, safe at import time) |
| Everywhere else (views, forms, tests) | `from django.contrib.auth import get_user_model; User = get_user_model()` |
| Built-in forms | `UserCreationForm` and `UserChangeForm` work with custom models that keep `AbstractUser`'s fields; subclass them and set `Meta.model` if you added required fields |

### [08 · Sessions, Messages &amp; Middleware](level-2/08-sessions-middleware.md)

*Where the data lives*

| Engine | Stores data in | Notes |
|--------|----------------|-------|
| `db` (default) | database | simple; needs periodic `clearsessions` |
| `cache` | your cache (e.g. Redis) | fast; sessions lost if the cache is flushed |
| `cached_db` | cache, with database fallback | a good production default |
| `signed_cookies` | the cookie itself | no server storage; data visible to the user; size-limited; can't be revoked server-side |
| `file` | files on disk | rarely the right choice with several servers |

### [09 · Testing Django Apps](level-2/09-testing.md)

*Choosing a test class*

| Class | Database | Use for |
|-------|----------|---------|
| `SimpleTestCase` | blocked | forms without DB lookups, utilities, templates |
| `TestCase` | yes; each test in a rolled-back transaction | almost everything |
| `TransactionTestCase` | yes; tables truncated after each test (slow) | code that depends on real commits: `on_commit` callbacks, `select_for_update` across connections |
| `LiveServerTestCase` | yes, plus a real server thread | browser tests with Playwright or Selenium |

## Level 3 · Advanced

### [01 · Django REST Framework: Serializers](level-3/01-drf-serializers.md)

*Representing relationships*

| Field | Renders as | Accepts on write |
|-------|------------|------------------|
| `PrimaryKeyRelatedField` (default for FKs) | `2` | an ID |
| `SlugRelatedField(slug_field="name")` | `"sci-fi"` | a name |
| `StringRelatedField` | `str(obj)` | read-only |
| `HyperlinkedRelatedField` | a URL | a URL |
| a nested serializer | a full object | read-only unless you write `create()`/`update()` |

### [02 · DRF Views, ViewSets &amp; Routers](level-3/02-drf-viewsets.md)

*What the API did*

| Request | Status | Response (trimmed) |
|---------|--------|--------------------|
| `GET /api/books/` | 200 | `{"count": 7, "next": "http://testserver/api/books/?page=2", "previous": null, "results": [...]}` |
| `GET /api/books/?search=chiang&ordering=-pages` | 200 | titles: `Exhalation`, `Stories of Your Life and Others` |
| `GET /api/books/?page=9` | 404 | `{"detail": "Invalid page."}` |
| `GET /api/books/3/` | 200 | keys include `reviews` and `average_rating` (detail serializer) |
| `GET /api/books/stats/` | 200 | `{"done": 2, "reading": 1, "want": 4}` |
| `POST /api/books/` | 403 | `{"detail": "Authentication credentials were not provided."}` |
| authenticated `POST /api/books/` | 201 | the new book, `"id": 8` |
| authenticated `PATCH /api/books/8/` with `{"pages": "lots"}` | 400 | `{"pages": ["A valid integer is required."]}` |
| authenticated `POST /api/books/8/finish/` | 200 | `"status": "done"`, `"published": null` |
| authenticated `DELETE /api/books/8/` | 204 | — |

### [03 · API Auth, Permissions, Throttling &amp; Pagination](level-3/03-drf-auth-permissions.md)

*Permissions: what may they do?*

| Built-in | Allows |
|----------|--------|
| `AllowAny` | everyone (the default if you set nothing) |
| `IsAuthenticated` | logged-in users |
| `IsAuthenticatedOrReadOnly` | anyone can read; writes need login |
| `IsAdminUser` | `is_staff` users |
| `DjangoModelPermissions` | Django's add/change/delete model permissions (Level 2 · 06) |

### [04 · Signals — and When Not to Use Them](level-3/04-signals.md)

*The built-in signals*

| Signal | Sent |
|--------|------|
| `pre_save` / `post_save` | around `Model.save()` |
| `pre_delete` / `post_delete` | around deleting each object |
| `m2m_changed` | when a many-to-many relation changes (`add`, `remove`, `clear`, `set`) |
| `pre_migrate` / `post_migrate` | around `migrate` (auth uses this to create permissions) |
| `request_started` / `request_finished` | around each request |
| `user_logged_in` / `user_logged_out` / `user_login_failed` | in `django.contrib.auth` |

### [07 · Caching](level-3/07-caching.md)

*Configuring a backend*

| Backend | Shared between processes/servers? | Use |
|---------|-----------------------------------|-----|
| `LocMemCache` | **no**, each process has its own | development, tests |
| `RedisCache` (built in since Django 4.0) | yes | the usual production choice |
| `PyMemcacheCache` | yes | production; simple and fast |
| `DatabaseCache` | yes | when you can't run a cache server; slower |
| `FileBasedCache` | per machine | rarely |
| `DummyCache` | — | disables caching (useful in tests) |

### [08 · Background Work: The Tasks Framework &amp; Celery](level-3/08-background-tasks.md)

*Celery*

| Situation | Reasonable choice |
|-----------|-------------------|
| New project, simple "do this later" jobs | the built-in tasks API with a production backend package |
| Existing Celery deployment | keep Celery |
| Complex workflows, periodic schedules, heavy routing | Celery (or another mature queue) |
| A tiny site with one nightly job | a management command run by cron (Level 4 · 05) |

### [10 · Project — A REST API for the Task Board](level-3/10-project-rest-api.md)

*API design*

| Method & path | Who | Does |
|---------------|-----|------|
| `GET /api/boards/` | any member | my boards, with my role and open-task count |
| `POST /api/boards/` | any user | create a board; creator becomes owner |
| `GET /api/boards/{id}/` | members | board with member list |
| `PATCH/DELETE /api/boards/{id}/` | owner | rename / delete |
| `GET /api/boards/{id}/tasks/?status=` | members | tasks, paginated |
| `POST /api/boards/{id}/tasks/` | editors, owner | create task |
| `GET/PATCH/DELETE /api/boards/{id}/tasks/{tid}/` | members read; editors+ write | task detail |
| `POST /api/boards/{id}/tasks/{tid}/move/` | editors, owner | change status |

## Level 4 · Master

### [02 · Production Settings &amp; Configuration](level-4/02-production-settings.md)

*What must differ between environments*

| Setting | Development | Production |
|---------|-------------|------------|
| `DEBUG` | `True` | `False`, always |
| `SECRET_KEY` | anything | long, random, secret, unique per environment |
| `ALLOWED_HOSTS` | `localhost` | your real domain(s) |
| `DATABASES` | SQLite or local PostgreSQL | managed PostgreSQL with credentials |
| Email | console backend | SMTP or a provider's API |
| HTTPS settings | off | on |
| Static files storage | default | hashed and compressed |

### [07 · Logging, Error Reporting &amp; Health Checks](level-4/07-logging-errors.md)

*What Django logs by itself*

| Logger | Logs |
|--------|------|
| `django.request` | `WARNING` for 4xx responses, `ERROR` (with traceback) for 5xx |
| `django.server` | `runserver` request lines |
| `django.security.*` | `DisallowedHost`, CSRF failures, suspicious operations |
| `django.db.backends` | every SQL query at `DEBUG` level, **only when `settings.DEBUG` is True** |
| `django.template` | missing-variable details at `DEBUG` level |

### [10 · Capstone — A Production-Ready Book Club App](level-4/10-capstone.md)

*Tests*

| Test | Proves |
|------|--------|
| `test_outsider_gets_404` | non-members can't see clubs or meetings |
| `test_only_host_schedules` | member 403, host 200 on the schedule page |
| `test_rsvp_is_created_then_updated_not_duplicated` | answering twice updates one row |
| `test_rsvp_requires_post` | GET → 405 |
| `test_cannot_rsvp_to_past_meeting` | past meetings → 404 |
| `test_meeting_time_shown_in_each_users_zone` | same instant, each user's local clock time on the page |
| `test_reminder_skips_members_who_declined_and_is_idempotent` | one email, to the right person; second run sends none |
| `test_command_dry_run_and_real_run` | dry run sends nothing; real run (with `captureOnCommitCallbacks`) emails both members and sets `reminder_sent_at` |
| `test_club_page_query_count_does_not_grow_with_meetings` | 6 queries with 21 meetings |
| `test_health_endpoints` | `/healthz` and `/readyz` return ok |

---

Go deeper with the full [Django Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
