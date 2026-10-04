---
description: "Level 2 of Django: relationships, advanced QuerySets, N+1 fixes, class-based views, auth, permissions, custom users, sessions, testing and a project."
---

# Level 2 · Intermediate <span class="level-badge">Real applications</span>

Goal: build a Django application that several people use at once, where data is
connected, pages stay fast as data grows, and each user can see and do exactly what
they're allowed to, with tests that prove it.

Level 1 was about getting a request to produce a page. Level 2 is about the things that
separate a demo from a product:

1. **Data is relational, and the ORM can do the heavy lifting.** Relationships,
   aggregation, subqueries and prefetching let the database answer questions in one or
   two queries instead of hundreds.
2. **Authorization is a property of every view, not of the navigation bar.** You'll
   centralise "who can see this" in QuerySets and helper functions, and test it with
   users who *shouldn't* have access.
3. **Tests are how you know.** Query budgets, permission checks and form rules all get
   tests, and the Level 2 project ships with its suite.

## Modules

1. [Model Relationships: ForeignKey, ManyToMany & OneToOne](01-relationships.md) — related managers, `on_delete`, through models, and 6.1's database-level cascades
2. [Advanced QuerySets: Q, F, annotate & aggregate](02-advanced-querysets.md) — conditional counts, `Subquery`, `Exists`, `Case`, window functions, and two real traps
3. [Finding and Fixing N+1 Queries](03-n-plus-one.md) — `select_related`, `prefetch_related`, `Prefetch` objects and query budgets
4. [Class-Based & Generic Views](04-class-based-views.md) — `as_view()`, generic CRUD views, mixins, the MRO, and when to use functions
5. [Authentication: Login, Logout & Sign-up](05-authentication.md) — built-in views, `LoginRequiredMiddleware`, password hashing and session rotation
6. [Permissions, Groups & Object-Level Access](06-permissions.md) — model permissions, groups, caching surprises, and owner-only access
7. [The Custom User Model](07-custom-user-model.md) — why to start with one, how, and what breaks if you switch late
8. [Sessions, Messages & Middleware](08-sessions-middleware.md) — session storage, the mutation trap, flash messages, writing middleware
9. [Testing Django Apps](09-testing.md) — test classes, the test client, `setUpTestData`, query budgets, how isolation works
10. [Project — A Multi-User Task Board](10-project-task-board.md) — roles, scoped lookups, a fixed query count, and ten passing tests

## What you need before starting

- **Level 1 of this course**, or equivalent experience: models, migrations, the ORM
  basics, function views, templates and ModelForms.
- **Comfort reading SQL.** Several lessons judge code by the SQL it produces. The
  [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers joins,
  grouping and subqueries.
- **Python classes and inheritance** for lesson 04 (the method resolution order matters).

## How the examples were checked

As in Level 1, every output was produced by running the code on **Django 6.1.1**
(Python 3.14, SQLite): the catalogue sample data for lessons 01–08, the Level 1
reading-list app for the test suite in lesson 09, and a fresh project for the task
board. Query counts were measured with Django's `CaptureQueriesContext` and
`assertNumQueries`, not estimated. Several of the "surprises" in this level, such as the
SQLite integer division, the lost session write and the permission cache, were found by
running the examples rather than planned in advance.

!!! note "Version notes"
    `LoginRequiredMiddleware` needs Django 5.1+. POST-only logout has been the rule since
    5.0. The database-level `on_delete` options (`DB_CASCADE` and friends) and the
    `MAILERS` email setting are new in **6.1**; on earlier versions, skip those
    paragraphs. Everything else in this level applies to any currently supported release.
