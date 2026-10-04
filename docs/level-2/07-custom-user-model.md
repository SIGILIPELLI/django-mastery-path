---
description: "Why every Django project should start with a custom user model: AbstractUser with a unique email, referencing users correctly, and switching late."
---

# 07 · The Custom User Model

Django's documentation gives one piece of advice more emphatically than almost any
other: if you're starting a new project, set up a custom user model, even if the default
one is sufficient for now. This lesson explains why, sets one up properly, and shows
exactly what goes wrong when a project tries to switch after its first migration. We ran
both scenarios on Django 6.1.1.

## Why bother if the default works?

The default `django.contrib.auth.models.User` has a fixed set of fields: `username`,
`first_name`, `last_name`, `email`, `password`, `is_staff`, `is_active`,
`is_superuser`, `last_login`, `date_joined`, plus groups and permissions. Sooner or later
most products want to change something about it: make `email` unique, log in by email,
add a display name or a time zone, drop `first_name`/`last_name` in favour of one name
field.

You can always add a separate `Profile` model with a one-to-one link (lesson 01). But
some changes, such as uniqueness of email, the login field and the table itself, can only
be made on the user model. And `AUTH_USER_MODEL`, the setting that says which model is
the user, is referenced by the very first migrations of `admin`, `auth` and every app
with a foreign key to users. It's effectively fixed once the database exists. Starting
with your own subclass costs ten lines and keeps every option open.

## Setting it up (before the first `migrate`)

Create an app for it:

```bash
python manage.py startapp accounts
```

```python title="accounts/models.py"
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    email = models.EmailField("email address", unique=True)
    display_name = models.CharField(max_length=80, blank=True)

    def __str__(self):
        return self.display_name or self.username
```

```python title="config/settings.py"
INSTALLED_APPS = [
    # ...
    "accounts",
]
AUTH_USER_MODEL = "accounts.User"
```

Register it with the admin, reusing Django's `UserAdmin` (which handles password
hashing and the two-step add form):

```python title="accounts/admin.py"
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import User


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    fieldsets = UserAdmin.fieldsets + (("Profile", {"fields": ["display_name"]}),)
    add_fieldsets = UserAdmin.add_fieldsets + ((None, {"fields": ["email", "display_name"]}),)
    list_display = ["username", "email", "display_name", "is_staff"]
```

Then the first migrations:

```bash
python manage.py makemigrations accounts
python manage.py migrate
```

On our run, `accounts.0001_initial` was applied **before** `admin.0001_initial`, and
`sqlmigrate admin 0001` showed the admin log table pointing at the new model:
`REFERENCES "accounts_user"`.

The model behaves as you'd hope:

```python
>>> U = get_user_model(); U, U._meta.db_table
(<class 'accounts.models.User'>, 'accounts_user')
>>> U.objects.create_user("ana", email="ana@example.com", password="...")
<User: ana>
>>> U.objects.create_user("ana2", email="ana@example.com", password="...")
IntegrityError: UNIQUE constraint failed: accounts_user.email
```

The database now enforces unique emails. (Forms should catch it first with a friendly
message; the constraint is the guarantee.) Be aware that `unique=True` is
case-sensitive on most databases, so `Ana@example.com` and `ana@example.com` are
different values. Normalise emails to lower case on save, or add a unique constraint on
`Lower("email")`.

### `AbstractUser` vs `AbstractBaseUser`

- **`AbstractUser`**: the full default user, to which you add fields. Right for almost
  everyone.
- **`AbstractBaseUser`** (+ `PermissionsMixin`): only password and last-login handling.
  You define every other field, set `USERNAME_FIELD`, `REQUIRED_FIELDS`, and write a
  custom manager with `create_user` and `create_superuser`. Choose it only when the
  default fields genuinely don't fit (e.g. no username at all, login by phone number).

To log in by email with `AbstractUser`, you can set `USERNAME_FIELD = "email"` and
`REQUIRED_FIELDS = ["username"]` (or remove `username`, which requires a custom manager).
Do it before the first migration.

## Referencing the user model

Never import `User` from `django.contrib.auth.models` in a project with a custom user.
When we tried to use the default class after the swap:

```text
AttributeError: Manager isn't available; 'auth.User' has been swapped for 'accounts.User'
```

The right references:

| Where | Use |
|-------|-----|
| Model fields | `models.ForeignKey(settings.AUTH_USER_MODEL, ...)` (a string, safe at import time) |
| Everywhere else (views, forms, tests) | `from django.contrib.auth import get_user_model; User = get_user_model()` |
| Built-in forms | `UserCreationForm` and `UserChangeForm` work with custom models that keep `AbstractUser`'s fields; subclass them and set `Meta.model` if you added required fields |

Writing reusable apps this way also means they work in other people's projects, whatever
user model those use.

## What happens if you switch late

We simulated the common situation: a project that has already run `migrate` with the
default user, then adds `accounts.User` and sets `AUTH_USER_MODEL`.

First, if you define a model that subclasses `AbstractUser` but *forget* the setting,
the system check stops everything:

```text
accounts.User.groups: (fields.E304) Reverse accessor 'Group.user_set' for
'accounts.User.groups' clashes with reverse accessor for 'auth.User.groups'.
```

(Both user models are active and both want `group.user_set`.) With the setting in place,
`makemigrations accounts` succeeded, but `migrate` on the existing database failed:

```text
django.db.migrations.exceptions.InconsistentMigrationHistory: Migration admin.0001_initial
is applied before its dependency accounts.0001_initial on database 'default'.
```

The admin's first migration depends on whatever `AUTH_USER_MODEL` is. It was applied
long ago against `auth.User`, so the history is now contradictory. There is no
`makemigrations` fix for this.

Your options on a live project:

1. **Development only:** delete the database and migrations, start again.
2. **Production:** a careful manual migration. The common recipe is to create the new
   model *reusing the existing table* (`db_table = "auth_user"`), fake its initial
   migration, update the `django_content_type` row and the recorded migration history,
   then make further changes normally. It's fiddly, must be rehearsed on a copy of
   production, and is easy to get wrong. Many teams instead keep the default user and
   put new fields on a `Profile` model.

That cost, against ten lines on day one, is the whole argument.

## How It Actually Works

`AUTH_USER_MODEL` makes the user model **swappable**. `auth.User` declares
`swappable = "AUTH_USER_MODEL"` in its `Meta`. When the setting names another model, the
app registry marks `auth.User` as swapped out: it creates no table and its manager is
replaced by a descriptor that raises the `AttributeError` we saw.

Migrations reference the user through `migrations.swappable_dependency(settings.AUTH_USER_MODEL)`,
which resolves at migrate time to "the first migration of whichever app holds the user
model". That's how `admin.0001_initial` came to depend on `accounts.0001_initial` in the
new project and on `auth.0001_initial` in the old one, and why the recorded history
can't be reconciled automatically. Foreign keys declared with
`settings.AUTH_USER_MODEL` are stored in migrations as the setting reference, not a
fixed model, so the same migration files work in projects with different user models.

`get_user_model()` simply looks up `settings.AUTH_USER_MODEL` in the app registry, which
is why it must not be called at module import time in `models.py` (the registry may not
be ready); the string form exists for that case.

## Common mistakes

- **Running `migrate` before creating the custom user**, then discovering it's too late.
- **`from django.contrib.auth.models import User`** in app code.
- **Forgetting `AUTH_USER_MODEL`** after writing the model (the E304 clash).
- **Case-sensitive unique emails** letting `Ana@` and `ana@` register twice.
- **Choosing `AbstractBaseUser`** when `AbstractUser` plus a field would do, and then
  re-implementing the manager, admin and forms.
- **Putting frequently changing profile data on the user** (preferences, avatars); it's
  fine on a related `Profile` model, which keeps the user table small.

## Exercise

1. Start a new project with an `accounts.User(AbstractUser)` model before the first
   `migrate`. Add a `display_name` and a unique email.
2. Register it with a `UserAdmin` subclass and create a superuser.
3. Write a sign-up form subclassing `UserCreationForm` with `Meta.model = User` and
   `fields = ["username", "email", "display_name"]`; lower-case the email in
   `clean_email()`.
4. In a scratch copy, reproduce the `InconsistentMigrationHistory` error by switching
   user models after migrating. Read the error, then delete the scratch copy.
