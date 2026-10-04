---
description: "How Django sessions store per-visitor state, the mutation trap that loses data, flash messages across redirects, and writing correctly ordered middleware."
---

# 08 · Sessions, Messages & Middleware

HTTP is stateless: each request arrives with no memory of the last. Django layers state
on top with **sessions** (a per-visitor key/value store identified by a cookie), builds
**flash messages** on top of sessions, and lets you wrap every request with
**middleware**. All three are small APIs with a couple of sharp edges, which we'll find
by running them.

## Sessions

`request.session` behaves like a dictionary that persists between requests from the same
browser. A "recently viewed books" list:

```python
def add_recent(request, pk):
    recent = request.session.get("recent", [])
    recent = [pk] + [x for x in recent if x != pk]
    request.session["recent"] = recent[:5]
    return JsonResponse({"recent": request.session["recent"]})
```

Visiting books 3, 5, then 3 again returned `[3]`, `[5, 3]`... and finally `[3, 5]`;
a fresh request read back `{'recent': [3, 5]}`. Values must be JSON-serialisable (the
default serializer is JSON), so store IDs, not model instances.

### The cookie

The first response set a `sessionid` cookie. Its attributes, as Django set them by
default:

```text
{'httponly': True, 'samesite': 'Lax', 'secure': '', 'max-age': 1209600, 'path': '/'}
```

- **HttpOnly**: JavaScript can't read it, which limits what an XSS bug can steal.
- **SameSite=Lax**: the browser won't send it on most cross-site sub-requests (images,
  form POSTs from other sites), a second line of defence against CSRF.
- **max-age 1209600**: two weeks (`SESSION_COOKIE_AGE`).
- **secure** is off by default because development runs on plain HTTP. In production
  set `SESSION_COOKIE_SECURE = True` so the cookie is only sent over HTTPS (Level 4 · 01).

### Where the data lives

By default, in the database (`django.contrib.sessions`, table `django_session`). We
looked at the row for our session:

```text
session_data: eyJyZWNlbnQiOlszLDVdfQ:1xCYeI:...
get_decoded(): {'recent': [3, 5]}
```

That first segment is just base64 of `{"recent":[3,5]}`. Session data is **signed**
(tamper-evident, using `SECRET_KEY`) but **not encrypted**. Anyone with database access
can read it. Don't put secrets in sessions.

Other engines, set with `SESSION_ENGINE`:

| Engine | Stores data in | Notes |
|--------|----------------|-------|
| `db` (default) | database | simple; needs periodic `clearsessions` |
| `cache` | your cache (e.g. Redis) | fast; sessions lost if the cache is flushed |
| `cached_db` | cache, with database fallback | a good production default |
| `signed_cookies` | the cookie itself | no server storage; data visible to the user; size-limited; can't be revoked server-side |
| `file` | files on disk | rarely the right choice with several servers |

Expired sessions aren't deleted automatically from the database. Run
`python manage.py clearsessions` on a schedule.

### The mutation trap

Here's a shorter-looking version of the same feature:

```python
def add_recent_buggy(request, pk):
    request.session.setdefault("recent", []).insert(0, pk)
    return JsonResponse({"recent": request.session["recent"]})
```

We called it for 1 and then 2 with a fresh client, then read the session back:
`{'recent': [1]}`. The **2 was lost**. The first call *assigned* a new key (via
`setdefault`), which marks the session modified. The second only mutated a list already
inside it, which the session can't detect, so it was never saved. Either assign the key
again (`request.session["recent"] = recent`) or set `request.session.modified = True`
after mutating.

We also confirmed the flip side: a request that only reads the session doesn't resend
the cookie or rewrite the row (unless you set `SESSION_SAVE_EVERY_REQUEST = True`).

## Flash messages

The messages framework shows one-time notices ("Book saved.") on the *next* page,
which fits the Post/Redirect/Get pattern from Level 1:

```python
from django.contrib import messages

def flash(request):
    messages.success(request, "Book saved.")
    messages.info(request, "Tip: add tags to find it later.")
    return redirect("sess-show-msgs")
```

Following the redirect, the next page read both:

```text
success:Book saved. | info:Tip: add tags to find it later.
```

and loading that page **again** produced an empty string: iterating the messages marks
them as used, and they're deleted when the response is processed. In a template (the
`messages` context processor makes them available):

```django
{% if messages %}
<ul class="messages">
  {% for message in messages %}
    <li class="{{ message.tags }}" role="status">{{ message }}</li>
  {% endfor %}
</ul>
{% endif %}
```

Put that in `base.html` once. If a template never iterates `messages`, they accumulate
until some page does, which produces confusing "Book saved." notices much later.

## Middleware

Middleware wraps the whole request/response cycle. The modern form is a class:

```python title="config/timing.py"
import time


class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response          # runs once, at startup

    def __call__(self, request):
        start = time.perf_counter()               # before the view
        response = self.get_response(request)
        elapsed_ms = (time.perf_counter() - start) * 1000
        response["Server-Timing"] = f"app;dur={elapsed_ms:.1f}"   # after the view
        return response
```

```python title="config/settings.py"
MIDDLEWARE = [
    "config.timing.TimingMiddleware",
    "django.middleware.security.SecurityMiddleware",
    # ...
]
```

Every response we fetched afterwards carried a `Server-Timing: app;dur=...` header, which
browser DevTools display in the Network panel's timing tab. Because it's first in the
list, its timing includes every other middleware.

Middleware can also short-circuit (return a response without calling `get_response`),
which is how `CsrfViewMiddleware` rejects a bad POST and `SecurityMiddleware` redirects to
HTTPS. Optional hooks: `process_view()` (called after URL resolution, before the view,
with the view function and its arguments), `process_exception()` (when the view raises)
and `process_template_response()`.

### Order matters

The default list, and why it's in that order:

1. `SecurityMiddleware` — HTTPS redirect and security headers; first so it can redirect
   before doing other work.
2. `SessionMiddleware` — attaches `request.session`.
3. `CommonMiddleware` — `APPEND_SLASH`, disallowed user agents.
4. `CsrfViewMiddleware` — checks tokens on unsafe methods; must come before any view.
5. `AuthenticationMiddleware` — sets `request.user`; needs the session, so after (2).
6. `MessageMiddleware` — uses the session (default storage falls back to it).
7. `XFrameOptionsMiddleware` — sets the anti-clickjacking header on the way out.

Put your own middleware where its dependencies are satisfied: anything that reads
`request.user` must come after `AuthenticationMiddleware`.

## How It Actually Works

At startup, Django builds the middleware chain **from the bottom up**: it wraps the view
handler in the last middleware, wraps that in the second-to-last, and so on. The result
is one callable; calling it runs each middleware's "before" code top to bottom, the
view, then each "after" code bottom to top, like nested function calls. That's why the
`__init__` runs once per process and `__call__` once per request.

`SessionMiddleware.process_request` reads the cookie and attaches a lazy `SessionStore`
keyed by it; nothing is loaded until you access the session. On the way out,
`process_response` checks `request.session.modified` (set by `__setitem__`,
`__delitem__`, `pop`, `setdefault` when it inserts...) and, only if true, saves the data
and sets the cookie. A nested list mutation never calls those methods, which is the
whole mechanism behind the lost write. When a user logs in, `cycle_key()` copies the
data to a new key and deletes the old one, preventing session fixation.

The default message storage, `FallbackStorage`, tries a signed cookie first and falls
back to the session if the messages don't fit. Messages are added to a pending list;
when the response passes back through `MessageMiddleware`, any messages that were
iterated are dropped and the rest are stored again.

## Common mistakes

- **Mutating nested session data** without reassigning or setting `modified`.
- **Storing model instances or secrets in the session.**
- **Never running `clearsessions`** with the database engine; the table grows forever.
- **Using `signed_cookies`** and expecting to revoke sessions server-side.
- **Messages that never render** because `base.html` doesn't iterate them.
- **Custom middleware placed above `AuthenticationMiddleware`** that reads `request.user`.
- **Doing slow work (network calls) in middleware**; it runs on every request.

## Exercise

1. Build the "recently viewed" list on the catalogue's book detail page, showing the
   last five titles with one query (`in_bulk()` helps keep the order).
2. Write the buggy version and prove the lost update with a test or the shell; then fix
   it both ways.
3. Add flash messages to create, update and delete, rendered once in `base.html`.
4. Write a middleware that adds an `X-Request-ID` header (a `uuid4`) to every response
   and stores it on `request.id` so views and logs can use it (Level 4 · 07 uses this).
5. Switch `SESSION_ENGINE` to `signed_cookies`, log in, and decode the cookie's first
   segment with base64. What can you read?
