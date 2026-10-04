---
description: "Advanced Django migrations on PostgreSQL: RunPython data migrations, nullable-then-required, db_default, concurrent indexes and zero-downtime deploys."
---

# 06 · Migrations in Depth: Data Migrations & Zero-Downtime Changes

Level 1 covered `makemigrations` and `migrate` for a database only you use. Production
adds two complications: the tables already hold data that must be transformed, and the
application is running while the migration runs, possibly with old and new code side by
side during a rolling deploy. This lesson works through the patterns that handle both.
Everything here was run against **PostgreSQL 16**, because that's where these details
matter most and where SQLite would show you different SQL.

## Data migrations with `RunPython`

Schema migrations change tables; data migrations change rows. Create an empty migration
and fill it in:

```bash
python manage.py makemigrations catalog --empty -n fill_book_slugs
```

The task: books need a unique `slug` for URLs. You can't add a unique, non-null column to
a table full of rows in one step, because existing rows have no value. The standard
three-step pattern:

**Step 1 — add the column as nullable** (`slug = models.SlugField(max_length=220,
null=True)`), then `makemigrations`. Cheap and safe on a live table.

**Step 2 — fill it** in a data migration:

```python title="catalog/migrations/0004_fill_book_slugs.py"
from django.db import migrations
from django.utils.text import slugify


def fill_slugs(apps, schema_editor):
    Book = apps.get_model("catalog", "Book")
    seen = set()
    batch = []
    for book in Book.objects.filter(slug__isnull=True).only("id", "title").iterator(chunk_size=500):
        base = slugify(book.title)[:200] or "book"
        slug, n = base, 2
        while slug in seen or Book.objects.filter(slug=slug).exists():
            slug, n = f"{base}-{n}", n + 1
        seen.add(slug)
        book.slug = slug
        batch.append(book)
        if len(batch) == 500:
            Book.objects.bulk_update(batch, ["slug"])
            batch.clear()
    Book.objects.bulk_update(batch, ["slug"])


def clear_slugs(apps, schema_editor):
    apps.get_model("catalog", "Book").objects.update(slug=None)


class Migration(migrations.Migration):
    dependencies = [("catalog", "0003_book_slug_nullable")]
    operations = [migrations.RunPython(fill_slugs, clear_slugs)]
```

**Step 3 — make it required and unique**: change the field to
`models.SlugField(max_length=220, unique=True)` and run `makemigrations`. Django can't
know step 2 exists, so it asks:

```text
It is impossible to change a nullable field 'slug' on book to non-nullable without
providing a default. This is because the database needs something to populate existing rows.
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Ignore for now. Existing rows that contain NULL values will have to be handled manually,
    for example with a RunPython or RunSQL operation.
 3) Quit and manually define a default value in models.py.
```

Option 2 is right here: the data migration handles it. (With `--noinput`, Django picks
the same and prints a warning.) Applying all three:

```text
  Applying catalog.0003_book_slug_nullable... OK
  Applying catalog.0004_fill_book_slugs... OK
  Applying catalog.0005_book_slug_required... OK
```

and the slugs were `['the-dispossessed', 'a-wizard-of-earthsea', 'exhalation',
'stories-of-your-life-and-others', 'kindred']`. Step 3's SQL on PostgreSQL:

```sql
DROP INDEX IF EXISTS "catalog_book_slug_4eebf644_like";
ALTER TABLE "catalog_book" ALTER COLUMN "slug" SET NOT NULL;
ALTER TABLE "catalog_book" ADD CONSTRAINT "catalog_book_slug_4eebf644_uniq" UNIQUE ("slug");
CREATE INDEX "catalog_book_slug_4eebf644_like" ON "catalog_book" ("slug" varchar_pattern_ops);
```

(The `_like` index with `varchar_pattern_ops` is something Django adds on PostgreSQL for
indexed text columns, so `startswith` lookups can use an index.) Because we wrote a
reverse function, `migrate catalog 0002` unapplied all three cleanly.

Rules for `RunPython`:

- **Use `apps.get_model()`**, never import your models. The imported class is today's
  model; the migration must run against the model *as it was at that point in history*.
  Historical models also have no custom methods and only managers marked
  `use_in_migrations = True`.
- **Always write a reverse function**, or pass `migrations.RunPython.noop` if reversing
  is genuinely safe without doing anything. Otherwise the migration is irreversible.
- **Batch large tables** (`iterator()` + `bulk_update()`), and consider running huge
  backfills as a management command outside the deploy (Level 4 · 05).
- **Keep it in its own migration**: on PostgreSQL a migration is one transaction, and
  mixing schema and data changes can hold locks longer than you expect.

## `default` vs `db_default`

We added two fields to `Author` in one migration:

```python
country = models.CharField(max_length=60, default="")
active = models.BooleanField(db_default=True)
```

PostgreSQL got:

```sql
ALTER TABLE "catalog_author" ADD COLUMN "active" boolean DEFAULT true NOT NULL;
ALTER TABLE "catalog_author" ADD COLUMN "country" varchar(60) DEFAULT '' NOT NULL;
ALTER TABLE "catalog_author" ALTER COLUMN "country" DROP DEFAULT;
```

With a Python `default`, Django uses a database default only to fill existing rows and
**then drops it**: afterwards Django supplies the value on every insert. With
`db_default` (Django 5.0+), the default stays in the database. That matters during a
deploy: if old application code (which doesn't know about `country`) inserts an author
after the migration ran, the insert fails with a NOT NULL violation for `country`,
while `active` is filled in by the database. For columns added to busy tables, prefer
`db_default` or make the column nullable first.

## Indexes without locking the table

A normal `CREATE INDEX` on PostgreSQL blocks writes to the table until it finishes, which
on a large table can be minutes of failed requests. PostgreSQL's `CONCURRENTLY` builds
it without blocking writes, and Django exposes it:

```python title="catalog/migrations/0006_book_published_idx.py"
from django.contrib.postgres.operations import AddIndexConcurrently
from django.db import migrations, models


class Migration(migrations.Migration):
    atomic = False

    dependencies = [("catalog", "0005_book_slug_required")]
    operations = [
        AddIndexConcurrently("book", models.Index(fields=["published"], name="book_published_idx")),
    ]
```

`sqlmigrate` showed `CREATE INDEX CONCURRENTLY "book_published_idx" ON "catalog_book"
("published");`. `CONCURRENTLY` can't run inside a transaction, hence `atomic = False`.
Without it, Django refused:

```text
django.db.utils.NotSupportedError: The AddIndexConcurrently operation cannot be executed
inside a transaction (set atomic = False on the migration).
```

One more catch we hit: this hand-written migration added an index the **model** didn't
declare, so `makemigrations --check --dry-run` immediately proposed
`0007_remove_book_book_published_idx`. Add the same `models.Index(...)` to the model's
`Meta.indexes` so state and history agree. Run `makemigrations --check` in CI to catch
drift like this.

## Renames and moves

Rename a field in the model and, when the old and new fields look alike,
`makemigrations` asks interactively whether it was renamed (a `[y/N]` prompt). Answer yes and you get a `RenameField`; answer no
(or use `--noinput`) and you get remove-plus-add, which **deletes the column's data**.
Always read the generated operations before committing.

Renames are also not deploy-safe: old code still running during the deploy queries the
old column name. The zero-downtime way to rename is expand/contract (below), or to
rename only in Python with `db_column="old_name"`.

## Zero-downtime deploys: expand, migrate, contract

During a rolling deploy, old and new code run against the same database for a while. So
every migration must be compatible with **both**. The general pattern:

1. **Expand**: add new things (nullable columns, `db_default` columns, new tables,
   concurrent indexes). Old code ignores them; new code can use them.
2. **Migrate data and deploy code** that writes to both old and new, then reads from new.
3. **Contract** in a *later* deploy: remove old columns once no running code uses them.

Destructive operations (`RemoveField`, `DeleteModel`, renames, `NOT NULL` on existing
columns) belong in the contract step. Removing a field in one deploy breaks old code that
still `SELECT`s it, because Django lists every column explicitly in its queries.
`SeparateDatabaseAndState` helps here: stop using a field in Django's state first, drop
the column in a later migration.

## Squashing

Long-lived apps accumulate hundreds of migrations, which slows down test database
creation. `squashmigrations catalog 0005` combined 0001–0005 into one file. Its output
warned:

```text
Manual porting required
  Your migrations contained functions that must be manually copied over,
  as we could not safely copy their implementation.
```

and the generated file referenced the data migration's function in a form that wasn't
even valid Python (`code=catalog.migrations.0004_fill_book_slugs.fill_slugs`), so it
failed to import until fixed. Copy `RunPython` functions into the squashed file (or drop
one-off backfills that new databases don't need), keep the old migrations until every
environment has applied them, then delete them and remove the `replaces` list.

## How It Actually Works

`apps` in a `RunPython` function is a **historical app registry** rebuilt by replaying
migration operations up to that point, the same "project state" Level 1 described. Its
model classes are generated from that state, not imported from `models.py`, which is why
custom methods and non-migration managers are missing and why imports of real models in
migrations break when the model later changes.

`migrate` applies each migration inside a transaction when the database supports
transactional DDL (PostgreSQL does) and `atomic` is true. With `atomic = False`, each
operation runs in autocommit mode, so a failure midway leaves earlier operations applied;
keep non-atomic migrations to a single operation. A squashed migration has a `replaces`
list: on databases where all the replaced migrations are already recorded, Django marks
the squash as applied without running it; on new databases it runs the squash instead.

## Common mistakes

- **Importing models in migrations** instead of `apps.get_model()`.
- **Irreversible data migrations** with no reverse function.
- **One giant migration** mixing schema changes and a slow backfill.
- **Accepting remove-plus-add** when you meant a rename.
- **Dropping or renaming columns in the same deploy as the code change.**
- **Plain `AddIndex` on a large, busy PostgreSQL table.**
- **Hand-written migrations that drift from the models**; run `makemigrations --check`.

## Exercise

1. Add a unique slug to your books with the three-step pattern, including a reverse
   function, and roll it back and forward.
2. Add a column with `default=` and another with `db_default=` and compare `sqlmigrate`
   output on your production database engine.
3. Write an `AddIndexConcurrently` migration (PostgreSQL), add the matching
   `Meta.indexes` entry, and confirm `makemigrations --check` is clean.
4. Plan, as three separate deploys, renaming `Book.pages` to `Book.page_count` with no
   downtime. Write the migrations for each deploy.
