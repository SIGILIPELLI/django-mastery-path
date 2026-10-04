---
description: "Cache Django apps safely: the low-level cache API, per-view and fragment caching, a real per-user cache leak and its fix, and invalidation strategies."
---

# 07 · Caching

Caching trades freshness for speed: store the result of expensive work and serve it
again instead of recomputing. Done well, it turns a page that runs twenty queries into
one that runs none. Done carelessly, it shows users stale data or, worse, **other users'
data**. We reproduced exactly that leak below with Django's own `cache_page` decorator.
This lesson covers the API from the bottom up and the invalidation habits that keep
caches correct.

## Configuring a backend

With no `CACHES` setting, Django uses a per-process in-memory cache. We printed it:

```text
{'default': {'BACKEND': 'django.core.cache.backends.locmem.LocMemCache'}}
```

Backends:

| Backend | Shared between processes/servers? | Use |
|---------|-----------------------------------|-----|
| `LocMemCache` | **no**, each process has its own | development, tests |
| `RedisCache` (built in since Django 4.0) | yes | the usual production choice |
| `PyMemcacheCache` | yes | production; simple and fast |
| `DatabaseCache` | yes | when you can't run a cache server; slower |
| `FileBasedCache` | per machine | rarely |
| `DummyCache` | — | disables caching (useful in tests) |

```python title="config/settings.py (production)"
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL"),   # e.g. redis://cache:6379/1
    }
}
```

We didn't have a Redis server in the environment used to write this lesson, so all
outputs below come from `LocMemCache`. The API and behaviour are identical across
backends; what differs is that LocMem isn't shared. With four Gunicorn workers, a
LocMem cache means four separate caches, each warmed and invalidated separately, which is
a frequent cause of "I cleared the cache but some requests still show old data".

## The low-level API

```python
from django.core.cache import cache

cache.get("missing")                 # None
cache.get("missing", "dflt")         # 'dflt'
cache.set("k", {"a": 1}, timeout=1)  # seconds; None = forever; 0 = don't cache
cache.get("k")                       # {'a': 1}   ... and after 1.1 s: None
cache.set("n", 0); cache.incr("n"); cache.incr("n", 10)   # 1, then 11
cache.get_or_set("answer", expensive, 60)
```

Our `get_or_set` called twice ran `expensive()` **once**. Values are pickled, so you can
cache dicts, lists and model instances (though caching IDs or plain data is more robust
across deploys).

One subtlety we confirmed: storing `None` is indistinguishable from a miss with `get()`.
After `cache.set("none", None)`, `cache.get("none", "sentinel-default")` returned `None`
(the stored value) while `has_key("none")` was `True`; but a plain `cache.get("none")`
would look like a miss. If `None` is a legitimate result, cache a sentinel or wrap it.

A typical pattern caches a computed summary:

```python
def stats(request):
    data = cache.get("book-stats")
    if data is None:
        data = dict(Book.objects.values_list("status").annotate(n=Count("id")))
        cache.set("book-stats", data, timeout=300)
    return JsonResponse(data)
```

First request: 1 query. Second: **0**.

## Per-view caching, and the leak

`cache_page` caches a view's entire response:

```python
from django.views.decorators.cache import cache_page

@cache_page(60)
def cached_list(request):
    return JsonResponse({"titles": list(Book.objects.values_list("title", flat=True))})
```

First request: 1 query, response headers `Cache-Control: max-age=60` plus `Expires`.
Second: 0 queries. Then we inserted a book and requested again: still **0 queries and 7
titles**. The new book was invisible for up to a minute. That's the deal you make.

Now a view whose output depends on **who** is asking:

```python
@cache_page(60)
def greeting(request):
    name = request.user.username if request.user.is_authenticated else "stranger"
    return HttpResponse(f"Hello, {name}")
```

We requested it as an anonymous visitor, then as `ana`, then as `raj`:

```text
b'Hello, stranger'   b'Hello, stranger'   b'Hello, stranger'
```

Both logged-in users were served the anonymous page. Reverse the order (first visitor
logged in) and everyone would see `ana`'s name, or her account details on a real page.
The cache key is built from the URL plus the headers named in the response's `Vary`
header. `SessionMiddleware` does add `Vary: Cookie` when the session is accessed, but
it does so **after** the view returns, and `cache_page` stores the response before that,
so the cached entry had no `Vary` and one key served everyone. Declaring the dependency
inside the decorator stack fixes it:

```python
from django.views.decorators.vary import vary_on_cookie

@cache_page(60)
@vary_on_cookie
def greeting(request):
    ...
```

```text
b'Hello, stranger'   b'Hello, ana'   b'Hello, raj'      Vary: Cookie
```

Better still: don't page-cache personalised pages at all. Cache the expensive,
shared parts (with the low-level API or fragment caching) and render the personal bits
fresh. Also beware that `cache_page` sets `Cache-Control: max-age`, so **browsers and
CDNs** may cache the response too; for anything personal, mark it `private`
(`@cache_control(private=True)`) or don't cache it.

## Template fragment caching

Cache part of a template, keyed by whatever it depends on:

```django
{% load cache %}
{% cache 600 sidebar request.LANGUAGE_CODE %}
  ... expensive "popular books" sidebar ...
{% endcache %}

{% cache 3600 book_card book.pk book.updated %}
  ... one book's card ...
{% endcache %}
```

Extra arguments become part of the key. Including `book.updated` (an `auto_now`
timestamp) means editing the book automatically uses a new key: no explicit
invalidation needed.

## Invalidation

Phil Karlton's well-known line puts cache invalidation among the two hard things in
computer science. Practical strategies, from simplest:

1. **Short timeouts.** Accept up to N seconds of staleness. Often the right answer.
2. **Key versioning by content.** Put a version or timestamp in the key (`book.updated`
   above, or `f"stats:v{settings.STATS_VERSION}"`). Old entries simply expire unused.
3. **Explicit deletion on write.** `cache.delete("book-stats")` after changes, ideally in
   `transaction.on_commit` so a rollback doesn't leave the cache cleared for data that
   didn't change, and a slow commit doesn't let a reader re-cache old data before it lands.
   Remember Level 3 · 04: `update()` and `bulk_create()` don't fire signals, so
   signal-based invalidation misses them.
4. **Generation counters.** Keep `cache.incr("books:generation")` and include the
   generation in every book-related key; bumping it invalidates them all at once.

Whatever you choose, write a test that changes the data and asserts the new value is
served.

## Other caching layers

- **`select_related`/`prefetch_related` and query fixes come first.** Caching a slow
  page hides the slow page.
- **`@cached_property`** caches a computed attribute for one object's lifetime (one
  request), no backend involved.
- **HTTP caching** with `ETag`/`Last-Modified` (`@condition`, `ConditionalGetMiddleware`)
  lets browsers revalidate cheaply with `304 Not Modified`.
- **The per-site cache middleware** (`UpdateCacheMiddleware`/`FetchFromCacheMiddleware`)
  caches every anonymous GET site-wide. Powerful for mostly-static sites; dangerous if
  any page varies per user without declaring it.

## How It Actually Works

Every backend implements the same `BaseCache` interface. Keys are transformed by
`make_key()` into `"<KEY_PREFIX>:<VERSION>:<key>"`, so you can isolate environments with
`KEY_PREFIX` and invalidate everything at once by bumping `VERSION`. Values are pickled
before storage (Redis and Memcached store bytes).

`cache_page` wraps the view in `CacheMiddleware` logic. On a request, it builds a
**header key** from the URL; stored under that key is the list of headers from the
response's `Vary`. It then builds the real cache key from the URL plus the *values* of
those headers in the current request (e.g. the cookie). On a miss, it calls the view and,
if the response is cacheable (GET/HEAD, status 200, no `Cache-Control: private` or
`no-store`), stores it and records its `Vary` list. Because the decorator sits inside
the middleware stack, it sees the response before `SessionMiddleware` adds `Vary:
Cookie`, which is the precise mechanism behind the leak we observed.

## Common mistakes

- **`cache_page` on personalised views** without `vary_on_cookie`.
- **LocMem in production** with several processes, giving inconsistent caches.
- **Invalidating before commit**, or relying on signals that `update()` skips.
- **Caching `None`** and treating it as a miss.
- **Caching instead of fixing N+1 queries.**
- **Unbounded key spaces** (`f"search:{user_query}"`) filling memory with one-off keys.

## Exercise

1. Add `@cache_page(60)` to your catalogue's list view. Measure queries on the first and
   second request, then add a book and observe the staleness.
2. Reproduce the per-user leak with a "Hello, name" view, then fix it with
   `vary_on_cookie`. Write a test that logs in two different users and asserts each sees
   their own name.
3. Cache the per-status counts with the low-level API and invalidate them in
   `on_commit` whenever a book is created or changes status. Test it.
4. Add a fragment cache around each book card keyed by `book.pk` and `book.updated`.
