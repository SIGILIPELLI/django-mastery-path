---
description: "Authorize users in Django with model permissions, groups, custom permissions, checks in views and templates, permission caching and object-level rules."
---

# 06 · Permissions, Groups & Object-Level Access

Authentication tells you *who* someone is. Authorization decides *what they may do*.
Django gives you a model-level permission system out of the box and leaves object-level
rules ("only the author can edit this review") to you. Most real applications need both,
and most authorization bugs come from mixing them up or forgetting one view. This lesson
covers the built-in system precisely, then the patterns for per-object rules.

## The permissions Django creates for you

For every model, after `migrate`, Django creates four permissions. For `Book`:

```text
['add_book', 'change_book', 'delete_book', 'view_book']
```

They're referred to as `"<app_label>.<codename>"`, e.g. `"catalog.change_book"`. They're
rows in the `auth_permission` table, tied to the model's content type, and they say
nothing about *which* books: "change_book" means "may change books" in general.

Add your own in the model's `Meta`:

```python
class Book(models.Model):
    ...
    class Meta:
        permissions = [
            ("publish_book", "Can publish a book"),
            ("feature_book", "Can feature a book on the home page"),
        ]
```

They're created by the next `migrate`.

## Groups

Assigning permissions user by user doesn't scale. Put them on a **group** and add users
to it:

```python
from django.contrib.auth.models import Group, Permission

editors, _ = Group.objects.get_or_create(name="Editors")
editors.permissions.add(
    Permission.objects.get(codename="change_book"),
    Permission.objects.get(codename="view_book"),
)
raj.groups.add(editors)
```

Groups map to roles in your organisation ("Editors", "Support", "Billing admins"). A
user's effective permissions are the union of their own and all their groups'. Create
groups in a data migration (Level 3 · 06) so every environment has the same roles, rather
than clicking them together in each environment's admin.

## Checking permissions

```python
user.has_perm("catalog.change_book")          # one permission
user.has_perms(["catalog.change_book", "catalog.view_book"])
user.has_module_perms("catalog")              # any permission in the app
```

In views:

```python
from django.contrib.auth.decorators import permission_required

@permission_required("catalog.publish_book", raise_exception=True)
def publish(request, pk):
    ...
```

```python
from django.contrib.auth.mixins import PermissionRequiredMixin

class BookUpdateView(PermissionRequiredMixin, UpdateView):
    permission_required = "catalog.change_book"
```

In templates, through the `perms` variable from the auth context processor:

```django
{% if perms.catalog.change_book %}<a href="{% url 'cbv-update' book.pk %}">Edit</a>{% endif %}
```

We rendered three checks for `raj` once he was in Editors:

```text
can edit||some catalog perm
```

`change_book` true, `delete_book` false, and `perms.catalog` true because he has at least
one permission in the app. **Hiding a link is not authorization.** The view must check
too, because anyone can type the URL.

What we observed in views: before joining Editors, `raj` got **403** on the update view;
after, **200**. An anonymous user got a redirect to login instead of 403 (the mixin
treats "not logged in" and "logged in but not allowed" differently, which is what users
expect). And `raj` still couldn't use the admin: permissions don't grant admin access
without `is_staff=True`.

## Three behaviours that surprise people

We tested each one:

**1. Permissions are cached on the user object.**

```python
>>> raj.has_perm("catalog.change_book")
False
>>> raj.groups.add(editors)
>>> raj.has_perm("catalog.change_book")      # same object
False
>>> raj = User.objects.get(pk=raj.pk)
>>> raj.has_perm("catalog.change_book")      # re-fetched
True
```

The first check loaded and cached the permission set on that instance. This matters in
tests and scripts that grant a permission and check it in the same breath; in normal
requests, each request loads a fresh user.

**2. Superusers have every permission, even ones that don't exist.**

```python
>>> admin.has_perm("catalog.nonexistent_perm")
True
```

A typo in a permission name is invisible when testing as a superuser. Test authorization
with an ordinary user.

**3. Inactive users have none.** Setting `is_active = False` made `has_perm` return
`False` for `raj` even though his groups were unchanged. Deactivate rather than delete
accounts you want to keep for history.

## Object-level access

"Only the reviewer can edit their review." The built-in `ModelBackend` doesn't do this;
we checked, and `raj.has_perm("catalog.change_book", book)` returned **False** even though
he has `change_book`, because the default backend returns no permissions for object
checks. You have three good options.

**Option A — scope the QuerySet.** The simplest and most robust: users can only ever
*find* their own objects.

```python
class ReviewUpdateView(LoginRequiredMixin, UpdateView):
    model = Review
    fields = ["rating", "body"]

    def get_queryset(self):
        return Review.objects.filter(user=self.request.user)
```

Someone else's review is a 404, which also doesn't reveal that it exists. Apply the same
filter to every view of that model (detail, update, delete, any API endpoint).

**Option B — an explicit check function** when the rule is richer than ownership:

```python
from django.core.exceptions import PermissionDenied

def can_edit_review(user, review):
    return review.user_id == user.id or user.has_perm("catalog.moderate_review")

def review_edit(request, pk):
    review = get_object_or_404(Review, pk=pk)
    if not can_edit_review(request.user, review):
        raise PermissionDenied
    ...
```

`PermissionDenied` becomes a 403 response. Keeping rules in named functions (or a
`rules.py` module) gives you one place to read and test them.

**Option C — an object-permission backend.** Third-party packages such as
`django-guardian` (per-object permission rows) or `django-rules` (predicates) plug into
`has_perm(perm, obj)` so the same API works for objects. Worth it when you have many
object types with sharing rules; overkill for "owner can edit".

## How It Actually Works

`user.has_perm()` first returns `True` for active superusers and `False` for inactive
users, then asks each backend in `AUTHENTICATION_BACKENDS` in turn; any backend saying
yes grants it. `ModelBackend.has_perm` calls `get_all_permissions()`, which runs two
queries (user permissions, and group permissions via the join tables), builds a set of
`"app_label.codename"` strings, and caches it on the user instance as
`_perm_cache`. That attribute is the cache we tripped over. Later checks are set lookups.
When an object is passed, `ModelBackend` returns an empty set by design, leaving
object-level logic to other backends.

The four default permissions are created by a `post_migrate` signal handler in
`django.contrib.auth`, which compares each model's `Meta.default_permissions` and
`Meta.permissions` with what's in the table and inserts what's missing. That's why
permissions appear after `migrate` rather than in a migration file, and why deleting
`auth_permission` rows by hand is undone the next time you migrate.

The `perms` template variable is a `PermWrapper`. `perms.catalog` returns a
`PermLookupDict` for the app, whose truthiness calls `has_module_perms`, and
`perms.catalog.change_book` calls `has_perm("catalog.change_book")`.

## Common mistakes

- **Checking only in the template.** Every view and API endpoint needs its own check.
- **Fetching with `Model.objects.get(pk=pk)`** in an owner-only view. Scope the QuerySet.
- **Testing as a superuser**, which passes every check including misspelled ones.
- **Granting permissions in code and checking on the same user object** in a test.
- **Using `is_staff` as an authorization flag** for non-admin features. It means "can log
  into the admin", nothing more.
- **Creating groups by hand in production**; script them in a data migration.

## Exercise

1. Add a custom permission `catalog.publish_book`, migrate, and confirm it appears in
   the admin's permission picker.
2. Create "Editors" and "Readers" groups in a data migration with sensible permissions.
3. Protect the book update view with `PermissionRequiredMixin` and show the Edit link
   only to users with the permission. Test with an editor, a reader and an anonymous user.
4. Build "edit my review" so another user's review returns 404. Write a test that proves
   it (Level 2 · 09 shows how).
5. Reproduce the permission-cache surprise in the shell, then explain it.
