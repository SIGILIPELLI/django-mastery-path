---
description: "Level 1 project: a complete Django reading-list app with a model, filtered lists with counts, create/edit/finish/delete views, templates and the admin."
---

# 10 · Project — A Reading-List App

Time to put Level 1 together. You'll build a personal reading list: add books, filter
by status, mark one finished, rate it, edit and delete. It's small, but it uses every
lesson from this level: a project and app, URLs and views, template inheritance, a model
with choices and migrations, ORM queries with an aggregate, the admin, a validated
ModelForm, CSRF-protected POSTs and a static stylesheet.

Everything below was built and exercised end to end on Django 6.1.1 with the test
client; the results are listed at the end so you can compare.

## Requirements

1. A list page showing all books, newest first, with tabs for each status and a count
   per status. An unknown `?status=` value should show everything, not crash or show
   nothing.
2. Add and edit pages sharing one form. Title and author are required. A rating (1–5)
   is allowed only on finished books.
3. A detail page with a **Mark finished** button that sets the status and today's date.
   It must only work via POST.
4. Delete with a confirmation page; the actual delete must be a POST.
5. All entries manageable in the admin, with search and a status filter.

## Step 1 — project and app

```bash
mkdir readinglist && cd readinglist
python3 -m venv .venv && source .venv/bin/activate
python -m pip install django
django-admin startproject config .
python manage.py startapp books
```

Add `"books"` to `INSTALLED_APPS` and set `"DIRS": [BASE_DIR / "templates"]` in
`TEMPLATES`.

## Step 2 — the model

```python title="books/models.py"
from django.db import models
from django.urls import reverse
from django.utils import timezone


class Entry(models.Model):
    class Status(models.TextChoices):
        WANT = "want", "Want to read"
        READING = "reading", "Reading"
        DONE = "done", "Finished"

    title = models.CharField(max_length=200)
    author = models.CharField(max_length=200)
    status = models.CharField(max_length=10, choices=Status, default=Status.WANT)
    rating = models.PositiveSmallIntegerField(null=True, blank=True,
                                              help_text="1–5, only for finished books")
    notes = models.TextField(blank=True)
    added = models.DateTimeField(auto_now_add=True)
    finished_on = models.DateField(null=True, blank=True)

    class Meta:
        ordering = ["-added"]
        verbose_name_plural = "entries"

    def __str__(self):
        return f"{self.title} — {self.author}"

    def get_absolute_url(self):
        return reverse("books:detail", args=[self.pk])

    def mark_finished(self, rating=None):
        self.status = self.Status.DONE
        self.finished_on = timezone.localdate()
        if rating is not None:
            self.rating = rating
        self.save(update_fields=["status", "finished_on", "rating"])
```

Design notes:

- `author` is a plain `CharField` here. A separate `Author` model with a foreign key is
  better once you want "all books by this author"; Level 2 · 01 makes that change.
- `rating` is nullable because "not rated" is genuinely different from any number.
- `mark_finished()` lives on the model, not the view. Business rules belong with the
  data, where the admin, a management command or an API can reuse them.
- `verbose_name_plural` fixes the admin's "Entrys".

```bash
python manage.py makemigrations books
python manage.py migrate
```

## Step 3 — the form

```python title="books/forms.py"
from django import forms
from .models import Entry


class EntryForm(forms.ModelForm):
    class Meta:
        model = Entry
        fields = ["title", "author", "status", "rating", "notes"]
        widgets = {"notes": forms.Textarea(attrs={"rows": 4})}

    def clean_rating(self):
        rating = self.cleaned_data.get("rating")
        if rating is not None and not 1 <= rating <= 5:
            raise forms.ValidationError("Rating must be between 1 and 5.")
        return rating

    def clean(self):
        cleaned = super().clean()
        if cleaned.get("rating") and cleaned.get("status") != Entry.Status.DONE:
            self.add_error("rating", "Only finished books can be rated.")
        return cleaned
```

`self.add_error("rating", ...)` attaches a cross-field error to a specific field instead
of the form-wide `__all__` list, so it appears next to the rating input where the user
will look for it.

## Step 4 — views

```python title="books/views.py"
from django.db.models import Count
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST

from .forms import EntryForm
from .models import Entry


def entry_list(request):
    entries = Entry.objects.all()
    status = request.GET.get("status")
    if status in Entry.Status.values:
        entries = entries.filter(status=status)
    else:
        status = None
    counts = dict(Entry.objects.values_list("status").annotate(n=Count("id")))
    tabs = [(value, label, counts.get(value, 0)) for value, label in Entry.Status.choices]
    return render(request, "books/entry_list.html",
                  {"entries": entries, "status": status, "tabs": tabs})


def entry_detail(request, pk):
    entry = get_object_or_404(Entry, pk=pk)
    return render(request, "books/entry_detail.html", {"entry": entry})


def entry_form(request, pk=None):
    entry = get_object_or_404(Entry, pk=pk) if pk else None
    form = EntryForm(request.POST or None, instance=entry)
    if request.method == "POST" and form.is_valid():
        entry = form.save()
        return redirect(entry)
    return render(request, "books/entry_form.html", {"form": form, "entry": entry})


@require_POST
def entry_finish(request, pk):
    entry = get_object_or_404(Entry, pk=pk)
    entry.mark_finished()
    return redirect(entry)


def entry_delete(request, pk):
    entry = get_object_or_404(Entry, pk=pk)
    if request.method == "POST":
        entry.delete()
        return redirect("books:list")
    return render(request, "books/entry_confirm_delete.html", {"entry": entry})
```

The counts query is worth reading closely. `values_list("status").annotate(n=Count("id"))`
groups by status and counts each group in **one** query, producing pairs like
`("want", 1)`; `dict()` turns them into a lookup. Statuses with no books are missing from
that result, so `counts.get(value, 0)` fills them in. Validating `status` against
`Entry.Status.values` means a hand-edited URL can't produce a confusing empty page.

`EntryForm(request.POST or None, instance=entry)` is a compact idiom: on GET,
`request.POST` is an empty (falsy) QueryDict, so the form is unbound. One view handles
both create (`pk=None`) and edit.

## Step 5 — URLs

```python title="books/urls.py"
from django.urls import path
from . import views

app_name = "books"
urlpatterns = [
    path("", views.entry_list, name="list"),
    path("new/", views.entry_form, name="create"),
    path("<int:pk>/", views.entry_detail, name="detail"),
    path("<int:pk>/edit/", views.entry_form, name="update"),
    path("<int:pk>/finish/", views.entry_finish, name="finish"),
    path("<int:pk>/delete/", views.entry_delete, name="delete"),
]
```

```python title="config/urls.py"
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("books.urls")),
]
```

## Step 6 — templates and CSS

```django title="templates/base.html"
{% load static %}<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Reading list{% endblock %}</title>
  <link rel="stylesheet" href="{% static 'books/site.css' %}">
</head>
<body>
  <header><a href="{% url 'books:list' %}">📚 Reading list</a>
    <a class="button" href="{% url 'books:create' %}">Add a book</a></header>
  <main>{% block content %}{% endblock %}</main>
</body>
</html>
```

```django title="books/templates/books/entry_list.html"
{% extends "base.html" %}
{% block content %}
<nav class="tabs">
  <a href="{% url 'books:list' %}"{% if not status %} aria-current="page"{% endif %}>All</a>
  {% for value, label, n in tabs %}
    <a href="?status={{ value }}"{% if status == value %} aria-current="page"{% endif %}>{{ label }} ({{ n }})</a>
  {% endfor %}
</nav>
<ul class="entries">
{% for entry in entries %}
  <li>
    <a href="{{ entry.get_absolute_url }}">{{ entry.title }}</a> — {{ entry.author }}
    <span class="status">{{ entry.get_status_display }}</span>
    {% if entry.rating %}<span>{{ entry.rating }}/5</span>{% endif %}
  </li>
{% empty %}
  <li>Nothing here yet. <a href="{% url 'books:create' %}">Add your first book.</a></li>
{% endfor %}
</ul>
{% endblock %}
```

```django title="books/templates/books/entry_detail.html"
{% extends "base.html" %}
{% block title %}{{ entry.title }}{% endblock %}
{% block content %}
<h1>{{ entry.title }}</h1>
<p>by {{ entry.author }} · {{ entry.get_status_display }}{% if entry.finished_on %} on {{ entry.finished_on|date:"j M Y" }}{% endif %}</p>
{% if entry.notes %}<p>{{ entry.notes|linebreaksbr }}</p>{% endif %}
{% if entry.status != "done" %}
<form method="post" action="{% url 'books:finish' entry.pk %}">{% csrf_token %}<button>Mark finished</button></form>
{% endif %}
<a href="{% url 'books:update' entry.pk %}">Edit</a> · <a href="{% url 'books:delete' entry.pk %}">Delete</a>
{% endblock %}
```

`entry_form.html` is the form template from lesson 08 with a heading that says "Add a
book" or "Edit “…”" depending on whether `entry` is set. `entry_confirm_delete.html` is a
one-line question plus a POST form with `{% csrf_token %}` and a Cancel link.
`|linebreaksbr` keeps the user's line breaks in notes **and** escapes the text, so notes
can't inject HTML.

A small stylesheet at `books/static/books/site.css` (layout, tab styling, red
`.errorlist`) completes it. Notice `aria-current="page"` on the active tab: it's both
the styling hook and how screen readers announce the current tab.

## Step 7 — admin

```python title="books/admin.py"
from django.contrib import admin
from .models import Entry


@admin.register(Entry)
class EntryAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "status", "rating", "added"]
    list_filter = ["status"]
    search_fields = ["title", "author"]
```

## What we observed

We drove the finished app with Django's test client, in this order:

| Action | Result |
|--------|--------|
| `GET /` on an empty database | 200, shows "Add your first book" |
| POST a *reading* book with rating 4 | 200, form re-shown: "Only finished books can be rated." |
| POST the same book without a rating | 302 → `/1/` |
| `GET /1/finish/` | **405** (the view is POST-only) |
| `POST /1/finish/` | 302; status became `done`, `finished_on` set |
| Edit with rating 9 | "Rating must be between 1 and 5." |
| `GET /` after adding a second, unread book | tabs read `Want to read (1)`, `Reading (0)`, `Finished (1)` |
| `GET /?status=bogus` | 200, all entries listed, no tab highlighted |
| `GET /2/delete/` then `POST /2/delete/` | confirmation page, then 302 → `/`; one entry left |
| `GET /999/` | 404 |

The rating check in `clean_rating()` duplicates what a database `CHECK` constraint could
enforce. Forms protect the UI; constraints protect the data from every other path (bulk
updates, scripts). Level 3 · 05 adds that constraint.

## How It Actually Works

Follow one click on **Mark finished**. The browser POSTs `/1/finish/` with the CSRF
token from the hidden field and the `csrftoken` cookie. Middleware runs top-down:
`SessionMiddleware`, then `CsrfViewMiddleware`, which compares token and cookie before
any of your code runs. The resolver walks `config.urls`, matches the empty prefix of the
`include()`, then `<int:pk>/finish/` in `books.urls`, converting `"1"` to `1`.
`require_POST` checks the method (which is how the GET got a 405 without touching the
database), `get_object_or_404` runs one `SELECT`, `mark_finished()` runs one `UPDATE`
touching three columns because of `update_fields`, and `redirect(entry)` calls
`get_absolute_url()` to build the `Location` header. The browser follows it with a GET,
so refreshing the page now just reloads the detail view, never re-finishes the book.

## Common mistakes

- **Making "finish" or "delete" a link (GET).** Browser prefetchers and crawlers follow
  links; a GET that changes data will eventually be triggered by something that isn't a
  person.
- **Using two different templates for create and edit** that drift apart.
- **Counting each status with a separate `count()` query**, three queries where one
  grouped query will do.
- **Rendering notes with `|safe`** to keep formatting.

## Exercise

Build the app, then extend it:

1. Add a `?q=` search box on the list page that matches title **or** author (you'll need
   a `Q` object: look ahead to Level 2 · 02 or the docs).
2. Add a "Currently reading" section at the top of the list that only shows when there's
   at least one book in that status. Use one query, not two.
3. Add a `started_on` date that's set automatically when status changes to "reading".
   Where should that logic live?
4. Show the average rating of finished books on the list page using `aggregate(Avg(...))`.
