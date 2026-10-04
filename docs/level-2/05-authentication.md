---
description: "Add login, logout, sign-up and password reset with Django's built-in auth views, protect pages, and see how password hashing and sessions work."
---

# 05 · Authentication: Login, Logout & Sign-up

Authentication answers "who is this?" Django ships a complete, battle-tested
implementation: a `User` model, password hashing, login/logout views, password change
and reset flows, and helpers to protect views. Writing your own is how sites end up
storing passwords badly. This lesson wires up the built-in pieces, adds sign-up (the one
thing Django doesn't give you a view for), and explains what happens under the hood.

## Wiring the built-in views

One line in the root URLconf:

```python title="config/urls.py"
path("accounts/", include("django.contrib.auth.urls")),
```

We listed the URL names it provides:

```text
['login', 'logout', 'password_change', 'password_change_done', 'password_reset',
 'password_reset_complete', 'password_reset_confirm', 'password_reset_done']
```

Each view renders a template under `registration/` that you supply. The login template
is the only one you need on day one:

```django title="templates/registration/login.html"
{% extends "base.html" %}
{% block content %}
<form method="post">
  {% csrf_token %}
  {{ form.as_div }}
  <input type="hidden" name="next" value="{{ next }}">
  <button>Log in</button>
</form>
{% endblock %}
```

And three settings control where people land:

```python title="config/settings.py"
LOGIN_URL = "login"                # where @login_required sends anonymous users
LOGIN_REDIRECT_URL = "catalog:book-list"   # after login, if no ?next=
LOGOUT_REDIRECT_URL = "catalog:book-list"  # after logout
```

## What the login view does (observed)

| Request | Result |
|---------|--------|
| POST wrong password | 200, form re-rendered with "Please enter a correct username and password. Note that both fields may be case-sensitive." |
| POST correct password, `next=/cbv/new/` | 302 → `/cbv/new/` |
| POST correct password, `next=https://evil.example/` | 302 → `/accounts/profile/` (the default `LOGIN_REDIRECT_URL`) |

The last row is important. The `next` parameter is user-controlled, so an attacker could
send someone a link to your real login page that redirects to a look-alike site after
login. Django checks that `next` points to the same host and ignores it otherwise. If
you ever write your own redirect-after-action code, use
`django.utils.http.url_has_allowed_host_and_scheme()` for the same check.

The error message deliberately doesn't say whether the username exists, so the form
can't be used to discover accounts.

## Logout must be a POST

Since Django 5.0, `LogoutView` only accepts POST. We confirmed it: `GET
/accounts/logout/` returned **405**. Logging out changes state, and a GET could be
triggered by an `<img>` tag on another site. Put a small form in your navigation:

```django
{% if user.is_authenticated %}
  <form method="post" action="{% url 'logout' %}">
    {% csrf_token %}
    <button>Log out {{ user.username }}</button>
  </form>
{% else %}
  <a href="{% url 'login' %}?next={{ request.path|urlencode }}">Log in</a>
{% endif %}
```

`user` is available in every template through the `auth` context processor; anonymous
visitors get an `AnonymousUser` object whose `is_authenticated` is `False`.

## Protecting views

```python
from django.contrib.auth.decorators import login_required

@login_required
def my_reviews(request):
    reviews = Review.objects.filter(user=request.user).select_related("book")
    return render(request, "catalog/my_reviews.html", {"reviews": reviews})
```

For class-based views, `LoginRequiredMixin` (lesson 04). An anonymous request to a
protected page gets a 302 to `LOGIN_URL?next=<path>`; we saw
`/accounts/login/?next=/cbv/new/`.

### Login required by default

For sites where almost everything is private, Django 5.1 added
`LoginRequiredMiddleware`, which inverts the default: every view requires login unless
marked otherwise.

```python title="config/settings.py"
MIDDLEWARE = [
    # ... after AuthenticationMiddleware ...
    "django.contrib.auth.middleware.LoginRequiredMiddleware",
]
```

With it enabled we requested four URLs anonymously:

```text
/hello/            302 /accounts/login/?next=/hello/
/accounts/login/   200
/admin/            302 /admin/login/?next=/admin/
/cbv/              302 /accounts/login/?next=/cbv/
```

The login view (and the admin's) are already exempt. Mark your own public views with
`@login_not_required`. Forgetting to exempt a public page is a visible bug; forgetting to
protect a private one, in the opt-in model, is a silent leak. That asymmetry is the
argument for the middleware.

## Sign-up

Django provides `UserCreationForm` but no view. A function view is short:

```python title="accounts/views.py"
from django.contrib.auth import login
from django.contrib.auth.forms import UserCreationForm
from django.shortcuts import redirect, render


def signup(request):
    form = UserCreationForm(request.POST or None)
    if request.method == "POST" and form.is_valid():
        user = form.save()
        login(request, user)
        return redirect("catalog:book-list")
    return render(request, "registration/signup.html", {"form": form})
```

The form enforces your `AUTH_PASSWORD_VALIDATORS`. With the defaults, we tried the
password `password`:

```text
This password is too common.
```

(The common-password validator checks a bundled list of about 19,600 common passwords; the others
reject passwords that are short, entirely numeric or too similar to the username.) In a
real product you'll usually also want an email field and email verification, which is
where packages like `django-allauth` save time.

## Password reset

The reset views are included by `auth.urls`. They need email configured, and this is
one place where the Django version matters. A project generated by Django 6.1 already
contains:

```python title="config/settings.py (Django 6.1)"
MAILERS = {
    "default": {
        "BACKEND": "django.core.mail.backends.console.EmailBackend",
    },
}
```

The console backend prints emails to the `runserver` terminal instead of sending them.
6.1 introduced `MAILERS` and deprecated the older `EMAIL_BACKEND`, `EMAIL_HOST`,
`EMAIL_PORT`... settings (they're scheduled for removal in 7.0). The two styles can't be
mixed: Django raises `ImproperlyConfigured: Deprecated email settings are not allowed
when MAILERS is defined` if both are present. On Django 6.0 or earlier, the equivalent
is the single setting `EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"`.

The flow: the user enters an email; if a matching active user exists, Django emails a
one-time link containing a token; the link lets them set a new password. The view shows
the same "check your email" page whether or not the address exists, again to avoid
revealing accounts.

## How It Actually Works

**Passwords are never stored.** `make_password()` produces a string like
`pbkdf2_sha256$1500000$<salt>$<hash>`. We checked a fresh hash on Django 6.1.1: the
algorithm was `pbkdf2_sha256` with **1,500,000** iterations. A random salt makes
identical passwords hash differently; the high iteration count makes each guess
expensive for an attacker with a stolen database. The iteration count rises in most
Django releases, and when a user logs in with a hash made under an older count, Django
transparently re-hashes it with the current settings. You can switch to Argon2 (the
first entry in `PASSWORD_HASHERS` is used for new hashes) after installing
`argon2-cffi`.

**Logging in** (`django.contrib.auth.login()`) does three things: it **rotates the
session key** (so an attacker who planted a session ID before login can't ride the
logged-in session: session fixation), stores the user's ID, the backend used, and a
hash derived from the password hash in the session, and rotates the CSRF token. On each
request, `AuthenticationMiddleware` sets `request.user` to a lazy object; the first time
code touches it, Django loads the user from the session's ID and compares the stored
password-derived hash. That's why **changing a password logs out other sessions**: the
hash no longer matches. (Use `update_session_auth_hash()` after a password change to keep
the *current* session logged in.)

`authenticate(username=..., password=...)` loops over `AUTHENTICATION_BACKENDS`; the
default `ModelBackend` looks the user up and calls `check_password()`. It returned the
user `raj` for the right password and `None` for a wrong one, never an exception, so
callers can't distinguish "no such user" from "wrong password" either.

## Common mistakes

- **Building your own password storage or login check.** Use `authenticate()` and `login()`.
- **Logout links** (GET) instead of forms.
- **Redirecting to `request.GET["next"]` unchecked** in custom views.
- **Messages like "No account with that email"** on login or reset pages.
- **Not setting `LOGIN_REDIRECT_URL`**, sending users to the non-existent
  `/accounts/profile/`.
- **Forgetting `update_session_auth_hash`** after a custom password-change view, which
  logs the user out.

## Exercise

1. Add `auth.urls`, a login template and a logout button to the catalogue's base
   template. Set the three redirect settings.
2. Protect "add a book" with `@login_required` / `LoginRequiredMixin`, log out, and
   follow the redirect round-trip back to the form.
3. Add a sign-up page that logs the new user in. Try `password`, `12345678` and your
   username as passwords and read each validator's message.
4. Configure the console email backend, request a password reset and complete it using
   the link printed in the terminal.
5. Turn on `LoginRequiredMiddleware`, then fix every public page you broke with
   `@login_not_required`.
