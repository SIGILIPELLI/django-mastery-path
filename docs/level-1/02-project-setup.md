---
description: "Install Django in a virtual environment, create a project and an app, read every line of the generated settings, and run the development server."
---

# 02 · Installing Django & Your First Project

By the end of this lesson you'll have a running Django project with one app, and you'll
know what every generated file is for. We'll build the same project throughout Level 1:
a small book catalogue called **shelf**.

## Prerequisites

- **Python 3.12 or newer.** Django 6.x declares `Requires-Python: >=3.12`. Check with
  `python3 --version`. If you need a refresher on Python itself, see
  [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/).
- A terminal and an editor. Nothing else: the default database is SQLite, which ships
  with Python.

## Step 1 — a virtual environment

Every Django project should have its own virtual environment so its dependencies don't
collide with other projects or the system Python.

```bash
mkdir shelf && cd shelf
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python -m pip install django
python -m django --version
```

When this course was written, that installed **6.1.1**. Django has a fixed release
rhythm: a feature release roughly every eight months, with every third one (x.2) being a
long-term-support release that gets security fixes for about three years. If you're
choosing a version for a long-lived project, an LTS is the safer default; check the
official "Supported versions" table on djangoproject.com for current dates rather than
trusting a number in a tutorial. Pin what you install:

```bash
python -m pip freeze > requirements.txt
```

## Step 2 — create the project

```bash
django-admin startproject config .
```

The trailing `.` means "put it in the current directory" instead of creating another
nested `config/` folder. Naming the project package `config` is a common convention: it
makes clear that the package holds configuration, not features. You get:

```text
.
├── config/
│   ├── __init__.py
│   ├── asgi.py        # entry point for ASGI servers (Uvicorn, Daphne)
│   ├── settings.py    # all configuration
│   ├── urls.py        # the root URLconf
│   └── wsgi.py        # entry point for WSGI servers (Gunicorn, mod_wsgi)
└── manage.py          # command-line entry point for this project
```

`manage.py` is a thin wrapper around `django-admin` that sets
`DJANGO_SETTINGS_MODULE=config.settings` first, so every command knows which project it
belongs to.

## Step 3 — read the settings

Open `config/settings.py`. Here is what the generated file contains in Django 6.1
(comments removed), with what each part means:

```python title="config/settings.py"
from pathlib import Path
BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = 'django-insecure-...'   # signs sessions, password reset tokens, CSRF
DEBUG = True                          # detailed error pages; NEVER True in production
ALLOWED_HOSTS = []                    # host names this site may serve

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'config.urls'
TEMPLATES = [ ... ]                  # template engine config (lesson 04)
WSGI_APPLICATION = 'config.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

AUTH_PASSWORD_VALIDATORS = [ ... ]   # min length, common-password check, etc.

LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True                         # store datetimes as aware UTC (Level 4 · 08)

STATIC_URL = 'static/'
```

The 6.1 template also adds a `MAILERS` block that configures the console email backend
for development. If you're on an older version you won't see it; email is covered where
we need it (password resets in Level 2 · 05).

Two settings matter immediately:

- **`SECRET_KEY`** is generated randomly and prefixed `django-insecure-` as a reminder.
  It must be secret in production, because anyone with it can forge signed data such as
  session cookies. Level 4 · 02 moves it into an environment variable.
- **`ALLOWED_HOSTS = []`** looks like "allow nothing", but while `DEBUG` is `True`
  Django substitutes `['.localhost', '127.0.0.1', '[::1]']` so local development works.
  We saw that list in the error page when a request arrived with an unexpected host name.

## Step 4 — the first migration and the server

The built-in apps (auth, sessions, admin) have database tables. Create them:

```bash
python manage.py migrate
```

This printed one `Applying ... OK` line per migration — 18 of them across
`contenttypes`, `auth`, `admin` and `sessions` on our run — and created `db.sqlite3`.
Now start the development server:

```bash
python manage.py runserver
```

Our run printed:

```text
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
October 02, 2026 - 08:11:45
Django version 6.1.1, using settings 'config.settings'
Starting WSGI development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.

WARNING: This is a development server. Do not use it in a production setting. ...
```

Visit `http://127.0.0.1:8000/` and you'll see Django's welcome page (it only appears
while `DEBUG` is on and you haven't defined any URLs of your own). The server reloads
itself when you save a Python file.

## Step 5 — your first app

```bash
python manage.py startapp catalog
```

```text
catalog/
├── __init__.py
├── admin.py        # register models with the admin (lesson 07)
├── apps.py         # the AppConfig: the app's name and startup hooks
├── migrations/     # generated schema migrations (lesson 05)
│   └── __init__.py
├── models.py       # your data model
├── tests.py        # tests (Level 2 · 09)
└── views.py        # request handlers
```

An app does nothing until the project knows about it. Add it to `INSTALLED_APPS`:

```python title="config/settings.py"
INSTALLED_APPS = [
    # ... the six django.contrib apps ...
    "catalog",
]
```

Then confirm the project is healthy:

```bash
python manage.py check
```

```text
System check identified no issues (0 silenced).
```

## The commands you'll use most

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

`python manage.py help` lists every command, including those added by installed apps.

Try `python manage.py shell`. Since Django 5.2 it auto-imports your models and some
common helpers; on our project it announced `16 objects imported automatically (use -v
2 for details).` so you can type `Book.objects.count()` straight away.

## How It Actually Works

`manage.py` sets the `DJANGO_SETTINGS_MODULE` environment variable and calls
`django.core.management.execute_from_command_line`. Before any command runs, Django
calls `django.setup()`, which does two things in order: it imports your settings module
(so any error in `settings.py` appears first), then it loads the **app registry**,
importing each entry in `INSTALLED_APPS`, its `AppConfig`, and then each app's
`models` module. Only after every app's models are loaded is the registry marked ready.

That ordering explains a classic error: importing a model at the top of `settings.py`,
or in some module imported by an app's `__init__.py`, raises `AppRegistryNotReady`
("Apps aren't loaded yet"), because you've asked for a model before the registry has
finished loading.

The development server's auto-reload works by running your project in a child process
while the parent watches the files of every imported module (`StatReloader` polls
modification times; if the `pywatchman` package and a Watchman service are available,
Django uses the more efficient `WatchmanReloader` instead). When a file changes, the child is killed and restarted, which is why a syntax
error mid-edit stops the server until you fix it.

## Common mistakes

- **Installing Django globally** with `sudo pip install`. Always use a virtual
  environment per project.
- **Running `startproject config` without the dot**, ending up with `config/config/`
  and confusion about which directory is the project root.
- **Forgetting to add the app to `INSTALLED_APPS`.** Its models won't get migrations,
  its templates won't be found, and its admin registrations are ignored.
- **Committing `db.sqlite3` and `.venv/`** to git. Add them to `.gitignore`.
- **Using `python manage.py` from the wrong directory**, or with the venv not activated
  (you'll get `ModuleNotFoundError: No module named 'django'`).

## Exercise

1. Create the `shelf` project and the `catalog` app exactly as above, and get
   `runserver` showing the welcome page.
2. Break `settings.py` on purpose (misspell a module in `INSTALLED_APPS`), run
   `python manage.py check`, and read the error. Fix it.
3. Run `python manage.py shell -v 2` and read the list of auto-imported names.
4. Write a `.gitignore` for the project and make your first commit.
