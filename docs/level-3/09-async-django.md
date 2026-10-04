---
description: "Async Django measured: ASGI vs WSGI, async views, the async ORM, SynchronousOnlyOperation, sync_to_async, and when async helped and when it didn't."
---

# 09 · Async Django: ASGI, Async Views & the Async ORM

Django can run `async def` views, has an async-capable ORM interface, and ships an ASGI
entry point (`config/asgi.py`) next to the WSGI one. It's tempting to conclude that
switching to async makes a site faster. Sometimes it does, dramatically; often it makes
no difference; occasionally it makes things worse. This lesson measures both cases on
Django 6.1.1 under Uvicorn 0.54 so you can tell them apart.

## WSGI vs ASGI

- **WSGI** (Gunicorn, uWSGI, mod_wsgi): each request occupies a worker (process or
  thread) from start to finish. Synchronous all the way.
- **ASGI** (Uvicorn, Daphne, Hypercorn): an event loop handles many connections; code
  can `await` I/O and let other requests run meanwhile. Also supports WebSockets and
  long-lived connections (with Django Channels).

Django supports both from the same project. Async views work under WSGI too, but each
one then runs in its own short-lived event loop, so you get the async syntax without the
concurrency benefits. To benefit, serve with ASGI:

```bash
python -m pip install uvicorn
uvicorn config.asgi:application --workers 4
```

## Async views

Any view can be `async def`:

```python
import asyncio, time
from django.http import JsonResponse


async def fetch_price(source, delay):
    await asyncio.sleep(delay)          # stands in for an HTTP call to a price API
    return {"source": source, "delay": delay}


async def prices_sequential(request):
    start = time.perf_counter()
    results = [await fetch_price(s, 0.3) for s in ("a", "b", "c")]
    return JsonResponse({"results": results, "seconds": round(time.perf_counter() - start, 2)})


async def prices_concurrent(request):
    start = time.perf_counter()
    results = await asyncio.gather(*(fetch_price(s, 0.3) for s in ("a", "b", "c")))
    return JsonResponse({"results": list(results), "seconds": round(time.perf_counter() - start, 2)})
```

Measured with Django's `AsyncClient`:

```text
/a/seq/ 200 0.9
/a/con/ 200 0.3
```

Same three 0.3-second waits; the concurrent version overlapped them. **This is the case
where async clearly wins: one request waiting on several independent I/O operations.**
(For real HTTP calls, use an async client such as `httpx.AsyncClient`; calling the
synchronous `requests` library inside an async view blocks the whole event loop.)

## Async vs sync under load: what we measured

The common belief is that async views let one server handle more simultaneous requests.
We tested it: a view that waits 0.5 seconds, written both ways, served by one Uvicorn
process, hit with 200 simultaneous requests:

```python
async def slow_async(request):
    await asyncio.sleep(0.5)
    return JsonResponse({"ok": True})

def slow_sync(request):
    time.sleep(0.5)
    return JsonResponse({"ok": True})
```

```text
/a/slow-async/ 200 concurrent requests: 1.66 {200}
/a/slow-sync/ 200 concurrent requests: 1.54 {200}
/a/slow-async/ 200 concurrent requests: 1.58 {200}
/a/slow-sync/ 200 concurrent requests: 1.56 {200}
```

(Seconds for all 200 to complete, two runs, on an 8-core laptop with the client on the
same machine.) **No meaningful difference.** Under ASGI, Django runs synchronous views in
a thread pool, so blocking sync views still overlap; the remaining time is per-request
overhead in a single process. Your numbers will differ with hardware and workload, but
the lesson generalises: converting ordinary views to `async def` is not a free
performance upgrade. Async pays off for fan-out within a request, for very many long-held
connections (streaming, server-sent events, WebSockets), and for calling async-native
libraries. For typical "query the database, render a template" views, sync views under
Gunicorn remain a perfectly good choice.

## The async ORM

Database access from async code must not block the event loop. Django's guard catches
the mistake:

```python
async def book_count_wrong(request):
    return JsonResponse({"n": Book.objects.count()})
```

```text
django.core.exceptions.SynchronousOnlyOperation: You cannot call this from an async
context - use a thread or sync_to_async.
```

The ORM has async versions of every method that executes a query, prefixed with `a`:

```python
async def book_titles(request):
    n = await Book.objects.acount()
    titles = [b.title async for b in Book.objects.filter(pages__gt=300)]
    first = await Book.objects.select_related("author").afirst()
    return JsonResponse({"n": n, "long": titles, "first_author": first.author.name})
```

```text
{'n': 7, 'long': ['Exhalation', 'Parable of the Sower', 'The Dispossessed'],
 'first_author': 'Ursula K. Le Guin'}
```

Building QuerySets (`filter`, `select_related`) doesn't touch the database, so those stay
synchronous; only evaluation is awaited: `aget`, `afirst`, `acount`, `aexists`,
`acreate`, `aupdate`, `adelete`, `async for`, `instance.asave()`, `instance.adelete()`.
Accessing a relation that isn't loaded (`first.author` without `select_related`) would
trigger a synchronous query and raise `SynchronousOnlyOperation`, so preload relations.

Important caveat: in current Django the async ORM methods largely run the same
synchronous database code **in a thread** for you. It's a correct interface, not
(yet) a fundamentally faster one. Transactions (`atomic()`) don't have an async API;
wrap transactional code in a sync function and call it with `sync_to_async`.

## Bridging with `sync_to_async` and `async_to_sync`

```python
from asgiref.sync import sync_to_async

def _legacy_report():
    return {"done": Book.objects.filter(status="done").count()}

async def legacy(request):
    return JsonResponse(await sync_to_async(_legacy_report)())
```

returned `{'done': 2}`. `sync_to_async` runs the function in a thread (by default the
request's single "thread-sensitive" thread, which keeps database connections and other
thread-bound state consistent). `async_to_sync` goes the other way, for calling async
code from a sync view or management command.

Each switch costs a little (a thread hop). A view that bounces between sync and async a
dozen times can be slower than a plain sync view. The same goes for middleware: if any
middleware in the stack is sync-only, Django adapts around it for every request. All of
Django's built-in middleware supports both modes; check third-party ones
(`sync_capable`/`async_capable` attributes).

## Streaming

Async shines for responses that stay open:

```python
from django.http import StreamingHttpResponse

async def ticker(request):
    async def events():
        for i in range(5):
            yield f"data: tick {i}\n\n"
            await asyncio.sleep(1)
    return StreamingHttpResponse(events(), content_type="text/event-stream")
```

Under ASGI, a thousand clients waiting on this cost a thousand coroutines, not a thousand
threads. Under WSGI, each one would hold a worker for its whole duration.

## How It Actually Works

`ASGIHandler` is an async callable. For each request it creates a
`ThreadSensitiveContext`, builds the `HttpRequest`, and calls the middleware chain, which
was built at startup in async mode where possible. When it reaches a sync view (or sync
middleware), it wraps it with `sync_to_async(thread_sensitive=True)`: the call is sent to
a thread associated with that request's context and the event loop awaits the result.
That's why our sync view still overlapped 200 requests: each request's sync code ran in
its own thread while the loop kept accepting connections.

`SynchronousOnlyOperation` comes from a decorator (`async_unsafe`) on the database
connection's methods: it checks whether an event loop is running in the current thread
and raises if so. The async QuerySet methods are thin wrappers, essentially
`return await sync_to_async(self.count)()`, which is why they're safe today and why the
interface can later gain truly async database drivers without your code changing.

## Common mistakes

- **Converting everything to async** expecting a speed-up without measuring.
- **Blocking calls in async views**: `requests.get`, `time.sleep`, sync ORM calls, file I/O.
- **Lazy relation access** in async code, causing `SynchronousOnlyOperation`.
- **Serving async views with WSGI** and expecting concurrency.
- **Sync-only third-party middleware** forcing an adaptation hop on every request.
- **Using `sync_to_async(thread_sensitive=False)`** for ORM code, which can break
  connection handling.

## Exercise

1. Write the sequential and concurrent "three prices" views and time them with
   `AsyncClient`.
2. Install Uvicorn, serve your project with `uvicorn config.asgi:application`, and repeat
   the 200-request experiment with your own async and sync views. Record your numbers.
3. Rewrite a detail view with `aget()`/`async for`, deliberately forget `select_related`,
   read the error, then fix it.
4. Build a server-sent events endpoint that streams the count of books every second, and
   watch it with `curl -N`.
