---
description: "Make a Django app observable: the LOGGING setting, request IDs on log lines, what django.request logs, error emails, and liveness vs readiness checks."
---

# 07 · Logging, Error Reporting & Health Checks

In development, problems announce themselves with a yellow debug page. In production,
with `DEBUG = False`, a user sees "Server Error (500)" and *you* see nothing, unless
you set up logging and error reporting. This lesson configures Django's logging, adds a
request ID so you can follow one request through the logs, shows exactly what Django logs
for 404s and 500s, and adds the health-check endpoints that load balancers and
orchestrators need. We ran everything with `DEBUG = False` on the reading-list project.

## The `LOGGING` setting

Django uses Python's standard `logging` module, configured from a dictionary:

```python title="config/settings.py"
ADMINS = [("Ops", "ops@example.com")]

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {"request_id": {"()": "config.request_id.RequestIDFilter"}},
    "formatters": {
        "plain": {"format": "{asctime} {levelname} {name} [{request_id}] {message}", "style": "{"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "plain", "filters": ["request_id"]},
        "mail_admins": {"class": "django.utils.log.AdminEmailHandler", "level": "ERROR"},
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django.request": {"handlers": ["console", "mail_admins"], "level": "WARNING", "propagate": False},
    },
}
```

The pieces:

- **Loggers** are named, usually after modules (`logging.getLogger(__name__)` gives
  `books.health`). Names are hierarchical: `books.health` propagates to `books`, then to
  the root.
- **Handlers** decide where records go: the console (stdout/stderr, which container
  platforms collect), files, email, external services.
- **Formatters** decide how a record looks; **filters** can drop records or add fields.
- **`disable_existing_loggers: False`** keeps Django's own loggers working. Setting it to
  `True` (the `dictConfig` default) silently disables loggers created before your config
  is applied.

In containers, log to the **console** and let the platform ship logs. Writing log files
inside a container loses them when it restarts. Many teams switch the formatter to JSON
(one object per line) so log search tools can filter on fields; the
`python-json-logger` package or a small custom formatter does that.

## Logging from your code

```python title="books/health.py"
import logging

logger = logging.getLogger(__name__)

def boom(request):
    logger.info("About to divide by zero for entry count %s", 3)
    return 1 / 0
```

Pass values as **arguments**, not f-strings: the message is only formatted if a handler
actually emits it, and log aggregators can group identical templates.

## Request IDs

A busy site interleaves log lines from many requests. Tagging each line with a request ID
lets you pull out one request's story. A middleware plus a logging filter, using a
`contextvars.ContextVar` so it works for both threads and async tasks:

```python title="config/request_id.py"
import logging
import uuid
from contextvars import ContextVar

request_id_var = ContextVar("request_id", default="-")


class RequestIDMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        rid = request.headers.get("X-Request-ID") or uuid.uuid4().hex[:12]
        # Not reset afterwards: Django logs 4xx/5xx responses *after* the middleware
        # chain returns, and those log lines should still carry the ID. The next
        # request on this thread (or task) sets a fresh value first.
        request_id_var.set(rid)
        response = self.get_response(request)
        response["X-Request-ID"] = rid
        return response


class RequestIDFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get()
        return True
```

Put the middleware **first** in `MIDDLEWARE`. It reuses an incoming `X-Request-ID` if a
proxy or load balancer already assigned one, so IDs match across systems. (In production,
only trust that header from your own proxy.)

That comment in the code is a lesson we learned by running it. Our first version reset
the variable in a `finally:` block, the textbook pattern. The 500 error line carried the
ID, but the 404 line showed `[-]`. Django logs "Not Found" warnings from
`log_response()` *after* the whole middleware chain has returned, by which time the value
had been reset. Without the reset:

```text
2026-10-02 08:57:10,678 WARNING django.request [abc123] Not Found: /missing/
2026-10-02 08:57:10,678 INFO books.health [9633fb8bc388] About to divide by zero for entry count 3
2026-10-02 08:57:10,678 ERROR django.request [9633fb8bc388] Internal Server Error: /boom/
Traceback (most recent call last):
  ...
ZeroDivisionError: division by zero
```

The 404 request supplied its own ID (`abc123`), and the response echoed it back in an
`X-Request-ID` header. The two lines from the failing request share an ID, so the info
message and the traceback can be connected. Show the ID on your 500 page so users can
quote it in support requests.

## What Django logs by itself

| Logger | Logs |
|--------|------|
| `django.request` | `WARNING` for 4xx responses, `ERROR` (with traceback) for 5xx |
| `django.server` | `runserver` request lines |
| `django.security.*` | `DisallowedHost`, CSRF failures, suspicious operations |
| `django.db.backends` | every SQL query at `DEBUG` level, **only when `settings.DEBUG` is True** |
| `django.template` | missing-variable details at `DEBUG` level |

Watch `django.security.DisallowedHost` in particular: bots constantly send requests with
random `Host` headers, so many teams route that logger to a null handler to stop it
flooding email or alerts.

## Error emails

`AdminEmailHandler` emails `ADMINS` for each `ERROR` on `django.request`. Our `/boom/`
request produced (via the console email backend) a message with subject:

```text
Subject: [Django] ERROR (EXTERNAL IP): Internal Server Error: /boom/
```

and a body with the traceback, request details, and **settings**, sensitive ones masked:

```text
AUTH_PASSWORD_VALIDATORS = '********************'
DATABASES = {'default': {..., 'PASSWORD': '********************', ...}}
PASSWORD_HASHERS = '********************'
```

Django masks settings whose names match `API|AUTH|TOKEN|KEY|SECRET|PASS|SIGNATURE|HTTP_COOKIE`
(case-insensitive), which is why even the harmless `AUTH_PASSWORD_VALIDATORS` was starred
out. It can't know about request data: mark views that receive passwords or card
numbers with `@sensitive_post_parameters("password")` and functions with
`@sensitive_variables(...)` so those values are masked in reports.

Error emails work for small sites. Beyond a handful of errors a day they become noise:
the same error emailed 500 times. Dedicated error-tracking services group identical errors,
keep history and show release and user context; most provide a Django integration that
hooks into the same logging path. We didn't connect one for this course, so there's no
output to show.

## Health checks

Load balancers and orchestrators ask each instance "are you OK?" and stop sending traffic
or restart it when the answer is no. Two different questions:

```python title="books/health.py"
from django.db import connection
from django.http import JsonResponse


def healthz(request):
    """Liveness: the process is up and can answer HTTP."""
    return JsonResponse({"status": "ok"})


def readyz(request):
    """Readiness: dependencies this instance needs are reachable."""
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
    except Exception:
        logger.exception("Readiness check failed: database unreachable")
        return JsonResponse({"status": "error", "database": "unreachable"}, status=503)
    return JsonResponse({"status": "ok", "database": "ok"})
```

```text
healthz 200 {'status': 'ok'}
readyz 200 {'status': 'ok', 'database': 'ok'}
```

- **Liveness** must be cheap and must not depend on the database. If it did, a database
  outage would make the orchestrator restart every web process, which fixes nothing and
  adds load.
- **Readiness** checks dependencies; failing it takes the instance out of rotation until
  it recovers.
- Health endpoints must be reachable by the checker: add the checker's `Host` to
  `ALLOWED_HOSTS` (or check by IP with a matching entry), exempt them from
  `SECURE_SSL_REDIRECT` if the checker uses plain HTTP inside the network
  (`SECURE_REDIRECT_EXEMPT = [r"^healthz$", r"^readyz$"]`), and keep them out of
  login requirements.

## How It Actually Works

At startup `django.setup()` calls `configure_logging()`, which first applies Django's
default config (console output for `django` loggers when `DEBUG` is on, admin emails for
`django.request` errors when it's off) and then your `LOGGING` with `dictConfig()`.

When a view raises, the exception travels up to the nearest
`convert_exception_to_response` wrapper (each middleware is wrapped in one). It calls
`response_for_exception()`, which builds the 500 response (via `handler500`) and calls
`log_response()`, logging to `django.request` at `ERROR` with `exc_info`, *inside* the
middleware chain. `Http404` is the exception: `response_for_exception()` builds the 404
page without logging it, and `BaseHandler.get_response()` logs every response with status
400 or above that hasn't been logged yet *after* the whole middleware chain returns.
That's why the 404 line saw the reset context variable while the 500 line didn't.
`AdminEmailHandler.emit()` renders the same technical report as the debug page, using a
`SafeExceptionReporterFilter` that applies the masking rules, and sends it as an email.

## Common mistakes

- **No logging configuration**, so 500s vanish in production.
- **`disable_existing_loggers: True`** silencing Django's own loggers.
- **f-strings in log calls**, defeating lazy formatting and grouping.
- **Logging secrets** (request bodies, tokens) without sensitive-data decorators.
- **Liveness checks that query the database.**
- **Emailing every error** from a busy site instead of using an error tracker.
- **Health endpoints blocked by `ALLOWED_HOSTS` or HTTPS redirects.**

## Exercise

1. Add the `LOGGING` config and request-ID middleware to your project. Trigger a 404 and
   a 500 and confirm both lines carry an ID.
2. Reproduce the `finally: reset()` version and observe the `[-]` on the 404 line.
3. Add a custom 500 template (`templates/500.html`) that shows the request ID. (Hint: the
   500 handler renders with an empty context, so read the context variable in a custom
   `handler500` view.)
4. Add `/healthz` and `/readyz`, then stop your database and check that readiness returns
   503 while liveness still returns 200.
5. Write a JSON formatter (subclass `logging.Formatter`) and switch the console handler
   to it.
