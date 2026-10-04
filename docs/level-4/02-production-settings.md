---
description: "Configure Django for production from environment variables: secrets, DEBUG, hosts, database URLs, HTTPS and cookies, and a clean check --deploy."
---

# 02 · Production Settings & Configuration

The generated `settings.py` is a development file: a hard-coded insecure key, `DEBUG =
True`, SQLite, a console email backend. Production needs different values, and the same
code must run in development, CI, staging and production without edits. The widely used
approach (often summarised as the "twelve-factor" config rule) is to keep **code** in
the repository and **configuration** in the environment. This lesson rewrites the
reading-list project's settings that way and takes `check --deploy` from eight issues to
two deliberate, documented ones.

## What must differ between environments

| Setting | Development | Production |
|---------|-------------|------------|
| `DEBUG` | `True` | `False`, always |
| `SECRET_KEY` | anything | long, random, secret, unique per environment |
| `ALLOWED_HOSTS` | `localhost` | your real domain(s) |
| `DATABASES` | SQLite or local PostgreSQL | managed PostgreSQL with credentials |
| Email | console backend | SMTP or a provider's API |
| HTTPS settings | off | on |
| Static files storage | default | hashed and compressed |

With `DEBUG = True` in production, any error shows a page with your settings, local
variables and SQL, and Django keeps every query in memory. It's the single most
damaging setting to get wrong.

## Reading the environment

A few small helpers keep settings readable. No third-party package is required, though
`django-environ` and `python-decouple` are popular alternatives that do the same thing:

```python title="config/settings.py"
import os
from pathlib import Path

import dj_database_url
from django.core.exceptions import ImproperlyConfigured

BASE_DIR = Path(__file__).resolve().parent.parent


def env(name, default=None, *, required=False):
    value = os.environ.get(name, default)
    if required and not value:
        raise ImproperlyConfigured(f"Set the {name} environment variable.")
    return value


def env_bool(name, default=False):
    return env(name, str(default)).lower() in {"1", "true", "yes", "on"}


def env_list(name, default=""):
    return [item.strip() for item in env(name, default).split(",") if item.strip()]


DEBUG = env_bool("DJANGO_DEBUG", False)
SECRET_KEY = env("DJANGO_SECRET_KEY", required=not DEBUG) or "dev-only-insecure-key"
ALLOWED_HOSTS = env_list("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1" if DEBUG else "")
CSRF_TRUSTED_ORIGINS = env_list("DJANGO_CSRF_TRUSTED_ORIGINS")
```

Two design choices matter:

- **Secure by default.** `DEBUG` is `False` unless explicitly enabled, so a forgotten
  variable fails safe.
- **Fail loudly.** Without a secret key in production mode, the project refuses to start.
  We ran `check` with no environment variables set:

```text
django.core.exceptions.ImproperlyConfigured: Set the DJANGO_SECRET_KEY environment variable.
```

With `DJANGO_DEBUG=1` it ran with `System check identified no issues (0 silenced).`
A crash at startup is far better than a production site quietly signing sessions with a
known key.

Generate a key with Django's own helper (we checked: 50 characters):

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

## The database from a URL

Hosting platforms usually provide a single `DATABASE_URL`. The `dj-database-url` package
(3.1.2 when we tested) parses it into Django's dictionary:

```python
DATABASES = {
    "default": dj_database_url.parse(
        env("DATABASE_URL", f"sqlite:///{BASE_DIR / 'db.sqlite3'}"),
        conn_max_age=int(env("DJANGO_CONN_MAX_AGE", "60")),
        conn_health_checks=True,
    )
}
```

With `DATABASE_URL=postgres://postgres@/reading?host=<socket dir>` the project connected
to our local PostgreSQL; the shell reported `postgresql 60 True` for vendor,
`CONN_MAX_AGE` and `CONN_HEALTH_CHECKS`.

- **`CONN_MAX_AGE`** keeps a database connection open for that many seconds and reuses it
  across requests, instead of reconnecting every request (the default, `0`). Size it with
  your database's connection limit in mind: each worker process or thread holds one.
- **`CONN_HEALTH_CHECKS`** makes Django verify a reused connection before using it, so a
  database restart doesn't produce one failed request per worker.
- Django 5.1+ also supports PostgreSQL **connection pooling** through psycopg 3
  (`"OPTIONS": {"pool": True}`), an alternative to persistent connections. We didn't
  benchmark it here.

## HTTPS and cookie settings

```python
SECURE_SSL_REDIRECT = env_bool("DJANGO_SECURE_SSL_REDIRECT", not DEBUG)
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https") if env_bool("DJANGO_BEHIND_PROXY") else None
SESSION_COOKIE_SECURE = not DEBUG
CSRF_COOKIE_SECURE = not DEBUG
SECURE_HSTS_SECONDS = int(env("DJANGO_HSTS_SECONDS", "0" if DEBUG else "3600"))
```

`SECURE_PROXY_SSL_HEADER` is only safe when a proxy you control always sets (and
overwrites) `X-Forwarded-Proto`. Otherwise a client could send that header itself and
make Django believe a plain-HTTP request was secure. That's why it's opt-in here.

## Email

On Django 6.1, the `MAILERS` setting replaces the deprecated `EMAIL_*` settings
(Level 2 · 05):

```python
MAILERS = {
    "default": {
        "BACKEND": env("DJANGO_EMAIL_BACKEND",
                       "django.core.mail.backends.console.EmailBackend" if DEBUG
                       else "django.core.mail.backends.smtp.EmailBackend"),
    },
}
```

Provider credentials (host, user, password) also come from the environment. Check your
Django version's documentation for the exact `MAILERS` option names before adding them;
this course only exercised the backend selection.

## The result: `check --deploy`

With a generated secret key and `DJANGO_ALLOWED_HOSTS=reading.example`:

```text
WARNINGS:
?: (security.W005) You have not set the SECURE_HSTS_INCLUDE_SUBDOMAINS setting to True. ...
?: (security.W021) You have not set the SECURE_HSTS_PRELOAD setting to True. ...

System check identified 2 issues (0 silenced).
```

From eight issues (lesson 01) to two, and those two are **decisions**, not oversights:
including subdomains and preloading are hard to reverse, so we leave them off until the
whole domain is known to be HTTPS-only. Record that decision in code so the check stays
useful:

```python
# We serve legacy subdomains over HTTP until 2027; see ADR-007.
SILENCED_SYSTEM_CHECKS = ["security.W005", "security.W021"]
```

Then make `python manage.py check --deploy --fail-level WARNING` part of CI, run with
production-like environment variables. Any new warning fails the build.

## One settings file or several?

Two common layouts:

- **One `settings.py` driven by environment variables** (this lesson). Every environment
  runs the same code path; differences are visible in one place.
- **A `settings/` package** with `base.py`, `dev.py`, `prod.py`, selected by
  `DJANGO_SETTINGS_MODULE`. Useful when environments differ structurally (e.g. extra
  development-only apps like django-debug-toolbar), but it's easy for production-only
  code paths to go untested.

A combination works well: one base driven by the environment, plus a small `dev.py` that
imports it and adds development tools.

## How It Actually Works

`settings.py` is an ordinary Python module, imported once per process when
`django.setup()` first accesses `django.conf.settings`. The `settings` object is a
`LazySettings` proxy: on first attribute access it imports the module named by
`DJANGO_SETTINGS_MODULE`, copies every UPPERCASE name onto a `Settings` object, and fills
in anything missing from `django.conf.global_settings`. That's why environment variables
are read exactly once, at startup (changing them requires restarting the process), and
why lowercase helper functions like `env()` don't become settings.

`dj_database_url.parse()` is a pure function: it splits the URL into scheme, user,
password, host, port, path and query, maps the scheme (`postgres`) to an engine path
(`django.db.backends.postgresql`), and returns the dictionary Django expects. A `host`
query parameter becomes `HOST`, which psycopg treats as a Unix socket directory when it
starts with `/`.

## Common mistakes

- **`DEBUG = True` in production**, or `DEBUG` defaulting to `True` when a variable is
  missing.
- **Secrets in the repository**, including in old commits. Rotate any that were ever
  committed.
- **`ALLOWED_HOSTS = ["*"]`** to make an error go away.
- **`SECURE_PROXY_SSL_HEADER` without a proxy that sets it**, or a redirect loop without it.
- **Silencing checks without a recorded reason.**
- **Production-only settings code that never runs in CI.**

## Exercise

1. Convert your project's settings to read `DEBUG`, `SECRET_KEY`, `ALLOWED_HOSTS` and
   `DATABASE_URL` from the environment, secure by default.
2. Confirm the project refuses to start without `DJANGO_SECRET_KEY` when `DEBUG` is off.
3. Get `check --deploy` down to only deliberate warnings, silence those with a comment
   explaining why, and add the `--fail-level WARNING` check to your CI.
4. Create a `.env.example` file listing every variable with a safe example value (never
   real secrets), and document it in your README.
