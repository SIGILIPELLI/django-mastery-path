---
description: "What Django's security features really protect against: SQL injection, XSS, CSRF, clickjacking, the built-in CSP with nonces, and check --deploy."
---

# 01 · Security: What Django Protects and What It Can't

Django has a strong security reputation, and it's earned: most of the classic web
vulnerabilities are prevented by default. But "by default" carries a lot of weight in
that sentence. Each protection has an escape hatch (`raw()`, `|safe`, `@csrf_exempt`),
and some risks, like authorization bugs and leaked secrets, are entirely yours. This
lesson demonstrates each protection on Django 6.1.1, shows the escape hatch that defeats
it, and ends with the deployment checklist.

For broader security concepts (threat modelling, OWASP categories, incident response),
see the [Cybersecurity Mastery Path](https://sigilipelli.github.io/cybersecurity-mastery-path/).

## SQL injection

The ORM always sends values as query parameters, separately from the SQL text. Raw SQL
is safe **only** if you do the same. We searched for a book title containing a classic
injection string, `x' OR '1'='1`, three ways:

```python
evil = "x' OR '1'='1"

len(list(Book.objects.raw(f"SELECT * FROM catalog_book WHERE title = '{evil}'")))   # 7
len(list(Book.objects.raw("SELECT * FROM catalog_book WHERE title = %s", [evil])))   # 0
Book.objects.filter(title=evil).count()                                             # 0
```

The f-string version returned **every book in the table**: the input closed the quote
and added `OR '1'='1'`. With a parameter (`%s` plus a list), the database treated the
whole string as a title to look for, and found none. The same rule applies to
`cursor.execute()`, `RawSQL()`, and `extra()`. Never build SQL with string formatting,
and never put user input into identifiers (table or column names), which can't be
parameterised; map them through an allow-list instead.

## Cross-site scripting (XSS)

Templates escape variables automatically (Level 1 · 04). The escape hatches are `|safe`,
`{% autoescape off %}` and `mark_safe()`. When you need to build HTML in Python, use
`format_html`, which escapes the arguments and trusts only the template string:

```python
>>> format_html('<a href="{}">{}</a>', "/books/1/", "<script>x</script>")
'<a href="/books/1/">&lt;script&gt;x&lt;/script&gt;</a>'
```

Places auto-escaping doesn't cover:

- Values inside `<script>` blocks or inline event handlers. Use `json_script`.
- URLs from users in `href`: escaping doesn't stop `javascript:` URLs. Validate the scheme.
- Uploaded HTML or SVG served from your domain (Level 1 · 09).
- JSON APIs consumed by front-end code that inserts it with `innerHTML`.

### Content Security Policy, built in

A CSP header tells the browser which scripts may run, so an injected `<script>` is
refused even if one slips through. Django 6.0 added built-in support:

```python title="config/settings.py"
from django.utils.csp import CSP

MIDDLEWARE.append("django.middleware.csp.ContentSecurityPolicyMiddleware")
TEMPLATES[0]["OPTIONS"]["context_processors"].append("django.template.context_processors.csp")

SECURE_CSP = {
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],
    "img-src": [CSP.SELF, "https:"],
    "frame-ancestors": [CSP.NONE],
}
```

Inline scripts you trust get the per-request nonce:

```django
<script nonce="{{ csp_nonce }}">console.log('allowed');</script>
```

Two consecutive requests produced:

```text
default-src 'self'; script-src 'self' 'nonce-e4B77Pv6QvfVZ1jm4BfEew'; img-src 'self' https:; frame-ancestors 'none'
<script nonce="e4B77Pv6QvfVZ1jm4BfEew"
default-src 'self'; script-src 'self' 'nonce-lfenbsUdmmPFOwvDtGXdAQ'; img-src 'self' https:; frame-ancestors 'none'
<script nonce="lfenbsUdmmPFOwvDtGXdAQ"
```

A fresh random nonce on every response, matching the one in the page. An attacker's
injected script can't know it. Roll out a new policy with `SECURE_CSP_REPORT_ONLY` first:
browsers report violations without blocking, so you find the inline scripts and
third-party resources you forgot. (On Django 5.x and earlier, the third-party
`django-csp` package provides similar functionality.)

## CSRF

`CsrfViewMiddleware` rejects unsafe requests without a valid token (Level 1 · 08 showed the
403). The escape hatch is `@csrf_exempt`; legitimate uses are rare (webhooks from other
services, which should be authenticated another way, such as by a signature). For HTTPS
sites served on several domains or behind proxies, `CSRF_TRUSTED_ORIGINS` must list the
origins that may POST, e.g. `["https://shelf.example"]`.

## Clickjacking and other headers

A default project already sent these headers on every response:

```text
X-Frame-Options = DENY
X-Content-Type-Options = nosniff
Referrer-Policy = same-origin
Cross-Origin-Opener-Policy = same-origin
Strict-Transport-Security = None
```

`X-Frame-Options: DENY` (and the CSP `frame-ancestors 'none'` above) stops other sites
from loading yours in an invisible frame to trick clicks. `nosniff` stops browsers
guessing content types. HSTS is absent until you configure it, deliberately, because it's
hard to undo (below).

## Open redirects

Any view that redirects to a URL from the request (`?next=`) can be abused to send users
from your trusted domain to a phishing site. Django's login view checks `next`
(Level 2 · 05). In your own code, use the same helper:

```python
>>> from django.utils.http import url_has_allowed_host_and_scheme as ok
>>> [ok(u, allowed_hosts={"shelf.example"}) for u in
...  ["/books/1/", "https://evil.example/", "//evil.example/", "javascript:alert(1)"]]
[True, False, False, False]
```

Note `//evil.example/`: a "relative" URL that browsers treat as another host.

## Secrets

`SECRET_KEY` signs sessions, password-reset tokens, messages and anything using
`django.core.signing`. If it leaks, attackers can forge them. Keep it out of the
repository (lesson 02), and if it may have leaked, rotate it with
`SECRET_KEY_FALLBACKS`: set the new key as `SECRET_KEY`, the old one in the fallbacks so
existing sessions keep working, then remove the fallback after the session lifetime.

## The deployment checklist: `check --deploy`

Run against an unmodified project, `python manage.py check --deploy` reported one error
and seven warnings:

```text
ERRORS:
?: (mail.E001) Your MAILERS setting uses a development-only email backend in the 'default'
   entry (django.core.mail.backends.console.EmailBackend).

WARNINGS:
?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting. ...
?: (security.W008) Your SECURE_SSL_REDIRECT setting is not set to True. ...
?: (security.W009) Your SECRET_KEY has less than 50 characters, less than 5 unique
   characters, or it's prefixed with 'django-insecure-' ...
?: (security.W012) SESSION_COOKIE_SECURE is not set to True. ...
?: (security.W016) ... you have not set CSRF_COOKIE_SECURE to True. ...
?: (security.W018) You should not have DEBUG set to True in deployment.
?: (security.W020) ALLOWED_HOSTS must not be empty in deployment.
```

(The `mail.E001` error is from 6.1's `MAILERS` check; older versions don't report it.)
Lesson 02 fixes every item. Two deserve care:

- **`SECURE_SSL_REDIRECT`**: if a proxy terminates TLS, Django sees plain HTTP and
  redirects forever unless you also set `SECURE_PROXY_SSL_HEADER` (and only if the proxy
  sets that header and strips it from clients).
- **HSTS** tells browsers to refuse plain HTTP for your domain for `SECURE_HSTS_SECONDS`.
  Start small (e.g. 3600), confirm everything works over HTTPS, then raise it.
  `SECURE_HSTS_INCLUDE_SUBDOMAINS` and `SECURE_HSTS_PRELOAD` extend it to every subdomain
  and into browsers' built-in lists, which can take months to undo.

## What Django can't do for you

- **Authorization.** Every "user A sees user B's data" bug in Levels 2 and 3 was in
  application code. Test with users who shouldn't have access.
- **Dependency vulnerabilities.** Keep Django and packages updated (Django publishes
  security releases; subscribe to the announcements) and run `pip-audit` or similar.
- **Business logic abuse**: unlimited password-reset emails, coupon reuse, enumeration.
  Rate-limit and design for it.
- **Logging secrets.** Error reports can include request data; Django's
  `@sensitive_post_parameters` and `@sensitive_variables` decorators mask named values.

## How It Actually Works

Most protections are middleware or defaults that run without your code asking.
`SecurityMiddleware` reads settings like `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS`,
`SECURE_CONTENT_TYPE_NOSNIFF`, `SECURE_REFERRER_POLICY` and
`SECURE_CROSS_ORIGIN_OPENER_POLICY`, redirects in `process_request` and adds headers in
`process_response`. `XFrameOptionsMiddleware` adds its header unless a view is decorated
with `@xframe_options_exempt`. The CSP middleware attaches a `LazyNonce` to each request
in `process_request`; the nonce is generated only if a template or view reads it, and
`build_policy()` substitutes it for the `CSP.NONCE` sentinel when writing the header.

`check --deploy` runs the system-check framework with the `deploy` tag: registered check
functions inspect settings and return `Warning` or `Error` objects. Anything that returns
an error stops `check` with a non-zero exit code, which is why running it in CI catches
misconfigurations before release.

## Common mistakes

- **f-strings or `%` formatting in raw SQL.**
- **`|safe` on user content, or user URLs in `href` without scheme checks.**
- **`@csrf_exempt` to silence a 403.**
- **`SECURE_SSL_REDIRECT` behind a proxy without `SECURE_PROXY_SSL_HEADER`.**
- **Enabling HSTS preload on day one.**
- **Committing `SECRET_KEY`** or reusing one across environments.
- **Treating `check --deploy` as optional.** Run it in CI against production settings.

## Exercise

1. Reproduce the raw-SQL injection on your catalogue, then fix it with parameters.
2. Add the built-in CSP in report-only mode, open your pages in a browser, and read the
   console's violation reports. Fix them, then switch to enforcing.
3. Write a view that redirects to `?next=` safely using `url_has_allowed_host_and_scheme`,
   and test it with the four URLs above.
4. Run `check --deploy` against your project and write down what each warning means and
   the setting that fixes it (lesson 02 will check your answers).
