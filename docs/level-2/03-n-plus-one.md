---
description: "Find and fix N+1 queries in Django: how they arise, select_related vs prefetch_related, Prefetch objects, to_attr, and query budgets in tests."
---

# 03 · Finding and Fixing N+1 Queries

The single most common performance problem in Django applications has a name: **N+1
queries**. You run one query to get a list of N things, then one more query *per thing*
to fetch something related. It hides well: on a development database with five rows the
page feels instant, and in production with five hundred rows it makes five hundred and
one round trips to the database. This lesson shows how to see it, the two tools that fix
it, and how to stop it coming back.

## Seeing it

Here's a loop that looks innocent:

```python
for b in Book.objects.all():
    print(b.title, b.author.name)
```

We ran it inside Django's `assertNumQueries(1)` on a test database with 12 books by 3
authors. It failed, and the failure lists every query:

```text
AssertionError: 13 != 1 : 13 queries executed, 1 expected
Captured queries were:
1. SELECT "catalog_book"."id", "catalog_book"."title", ... FROM "catalog_book" ORDER BY "catalog_book"."title" ASC
2. SELECT "catalog_author"."id", "catalog_author"."name", "catalog_author"."born" FROM "catalog_author" WHERE "catalog_author"."id" = 1 LIMIT 21
3. SELECT ... FROM "catalog_author" WHERE "catalog_author"."id" = 1 LIMIT 21
4. SELECT ... FROM "catalog_author" WHERE "catalog_author"."id" = 1 LIMIT 21
...
13. SELECT ... FROM "catalog_author" WHERE "catalog_author"."id" = 3 LIMIT 21
```

One query for the books, then one per book for its author. Notice queries 2, 3 and 4 are
**identical**: the same author fetched again for each of their books. Django's ORM has
no identity map; each `Book` instance caches its own `author`, but two books don't share
it. (The `LIMIT 21` is how `get()` detects "more than one row" while still reporting a
useful count in the error message.)

The same pattern appears in templates (`{{ book.author.name }}` in a `{% for %}`), in
`__str__` methods that touch relations, in admin `list_display` methods, and in API
serializers. Anywhere you loop and touch a relation.

## Fix 1 — `select_related` for "to-one" relations

For `ForeignKey` and `OneToOneField` (one related object per row), join it into the
same query:

```python
for b in Book.objects.select_related("author"):
    print(b.title, b.author.name)
```

On our catalogue that took **1** query instead of 6:

```sql
SELECT "catalog_book"."id", ..., "catalog_author"."id", "catalog_author"."name", "catalog_author"."born"
FROM "catalog_book" INNER JOIN "catalog_author" ON ("catalog_book"."author_id" = "catalog_author"."id")
ORDER BY "catalog_book"."title" ASC
```

You can follow chains (`select_related("author__publisher")`) and list several. Django
uses an `INNER JOIN` when the FK is non-nullable and a `LEFT OUTER JOIN` when it's
nullable, so rows aren't lost.

## Fix 2 — `prefetch_related` for "to-many" relations

A join can't sensibly fetch many tags per book (it would multiply rows), so
`prefetch_related` runs **one extra query per relation** and stitches the results
together in Python:

```python
for b in Book.objects.prefetch_related("tags"):
    print(b.title, [t.name for t in b.tags.all()])
```

Naively that loop took 6 queries on our data; prefetched it took **2**:

```sql
SELECT ... FROM "catalog_book" ORDER BY "catalog_book"."title" ASC
SELECT ("catalog_book_tags"."book_id") AS "_prefetch_related_val_book_id", "catalog_tag"."id", "catalog_tag"."name"
FROM "catalog_tag" INNER JOIN "catalog_book_tags" ON ("catalog_tag"."id" = "catalog_book_tags"."tag_id")
WHERE "catalog_book_tags"."book_id" IN (2, 3, 5, 4, 1)
```

Two queries regardless of how many books, because the second uses `IN (...)` with every
book ID.

### The prefetch cache only serves `.all()`

After prefetching, `b.tags.all()` reads from the cache. But any *new* query on the
related manager bypasses it:

```python
for b in Book.objects.prefetch_related("tags"):
    list(b.tags.filter(name="sci-fi"))     # 7 queries: the prefetch was wasted
```

To prefetch a filtered or annotated set, describe it with a `Prefetch` object.

## `Prefetch` objects and `to_attr`

A page listing authors, their books, each book's tags and review count. Written
naively, it took **16** queries for 3 authors and 6 books:

```python
for a in Author.objects.all():
    for b in a.books.all():
        [t.name for t in b.tags.all()]
        b.reviews.count()
```

With a `Prefetch` that carries its own QuerySet (including an annotation and a nested
prefetch) it took **3**:

```python
from django.db.models import Count, Prefetch

authors = Author.objects.prefetch_related(
    Prefetch(
        "books",
        queryset=Book.objects.annotate(n_reviews=Count("reviews")).prefetch_related("tags"),
    )
)
for a in authors:
    for b in a.books.all():
        [t.name for t in b.tags.all()]
        b.n_reviews
```

Three queries: authors; their books with review counts; those books' tags.

`to_attr` stores a filtered prefetch under its own name as a plain list, so it can't be
confused with the full relation:

```python
books = Book.objects.prefetch_related(
    Prefetch("reviews",
             queryset=Review.objects.filter(rating__gte=4).select_related("user"),
             to_attr="good_reviews")
)
[(b.title, [r.user.username for r in b.good_reviews]) for b in books]
# 2 queries:
# [('A Wizard of Earthsea', []), ('Exhalation', ['ana']), ..., ('The Dispossessed', ['ana', 'raj']), ...]
```

### An error you'll meet

We first wrote the author query as
`prefetch_related("books__tags").prefetch_related(Prefetch("books", queryset=...))`:
two different definitions of `books`. Django refused:

```text
ValueError: 'books' lookup was already seen with a different queryset. You may need to
adjust the ordering of your lookups.
```

Define each relation once, with nested prefetches inside the `Prefetch` QuerySet.

## Other ways to accidentally multiply queries

- **`only()` / `defer()` then touching a deferred field.** We loaded
  `Book.objects.only("title")` and read `b.pages` in the loop: **7** queries for 6 books.
  Each deferred access is a query.
- **`.iterator()` with prefetching.** Since Django 4.1, prefetching works with
  `iterator(chunk_size=...)`, one prefetch query per chunk. With 6 books and
  `chunk_size=2` we measured 4 queries (1 for books + 3 chunked tag queries). Bigger
  chunks mean fewer queries but more memory.
- **`count()` in a loop**; annotate it instead, as above.

## Keeping it fixed: query budgets in tests

Fixes regress when someone adds `{{ book.publisher.name }}` to a template next month.
Lock the count in a test:

```python
from django.test import TestCase
from django.urls import reverse

class BookListQueryTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        for i in range(3):
            a = Author.objects.create(name=f"Author {i}")
            for j in range(4):
                Book.objects.create(title=f"Book {i}-{j}", author=a, pages=100 + j)

    def test_book_list_query_count_is_constant(self):
        with self.assertNumQueries(1):
            self.client.get(reverse("catalog:book-list"))
```

That test passed for our `select_related("author")` list view. The important detail is
`setUpTestData` creating **more than one** row per relation: a query budget measured with
one book can't detect N+1.

During development, the third-party **django-debug-toolbar** shows every query for a page,
highlights duplicates, and shows where in your code each was triggered. It's worth
installing in every project's development settings.

## How It Actually Works

`select_related` changes the SQL: the compiler adds joins and extra columns, and the row
iterator splits each row into the main object and its related objects, calling
`setattr` on the main object's **field cache** so the descriptor finds it without a
query.

`prefetch_related` doesn't change the main query at all. After the main QuerySet is
evaluated, `prefetch_related_objects()` collects the primary keys of every fetched
object, asks the relation's descriptor for a "prefetch queryset" (essentially
`related_model.objects.filter(fk__in=pks)`), runs it, groups the results by the foreign
key value, and stores each group in the instance's `_prefetched_objects_cache` under the
relation name. When you call `b.tags.all()`, the related manager checks that cache
first. Calling `.filter()` creates a new QuerySet that has no access to that cache, so it
queries. `to_attr` simply stores the grouped list on a plain attribute instead.

Nested lookups like `"books__tags"` repeat the process level by level: prefetch books for
all authors, then tags for all those books, which is why the query count depends on the
depth of the lookup, not on the number of rows.

## Common mistakes

- **Using `select_related` on a many-to-many or reverse FK.** It raises
  `FieldError: Invalid field name(s) given in select_related`. Use `prefetch_related`.
- **Prefetching, then filtering the related manager**, which silently re-queries.
- **Testing query counts with one row per relation.**
- **Fixing it in the view, re-breaking it in the template** with a new relation access.
- **Over-joining:** `select_related()` with no arguments follows every non-null FK; on
  wide models that's a lot of columns. Name what you need.

## Exercise

1. Write a view that lists authors, each with their books and each book's tags. Measure
   queries with `CaptureQueriesContext` (from `django.test.utils`), then get it to 3.
2. Add the average rating per book using an annotation inside a `Prefetch` QuerySet.
3. Add a `to_attr="recent_reviews"` prefetch for each book's three newest reviews. (Hint:
   slicing in a `Prefetch` QuerySet is supported since Django 4.2.)
4. Write an `assertNumQueries` test for the page with at least three authors and three
   books each, then add `{{ book.author.born }}` somewhere it isn't joined and watch the
   test fail.
