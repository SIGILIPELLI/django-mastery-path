---
description: "Keep data correct under concurrency in Django: atomic blocks, savepoints, a measured lost-update race, F() and select_for_update, and constraints."
---

# 05 · Transactions, Constraints & Race Conditions

Everything so far assumed one request at a time. Production doesn't work that way: two
people click "reserve" on the last seat at the same moment; a retry runs a payment twice;
a crash leaves half an order written. This lesson covers the three tools that keep data
correct anyway: **transactions** (all or nothing), **locking and atomic updates** (no
lost writes), and **database constraints** (rules that hold no matter which code path
writes). Unlike most of the course, the race-condition experiments here ran against
**PostgreSQL 16** (a local server started through the `pgserver` Python package), because
SQLite serialises writes and hides exactly the problems we want to see.

## Transactions with `atomic()`

By default Django runs in **autocommit** mode: each query is committed immediately.
To group writes, use `transaction.atomic()` as a context manager or decorator:

```python
from django.db import transaction

@transaction.atomic
def place_order(user, cart):
    order = Order.objects.create(user=user)
    for item in cart:
        OrderLine.objects.create(order=order, product=item.product, qty=item.qty)
        Product.objects.filter(pk=item.product.pk).update(stock=F("stock") - item.qty)
    return order
```

If anything inside raises, everything inside is rolled back. If the block finishes, it
commits. That's it; most of the subtlety is in the edges below.

### Catching errors inside `atomic()`

A database error inside a transaction leaves it unusable until it's rolled back. We
caught an `IntegrityError` (a duplicate review) inside an `atomic()` block and then ran
one more query:

```text
caught: UNIQUE constraint failed: catalog_review.book_id, catalog_review.user_id
TransactionManagementError: An error occurred in the current transaction. You can't
execute queries until the end of the 'atomic' block.
```

The fix is a **nested** `atomic()` around the statement that might fail. Inner blocks are
savepoints, so only they roll back:

```python
with transaction.atomic():
    try:
        with transaction.atomic():
            Review.objects.create(book=b, user=ana, rating=5)
    except IntegrityError:
        ...  # handle the duplicate; the outer transaction is still usable
    Review.objects.count()   # works
```

That ran cleanly. Never wrap a `try/except IntegrityError` directly around ORM calls
inside a transaction without that inner block.

### `ATOMIC_REQUESTS`

Setting `"ATOMIC_REQUESTS": True` in a database's config wraps every view in a
transaction. It's a simple safety net, with costs: every request holds a transaction for
its whole duration (including slow template rendering), and you can't commit partway. Many
teams prefer explicit `atomic()` around the writes that need it.

### Side effects after commit

Emails, webhooks and task queueing should happen only if the data actually committed.
Lesson 04 showed a rollback that still "sent" an email. Use:

```python
transaction.on_commit(lambda: send_receipt.enqueue(order.pk))
```

Outside a transaction, `on_commit` runs the callback immediately.

## The lost update, measured

The classic bug: read a value, change it in Python, write it back. Two requests that
interleave both read the same starting value and one increment vanishes. We ran four
threads, each incrementing the same row 50 times (so the answer should be 200), with
three implementations:

```python
def naive():
    b = Book.objects.get(pk=1); time.sleep(0.001)
    b.pages += 1
    b.save(update_fields=["pages"])

def f_expr():
    Book.objects.filter(pk=1).update(pages=F("pages") + 1)

def locked():
    with transaction.atomic():
        b = Book.objects.select_for_update().get(pk=1); time.sleep(0.001)
        b.pages += 1
        b.save(update_fields=["pages"])
```

Two runs on PostgreSQL:

```text
naive expected 200, got 51
f_expr expected 200, got 200
locked expected 200, got 200
naive expected 200, got 50
f_expr expected 200, got 200
locked expected 200, got 200
```

The naive version lost three quarters of the updates. (The 1 ms sleep widens the window
between read and write to make the race reliable; in production the window is smaller,
so the bug is rarer and much harder to reproduce, which is worse.) `update_fields` didn't
help: it limits *which* columns are written, not *when*.

### Fix 1: do the arithmetic in the database

`F("pages") + 1` compiles to `SET pages = pages + 1`, which the database applies
atomically per row. For counters, balances and stock, this is the first choice. Add a
condition to make it a safe "decrement if available":

```python
updated = Product.objects.filter(pk=pk, stock__gte=qty).update(stock=F("stock") - qty)
if updated == 0:
    raise OutOfStock
```

We tried it on a row with value 1, decrementing twice: the first `update()` returned
`1` (row changed), the second `0` (condition no longer true), and the value ended at 0,
never -1.

### Fix 2: lock the row with `select_for_update()`

When the logic between read and write is too complex for an expression, lock the row:
other transactions that try to lock or write it wait until you commit. On PostgreSQL the
query ended with:

```sql
... WHERE "catalog_book"."id" = 1 ORDER BY "catalog_book"."title" ASC FOR UPDATE
```

Rules and options:

- It **must** be inside a transaction. Outside one, PostgreSQL raised
  `TransactionManagementError: select_for_update cannot be used outside of a transaction.`
- `of=("self",)` locks only the main table when you also `select_related`;
  `skip_locked=True` skips rows another transaction holds (great for job queues:
  `... FOR UPDATE OF "catalog_book" SKIP LOCKED`); `nowait=True` errors instead of waiting.
- Lock rows in a consistent order (e.g. by primary key) to avoid deadlocks.
- Keep the transaction short. Every waiting request is a blocked worker.

**SQLite ignores it.** On SQLite the same query had no `FOR UPDATE` clause at all, and
calling it outside a transaction raised nothing, because SQLite locks the whole database
for writes instead. Code that relies on row locks must be tested on the database you run
in production.

## Constraints: rules the database enforces

Validation in forms and serializers protects one entry point. Constraints protect the data
from *every* entry point: `update()`, bulk imports, the shell, other services.

```python
class Review(models.Model):
    ...
    class Meta:
        constraints = [
            models.CheckConstraint(condition=models.Q(rating__gte=1, rating__lte=5),
                                   name="rating_1_to_5"),
            models.UniqueConstraint(fields=["book", "user"], name="one_review_per_user"),
        ]
```

(`condition=` replaced the older `check=` argument in Django 5.1.) Other useful forms:

```python
from django.db.models.functions import Lower

# case-insensitive uniqueness
models.UniqueConstraint(Lower("email"), name="unique_lower_email")

# partial uniqueness: only one *active* subscription per user
models.UniqueConstraint(fields=["user"], condition=models.Q(active=True),
                        name="one_active_subscription")
```

Since Django 4.1, model validation also checks constraints. Calling `full_clean()` on a
review with rating 9 for a book the user had already reviewed reported both:

```text
{'__all__': ['Constraint “rating_1_to_5” is violated.',
             'Review with this Book and User already exists.']}
```

ModelForms call this automatically. DRF serializers don't (lesson 03), and neither do
`save()`, `update()` or `bulk_create()`. Treat validation as the friendly message and the
constraint as the guarantee, and keep both.

## A note on database arithmetic

Level 2 · 02 showed SQLite computing `15.00 / 350` as `0`, because it had stored the
decimal as an integer. The same annotation on PostgreSQL returned
`Decimal('0.04285714285714285714')`. Same Django code, different answer: one more reason
to run your test suite against the production database engine.

## How It Actually Works

`atomic()` keeps a stack per database connection. Entering the outermost block turns off
autocommit and begins a transaction; entering a nested block creates a **savepoint**.
Leaving a block normally releases the savepoint or commits; leaving with an exception
rolls back to the savepoint or rolls back the transaction. When a query fails inside a
block, Django marks the connection as `needs_rollback`, which is what produces
`TransactionManagementError` on the next query until the block ends. `on_commit`
callbacks are stored on the connection along with the savepoint they belong to, run in
order after the outermost commit, and discarded if their savepoint rolls back.

`select_for_update()` just adds the `FOR UPDATE` clause; the waiting is done by
PostgreSQL's row-level locks, which are held until the transaction ends. `F()` updates
work without explicit locks because an `UPDATE` statement itself takes a row lock, and
under PostgreSQL's default `READ COMMITTED` isolation, a second `UPDATE` waiting on that
lock re-reads the committed row value before applying `pages + 1`.

## Common mistakes

- **Read-modify-write in Python** for counters, balances and stock.
- **`try/except IntegrityError` inside `atomic()`** without a nested block.
- **Long transactions with `select_for_update`**, including network calls while holding
  a lock.
- **Testing concurrency on SQLite** and assuming it holds on PostgreSQL or MySQL.
- **Side effects before commit**; use `on_commit`.
- **Validation without constraints**, or constraints without validation.

## Exercise

1. Write a `reserve_seat(event_id, user)` function that can never oversell: use a
   conditional `update()`, then rewrite it with `select_for_update()`.
2. Reproduce the lost-update experiment on your database with threads (each thread must
   close its connection when done: `connection.close()`). If you only have SQLite, note
   what happens instead and why.
3. Add a case-insensitive unique constraint on a username or email field and test that
   `full_clean()` reports it.
4. Write a test using `captureOnCommitCallbacks` proving an email is only sent when the
   order commits.
