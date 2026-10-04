---
description: "Level 1 of Django: the request cycle, project setup, URLs and views, templates, models, migrations, the ORM, admin, forms, static files and a project."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build a small, complete database-backed web application with Django, and be able
to explain what happens between a browser's request and the HTML that comes back.

Three ideas carry you through all ten modules:

1. **A request is a function call.** The server hands Django a request; middleware wraps
   it; the URL resolver picks a view; the view returns a response. If you can trace that
   path, you can debug almost anything at this level.
2. **The model is the source of truth.** Tables, migrations, admin screens and form
   fields are all generated from your model classes. Spend your care there.
3. **Never trust input; never hand-build output.** Forms validate everything that comes
   in, and the template engine escapes everything that goes out. Both are on by default:
   your job is not to switch them off.

## Modules

1. [What Django Is: The Request/Response Cycle](01-what-django-is.md) — what's in the box, MTV, and the path of one request
2. [Installing Django & Your First Project](02-project-setup.md) — virtual environment, `startproject`, every line of `settings.py`, `startapp`
3. [URLs and Views](03-urls-views.md) — `path()`, converters, the request object, responses, reversing and namespaces
4. [Templates: Inheritance, Tags, Filters & Autoescaping](04-templates.md) — lookup order, blocks, the dot lookup, and why `|safe` is dangerous
5. [Models and Migrations](05-models-migrations.md) — field types, `null` vs `blank`, reading the SQL a migration runs, rolling back
6. [Querying with the ORM](06-orm-basics.md) — QuerySets, lookups, `get()` vs `filter()`, updates, laziness and caching
7. [The Django Admin](07-admin.md) — `ModelAdmin`, search, filters, inlines, actions, and measuring its queries
8. [Forms, ModelForms & CSRF](08-forms.md) — the validation pipeline, the view pattern, Post/Redirect/Get, CSRF tokens
9. [Static Files & User Uploads](09-static-media.md) — `collectstatic`, hashed filenames, `FileField`, upload security
10. [Project — A Reading-List App](10-project-reading-list.md) — a complete CRUD app with filters, validation and the admin

## What you need before starting

- **Python fundamentals:** functions, classes, modules, virtual environments, `pip`.
  Django is plain Python, and every error message assumes you can read a traceback. If
  that's shaky, start with [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/).
- **Basic HTML:** forms, links, the difference between GET and POST. The
  [HTML & CSS Mastery Path](https://sigilipelli.github.io/html-css-mastery-path/) covers
  this.
- **A little SQL helps** for reading the queries the ORM generates; see
  [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/).
- Python 3.12 or newer. SQLite is built in, so no database server is needed for Level 1.

## How the examples were checked

Every command, query and error message in this level was run on **Django 6.1.1** with
Python 3.14 and SQLite, against a small sample catalogue (three authors, five books, a
few tags and reviews). Outputs are copied from those runs, sometimes trimmed. Things
that change between machines, such as absolute paths, the generated `SECRET_KEY` and
the random suffixes Django adds to duplicate upload names, will look different for you.

!!! note "About versions"
    Django's core concepts here have been stable for many releases, so most of this level
    applies to any supported version. Where a detail is version-specific (the shell's
    auto-imports arrived in 5.2; `BigAutoField` became the default primary key type in
    6.0; template partials arrived in 6.0) the lesson says so. Run
    `python -m django --version` to see what you have.
