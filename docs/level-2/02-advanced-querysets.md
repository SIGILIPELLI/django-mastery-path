---
description: "Push work into the database with Q objects, F expressions, annotate, aggregate, conditional counts, Subquery, Exists, Case/When and window functions."
---

# 02 · Advanced QuerySets: Q, F, annotate & aggregate

Level 1's ORM covered "fetch rows matching conditions". Real pages need more: counts per
group, averages, "the latest review for each book", "books longer than their author's
average". The naive way is to fetch rows and compute in Python, which gets slower with
every row you add. The ORM can express nearly all of it as SQL, so the database does
the work and returns only the answer.

All outputs below were produced on our catalogue sample data (six books by three
authors, a handful of reviews).

## `Q` objects: OR, NOT and dynamic filters

Keyword arguments to `filter()` are ANDed together. `Q` objects let you combine
conditions with `|` (OR), `&` (AND) and `~` (NOT):

```python
>>> from django.db.models import Q
>>> Book.objects.filter(Q(pages__lt=200) | Q(status="reading")).values_list("title", flat=True)
<QuerySet ['A Wizard of Earthsea', 'Exhalation', 'The Lathe of Heaven']>
>>> Book.objects.filter(~Q(status="done") & Q(pages__gt=200)).values_list("title", flat=True)
<QuerySet ['Exhalation', 'Kindred']>
```

`Q` objects are also how you build filters from user input, such as a search box where
every word must match:

```python
filters = Q()
for word in ["the", "of"]:
    filters &= Q(title__icontains=word)
Book.objects.filter(filters)
# <QuerySet ['Stories of Your Life and Others', 'The Lathe of Heaven']>
```

An empty `Q()` matches everything, so the loop works with zero words too.

## `F` expressions: refer to columns, not values

`F("pages")` means "the value of the `pages` column in this row". It lets the database
compare and compute with columns:

```python
>>> from django.db.models import F
>>> Book.objects.filter(pages__gt=F("author__born") / 5).values_list("title", flat=True)
<QuerySet ['The Dispossessed']>
```

(A silly comparison, but it shows an `F` that crosses a join.) The important everyday use
is **atomic updates**:

```python
Book.objects.filter(author__name="Ted Chiang").update(price=F("price") + 1)
```

This ran a single statement:

```sql
UPDATE "catalog_book" SET "price" = (CAST(("catalog_book"."price" + 1) AS NUMERIC))
WHERE "catalog_book"."id" IN (SELECT "U0"."id" FROM "catalog_book" "U0"
  INNER JOIN "catalog_author" "U1" ON ("U0"."author_id" = "U1"."id")
  WHERE "U1"."name" = 'Ted Chiang')
```

Compare `book.views = book.views + 1; book.save()`: two simultaneous requests both read
10 and both write 11, losing an increment. `F("views") + 1` makes the database do the
read-modify-write in one step. (Level 3 · 05 covers race conditions properly.)

### A real arithmetic pitfall

We tried price per page with `F("price") / F("pages")`:

```python
>>> Book.objects.annotate(per_page=F("price") / F("pages")).values_list("title", "per_page")[:2]
<QuerySet [('A Wizard of Earthsea', Decimal('0.0519125683060109')), ('Exhalation', Decimal('0'))]>
```

Exhalation's price was 15.00, yet the answer was exactly zero. SQLite had stored `15.00`
as the integer `15` (we checked with `typeof(price)`: `'integer'`, while `9.5` was
`'real'`), and integer divided by integer is integer division: `15 / 350 = 0`. Casting
fixes it:

```python
>>> from django.db.models import FloatField
>>> from django.db.models.functions import Cast
>>> Book.objects.annotate(pp=Cast("price", FloatField()) / F("pages")).values_list("title", "pp")[:2]
<QuerySet [('A Wizard of Earthsea', 0.05191256830601093), ('Exhalation', 0.045714285714285714)]>
```

(Exhalation's price was 16 by then, after the `F() + 1` update above.) The lesson
generalises: database arithmetic follows the database's type rules, which differ between
backends. We ran the same uncast annotation on PostgreSQL 16 and got
`Decimal('0.04285714285714285714')` for Exhalation: PostgreSQL keeps `numeric` values as
decimals. Test calculations on the database you deploy to.

## `aggregate()`: one answer for the whole QuerySet

```python
>>> from django.db.models import Avg, Count, Min, Sum
>>> Book.objects.aggregate(total=Sum("pages"), avg_price=Avg("price"), oldest=Min("published"))
{'total': 1465, 'avg_price': Decimal('12.5480000000000'), 'oldest': datetime.date(1968, 1, 1)}
```

(That run was on the original five books.) `aggregate()` ends the chain and returns a
dict.

## `annotate()`: one answer per row

`annotate()` adds a computed value to each object. With an aggregate function, it groups:

```python
>>> Author.objects.annotate(n=Count("books"), avg_pages=Avg("books__pages")).values("name", "n", "avg_pages")
<QuerySet [{'name': 'Ursula K. Le Guin', 'n': 2, 'avg_pages': 285.0},
           {'name': 'Ted Chiang', 'n': 2, 'avg_pages': 315.5},
           {'name': 'Octavia E. Butler', 'n': 1, 'avg_pages': 264.0}]>
```

The generated SQL is a `LEFT OUTER JOIN` plus `GROUP BY`, so authors with zero books still
appear with `n = 0`. You can filter on annotations (`.filter(n__gt=1)` becomes `HAVING`)
and order by them.

`values(...)` **before** `annotate()` changes the grouping to just those fields:

```python
>>> Book.objects.values("author__name").annotate(pages=Sum("pages")).order_by("-pages")
<QuerySet [{'author__name': 'Ursula K. Le Guin', 'pages': 754},
           {'author__name': 'Ted Chiang', 'pages': 631},
           {'author__name': 'Octavia E. Butler', 'pages': 264}]>
```

### Conditional aggregation

Pass `filter=Q(...)` to count only some related rows:

```python
>>> Author.objects.annotate(
...     done=Count("books", filter=Q(books__status="done")),
...     total=Count("books"),
... ).values_list("name", "done", "total")
<QuerySet [('Ursula K. Le Guin', 1, 3), ('Ted Chiang', 1, 2), ('Octavia E. Butler', 0, 1)]>
>>> Book.objects.aggregate(n=Count("id"), done=Count("id", filter=Q(status="done")))
{'n': 6, 'done': 2}
```

One query instead of one per status.

### The multiple-join trap

Annotating two different to-many relations in one query multiplies rows:

```python
>>> Author.objects.annotate(nb=Count("books"), nr=Count("books__reviews")).values("name", "nb", "nr")
<QuerySet [{'name': 'Ursula K. Le Guin', 'nb': 3, 'nr': 2}, ...]>
```

Le Guin had **two** books at that point, not three. The join to reviews duplicated the
book with two reviews, and `Count("books")` counted the duplicates. `distinct=True`
fixes simple counts:

```python
>>> Author.objects.annotate(nb=Count("books", distinct=True),
...                         nr=Count("books__reviews", distinct=True)).values("name", "nb", "nr")
<QuerySet [{'name': 'Ursula K. Le Guin', 'nb': 2, 'nr': 2}, ...]>
```

`distinct` doesn't fix `Sum` or `Avg` (two different books can have the same page count),
so for those use a `Subquery` per relation, shown next.

## `Subquery`, `OuterRef` and `Exists`

"For each book, the rating of its most recent review" needs a correlated subquery:

```python
from django.db.models import OuterRef, Subquery, Exists

latest = Review.objects.filter(book=OuterRef("pk")).order_by("-created")
Book.objects.annotate(last_rating=Subquery(latest.values("rating")[:1]))
# [('A Wizard of Earthsea', None), ('Exhalation', 5), ('Kindred', None),
#  ('Stories of Your Life and Others', 3), ('The Dispossessed', 4), ('The Lathe of Heaven', None)]
```

`OuterRef("pk")` refers to the outer query's book. The subquery must return one column
and at most one row, hence `.values("rating")[:1]`.

`Exists` is the efficient way to filter on "has at least one related row matching...":

```python
>>> Book.objects.filter(Exists(Review.objects.filter(book=OuterRef("pk"), rating=5))).values_list("title", flat=True)
<QuerySet ['Exhalation', 'The Dispossessed']>
```

Unlike `filter(reviews__rating=5)`, it never duplicates books, so no `distinct()` is
needed.

## `Case` / `When`: conditional values

```python
>>> from django.db.models import Case, When, Value
>>> Book.objects.annotate(size=Case(
...     When(pages__lt=200, then=Value("short")),
...     When(pages__lt=350, then=Value("medium")),
...     default=Value("long"),
... )).values_list("title", "size")
<QuerySet [('A Wizard of Earthsea', 'short'), ('Exhalation', 'long'), ('Kindred', 'medium'), ...]>
```

## Window functions

Rank each author's books by length without collapsing rows:

```python
>>> from django.db.models import Window
>>> from django.db.models.functions import Rank
>>> Book.objects.annotate(r=Window(Rank(), partition_by="author", order_by=F("pages").desc())
... ).values_list("author__name", "title", "pages", "r")
<QuerySet [('Ursula K. Le Guin', 'A Wizard of Earthsea', 183, 3), ('Ted Chiang', 'Exhalation', 350, 1),
           ('Octavia E. Butler', 'Kindred', 264, 1), ('Ted Chiang', 'Stories of Your Life and Others', 281, 2),
           ('Ursula K. Le Guin', 'The Dispossessed', 387, 1), ('Ursula K. Le Guin', 'The Lathe of Heaven', 184, 2)]>
```

## `alias()`: compute without selecting

`alias()` is `annotate()` for values you only filter or order by, so they aren't added
to the `SELECT`:

```python
>>> Book.objects.alias(n=Count("reviews")).filter(n__gte=1).values_list("title", flat=True)
<QuerySet ['The Dispossessed', 'Exhalation', 'Stories of Your Life and Others']>
```

Notice that list isn't alphabetical, although `Book.Meta.ordering = ["title"]`. That's
deliberate Django behaviour: since 3.1, `Meta.ordering` is **not applied to queries with
`GROUP BY`**, because it would change the grouping. Our `annotate(Count(...))` query
printed as `... GROUP BY "catalog_book"."id", ...` with no `ORDER BY` at all. If order
matters, add an explicit `order_by()`.

## How It Actually Works

Every one of these features is an **expression**: an object that knows how to compile
itself to SQL and what type its result has. `F("pages")` compiles to a column reference,
`Value("short")` to a query parameter, `Count("books")` to `COUNT(...)` plus the joins
needed to reach `books`. Arithmetic on expressions (`F("price") / F("pages")`) builds a
`CombinedExpression` tree. When you `annotate()`, Django adds the expression to the
`SELECT`; if it contains an aggregate, the query compiler switches into grouping mode
and adds a `GROUP BY` covering every non-aggregate selected column (which is why
`values()` before `annotate()` changes the groups).

Each expression also has an `output_field` used to convert the database's result back
into Python. When Django can't infer it, it refuses rather than guessing. Adding
`F("price") + Value(1.5)` raised `FieldError: Cannot infer type of '+' expression involving
these types: DecimalField, FloatField. You must set output_field.` Note that `output_field` only controls the Python conversion; the
SQL arithmetic itself still follows the database's rules, which is exactly how the
integer division slipped through. `Cast` changes the SQL.

`OuterRef` is a placeholder resolved when the subquery is embedded: it becomes a
reference to the outer query's table alias, producing a correlated subquery.

## Common mistakes

- **Computing in Python** what one `annotate()` could do in SQL.
- **Two `Count`s across different to-many relations** without `distinct=True`, or
  `Sum`/`Avg` across them at all.
- **`instance.field += 1; save()`** for counters under concurrency. Use `F()`.
- **Trusting `Meta.ordering`** on grouped queries.
- **`filter(related__x=...)` producing duplicates** where `Exists` would not.
- **Assuming arithmetic types are the same on SQLite and PostgreSQL.**

## Exercise

1. List authors with the number of finished books and the average rating of their
   reviews, in one query, ordered by average rating (authors without reviews last; look
   up `F(...).desc(nulls_last=True)`).
2. For each book, annotate `has_five_star` (boolean) using `Exists`.
3. Annotate each book with the username of its most recent reviewer using `Subquery`.
4. Bucket books into decades of publication with `Case`/`When` or `ExtractYear` and
   count books per decade.
5. Reproduce the multiple-join trap on your own data, then fix it with `Subquery` for a
   `Sum`.
