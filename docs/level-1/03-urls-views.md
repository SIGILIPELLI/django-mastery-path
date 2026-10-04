---
description: "Map URLs to views with path(), converters, include() and namespaces; read the request, return responses, and reverse URLs instead of hard-coding them."
---

# 03 · URLs and Views

A view is a Python callable that takes an `HttpRequest` and returns an `HttpResponse`.
A URLconf is a list that says which view handles which path. Those two ideas are the
whole of Django's routing, and once they're solid, everything from class-based views to
REST APIs is a variation on them.

## The smallest possible view

```python title="catalog/views.py"
from django.http import HttpResponse

def hello(request):
    return HttpResponse(f"Hello from {request.method} {request.path}")
```

A view doesn't need to render templates or touch the database. This one reads two
attributes of the request and returns plain text. To reach it, the app gets its own
URLconf:

```python title="catalog/urls.py"
from django.urls import path
from . import views

app_name = "catalog"
urlpatterns = [
    path("hello/", views.hello, name="hello"),
]
```

and the project's root URLconf includes it:

```python title="config/urls.py"
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("catalog.urls")),
]
```

Requesting `/hello/` returned `b'Hello from GET /hello/'` when we ran it through
Django's test client.

## Capturing values with path converters

Angle brackets in a route capture part of the path and pass it to the view as a keyword
argument. The text before the colon is a **converter**:

| Converter | Matches | Python type |
|-----------|---------|-------------|
| `str` (default) | anything except `/` | `str` |
| `int` | `0` or a positive integer | `int` |
| `slug` | letters, digits, hyphens, underscores | `str` |
| `uuid` | a formatted UUID | `uuid.UUID` |
| `path` | anything, including `/` | `str` |

```python title="catalog/urls.py"
urlpatterns = [
    path("hello/", views.hello, name="hello"),
    path("", views.book_list, name="book-list"),
    path("books/<int:pk>/", views.book_detail, name="book-detail"),
    path("authors/<slug:name>/", views.author_echo, name="author-echo"),
]
```

```python title="catalog/views.py"
def author_echo(request, name):
    return HttpResponse(f"name={name!r} type={type(name).__name__}")
```

`/authors/ted-chiang/` returned `name='ted-chiang' type=str`, and `/books/abc/` was a
404 without the view ever being called, because `abc` doesn't match `int`. Converters
are your first layer of input validation.

## The request object

Everything about the incoming request hangs off `request`:

| Attribute | Contains |
|-----------|----------|
| `request.method` | `"GET"`, `"POST"`, ... |
| `request.path` | `/books/3/` (no query string) |
| `request.GET` | query-string parameters (a `QueryDict`) |
| `request.POST` | form-encoded body parameters |
| `request.FILES` | uploaded files (lesson 09) |
| `request.headers` | case-insensitive header access: `request.headers["User-Agent"]` |
| `request.COOKIES` | cookies as a dict |
| `request.user` | the logged-in user, or `AnonymousUser` (added by auth middleware) |
| `request.session` | the session (added by session middleware) |

`QueryDict` exists because HTTP allows repeated keys (`?q=1&q=2`). In our shell:

```python
>>> from django.http import QueryDict
>>> q = QueryDict("q=1&q=2&tag=sci-fi")
>>> q["q"], q.getlist("q"), q.get("missing", "dflt")
('2', ['1', '2'], 'dflt')
```

Indexing returns the **last** value; use `getlist()` when you expect several. Use
`.get()` with a default for optional parameters: `request.GET["status"]` raises
`MultiValueDictKeyError`, a `KeyError` subclass that Django doesn't treat specially, so a
missing parameter becomes a 500 server error rather than a 400.

A list view that filters by an optional query parameter:

```python title="catalog/views.py"
from django.shortcuts import render
from .models import Book

def book_list(request):
    books = Book.objects.select_related("author").order_by("title")
    status = request.GET.get("status")
    if status:
        books = books.filter(status=status)
    return render(request, "catalog/book_list.html", {"books": books, "status": status})
```

## Responses

Every view must return an `HttpResponse` (or a subclass) or raise an exception.

```python
from django.http import HttpResponse, JsonResponse, Http404, HttpResponseNotAllowed
from django.shortcuts import redirect, render, get_object_or_404

HttpResponse("text", content_type="text/plain", status=200)
JsonResponse({"count": 3})                  # dicts by default; safe=False for lists
redirect("catalog:book-detail", pk=3)       # 302 to a reversed URL
redirect(book)                              # uses book.get_absolute_url()
render(request, "template.html", context)   # renders a template into a response
raise Http404("No such book")              # becomes a 404 page
```

`get_object_or_404(Model, **lookups)` is the idiom for detail views: it runs `.get()`
and converts `DoesNotExist` into `Http404`. Without it, a missing object raises an
uncaught exception and the user gets a 500.

Restrict methods with decorators instead of `if` ladders when a view only makes sense
for some:

```python
from django.views.decorators.http import require_GET, require_POST

@require_POST
def archive_book(request, pk):
    ...
```

Other methods get a `405 Method Not Allowed` automatically.

## Naming and reversing URLs

Never hard-code `/books/3/` in Python or templates. Name the pattern and **reverse** it:

```python
>>> from django.urls import reverse, resolve
>>> reverse("catalog:book-detail", args=[7])
'/books/7/'
>>> resolve("/books/7/")
ResolverMatch(func=catalog.views.book_detail, args=(), kwargs={'pk': 7},
              url_name='book-detail', app_names=['catalog'],
              namespaces=['catalog'], route='books/<int:pk>/', ...)
```

In templates the same thing is `{% url 'catalog:book-detail' book.pk %}`. Now you can
change `books/` to `library/` in one place.

The `catalog:` prefix is the **namespace** from `app_name = "catalog"`. It stops two
apps that both have a `detail` URL from colliding. The errors are worth recognising.
Forgetting the namespace:

```text
NoReverseMatch: Reverse for 'book-detail' not found. 'book-detail' is not a valid
view function or pattern name.
```

Passing an argument the converter can't accept:

```text
NoReverseMatch: Reverse for 'book-detail' with arguments '('x',)' not found.
1 pattern(s) tried: ['books/(?P<pk>[0-9]+)/\\Z']
```

That second message shows you the regular expression the route compiled into.

## Trailing slashes and `APPEND_SLASH`

With the default `CommonMiddleware` and `APPEND_SLASH = True`, a request for `/hello`
(no slash) that doesn't match, but would match with a slash, is redirected. We saw
`301 /hello/`. That redirect can't preserve a POST body, so for a POST Django raises an
error in development instead of silently dropping data:

```text
RuntimeError: You called this URL via POST, but the URL doesn't end in a slash and you
have APPEND_SLASH set. Django can't redirect to the slash URL while maintaining POST
data. Change your form to point to testserver/hello/ (note the trailing slash), ...
```

Pick one convention (Django's is trailing slashes) and use `{% url %}` everywhere so you
never type a path by hand.

## How It Actually Works

When Django starts handling URLs, each `path()` becomes a `URLPattern` whose route is
compiled to a regex: `books/<int:pk>/` becomes `^books/(?P<pk>[0-9]+)/\Z`, exactly as
the error message showed. `include()` creates a `URLResolver`, a node that matches a
prefix and then hands the *rest* of the path to its own list of patterns. Resolution is
a depth-first walk: try each entry in order, descend into resolvers whose prefix
matches, and stop at the first full match. After a match, each captured string is passed
through the converter's `to_python()` (so `"7"` becomes `7`) and the view is called as
`view(request, **kwargs)`.

Reversing runs the process backwards. Django builds a lookup table from names to
patterns, and for each candidate it calls the converter's `to_url()` and checks that the
result matches the pattern's regex. That's why `reverse()` with `"x"` failed: the string
can't produce a path the `int` pattern would accept. The lookup table is built lazily
and cached per URLconf, so reversing is cheap after the first call.

No match at all raises `Resolver404`, which becomes a 404 response. With `DEBUG=True`
you see a page listing every pattern that was tried, which is the quickest way to debug
routing.

## Common mistakes

- **Ordering a catch-all before specific routes.** `path("<slug:page>/", ...)` placed
  above `path("about/", ...)` swallows `/about/`. Put specific routes first.
- **Forgetting `app_name`** and then using a namespaced name, or vice versa.
- **Reading `request.GET["x"]`** for optional parameters. Use `.get("x")`.
- **Returning `None` from a view** (a forgotten `return`). Django raises "The view ...
  didn't return an HttpResponse object".
- **Changing data on GET.** Anything that modifies state should be POST (crawlers and
  link prefetchers follow GET links).
- **Hard-coding paths** in templates and redirects instead of `{% url %}` and `reverse()`.

## Exercise

1. Add `path("books/<int:pk>/", ...)` and `path("authors/<slug:name>/", ...)` to your
   catalog and confirm in the browser that `/books/abc/` 404s before reaching your view.
2. Write a view `stats(request)` that returns `JsonResponse({"books": Book.objects.count()})`
   (you'll have a `Book` model after lesson 05; until then return a constant).
3. Write a view that accepts `?q=` repeated several times and returns the values joined
   with commas. Test it with `/search/?q=a&q=b&q=c`.
4. Deliberately call `reverse()` with a wrong name and a wrong argument type, and read
   both error messages.
