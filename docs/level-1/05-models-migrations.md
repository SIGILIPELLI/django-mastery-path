---
description: "Define Django models with the right field types and options, generate and read migrations, inspect the SQL they run, and roll a change back."
---

# 05 · Models and Migrations

A model is a Python class that describes one database table. Django uses it in three
directions: to generate the table (migrations), to read and write rows (the ORM), and
to build forms and admin screens. Getting the model right is the highest-leverage
decision in a Django project, because everything else is generated from it.

## The catalogue's models

```python title="catalog/models.py"
from django.conf import settings
from django.db import models
from django.urls import reverse


class Author(models.Model):
    name = models.CharField(max_length=200)
    born = models.IntegerField(null=True, blank=True)

    class Meta:
        ordering = ["name"]

    def __str__(self):
        return self.name


class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):
        return self.name


class Book(models.Model):
    class Status(models.TextChoices):
        WANT = "want", "Want to read"
        READING = "reading", "Reading"
        DONE = "done", "Finished"

    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.PROTECT, related_name="books")
    tags = models.ManyToManyField(Tag, blank=True, related_name="books")
    pages = models.PositiveIntegerField()
    price = models.DecimalField(max_digits=6, decimal_places=2, default=0)
    status = models.CharField(max_length=10, choices=Status, default=Status.WANT)
    published = models.DateField(null=True, blank=True)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["title"]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("catalog:book-detail", args=[self.pk])
```

Relationships (`ForeignKey`, `ManyToManyField`) get a full lesson in Level 2 · 01. For
now: each book has one author; a book can have many tags and a tag many books.

## Choosing field types

| Data | Field | Notes |
|------|-------|-------|
| Short text | `CharField(max_length=...)` | `max_length` is required |
| Long text | `TextField()` | no length limit in the database |
| Whole numbers | `IntegerField`, `PositiveIntegerField`, `BigIntegerField` | positive fields add a `CHECK (>= 0)` |
| Money | `DecimalField(max_digits, decimal_places)` | never `FloatField` for money |
| Dates/times | `DateField`, `DateTimeField` | `auto_now_add` sets on create, `auto_now` on every save |
| True/false | `BooleanField(default=False)` | |
| Fixed set of values | `CharField(choices=...)` with `TextChoices` | `get_status_display()` returns the label |
| Email / URL | `EmailField`, `URLField` | `CharField` plus form validation |
| Structured data | `JSONField` | supported on all built-in backends |
| Files | `FileField`, `ImageField` | lesson 09 |

`TextChoices` gives you an enum: `Book.Status.DONE` is the string `"done"`, and
`Book.Status.DONE.label` is `"Finished"`. Refer to the enum in code, never the raw
string, so a typo fails at import time rather than silently matching nothing.

## `null` vs `blank`

These two options are the most commonly confused pair in Django:

- **`null=True`** is about the **database**: the column allows `NULL`.
- **`blank=True`** is about **validation**: forms and the admin allow the field to be
  empty.

For `born` (an optional number) we want both: no value is stored as `NULL`, and forms
don't require it. For optional **text** fields, Django's convention is `blank=True` with
no `null`, storing an empty string. Having both `NULL` and `""` mean "no value" leads to
queries that miss half the empty rows.

## Migrations: from class to table

Models change; databases have data in them. Migrations are versioned Python files that
describe each change, so every developer's database and production can be moved forward
identically.

```bash
python manage.py makemigrations catalog
```

```text
Migrations for 'catalog':
  catalog/migrations/0001_initial.py
    + Create model Author
    + Create model Tag
    + Create model Book
    + Create model Review
```

(`Review` is a fourth model we use from Level 2.) `makemigrations` doesn't touch the
database. It compares your models to the state recorded by existing migration files and
writes a new file with the difference. To see the SQL a migration will run, use
`sqlmigrate`. On SQLite, part of ours read:

```sql
CREATE TABLE "catalog_book" (
  "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
  "title" varchar(200) NOT NULL,
  "pages" integer unsigned NOT NULL CHECK ("pages" >= 0),
  "price" decimal NOT NULL,
  "status" varchar(10) NOT NULL,
  "published" date NULL,
  "created" datetime NOT NULL,
  "author_id" bigint NOT NULL REFERENCES "catalog_author" ("id") DEFERRABLE INITIALLY DEFERRED);
CREATE TABLE "catalog_book_tags" (... "book_id" bigint ..., "tag_id" bigint ...);
CREATE INDEX "catalog_book_author_id_b0849980" ON "catalog_book" ("author_id");
```

Things to notice:

- Table names are `<app>_<model>` in lower case.
- Django added an `id` primary key automatically. Since Django 6.0 the default
  auto-field type is `BigAutoField` (a 64-bit integer); on SQLite every `integer primary
  key` is already 64-bit, which is why it prints as `integer`.
- `ForeignKey` became an `author_id` column **plus an index**.
- The many-to-many field became a separate join table, `catalog_book_tags`.
- `PositiveIntegerField` added a `CHECK` constraint.
- `choices` did **not** become a database constraint. It's enforced by forms and model
  validation only. (Level 3 · 05 shows how to add a real `CheckConstraint`.)

Apply it:

```bash
python manage.py migrate
```

Django records each applied migration in a table called `django_migrations`;
`showmigrations` reads it:

```text
catalog
 [X] 0001_initial
```

## Changing a model

Suppose authors need a country. We first added
`country = models.CharField(max_length=60)` and ran `makemigrations --noinput`:

```text
Field 'country' on model 'author' not migrated: it is impossible to add a non-nullable
field without specifying a default.
```

Existing rows need *some* value. (Without `--noinput`, Django prompts you to type a
one-off default instead.) For an optional text field the right fix is in the model:

```python
country = models.CharField(max_length=60, blank=True, default="")
```

```text
Migrations for 'catalog':
  catalog/migrations/0002_author_country.py
    + Add field country to author
```

The generated file is readable Python:

```python title="catalog/migrations/0002_author_country.py"
class Migration(migrations.Migration):
    dependencies = [
        ("catalog", "0001_initial"),
    ]
    operations = [
        migrations.AddField(
            model_name="author",
            name="country",
            field=models.CharField(blank=True, default="", max_length=60),
        ),
    ]
```

and `sqlmigrate catalog 0002` revealed something important about SQLite:

```sql
CREATE TABLE "new__catalog_author" (... "country" varchar(60) NOT NULL, ...);
INSERT INTO "new__catalog_author" ("id", "name", "born", "country")
  SELECT "id", "name", "born", '' FROM "catalog_author";
DROP TABLE "catalog_author";
ALTER TABLE "new__catalog_author" RENAME TO "catalog_author";
```

SQLite's `ALTER TABLE` is limited, so Django rebuilds the table and copies every row. On
PostgreSQL the same migration is a single `ALTER TABLE ... ADD COLUMN`. Always read
`sqlmigrate` for the database you deploy to, not just the one on your laptop.

To roll back, migrate to the previous migration by name:

```bash
python manage.py migrate catalog 0001
```

```text
  Rendering model states... DONE
  Unapplying catalog.0002_author_country... OK
```

## How It Actually Works

Django never inspects your database to decide what changed. Instead, `makemigrations`
**replays every existing migration file in memory** to build a "project state" (a
lightweight description of every model), builds a second state from your current
`models.py`, and diffs them with the **autodetector**. Each difference becomes an
operation (`AddField`, `AlterField`, `CreateModel`...). This is why editing the database
by hand is dangerous: the migration history no longer describes reality, and Django
won't notice until a later migration fails.

`migrate` builds a dependency graph from every migration's `dependencies` list, works
out which nodes aren't yet in `django_migrations`, and applies them in order. Each
operation has a `database_forwards` method that asks the database backend's **schema
editor** to emit SQL. The schema editor is per-backend, which is why the same
`AddField` produced a table rebuild on SQLite and would produce an `ADD COLUMN` on
PostgreSQL. Most operations also know how to reverse themselves, which is what makes
`migrate catalog 0001` possible. On databases with transactional DDL (PostgreSQL,
SQLite), each migration runs inside a transaction, so a failure leaves the schema as it
was; MySQL can't roll back DDL, so a failed migration there can leave a half-applied
state.

## Common mistakes

- **Editing a migration that's already been applied elsewhere.** Others won't re-run it.
  Create a new migration instead.
- **Deleting migration files to "start fresh"** on a project that has a production
  database. The history is how production gets upgraded.
- **Forgetting to commit migration files.** They're source code.
- **`null=True` on `CharField`/`TextField`** without a reason, creating two kinds of empty.
- **Using `FloatField` for money.** `0.1 + 0.2` isn't `0.3` in binary floating point.
- **Assuming `choices` is enforced by the database.** It isn't; bulk updates and raw SQL
  can store anything.

## Exercise

1. Write the `Author`, `Tag` and `Book` models above, then `makemigrations` and
   `migrate`.
2. Run `sqlmigrate catalog 0001` and find the index Django created for the foreign key.
3. Add a non-nullable field with no default and read the error, then fix it two ways:
   with a model default, and by letting `makemigrations` prompt you for a one-off default.
   Compare the two migration files.
4. Roll the change back with `migrate catalog 0001`, delete the unapplied migration
   file, and confirm `showmigrations` is clean.
