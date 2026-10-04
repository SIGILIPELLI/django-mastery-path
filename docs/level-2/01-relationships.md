---
description: "Model one-to-many, many-to-many and one-to-one relations in Django: related managers, related_name, through models, on_delete and reverse lookups."
---

# 01 · Model Relationships: ForeignKey, ManyToMany & OneToOne

Real data is connected. A book has an author, a book has many tags, a user has one
profile, a club has members with roles. Django models these with three field types, and
each one gives you navigation in **both** directions without extra code. This lesson
covers how each relationship is stored, how you traverse it, and the decisions
(`on_delete`, `related_name`, through models) that are painful to change later.

The examples continue with the `catalog` app from Level 1 and add a profile and a book
club. Outputs come from running the code on our sample data.

## One-to-many: `ForeignKey`

```python
class Book(models.Model):
    author = models.ForeignKey(Author, on_delete=models.PROTECT, related_name="books")
```

Stored as an `author_id` column on the book table, with an index. You can navigate both
ways:

```python
>>> b = Book.objects.get(title="Exhalation")
>>> b.author, b.author_id
(<Author: Ted Chiang>, 2)
>>> le = Author.objects.get(name__startswith="Ursula")
>>> le.books.all()
<QuerySet [<Book: A Wizard of Earthsea>, <Book: The Dispossessed>, <Book: The Lathe of Heaven>]>
>>> type(le.books).__name__
'RelatedManager'
```

`le.books` is a **related manager**: a manager pre-filtered to this author, so every
QuerySet method works on it (`le.books.filter(pages__gt=200)`, `le.books.count()`). It
also has `add()`, `create()`, `remove()` and `set()` for changing the relationship.

### `related_name` — name the reverse side

Without `related_name`, the reverse accessor would be `author.book_set`. Name it
explicitly. It reads better and it's required when one model has two foreign keys to
the same model (e.g. `Message.sender` and `Message.recipient` both pointing at `User`),
which would otherwise clash.

### `_id` attributes are free

We counted queries for both forms:

```python
b = Book.objects.get(pk=1); b.author_id             # 1 query
b = Book.objects.get(pk=1); b.author.name; b.author.name   # 2 queries
```

`author_id` is already on the row. Accessing `author` loads the author with a second
query (then caches it, so the second `.name` was free). When you only need the ID, for
a comparison or a URL, use `author_id`.

### `on_delete` — decide what deletion means

| Option | When the author is deleted... | Use for |
|--------|-------------------------------|---------|
| `CASCADE` | their books are deleted too | parts that can't exist alone (a review without its book) |
| `PROTECT` | deletion raises `ProtectedError` | important records that must be dealt with first |
| `RESTRICT` | like `PROTECT`, but allows deletion if the object is also being deleted via another `CASCADE` path | complex graphs |
| `SET_NULL` | `author` becomes `NULL` (needs `null=True`) | optional links, keep history |
| `SET_DEFAULT` / `SET(...)` | set to a default or computed value | reassign to a placeholder |
| `DO_NOTHING` | nothing in Django; the database decides | only with explicit DB-level handling |

Our `Review.book` uses `CASCADE`. Deleting book 1 reported exactly what went with it:

```python
>>> Book.objects.get(pk=1).delete()
(5, {'catalog.Book_tags': 2, 'catalog.Review': 2, 'catalog.Book': 1})
```

Five rows: the book, its two reviews, and its two tag links. And because `Book.author`
is `PROTECT`, deleting an author who still has books fails with
`ProtectedError: Cannot delete some instances of model 'Author' because they are
referenced through protected foreign keys: 'Book.author'.` That's usually what you want
for anything a user would be upset to lose silently.

## Many-to-many: `ManyToManyField`

```python
class Book(models.Model):
    tags = models.ManyToManyField(Tag, blank=True, related_name="books")
```

No column on either table; Django creates a join table (`catalog_book_tags`) with a
unique pair of foreign keys. Both sides get a related manager:

```python
>>> b.tags.all()
<QuerySet [<Tag: sci-fi>, <Tag: short stories>]>
>>> Tag.objects.get(name="classic").books.values_list("title", flat=True)
<QuerySet ['A Wizard of Earthsea', 'Kindred', 'The Dispossessed']>
```

Change links with `b.tags.add(t)`, `remove(t)`, `set([t1, t2])` and `clear()`. These
write to the join table immediately; there's no `save()` involved.

## Relationships with data: a `through` model

When the link itself has attributes (a member's role, the date they joined), make the
join table an explicit model:

```python
class Club(models.Model):
    name = models.CharField(max_length=100)
    members = models.ManyToManyField(settings.AUTH_USER_MODEL, through="Membership",
                                     related_name="clubs")


class Membership(models.Model):
    class Role(models.TextChoices):
        MEMBER = "member"
        ORGANISER = "organiser"

    club = models.ForeignKey(Club, on_delete=models.CASCADE)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=Role, default=Role.MEMBER)
    joined = models.DateField(auto_now_add=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["club", "user"], name="unique_membership")]
```

```python
>>> Membership.objects.create(club=club, user=ana, role="organiser")
>>> club.members.add(raj, through_defaults={"role": "member"})
>>> club.members.all()
<QuerySet [<User: ana>, <User: raj>]>
>>> ana.clubs.all()
<QuerySet [<Club: Sunday SF>]>
>>> User.objects.filter(membership__role="organiser").values_list("username", flat=True)
<QuerySet ['ana']>
```

You keep the convenient `club.members` manager **and** can query the link's fields.
`through_defaults` supplies values for extra fields when using `add()`. If there's any
chance a link will need attributes later, start with a through model: converting an
auto-created join table later requires a careful migration.

Note `settings.AUTH_USER_MODEL` instead of importing `User`. That keeps the model working
if the project uses a custom user model (lesson 07).

## One-to-one: `OneToOneField`

A `ForeignKey` with `unique=True`, plus a reverse accessor that returns a single object
instead of a manager:

```python
class Profile(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                                related_name="profile")
    bio = models.TextField(blank=True)
```

The reverse side raises if the row doesn't exist:

```python
>>> ana.profile
RelatedObjectDoesNotExist: User has no profile.
```

Handle that with `hasattr(user, "profile")` or a `try/except`, or guarantee the profile
exists by creating it alongside the user (lesson 05 and Level 3 · 04 discuss the
options). Common uses: extending a model you don't own (like `User`), or splitting rarely
used, bulky columns into a separate table.

## Querying across relationships

Double underscores follow relationships in any direction, including reverse ones:

```python
>>> Author.objects.filter(books__tags__name="short stories").distinct()
<QuerySet [<Author: Ted Chiang>]>
```

That's author → book → tag through two joins. `distinct()` matters: Ted Chiang has two
books tagged "short stories", so without it he'd appear twice. Any filter across a
to-many relationship can multiply rows.

## How It Actually Works

Relationship fields install **descriptors** on both model classes. On `Book`, the
`author` attribute is a `ForwardManyToOneDescriptor`: reading it checks a per-instance
cache, and on a miss runs `Author.objects.get(pk=self.author_id)` and caches the result.
On `Author`, `related_name="books"` installs a `ReverseManyToOneDescriptor` whose
`__get__` builds a manager class on the fly, subclassing the default manager and adding
a filter `author=<this instance>` to every QuerySet. That's why `le.books` has all the
QuerySet methods: it *is* a manager.

Many-to-many descriptors do the same with the join table. When you don't give a
`through` model, Django generates one at class-creation time (that's the
`catalog.Book_tags` label in the delete output). `add()` and `remove()` are bulk
`INSERT`/`DELETE` statements on that hidden model, which is why they don't need
`save()`, and why they bypass any `save()` logic on your models.

`on_delete` is enforced by Django's `Collector`, not by the database: before deleting,
Django walks every relationship pointing at the object, gathers what to cascade, checks
for `PROTECT`, then runs the deletes in dependency order. That's how it could report the
full count dictionary. It also means raw SQL `DELETE` statements bypass these rules:
for the classic options, the database's foreign key has no `ON DELETE` action at all.

Django 6.1 added **database-level** variants: `DB_CASCADE`, `DB_SET_NULL` and
`DB_SET_DEFAULT`. We tried `DB_CASCADE` on a small `Folder`/`Doc` pair. The generated
table carried `REFERENCES "catalog_folder" ("id") ON DELETE CASCADE`, and deleting a
folder with two docs ran a single `DELETE FROM "catalog_folder" ...`, reported
`(1, {'catalog.Folder': 1})`, and the database removed both docs itself. That's faster
for large trees and works for raw SQL too, but the trade-off is real: Django no longer
knows what was deleted, so the count is incomplete and no `pre_delete`/`post_delete`
signals fire for the cascaded rows. The two styles also can't be mixed in one chain.
When we pointed a `DB_CASCADE` key at `Book` (which itself uses Python-level
`PROTECT`/`CASCADE` relations), the system check refused:

```text
catalog.Note.book: (fields.E323) Field specifies database-level on_delete variant, but
referenced model uses Python-level variant.
```

Treat the `DB_*` options as a tool for big, self-contained hierarchies; the Python-level
options remain the default for good reason.

## Common mistakes

- **Leaving `related_name` off** and later hitting a reverse accessor clash.
- **Choosing `CASCADE` by reflex.** Ask what a user would expect to happen. Losing an
  author shouldn't silently delete fifty books.
- **Importing `User` directly** in model fields instead of `settings.AUTH_USER_MODEL`.
- **Forgetting `distinct()`** after filtering across a to-many relationship.
- **Accessing `obj.related.field` when you only need `obj.related_id`**, costing a query.
- **Auto-created M2M when the link has meaning** (role, date, order); start with `through`.

## Exercise

1. Change the Level 1 reading list so `Entry.author` is a `ForeignKey` to a new `Author`
   model. Write the migration path: add the new model, add a nullable FK, then (in Level
   3 · 06's style, or by hand in the shell for now) fill it from the old text field.
2. Add a `Shelf` model with a many-to-many to `Entry` through `ShelfItem`, which has a
   `position` integer. List one shelf's books in position order.
3. Delete an author with `PROTECT` and read the error; switch to `SET_NULL` and observe
   what happens to their books.
4. Write a query for "all users who are organisers of a club with at least one other
   member".
