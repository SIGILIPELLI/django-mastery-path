---
description: "Django class-based and generic views: how as_view() and dispatch work, ListView, DetailView, editing views, mixins, the MRO, and when to use functions."
---

# 04 · Class-Based & Generic Views

Every view in Level 1 was a function. Django also lets a view be a **class**, and ships a
library of **generic views** that implement the patterns you keep writing: show a list,
show one object, create from a form, update, delete. Used well, they remove boilerplate.
Used badly, they turn a five-line function into a scavenger hunt through six parent
classes. This lesson shows both the mechanics and the judgment.

## The same view, twice

The Level 1 detail view as a function:

```python
def book_detail(request, pk):
    book = get_object_or_404(Book.objects.select_related("author"), pk=pk)
    return render(request, "catalog/book_detail.html", {"book": book})
```

As a generic class-based view:

```python title="catalog/cbv.py"
from django.views.generic import DetailView

class BookDetailView(DetailView):
    queryset = Book.objects.select_related("author").prefetch_related("tags")
```

```python title="catalog/cbv_urls.py"
path("<int:pk>/", cbv.BookDetailView.as_view(), name="cbv-detail"),
```

Our test request for `/cbv/1/` returned 200 rendered from `catalog/book_detail.html`,
the same template as before. `DetailView` worked out the template name from the model
(`<app>/<model>_detail.html`), fetched the object by the `pk` URL keyword, raised 404 if
missing, and put it in the context as both `object` and `book`.

## The core generic views

| View | Does | Default template | Context |
|------|------|------------------|---------|
| `ListView` | list a QuerySet, optional pagination | `<app>/<model>_list.html` | `object_list`, `<model>_list`, `page_obj`, `paginator`, `is_paginated` |
| `DetailView` | one object by `pk` or `slug` | `<app>/<model>_detail.html` | `object`, `<model>` |
| `CreateView` | ModelForm; save; redirect | `<app>/<model>_form.html` | `form` |
| `UpdateView` | ModelForm bound to an object | `<app>/<model>_form.html` | `form`, `object` |
| `DeleteView` | confirm on GET, delete on POST | `<app>/<model>_confirm_delete.html` | `object` |
| `TemplateView` | render a template | `template_name` (required) | URL kwargs |
| `RedirectView` | redirect | — | — |
| `FormView` | a non-model form | `template_name` | `form` |

## A paginated, filterable list

```python title="catalog/cbv.py"
from django.views.generic import ListView

class BookListView(ListView):
    model = Book
    paginate_by = 2
    context_object_name = "books"

    def get_queryset(self):
        qs = super().get_queryset().select_related("author")
        if status := self.request.GET.get("status"):
            qs = qs.filter(status=status)
        return qs
```

With six books, `/cbv/` rendered page 1 of 3 and the context contained `books`,
`object_list`, `page_obj`, `paginator` and `is_paginated`. `/cbv/?page=last` returned
page 3, and `/cbv/?page=99` returned **404**, not an empty page. A pagination block for
the template:

```django
{% if is_paginated %}
<nav aria-label="Pagination">
  {% if page_obj.has_previous %}<a href="?page={{ page_obj.previous_page_number }}">Previous</a>{% endif %}
  Page {{ page_obj.number }} of {{ paginator.num_pages }}
  {% if page_obj.has_next %}<a href="?page={{ page_obj.next_page_number }}">Next</a>{% endif %}
</nav>
{% endif %}
```

(If you also filter by `?status=`, include it in those links or the filter is lost on
page 2.)

Overriding **one method** (`get_queryset`) and setting attributes is the sweet spot for
generic views. The hooks you'll override most:

- `get_queryset()` — what rows; filter by user, add `select_related`.
- `get_context_data(**kwargs)` — add extra template variables (call `super()` first).
- `get_object()` — custom lookup for detail/update/delete.
- `form_valid(form)` — act before/after saving (e.g. set the owner).
- `get_success_url()` — where to go after a successful POST.

## Editing views and access control

```python title="catalog/cbv.py"
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, DeleteView, UpdateView


class BookCreateView(LoginRequiredMixin, CreateView):
    model = Book
    fields = ["title", "author", "pages", "status"]
    # success URL defaults to the new object's get_absolute_url()


class BookUpdateView(PermissionRequiredMixin, UpdateView):
    model = Book
    fields = ["title", "pages", "status"]
    permission_required = "catalog.change_book"


class BookDeleteView(LoginRequiredMixin, DeleteView):
    model = Book
    success_url = reverse_lazy("cbv-list")
```

What we observed with the test client:

| Request | Result |
|---------|--------|
| anonymous `GET /cbv/new/` | 302 → `/accounts/login/?next=/cbv/new/` |
| anonymous `GET /cbv/1/edit/` | 302 → login (anonymous users are sent to log in) |
| logged in as `raj`, no permission, `GET /cbv/1/edit/` | **403** |

Note `reverse_lazy`: class attributes are evaluated at import time, before the URLconf
is loaded, so `reverse()` there would fail. `reverse_lazy` defers the lookup until the
value is used.

To set fields the user shouldn't control, override `form_valid`:

```python
class ReviewCreateView(LoginRequiredMixin, CreateView):
    model = Review
    fields = ["rating", "body"]

    def form_valid(self, form):
        form.instance.user = self.request.user
        form.instance.book = get_object_or_404(Book, pk=self.kwargs["book_pk"])
        return super().form_valid(form)
```

And to restrict objects to their owner, filter in `get_queryset` so other people's
objects are simply 404s:

```python
def get_queryset(self):
    return super().get_queryset().filter(owner=self.request.user)
```

## Mixins and the MRO

Mixins must come **before** the view class. Python resolves methods left to right along
the class's MRO (method resolution order), and we printed the one for our list view:

```text
['BookListView', 'ListView', 'MultipleObjectTemplateResponseMixin', 'TemplateResponseMixin',
 'BaseListView', 'MultipleObjectMixin', 'ContextMixin', 'View', 'object']
```

Each generic view is assembled from small mixins: `MultipleObjectMixin` knows about
QuerySets and pagination, `TemplateResponseMixin` knows about templates, `View` knows
about HTTP methods. `LoginRequiredMixin` works by overriding `dispatch()`; if it came
after `CreateView`, `View.dispatch` would run first and your check would never execute.
When a generic view does something surprising, the MRO is the map: find which class in
the list defines the method you're curious about. (The third-party site
*ccbv.co.uk* shows every attribute and method of each generic view, flattened.)

## When to stay with functions

Generic views earn their keep for standard CRUD. Prefer a function when:

- the view does something non-standard (two forms on one page, a multi-step flow);
- you find yourself overriding four or more methods to bend a generic view;
- the reader would need to know the generic view's internals to follow the logic.

A plain `View` subclass with `get()` and `post()` methods is a reasonable middle ground:
explicit like a function, with method dispatch built in. Mixing styles in one project is
normal.

## How It Actually Works

`as_view()` is a class method that returns a **function**, because the URL dispatcher
only knows how to call functions. That function, on every request, creates a **new
instance** of your class (so instance attributes are per-request and safe), calls
`setup()` to store `request`, `args` and `kwargs` on `self`, then calls `dispatch()`.
`dispatch()` lowercases the HTTP method and looks for a method of that name: `get`,
`post`, and so on. If there isn't one, it calls `http_method_not_allowed()`, which returns
405.

For `ListView`, `get()` calls `get_queryset()`, then `get_context_data()` (which
paginates), then `render_to_response()`, which builds a `TemplateResponse`: a response
whose template isn't rendered until the very end of the request cycle, so middleware can
still modify its context. `CreateView.post()` builds the form with `get_form()`, calls
`form_valid()` or `form_invalid()`, and `form_valid()` saves and redirects to
`get_success_url()`.

Class attributes like `model` and `paginate_by` are shared by all requests. Assigning a
mutable class attribute (a list you `.append()` to in a request) is a bug that leaks
data between users; Django guards the most obvious version by rejecting keyword
arguments to `as_view()` that aren't existing class attributes.

## Common mistakes

- **Mixin order:** `class V(CreateView, LoginRequiredMixin)` silently skips the login check.
- **`reverse()` in a class attribute**; use `reverse_lazy`.
- **Forgetting `super()`** in `get_context_data` and losing `object_list` or `form`.
- **Checking ownership in `get_object()` only for update** but forgetting delete. Filter
  in `get_queryset()` on every view that touches the model.
- **`fields = "__all__"`** on create/update views, the same mass-assignment risk as forms.
- **Pagination links that drop the filter query string.**

## Exercise

1. Convert the Level 1 reading-list views to `ListView`, `DetailView`, `CreateView`,
   `UpdateView` and `DeleteView`. Keep "mark finished" as a function view.
2. Add pagination (10 per page) and make the status filter survive page changes.
3. Add `get_context_data` that adds the per-status counts from Level 1's project.
4. Print the MRO of your `UpdateView` and find which class defines `get_form_class`.
5. Put `LoginRequiredMixin` on the wrong side of `CreateView`, log out, and confirm the
   page is (wrongly) reachable. Then fix it.
