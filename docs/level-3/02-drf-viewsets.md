---
description: "Build REST endpoints with DRF: APIView, generic views, ModelViewSet, routers, custom actions, per-action querysets, and what the router generates."
---

# 02 · DRF Views, ViewSets & Routers

Lesson 01 turned models into JSON. This lesson exposes them over HTTP. DRF offers the
same ladder as Django's own views: plain functions and classes at the bottom, generic
views in the middle, and **ViewSets** with **routers** at the top, which generate a full
set of CRUD endpoints from one class. We'll build a books-and-reviews API on the
catalogue and exercise it with DRF's `APIClient`. All responses shown are what we
received.

## The ladder

```python
# 1. APIView: you write each method
from rest_framework.views import APIView
from rest_framework.response import Response

class BookCount(APIView):
    def get(self, request):
        return Response({"count": Book.objects.count()})

# 2. Generic views: one resource, standard behaviour
from rest_framework import generics

class BookList(generics.ListCreateAPIView):
    queryset = Book.objects.select_related("author")
    serializer_class = BookSerializer

# 3. ViewSet + router: list, create, retrieve, update, partial_update, destroy
from rest_framework import viewsets

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.select_related("author")
    serializer_class = BookSerializer
```

`APIView` differs from Django's `View` in useful ways: `request.data` parses JSON (and
forms) for any method, `Response` renders to JSON or the browsable HTML API depending on
the `Accept` header, exceptions like `ValidationError` and `PermissionDenied` become proper
error responses, and authentication, permissions and throttling run before your method.

## A real ViewSet

```python title="catalog/api.py"
from django.db.models import Avg, Count
from rest_framework import filters, permissions, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

from .models import Book
from .serializers import BookDetailSerializer, BookSerializer


class BookViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ["title", "author__name"]
    ordering_fields = ["title", "pages", "published"]

    def get_queryset(self):
        qs = Book.objects.select_related("author").prefetch_related("tags")
        if self.action == "retrieve":
            qs = (qs.prefetch_related("reviews__user")
                    .annotate(average_rating=Avg("reviews__rating")))
        if status_ := self.request.query_params.get("status"):
            qs = qs.filter(status=status_)
        return qs

    def get_serializer_class(self):
        return BookDetailSerializer if self.action == "retrieve" else BookSerializer

    @action(detail=True, methods=["post"])
    def finish(self, request, pk=None):
        book = self.get_object()
        book.status = Book.Status.DONE
        book.save(update_fields=["status"])
        return Response(self.get_serializer(book).data)

    @action(detail=False)
    def stats(self, request):
        return Response(dict(Book.objects.values_list("status").annotate(n=Count("id"))))
```

`self.action` tells you which operation is running (`"list"`, `"retrieve"`, `"create"`,
`"finish"`...), so one class can use a light serializer and QuerySet for lists and a
richer one for detail. (`BookDetailSerializer` here declares `average_rating` as a plain
read-only field, reading the annotation, per lesson 01's query-count fix.)

## Routers

```python title="catalog/api_urls.py"
from rest_framework.routers import DefaultRouter
from . import api

router = DefaultRouter()
router.register("books", api.BookViewSet, basename="book")
router.register("reviews", api.ReviewViewSet, basename="review")
urlpatterns = router.urls
```

```python title="config/urls.py"
path("api/", include("catalog.api_urls")),
```

We printed what the router generated:

```text
^books/$                                  -> book-list
^books\.(?P<format>[a-z0-9]+)/?$          -> book-list
^books/stats/$                            -> book-stats
^books/(?P<pk>[^/.]+)/$                   -> book-detail
^books/(?P<pk>[^/.]+)/finish/$            -> book-finish
^reviews/$                                -> review-list
^reviews/(?P<pk>[^/.]+)/$                 -> review-detail
                                          -> api-root
...plus a format-suffix variant of each
```

Two URL patterns per resource, not six: `book-list` handles GET (list) and POST
(create); `book-detail` handles GET, PUT, PATCH and DELETE. Custom `@action`s get their
own routes, named `<basename>-<action>`. `DefaultRouter` also adds an API root view
listing the resources, and `.json`-style format suffixes. (`SimpleRouter` omits both.)
`basename` is required here because the viewset defines `get_queryset()` rather than a
`queryset` attribute, so the router can't infer a name.

## What the API did

Requests made with `APIClient`, anonymous unless stated:

| Request | Status | Response (trimmed) |
|---------|--------|--------------------|
| `GET /api/books/` | 200 | `{"count": 7, "next": "http://testserver/api/books/?page=2", "previous": null, "results": [...]}` |
| `GET /api/books/?search=chiang&ordering=-pages` | 200 | titles: `Exhalation`, `Stories of Your Life and Others` |
| `GET /api/books/?page=9` | 404 | `{"detail": "Invalid page."}` |
| `GET /api/books/3/` | 200 | keys include `reviews` and `average_rating` (detail serializer) |
| `GET /api/books/stats/` | 200 | `{"done": 2, "reading": 1, "want": 4}` |
| `POST /api/books/` | 403 | `{"detail": "Authentication credentials were not provided."}` |
| authenticated `POST /api/books/` | 201 | the new book, `"id": 8` |
| authenticated `PATCH /api/books/8/` with `{"pages": "lots"}` | 400 | `{"pages": ["A valid integer is required."]}` |
| authenticated `POST /api/books/8/finish/` | 200 | `"status": "done"`, `"published": null` |
| authenticated `DELETE /api/books/8/` | 204 | — |

(Pagination and authentication are configured in lesson 03.)

Look at the `finish` row. Lesson 01's serializer insists that a finished book has a
publication date, yet the action finished a book without one. Custom actions that change
models directly **bypass serializer validation**. Either route the change through a
serializer (`self.get_serializer(book, data={"status": "done"}, partial=True)` plus
`is_valid(raise_exception=True)`), or put the rule somewhere every path respects: a model
method or a database constraint (lesson 05).

## Hooks you'll override

| Hook | Use for |
|------|---------|
| `get_queryset()` | scoping to the user, per-action optimisation |
| `get_serializer_class()` | different serializers per action |
| `perform_create(serializer)` | `serializer.save(owner=self.request.user)` |
| `perform_update` / `perform_destroy` | side effects, soft deletes |
| `get_permissions()` | different permissions per action |
| `get_object()` | custom lookups; also runs object-level permission checks |

For a resource that shouldn't support every operation, compose mixins instead of using
`ModelViewSet`:

```python
from rest_framework import mixins, viewsets

class ReviewViewSet(mixins.CreateModelMixin, mixins.ListModelMixin,
                    mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    ...   # no update or delete endpoints exist at all
```

An operation that doesn't exist can't be misused, which is better than one that exists
and is forbidden.

## How It Actually Works

A ViewSet is a class whose methods are named after **actions**, not HTTP verbs. The
router calls `BookViewSet.as_view({"get": "list", "post": "create"})` for the list route
and `as_view({"get": "retrieve", "put": "update", "patch": "partial_update", "delete":
"destroy"})` for the detail route. That mapping is the whole trick: `as_view()` produces a
view function that, on each request, binds the HTTP method to the mapped action method
and sets `self.action`. Routes are only generated for actions the class actually has,
which is why removing a mixin removes the endpoint.

DRF's `APIView.dispatch()` wraps Django's request in a DRF `Request` (lazy body
parsing, `request.data`, `request.query_params`, authenticated `request.user`), then
calls `initial()`, which runs **authentication, permission checks and throttling in that
order**, and only then the handler. Any `APIException` raised anywhere is turned into a
`Response` by the exception handler. `ModelViewSet.list()` itself is short: filter the
QuerySet, paginate it, serialize the page with `many=True`, return
`get_paginated_response()`.

## Common mistakes

- **`queryset = Book.objects.all()` as a class attribute** with relations in the
  serializer: N+1 on every list call. Optimise in `get_queryset()`.
- **Custom actions that skip validation** and break invariants.
- **`ModelViewSet` by reflex** for resources that should be read-only or append-only.
- **Filtering on unvalidated query parameters** (`?status=anything`). Validate them, or
  use `django-filter`, a third-party package that turns query params into validated
  filter forms.
- **Returning 403 vs 404 inconsistently** for objects the user can't see (lesson 03).

## Exercise

1. Build `BookViewSet` and register it with a `DefaultRouter`. Print `router.urls`.
2. Use the browsable API in a browser at `/api/books/` to create a book.
3. Add an `@action(detail=True, methods=["post"]) def tag(...)` that adds a tag by name,
   validating input with a small serializer.
4. Rewrite `finish` so it can't produce a finished book without a publication date.
5. Make `ReviewViewSet` create/list/retrieve only, and confirm `DELETE` now returns 405.
