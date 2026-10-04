---
description: "Fast Django on PostgreSQL: reading EXPLAIN ANALYZE, composite and functional indexes, why iexact skipped an index, and deep-offset pagination."
---

# 04 · Database Performance: Indexes, EXPLAIN & Query Budgets

Level 2 fixed the *number* of queries. This lesson is about the *cost* of each one. On a
table with a few hundred rows every query is fast, so nothing teaches you anything. We
loaded **200,005 books** into PostgreSQL 16, then measured real queries with `EXPLAIN
ANALYZE` before and after changes. The timings are from one laptop and will differ on
your hardware; the *plans* (which strategy PostgreSQL chose, and why) are what transfer.

## Reading a plan from Django

Every QuerySet has `.explain()`; on PostgreSQL, `analyze=True` actually runs the query and
reports real timings:

```python
qs = (Book.objects.filter(status="reading", published__year=2010)
                  .order_by("-published")[:20])
print(qs.explain(analyze=True))
```

```text
Limit  (cost=5334.93..5337.23 rows=20 width=61) (actual time=8.865..9.475 rows=20 loops=1)
  ->  Gather Merge  (...)
        Workers Planned: 1
        ->  Sort  (...)
              Sort Key: published DESC
              Sort Method: top-N heapsort  Memory: 29kB
              ->  Parallel Seq Scan on catalog_book  (... rows=80 loops=2)
                    Filter: ((published >= '2010-01-01'::date) AND (published <= '2010-12-31'::date)
                             AND ((status)::text = 'reading'::text))
                    Rows Removed by Filter: 99923
Planning Time: 0.076 ms
Execution Time: 9.487 ms
```

How to read it, bottom up:

- **`Parallel Seq Scan`**: PostgreSQL read the *whole table* (two processes, ~100,000
  rows each) and threw away almost everything (`Rows Removed by Filter: 99923` per
  worker) to find 160 matching rows.
- **`Sort ... top-N heapsort`**: then sorted the matches to find the newest 20.
- **`actual time`** is real milliseconds; `cost` is the planner's estimate in arbitrary
  units. Compare estimated `rows` with actual `rows`: big mismatches mean stale statistics
  (run `ANALYZE`) or a query the planner can't estimate well.

Also notice something good: `published__year=2010` became a **range**
(`published >= '2010-01-01' AND published <= '2010-12-31'`), not `EXTRACT(year ...)`.
Django rewrites `__year` lookups into ranges precisely so indexes on the column can be
used.

## Adding the right index

The query filters on `status` (equality) and `published` (range), and sorts by
`published DESC`. A composite index in that order matches it:

```python
class Book(models.Model):
    ...
    class Meta:
        indexes = [
            models.Index(fields=["status", "-published"], name="book_status_published"),
        ]
```

(In production, add it with `AddIndexConcurrently`, Level 3 · 06.) The same query
afterwards:

```text
Limit  (cost=0.42..65.85 rows=20 width=62) (actual time=0.013..0.027 rows=20 loops=1)
  ->  Index Scan using book_status_published on catalog_book  (... rows=20 loops=1)
        Index Cond: (((status)::text = 'reading'::text) AND (published >= '2010-01-01'::date)
                     AND (published <= '2010-12-31'::date))
Planning Time: 0.139 ms
Execution Time: 0.039 ms
```

**9.5 ms → 0.04 ms.** No sort step at all: the index is already in the requested order,
so PostgreSQL walks it and stops after 20 rows. Column order matters: equality columns
first, then the range or sort column. An index on `(published, status)` couldn't serve
this query nearly as well.

## When an index can't be used: `iexact`

Case-insensitive lookups are common for usernames, emails and titles:

```python
Book.objects.filter(title__iexact="generated book 123456")
```

`print(qs.query)` shows what Django generates on PostgreSQL:

```sql
... WHERE UPPER("catalog_book"."title"::text) = UPPER(generated book 123456) ...
```

(The printed query inlines the parameter without quotes; the real query sends it
separately.) A plain index on `title` indexes the *raw* values, not `UPPER(title)`, so
PostgreSQL fell back to a full scan:

```text
->  Parallel Seq Scan on catalog_book  (... rows=0 loops=2)
      Filter: (upper((title)::text) = 'GENERATED BOOK 123456'::text)
      Rows Removed by Filter: 100002
Execution Time: 19.830 ms
```

A **functional index** on the same expression fixes it:

```python
from django.db.models.functions import Upper

class Meta:
    indexes = [models.Index(Upper("title"), name="book_upper_title")]
```

```text
->  Index Scan using book_upper_title on catalog_book  (... rows=1 loops=1)
      Index Cond: (upper((title)::text) = 'GENERATED BOOK 123456'::text)
Execution Time: 0.024 ms
```

The general rule: an index helps only if the query's `WHERE` uses the **same expression**
the index was built on. Lookups like `icontains` (`UPPER(title) LIKE '%...%'`) can't use a
B-tree at all because of the leading wildcard; for substring and fuzzy search on
PostgreSQL, look at trigram indexes (`django.contrib.postgres` provides
`GinIndex` with `opclasses=["gin_trgm_ops"]`) or full-text search.

## Deep pagination

`OFFSET` has to walk past every skipped row:

```python
Book.objects.order_by("id")[150000:150020]       # OFFSET 150000 LIMIT 20
```

```text
Execution Time: 17.157 ms
```

**Keyset** (or "seek") pagination asks for rows after the last one seen instead:

```python
Book.objects.filter(id__gt=150000).order_by("id")[:20]
```

```text
Execution Time: 0.016 ms
```

Same 20 rows, about a thousand times less work, and the cost doesn't grow with page
number. That's what DRF's `CursorPagination` does (Level 3 · 03). The price is that you
can't jump to page 7,500 directly, which users rarely need. Separately, `COUNT(*)` for
the "page X of Y" display took 6.6 ms here and grows linearly with the table on
PostgreSQL; for huge tables, show "more results" instead of exact totals, or cache the
count.

## A performance workflow

1. **Find slow pages** from real data: logging of slow requests, an APM tool, or
   PostgreSQL's `pg_stat_statements` extension (not enabled in our local server, so no
   output here) to see which queries take the most total time.
2. **Count queries first** (Level 2 · 03). N+1 is more common than a missing index.
3. **`explain(analyze=True)`** the expensive query on production-sized data. Look for
   `Seq Scan` with large `Rows Removed by Filter`, big sorts, and estimate/actual
   mismatches.
4. **Add an index that matches the query's filters and ordering**, or rewrite the query
   (fetch fewer columns with `only()`/`values()`, move work into the database with
   annotations, cache the result).
5. **Re-measure**, then lock it in with an `assertNumQueries` test and, for critical
   queries, a test that asserts the plan uses the index if you're on PostgreSQL in CI.

## Indexes aren't free

Every index slows down inserts and updates (each write maintains every index), uses disk
and memory, and adds work to `VACUUM`. Don't index every column "just in case". Django
already indexes primary keys, foreign keys (`db_index=True` by default on `ForeignKey`)
and `unique` fields. Add others when a measured query needs them, and periodically look
for unused ones (PostgreSQL tracks index usage in `pg_stat_user_indexes`).

## How It Actually Works

PostgreSQL's planner estimates the cost of alternative plans (sequential scan, index scan,
bitmap scan, different join orders) using table statistics gathered by `ANALYZE`: row
counts, value distributions, most-common values. It picks the cheapest estimate. A B-tree
index stores the indexed values in sorted order with pointers to rows, so it can find an
exact value or a range in logarithmic time and return rows already ordered. That's why
the composite index removed the sort: `(status, published DESC)` keeps all "reading"
books together, ordered newest first.

A sequential scan isn't always wrong: when a query needs a large fraction of the table,
reading it straight through beats jumping around via an index, and the planner will
correctly ignore your index. That's also why a tiny development table tells you nothing:
PostgreSQL will sequentially scan a 50-row table regardless of indexes.

## Common mistakes

- **Optimising on development-sized data.** Load realistic volumes or copy anonymised
  production data.
- **Indexes whose column order doesn't match the query.**
- **Case-insensitive lookups without a matching functional index.**
- **Deep `OFFSET` pagination** on large tables.
- **Indexing everything**, slowing writes for indexes nothing uses.
- **Forgetting to declare manual indexes in `Meta.indexes`**, so migrations drift.

## Exercise

1. Generate 200,000 rows in a table of your own with `bulk_create` in batches of 10,000.
   Run `ANALYZE` (via `connection.cursor()`) afterwards.
2. Pick your slowest list query, run `explain(analyze=True)`, add a composite index in
   `Meta.indexes`, migrate, and measure again.
3. Find an `iexact` or `istartswith` lookup in your project and add a matching functional
   index. Confirm the plan changes.
4. Compare `OFFSET` and keyset pagination at page 1, 100 and 5,000.
