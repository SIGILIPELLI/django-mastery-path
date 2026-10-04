---
description: "Secure a DRF API: session vs token auth, CSRF for APIs, permission classes, object permissions, throttling, pagination and validating extra data."
---

# 03 · API Auth, Permissions, Throttling & Pagination

An API without access control is a data leak with a URL. This lesson configures who can
call the catalogue API, what they may change, how often they may call it and how much
they get back per request. Along the way we hit two real failures, a 500 error from a
unique constraint and a 500 from a check constraint, that come from the gap between
DRF validation and database rules.

Project-wide defaults go in one setting:

```python title="config/settings.py"
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 3,
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {"anon": "5/minute", "user": "100/minute"},
}
```

(Tiny page size and anonymous rate so the effects are easy to see.) Any view can
override each of these with class attributes.

## Authentication: who is calling?

**`SessionAuthentication`** uses Django's login session, so it's right for a JavaScript
front end served from the same site. Because the browser sends the cookie automatically,
DRF **enforces CSRF** for session-authenticated unsafe requests. We logged in a client
with CSRF checks enabled and posted without a token:

```text
403 {"detail": "CSRF Failed: CSRF cookie not set."}
```

Your JavaScript must send the `X-CSRFToken` header (read from the `csrftoken` cookie) on
POST, PUT, PATCH and DELETE.

**`TokenAuthentication`** (add `"rest_framework.authtoken"` to `INSTALLED_APPS` and
migrate) gives each user a random key, sent as a header:

```python
from rest_framework.authtoken.models import Token
token, _ = Token.objects.get_or_create(user=ana)   # a 40-character key
```

```text
Authorization: Token <40-character key>
```

With that header, `POST /api/books/` returned 201. Token auth suits scripts and mobile
apps. No cookie is involved, so no CSRF check is needed. DRF's built-in token model is
deliberately minimal: one token per user, no expiry, stored in plain text. For anything
serious, consider a package with hashed, expiring tokens (`django-rest-knox`) or JWTs
(`djangorestframework-simplejwt`), and always serve APIs over HTTPS.

### 401 or 403?

Our anonymous `POST` got **403**, not 401, with `"Authentication credentials were not
provided."` DRF returns 401 only when the *first* authentication class can produce a
`WWW-Authenticate` header. `SessionAuthentication` can't, so with it first in the list
the response is 403. Put `TokenAuthentication` first if your clients rely on 401 to know
when to log in again.

## Permissions: what may they do?

Permission classes answer two questions: `has_permission(request, view)` for every
request, and `has_object_permission(request, view, obj)` when `get_object()` fetches a
single object.

| Built-in | Allows |
|----------|--------|
| `AllowAny` | everyone (the default if you set nothing) |
| `IsAuthenticated` | logged-in users |
| `IsAuthenticatedOrReadOnly` | anyone can read; writes need login |
| `IsAdminUser` | `is_staff` users |
| `DjangoModelPermissions` | Django's add/change/delete model permissions (Level 2 · 06) |

The default being `AllowAny` is worth repeating: set `DEFAULT_PERMISSION_CLASSES` to
`IsAuthenticated` project-wide and open things up deliberately.

Object-level: only the reviewer may edit a review.

```python
class IsReviewerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.user_id == request.user.id


class ReviewViewSet(viewsets.ModelViewSet):
    serializer_class = ReviewSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsReviewerOrReadOnly]

    def get_queryset(self):
        return Review.objects.select_related("user")

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

`ana` created a review; `raj` tried to PATCH it:

```text
403 {"detail": "You do not have permission to perform this action."}
```

Two caveats that cause real leaks:

- **Object permissions only run when `get_object()` is called.** The list endpoint never
  calls it, so it shows every object `get_queryset()` returns. For private data, scope
  `get_queryset()` to the user; object permissions are for "can see but can't change".
- **Custom `@action`s and views that fetch objects themselves** (`Book.objects.get(...)`)
  skip object permissions. Always use `self.get_object()`.

## When the database knows more than the serializer

Reviews have a `UniqueConstraint(fields=["book", "user"])`. `ana` posted a second review
for the same book, and the API returned **500**:

```text
django.db.utils.IntegrityError: UNIQUE constraint failed: catalog_review.book_id, catalog_review.user_id
```

DRF does generate validators for unique constraints, but only when every field in the
constraint is a field on the serializer. `user` was read-only and set in
`perform_create()`, so the serializer didn't know about it. The fix is to make `user`
a hidden field with a default:

```python
class ReviewSerializer(serializers.ModelSerializer):
    user = serializers.HiddenField(default=serializers.CurrentUserDefault())

    class Meta:
        model = Review
        fields = ["id", "book", "user", "rating", "body", "created"]
        read_only_fields = ["created"]
```

`repr()` of that serializer now included
`validators = [<UniqueTogetherValidator(queryset=Review.objects.all(), fields=('book', 'user'))>]`,
and the duplicate returned a clean **400**:

```text
{"non_field_errors": ["The fields book, user must make a unique set."]}
```

(Drop the `perform_create` override; the hidden field supplies the user. If you also want
the username in responses, add a separate read-only field for it.)

The same review model has `CheckConstraint(condition=Q(rating__gte=1, rating__lte=5))`.
Posting `rating: 9` produced another 500:

```text
django.db.utils.IntegrityError: CHECK constraint failed: rating_1_to_5
```

DRF doesn't run Django's model validation (`full_clean()`), so check constraints aren't
enforced before saving. Mirror them on the serializer field: `rating =
serializers.IntegerField(min_value=1, max_value=5)`. The database constraint stays as the
last line of defence; the serializer turns violations into helpful 400s.

## Throttling

With the anonymous rate at 5/minute, the sixth anonymous request in a minute got:

```text
429 {"detail": "Request was throttled. Expected available in 60 seconds."}
Retry-After: 60
```

Throttles identify anonymous clients by IP address and authenticated ones by user ID,
and store counters in Django's **cache**. That has consequences:

- With the default local-memory cache, each server process counts separately, so the
  effective limit is multiplied by your process count. Use a shared cache (Redis,
  Memcached) in production.
- Behind a proxy or load balancer, every request appears to come from the proxy's IP
  unless you configure `NUM_PROXIES` so DRF reads the client IP from
  `X-Forwarded-For`. Get it wrong one way and everyone shares one limit; the other way
  and clients can spoof their IP.
- Throttling is a fairness and cost tool, not a security boundary against a determined
  attacker. Put coarse rate limits at the edge (load balancer, CDN) as well.

`ScopedRateThrottle` gives individual endpoints their own budget, e.g. a stricter limit
on a login or search endpoint.

## Pagination

Page-number pagination produced:

```json
{"count": 7, "next": "http://testserver/api/books/?page=2", "previous": null, "results": [...]}
```

and `?page=9` returned `404 {"detail": "Invalid page."}`. Options:

| Class | URLs look like | Notes |
|-------|----------------|-------|
| `PageNumberPagination` | `?page=2` | runs a `COUNT(*)`; simple; pages shift as rows are inserted |
| `LimitOffsetPagination` | `?limit=20&offset=40` | flexible; same count and drift issues; large offsets are slow |
| `CursorPagination` | `?cursor=cD0yMDI2...` | stable under inserts, no count, efficient on huge tables; requires a fixed, unique ordering; no jumping to page N |

For feeds and big tables, prefer cursor pagination. Always cap page size
(`max_page_size` on a subclass) if you let clients choose it.

## How It Actually Works

On every request, `APIView.initial()` runs three steps before your handler.
**Authentication** is lazy: `request.user` triggers each authentication class in order
until one returns a `(user, auth)` tuple; if none does, the user is `AnonymousUser`.
`SessionAuthentication` reads Django's session user and then runs Django's CSRF check
itself (DRF views are CSRF-exempt at the Django level so that token-auth clients aren't
blocked; the session class re-applies the check only when the session is what
authenticated the request). **Permissions** call `has_permission` on each class; any
`False` raises `NotAuthenticated` (if the request had no credentials) or
`PermissionDenied`. **Throttles** call `allow_request()`, which reads a list of
timestamps for the client's key from the cache, discards old ones, and compares the
remainder to the rate.

Object permissions run later, inside `get_object()`, via `check_object_permissions()`,
which is why anything that bypasses `get_object()` bypasses them.

## Common mistakes

- **No `DEFAULT_PERMISSION_CLASSES`**, leaving new endpoints open to everyone.
- **Relying on object permissions to hide data** in list endpoints.
- **Setting server-controlled fields in `perform_create`** and losing uniqueness validation.
- **Model constraints without matching serializer validation**, returning 500s.
- **Per-process throttle counters** from the local-memory cache.
- **DRF's plain `Token` model** for long-lived, high-value credentials.

## Exercise

1. Set `DEFAULT_PERMISSION_CLASSES` to `IsAuthenticated` and open only the book list and
   detail to anonymous users with a per-view override.
2. Reproduce the duplicate-review 500, then fix it with `HiddenField` and confirm the 400.
3. Add `min_value`/`max_value` to `rating` so the check constraint can never be the
   first thing to complain.
4. Switch reviews to `CursorPagination` ordered by `-created`. Add reviews between
   fetching page 1 and page 2: are any skipped or repeated?
5. Add a `ScopedRateThrottle` of 10/minute to the `stats` action and test it.
