---
description: "Create, read, update and delete rows with the Django ORM: managers, QuerySets, field lookups, get() vs filter(), laziness and caching, and seeing the SQL."
---

# 06 · Querying with the ORM

The ORM turns Python method calls into SQL. Learning it well means learning two things
at once: the API, and what SQL each call produces. This lesson does both, using the
catalogue models from lesson 05 and a small sample data set (three authors, five books,
three tags). Every output below was produced by running the code in
`python manage.py shell` on that data.

## Creating rows

```python
>>> le = Author.objects.create(name="Ursula K. Le Guin", born=1929)
>>> b = Book.objects.create(title="The Dispossessed", author=le, pages=387,
...                         price=Decimal("12.99"), status=Book.Status.DONE)
>>> b.pk
1
```

`create()` is `Book(...)` followed by `.save()`. Use the two-step form when you need to
set attributes conditionally before saving:

```python
book = Book(title="Kindred", author=butler, pages=264)
if imported_price:
    book.price = imported_price
book.save()
```

For many-to-many fields you must save first (the row needs an ID before join-table rows
can reference it), then use the related manager: `b.tags.add(scifi, classic)`.

## Managers and QuerySets

`Book.objects` is a **manager**, the entry point for table-level queries. Its methods
return a **QuerySet**, a lazy, chainable description of a query.

```python
>>> qs = Book.objects.filter(pages__gt=300)
>>> qs
<QuerySet [<Book: Exhalation>, <Book: The Dispossessed>]>
>>> print(qs.query)
SELECT "catalog_book"."id", "catalog_book"."title", "catalog_book"."author_id",
"catalog_book"."pages", "catalog_book"."price", "catalog_book"."status",
"catalog_book"."published", "catalog_book"."created" FROM "catalog_book"
WHERE "catalog_book"."pages" > 300 ORDER BY "catalog_book"."title" ASC
```

`print(qs.query)` is your best friend while learning. Notice the `ORDER BY title`: it
came from `Meta.ordering` on the model, not from anything we wrote.

The core methods:

| Method | Returns | SQL idea |
|--------|---------|----------|
| `all()` | QuerySet | `SELECT *` |
| `filter(**lookups)` | QuerySet | `WHERE ...` |
| `exclude(**lookups)` | QuerySet | `WHERE NOT (...)` |
| `order_by("-pages")` | QuerySet | `ORDER BY pages DESC` |
| `values("title")` / `values_list("title", flat=True)` | QuerySet of dicts / tuples / values | select specific columns |
| `get(**lookups)` | one object | exactly one row or an exception |
| `first()` / `last()` | object or `None` | `LIMIT 1` |
| `count()` | int | `SELECT COUNT(*)` |
| `exists()` | bool | `SELECT 1 ... LIMIT 1` |

## Field lookups

The double underscore separates a field name from a **lookup**:

```python
>>> Book.objects.filter(title__icontains="earth").values_list("title", flat=True)
<QuerySet ['A Wizard of Earthsea']>
>>> Book.objects.exclude(status="done").order_by("-pages").values("title", "pages")
<QuerySet [{'title': 'Exhalation', 'pages': 350}, {'title': 'Kindred', 'pages': 264},
           {'title': 'A Wizard of Earthsea', 'pages': 183}]>
>>> Book.objects.filter(published__year__lt=1980).values_list("title", flat=True)
<QuerySet ['A Wizard of Earthsea', 'Kindred', 'The Dispossessed']>
```

Common lookups: `exact` (the default), `iexact`, `contains`/`icontains`,
`startswith`, `in`, `gt`/`gte`/`lt`/`lte`, `range`, `isnull`, and date parts like
`year`, `month`, `date`.

Lookups can **follow relationships** with the same double underscore. This joins to the
author table:

```python
>>> Book.objects.filter(author__name__startswith="Ted").count()
2
```

Several keyword arguments in one `filter()` are combined with AND. For OR, use `Q`
objects (Level 2 · 02).

## `get()` and its two exceptions

`get()` insists on exactly one row:

```python
>>> Book.objects.get(pk=999)
catalog.models.Book.DoesNotExist: Book matching query does not exist.
>>> Book.objects.get(author__name="Ted Chiang")
catalog.models.Book.MultipleObjectsReturned: get() returned more than one Book -- it returned 2!
```

Use `get()` when "not exactly one" is a genuine error (lookup by primary key or a unique
field). In views, wrap it with `get_object_or_404`. When zero results are normal, use
`filter(...).first()`, which returns `None`.

## Updating and deleting

Two ways to update, with very different behaviour:

```python
# 1. Load, change, save: runs model save() logic, one UPDATE per object
book = Book.objects.get(pk=3)
book.status = Book.Status.DONE
book.save(update_fields=["status"])

# 2. Bulk update: one UPDATE statement, no save() called, no signals
Book.objects.filter(status="reading").update(status="done")
```

`save()` without `update_fields` writes **every** column, which can overwrite a change
another request made to a different field in the meantime. Passing `update_fields` is a
cheap habit that avoids that.

Deleting follows the same pattern: `book.delete()` or `qs.delete()`. Deletion respects
`on_delete` on related foreign keys. Our `Book.author` uses `PROTECT`, so deleting an
author who still has books raises `ProtectedError` instead of silently removing their
books. `CASCADE` (used on `Review.book`) deletes dependent rows.

Two helpers for "find or make":

```python
tag, created = Tag.objects.get_or_create(name="fantasy")
author, created = Author.objects.update_or_create(name="Ted Chiang", defaults={"born": 1967})
```

## QuerySets are lazy, and they cache

Building a QuerySet doesn't touch the database. We counted queries with Django's
`CaptureQueriesContext`:

```python
qs2 = Book.objects.all().filter(pages__gt=100).order_by("pages")   # 0 queries
list(qs2); list(qs2)                                                 # 1 query
```

Chaining `filter()` and `order_by()` ran **zero** queries. Evaluating it twice ran
**one**: the first `list()` filled the QuerySet's result cache, and the second reused it.
But indexing an *unevaluated* QuerySet doesn't fill the cache:

```python
qs = Book.objects.all(); qs[0]; qs[0]          # 2 queries (each is a LIMIT 1)
qs = Book.objects.all(); list(qs); qs[0]       # 1 query (the index reads the cache)
```

A QuerySet is evaluated when you iterate it, call `list()`, `len()` or `bool()` on it,
slice it with a step, or print its repr. `count()` and `exists()` always run their own
small query, and our `exists()` check produced:

```sql
SELECT 1 AS "a" FROM "catalog_book" WHERE "catalog_book"."status" = 'done' LIMIT 1
```

So prefer `if qs.exists():` over `if qs:` when you won't use the rows, and prefer
`if qs:` when you will (it fills the cache you're about to iterate).

## How It Actually Works

A QuerySet wraps a `Query` object: a structured description containing the model, a
tree of `WHERE` conditions, ordering, joins and limits. Each chained method **clones**
the QuerySet and modifies the clone's `Query`, which is why chaining never alters the
original and why it's cheap until evaluation.

When evaluated, the `Query` is handed to a **SQL compiler** for the current database
backend. The compiler resolves lookups like `author__name__startswith` into a join plus a
lookup class (`StartsWith`), which emits backend-specific SQL (a `LIKE 'Ted%'` pattern,
with special characters in your value escaped). Values are always sent as **query parameters**, never pasted
into the SQL string, which is why ORM filters are safe from SQL injection even with
user input. (`str(qs.query)` shows values inlined for readability, but that string is not
what's actually executed.) Rows come back as tuples and are turned into model instances
by the model iterable, and the list is stored in `qs._result_cache`.

## Common mistakes

- **Expecting `filter()` to raise when nothing matches.** It returns an empty QuerySet.
  Only `get()` raises.
- **`len(qs)` to count** when you don't need the rows; it loads them all. Use `count()`.
- **Looping and saving** when a single `update()` would do.
- **Forgetting `Meta.ordering` is applied everywhere**, including inside aggregation
  queries and on large tables where it forces a sort.
- **Calling `.all()` repeatedly** in a loop expecting caching; every `.all()` is a new
  QuerySet with an empty cache.

## Exercise

Load a few authors and books of your own (in the shell, or via the admin after lesson
07), then write ORM queries for:

1. All books with fewer than 300 pages, longest first, as a list of titles.
2. All books by authors born before 1950.
3. The number of books whose status is "want".
4. Whether any book has a price of zero (use `exists()`).
5. Mark every "reading" book as "done" in a single statement, and print the number of
   rows updated.

For each, print `qs.query` (where possible) and explain the SQL.
