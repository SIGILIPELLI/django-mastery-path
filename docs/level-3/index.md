---
description: "Level 3 of Django: Django REST Framework, signals, transactions, advanced migrations, caching, background tasks, async views and a REST API project."
---

# Level 3 · Advanced <span class="level-badge">Under load</span>

Goal: extend a working Django application with an API, keep its data correct when many
requests arrive at once, change its schema safely while it's running, and move slow work
out of the request, knowing in each case what Django is doing underneath.

Level 2 made an application multi-user. Level 3 is about what happens when it's busy:

1. **Every entry point must enforce the same rules.** An API, a background task and a
   data migration are all ways to write data. Rules that live only in a form or a view get
   bypassed; this level moves them into scoped QuerySets, serializers and database
   constraints.
2. **Concurrency is real, and SQLite hides it.** The race-condition and migration
   experiments in this level ran on PostgreSQL 16 so the failures are the ones you'd see in
   production.
3. **Measure before you optimise.** Caching, async and task queues each have a cost. You'll
   see one case where async was a 3x win, one where it made no difference, and a cache
   configuration that served one user's page to everyone.

## Modules

1. [Django REST Framework: Serializers](01-drf-serializers.md) — `ModelSerializer`, validation, partial updates, nesting, and 33 → 4 queries
2. [DRF Views, ViewSets & Routers](02-drf-viewsets.md) — the view ladder, routers, custom actions, and actions that skip validation
3. [API Auth, Permissions, Throttling & Pagination](03-drf-auth-permissions.md) — session vs token auth, object permissions, two real 500s and their fixes
4. [Signals — and When Not to Use Them](04-signals.md) — what fires, what silently doesn't, and `on_commit`
5. [Transactions, Constraints & Race Conditions](05-transactions-integrity.md) — a measured lost update (50 of 200), `F()`, `select_for_update`, constraints
6. [Migrations in Depth: Data Migrations & Zero-Downtime Changes](06-migrations-in-depth.md) — `RunPython`, `db_default`, concurrent indexes, expand/contract
7. [Caching](07-caching.md) — the cache API, `cache_page`, a per-user leak, invalidation strategies
8. [Background Work: The Tasks Framework & Celery](08-background-tasks.md) — Django 6's `@task`, what the default backend really does, idempotent tasks
9. [Async Django: ASGI, Async Views & the Async ORM](09-async-django.md) — when async helps, when it didn't, and the async ORM
10. [Project — A REST API for the Task Board](10-project-rest-api.md) — token-authenticated, role-aware API with 12 tests

## What you need before starting

- **Levels 1 and 2**, especially the task-board project (lesson 10 extends it), QuerySet
  annotations, and testing.
- **HTTP and JSON basics** for the DRF lessons. For REST design itself (resource naming,
  status codes, versioning) see the
  [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/).
- **Python's `asyncio` basics** (`async def`, `await`, `asyncio.gather`) for lesson 09.
- **Optional:** a local PostgreSQL to reproduce lessons 05 and 06 exactly.

## How the examples were checked

Outputs come from **Django 6.1.1** and **Django REST Framework 3.18.1** on Python 3.14.
Lessons 05 and 06 ran against **PostgreSQL 16.2** (started locally with the `pgserver`
Python package) because the behaviour differs on SQLite; everything else used SQLite.
Lesson 09's load test used **Uvicorn 0.54** and `httpx` 0.28 on an 8-core laptop; treat
its timings as one machine's results, not a benchmark. Two things were **not** run because
they need external services we didn't have: a Redis cache server (lesson 07 shows its
configuration but its outputs come from the local-memory cache) and a Celery broker and
worker (lesson 08 shows Celery code without output).

!!! note "Version notes"
    The built-in tasks framework (`django.tasks`) is new in **Django 6.0**. `db_default`
    needs 5.0+, `CheckConstraint(condition=...)` 5.1+, prefetching with `iterator()` 4.1+.
    DRF evolves on its own schedule; check its release notes when upgrading either package.
