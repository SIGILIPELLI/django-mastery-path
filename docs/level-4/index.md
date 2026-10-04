---
description: "Level 4 of Django: security, production settings, deployment, database performance, commands, template tags, logging, time zones and upgrades."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: take a Django application to production and keep it there: secure by default,
configured from the environment, deployed on real servers, fast on real data volumes,
observable when something goes wrong, correct for users in every time zone, and
upgradable without drama.

Levels 1–3 were about building features correctly. Level 4 is about everything around the
features:

1. **Defaults are not configuration.** A generated project fails `check --deploy` eight
   ways. You'll fix each one deliberately and record the two you choose not to.
2. **Measure on realistic data.** Index lessons use a 200,000-row PostgreSQL table, because
   a 50-row table teaches nothing about query plans.
3. **Assume everything runs twice and fails halfway.** Commands have dry runs, tasks are
   idempotent, migrations are reversible, deploys are rolling.

## Modules

1. [Security: What Django Protects and What It Can't](01-security.md) — SQL injection, XSS, the built-in CSP with nonces, open redirects, `check --deploy`
2. [Production Settings & Configuration](02-production-settings.md) — environment-driven, secure-by-default settings, from 8 deploy issues to 2 documented ones
3. [Deploying Django: Gunicorn, Uvicorn, Static Files & PostgreSQL](03-deployment.md) — workers, WhiteNoise with immutable caching, the release procedure, the proxy
4. [Database Performance: Indexes, EXPLAIN & Query Budgets](04-database-performance.md) — reading plans, composite and functional indexes, keyset pagination
5. [Custom Management Commands](05-management-commands.md) — arguments, dry runs, validation, exit codes, scheduling and tests
6. [Custom Template Tags, Filters & Template Partials](06-template-tags-partials.md) — filters, simple and inclusion tags, `format_html`, Django 6 partials
7. [Logging, Error Reporting & Health Checks](07-logging-errors.md) — `LOGGING`, request IDs, what Django logs, error emails, liveness vs readiness
8. [Time Zones & Internationalization](08-i18n-timezones.md) — aware datetimes, "today" per user, gettext, plurals and a gettext surprise
9. [Upgrading Django & Handling Deprecations](09-upgrading.md) — the release cycle, hidden warnings, `django-upgrade`, an upgrade routine
10. [Capstone — A Production-Ready Book Club App](10-capstone.md) — roles, time zones, idempotent reminders, CSP, health checks, 10 tests, served on Gunicorn + PostgreSQL

## What you need before starting

- **Levels 1–3**, especially testing, transactions and the task-board projects.
- **Basic command-line and server knowledge**: environment variables, processes, ports,
  HTTP headers. The [Shell/Bash Mastery Path](https://sigilipelli.github.io/shell-mastery-path/)
  and [Server Ops Mastery Path](https://sigilipelli.github.io/server-ops-mastery-path/)
  cover the operating-system side.
- **PostgreSQL** for lessons 03 and 04 if you want to reproduce the plans exactly. Any
  local installation works; we used a server started by the `pgserver` Python package.

## How the examples were checked

Outputs come from **Django 6.1.1** on Python 3.14, with **PostgreSQL 16.2** for the
deployment, performance and capstone runs, **Gunicorn 26.2.0**, **WhiteNoise 6.12.0**,
**dj-database-url 3.1.2** and **django-upgrade 1.32.0**. Query plans and timings were
measured on one laptop; your milliseconds will differ, the plan shapes shouldn't. We did
**not** run anything that needs an outside service or account: no TLS certificates, no
hosting platform, no error-tracking service, no SMTP server (emails were printed by the
console and in-memory backends). Those sections show configuration and say so instead of
inventing output.

!!! note "Version notes"
    Built-in CSP (lesson 01) and template partials (lesson 06) are new in **Django 6.0**;
    the `MAILERS` setting and its `check --deploy` error are new in **6.1**. On earlier
    versions the third-party packages `django-csp` and `django-template-partials` provide
    similar features, and email uses the `EMAIL_*` settings.
