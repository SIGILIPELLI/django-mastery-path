---
description: "Turn models into a working back office with the Django admin: superusers, ModelAdmin options, search, filters, inlines, custom actions, and its limits."
---

# 07 · The Django Admin

The admin is a complete create/read/update/delete interface generated from your models.
For an internal team managing content, orders or users, it can replace weeks of
back-office work. It's also the fastest way to put real data into a project you're
learning. This lesson takes it from the default to a properly configured screen, and is
honest about where it stops being the right tool.

## Turning it on

The admin is already in `INSTALLED_APPS` and `config/urls.py` (`path("admin/",
admin.site.urls)`) in a new project. You need a user who can log in:

```bash
python manage.py createsuperuser
```

It asks for a username, email and password, and runs the same password validators as
the rest of the site (too short or too common and it asks again). Visit `/admin/`. If
you're not logged in, you're redirected; we saw:

```text
302 → /admin/login/?next=/admin/catalog/book/
```

Nothing from `catalog` appears yet, because models have to be **registered**.

## Registering models

The minimum is one line per model:

```python title="catalog/admin.py"
from django.contrib import admin
from .models import Tag

admin.site.register(Tag)
```

That gives a list of tags (shown using `__str__`, which is why every model should define
it) and an add/edit form built from the model's fields. `blank`, `choices`, `max_length`
and `help_text` all flow through into the form.

## Configuring a `ModelAdmin`

For any model people will actually manage, write a `ModelAdmin`:

```python title="catalog/admin.py"
from django.contrib import admin
from .models import Author, Book, Review, Tag


class ReviewInline(admin.TabularInline):
    model = Review
    extra = 0


@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "status", "pages", "published"]
    list_filter = ["status", "tags"]
    search_fields = ["title", "author__name"]
    list_select_related = ["author"]
    autocomplete_fields = ["author"]
    filter_horizontal = ["tags"]
    inlines = [ReviewInline]
    actions = ["mark_done"]

    @admin.action(description="Mark selected books as finished")
    def mark_done(self, request, queryset):
        updated = queryset.update(status=Book.Status.DONE)
        self.message_user(request, f"{updated} book(s) marked finished.")


@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    search_fields = ["name"]


admin.site.register(Tag)
```

What each option does:

| Option | Effect |
|--------|--------|
| `list_display` | columns on the change list (fields, methods, or callables) |
| `list_filter` | the filter sidebar; works on choices, booleans, dates and relations |
| `search_fields` | the search box; `author__name` searches across the join |
| `list_select_related` | controls which relations are joined into the list query |
| `autocomplete_fields` | replaces a `<select>` of every author with an AJAX search box |
| `filter_horizontal` | a two-pane picker for many-to-many fields |
| `inlines` | edit related rows (reviews) on the parent's page |
| `actions` | bulk operations on selected rows |

Other options you'll reach for: `readonly_fields`, `fieldsets` (group fields into
sections), `prepopulated_fields` (fill a slug from a title), `date_hierarchy`,
`ordering`, `list_per_page`, and `list_editable` (edit cells in the list itself).

### The admin checks your configuration

`autocomplete_fields` needs the *related* model's admin to have `search_fields`, because
that's what the AJAX endpoint searches. When we left `search_fields` off `AuthorAdmin`,
`python manage.py check` refused to start:

```text
ERRORS:
<class 'catalog.admin.BookAdmin'>: (admin.E040) AuthorAdmin must define "search_fields",
because it's referenced by BookAdmin.autocomplete_fields.
```

Admin mistakes (a typo in `list_display`, a non-existent field in `search_fields`) are
caught at startup like this, not when someone clicks the page. Run `check` often.

## Measuring the change list

The admin is not magic about performance, so we measured it. Loading
`/admin/catalog/book/` as a superuser ran **6** queries: the session, the user, the tag
list for the filter sidebar, two `COUNT(*)` queries (the filtered count and the
full-table count the paginator shows), and one query for the rows with the author joined. We then
set `list_select_related = False` and got the same 6: when a plain foreign key appears
in `list_display`, the admin applies `select_related()` for it automatically.
`list_select_related` matters when you want to control *which* relations are joined.

The automatic join doesn't help with **methods**. We added a column to `AuthorAdmin`
that returned `obj.books.count()`, and the author list went to 8 queries for 3 authors:
one extra `COUNT` per row. On a page of 100 authors that's 100 extra queries. Override
`get_queryset()` to compute the value in the main query instead:

```python
from django.db.models import Count

@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    list_display = ["name", "born", "book_count"]
    search_fields = ["name"]

    def get_queryset(self, request):
        return super().get_queryset(request).annotate(n_books=Count("books"))

    @admin.display(ordering="n_books", description="Books")
    def book_count(self, obj):
        return obj.n_books
```

`@admin.display(ordering=...)` also makes the computed column sortable.

## Who can see what

Only users with `is_staff=True` can log into the admin. Inside it, a non-superuser sees
only models they have permissions for: Django creates `add`, `change`, `delete` and
`view` permissions for every model automatically, and you grant them to users or groups
(Level 2 · 06). Superusers bypass permission checks entirely, so don't hand that flag
out casually.

Custom actions should check permissions too. `@admin.action(permissions=["change"])`
hides the action from users who can't change the model.

## When not to use the admin

The admin is built for **trusted staff doing data management**. It's a poor fit for:

- Customer-facing pages (users, not staff). Build real views.
- Complex multi-step workflows. Every page is a form for one model plus inlines.
- Very large tables without tuning. The default change list counts the whole table for
  pagination; for tens of millions of rows, set `show_full_result_count = False` and
  consider a custom paginator.

Treat the admin URL as sensitive. Moving it off `/admin/` reduces noise from automated
login attempts, but it's not security on its own: strong passwords, and ideally
two-factor authentication via a third-party package, are what actually protect it.

## How It Actually Works

`admin.site` is an `AdminSite` instance holding a registry: a dict from model class to
`ModelAdmin` instance. `@admin.register` and `admin.site.register()` add entries. On
startup, `django.contrib.admin`'s `AppConfig.ready()` calls `autodiscover()`, which
imports the `admin` module of every installed app. That import is what runs your
registrations, and it's why `admin.py` must exist with that exact name.

`admin.site.urls` generates URL patterns from the registry: for each model, a change
list, add, change, delete and history view, all methods on its `ModelAdmin`. The change
form is a `ModelForm` built dynamically with `modelform_factory()` from the model's
fields, filtered by `fields`/`exclude`/`readonly_fields`. The change list is a
`ChangeList` object that builds a QuerySet from `get_queryset()`, applies the search
terms (as `icontains` lookups OR'd across `search_fields`), the filters and ordering,
and then paginates it. Everything is overridable because it's all plain methods on
`ModelAdmin`.

## Common mistakes

- **No `__str__` on models**, so the admin shows "Book object (3)".
- **Methods in `list_display` that query relations** (`obj.books.count()`), causing
  one extra query per row on every list page.
- **A `<select>` with 50,000 options** for a foreign key. Use `autocomplete_fields` or
  `raw_id_fields`.
- **Giving `is_superuser` to staff** who only need to edit a few models. Use groups.
- **Building customer features in the admin** because it's quick. It becomes the product.

## Exercise

1. Create a superuser, register `Author`, `Book` and `Tag`, and add five books through
   the admin.
2. Configure `BookAdmin` with `list_display`, `list_filter`, `search_fields` and
   `autocomplete_fields`. Remove `search_fields` from `AuthorAdmin` and read the check
   error.
3. Add the `book_count` column to `AuthorAdmin` and make it sortable.
4. Write an action "Mark as want to read" and use it on two books.
5. Create a staff user (not superuser) with only the "view book" permission and log in
   as them. What can they see?
