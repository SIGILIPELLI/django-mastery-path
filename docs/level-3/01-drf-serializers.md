---
description: "DRF serializers: ModelSerializer, read-only and source fields, related fields, field and object validation, partial updates, nesting and query costs."
---

# 01 · Django REST Framework: Serializers

Django renders HTML; many projects also need a JSON API for a mobile app, a JavaScript
front end or other services. Django itself has `JsonResponse` and nothing more. **Django
REST Framework** (DRF) is the long-established third-party package that fills the gap:
serializers, API views, authentication, permissions, throttling, pagination and a
browsable API. Level 3 uses it for four lessons. For general REST design (resource
naming, status codes, versioning), see the
[REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/); this course
stays on the Django side.

A **serializer** is DRF's equivalent of a Django form, pointed the other way as well:
it turns model instances into JSON-ready data (*serialization*) and validates incoming
JSON into Python values it can save (*deserialization*). Everything below ran on DRF
3.18.1 with Django 6.1.1.

## Setup

```bash
python -m pip install djangorestframework
```

```python title="config/settings.py"
INSTALLED_APPS = [
    # ...
    "rest_framework",
]
```

## A `ModelSerializer`

```python title="catalog/serializers.py"
from rest_framework import serializers
from .models import Author, Book, Review, Tag


class BookSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source="author.name", read_only=True)
    tags = serializers.SlugRelatedField(slug_field="name", many=True,
                                        queryset=Tag.objects.all(), required=False)
    status_label = serializers.CharField(source="get_status_display", read_only=True)

    class Meta:
        model = Book
        fields = ["id", "title", "author", "author_name", "tags", "pages", "price",
                  "status", "status_label", "published"]
```

Printing an instance shows the fields DRF generated from the model, which is the fastest
way to see what it will accept:

```text
BookSerializer():
    id = BigIntegerField(label='ID', read_only=True)
    title = CharField(max_length=200)
    author = PrimaryKeyRelatedField(queryset=Author.objects.all())
    author_name = CharField(read_only=True, source='author.name')
    tags = SlugRelatedField(many=True, queryset=<QuerySet [...]>, required=False, slug_field='name')
    pages = IntegerField(max_value=9223372036854775807, min_value=0)
    price = DecimalField(decimal_places=2, max_digits=6, required=False)
    status = ChoiceField(choices=[('want', 'Want to read'), ('reading', 'Reading'), ('done', 'Finished')], required=False)
    status_label = CharField(read_only=True, source='get_status_display')
    published = DateField(allow_null=True, required=False)
```

Everything the model knew became a validation rule: `max_length`, the
`PositiveIntegerField`'s minimum, decimal precision, choices, and which fields are
optional (those with defaults or `null=True`).

### Serializing

```python
>>> BookSerializer(Book.objects.get(title="Exhalation")).data
{"id": 3, "title": "Exhalation", "author": 2, "author_name": "Ted Chiang",
 "tags": ["sci-fi", "short stories"], "pages": 350, "price": "16.00",
 "status": "reading", "status_label": "Reading", "published": "2019-05-07"}
```

Notice `price` is a **string**. DRF renders `Decimal` as a string by default
(`COERCE_DECIMAL_TO_STRING`) because JSON numbers are usually parsed as binary floats,
which would turn `16.00` into an approximate value. Clients should treat it as a decimal
string.

The `source` argument is the workhorse: `source="author.name"` follows the relation,
`source="get_status_display"` calls a method. The API field name and the model attribute
don't have to match.

## Representing relationships

| Field | Renders as | Accepts on write |
|-------|------------|------------------|
| `PrimaryKeyRelatedField` (default for FKs) | `2` | an ID |
| `SlugRelatedField(slug_field="name")` | `"sci-fi"` | a name |
| `StringRelatedField` | `str(obj)` | read-only |
| `HyperlinkedRelatedField` | a URL | a URL |
| a nested serializer | a full object | read-only unless you write `create()`/`update()` |

A common pattern, used above: write by ID (`author`), read a friendly extra field
(`author_name`). For detail endpoints, a nested representation:

```python
class ReviewSerializer(serializers.ModelSerializer):
    user = serializers.StringRelatedField(read_only=True)

    class Meta:
        model = Review
        fields = ["id", "book", "user", "rating", "body", "created"]
        read_only_fields = ["created"]


class BookDetailSerializer(BookSerializer):
    author = AuthorSerializer(read_only=True)
    reviews = ReviewSerializer(many=True, read_only=True)
    average_rating = serializers.SerializerMethodField()

    class Meta(BookSerializer.Meta):
        fields = BookSerializer.Meta.fields + ["reviews", "average_rating"]

    def get_average_rating(self, obj):
        return obj.reviews.aggregate(avg=Avg("rating"))["avg"]
```

which produced, for Exhalation:

```json
{"id": 3, "title": "Exhalation", "author": {"id": 2, "name": "Ted Chiang", "born": 1967},
 "author_name": "Ted Chiang", "tags": ["sci-fi", "short stories"], ...,
 "reviews": [{"id": 3, "book": 3, "user": "ana", "rating": 5, "body": "",
              "created": "2026-10-02T08:10:33.601455Z"}],
 "average_rating": 5.0}
```

## Validation

Field-level rules go in `validate_<field>()`, cross-field rules in `validate()`:

```python
    def validate_title(self, value):
        if value.isupper():
            raise serializers.ValidationError("Please don't shout.")
        return value.strip()

    def validate(self, attrs):
        status = attrs.get("status", getattr(self.instance, "status", None))
        published = attrs.get("published", getattr(self.instance, "published", None))
        if status == Book.Status.DONE and published is None:
            raise serializers.ValidationError({"published": "Finished books need a publication date."})
        return attrs
```

We sent deliberately bad data:

```python
>>> s = BookSerializer(data={"title": "DUNE", "author": 99, "pages": -1,
...                          "tags": ["sci-fi", "nope"], "price": "12.345", "status": "done"})
>>> s.is_valid(), s.errors
False
{"title": ["Please don't shout."],
 "author": ["Invalid pk \"99\" - object does not exist."],
 "tags": ["Object with name=nope does not exist."],
 "pages": ["Ensure this value is greater than or equal to 0."],
 "price": ["Ensure that there are no more than 2 decimal places."]}
```

Every field-level problem at once, as a dict ready to return as a 400 response. The
cross-field rule in `validate()` didn't appear: in DRF, object-level validation only
runs **after every field passes**. That's a real difference from Django forms, whose
`clean()` runs even when some fields have failed.

Valid data becomes model instances and Python values in `validated_data`, and `save()`
calls `create()`:

```python
>>> s = BookSerializer(data={"title": "Parable of the Sower", "author": 3, "pages": 345,
...                          "tags": ["sci-fi"], "status": "want"})
>>> s.is_valid(), s.validated_data["author"], s.validated_data["tags"]
(True, <Author: Octavia E. Butler>, [<Tag: sci-fi>])
>>> obj = s.save(); obj.pk, list(obj.tags.values_list("name", flat=True))
(7, ['sci-fi'])
```

`ModelSerializer.create()` handled the many-to-many correctly: it creates the book, then
sets tags.

### Partial updates and `self.instance`

For PATCH, pass `partial=True`: missing fields are left alone.

```python
>>> BookSerializer(obj, data={"status": "done"}, partial=True).is_valid()
False   # {'published': ['Finished books need a publication date.']}
>>> s = BookSerializer(obj, data={"pages": 350}, partial=True); s.is_valid(), s.save().pages
(True, 350)
```

The cross-field rule worked on a partial update because `validate()` falls back to
`self.instance` for fields not in the payload. Writing `attrs["published"]` instead would
raise `KeyError` on every PATCH that doesn't include it, the serializer twin of the
`cleaned_data["x"]` mistake from Level 1.

One guard rail worth knowing: accessing `.data` on a serializer built with `data=` before
calling `is_valid()` raises:

```text
AssertionError: When a serializer is passed a `data` keyword argument you must call
`.is_valid()` before attempting to access the serialized `.data` representation.
```

## Serializers and query counts

Serializers are N+1 machines if you let them. We serialized all seven books with
`BookDetailSerializer(..., many=True)`:

| QuerySet | Queries |
|----------|---------|
| `Book.objects.all()` | **33** |
| `+ select_related("author").prefetch_related("tags", "reviews__user")` | **11** |
| `+ .annotate(average_rating=Avg("reviews__rating"))` and a plain `FloatField(read_only=True)` instead of the method field | **4** |

The prefetch removed the per-book author, tag and review queries, but the
`SerializerMethodField` still ran one `aggregate()` per book (7 of the 11). Moving the
computation into the QuerySet as an annotation, and declaring the field as a plain
read-only field that reads the annotated attribute, brought it to a constant 4. The rule:
the serializer decides *what* to output, the view's QuerySet decides *how it's fetched*.
Lesson 02 puts that QuerySet in `get_queryset()`.

## How It Actually Works

A serializer is a declarative class like a form: a metaclass collects field instances
into `_declared_fields`, and `ModelSerializer.get_fields()` adds generated fields by
inspecting the model's `_meta`, mapping each model field class to a serializer field via
`serializer_field_mapping` and copying constraints into keyword arguments. That's what
`repr()` printed.

Serializing calls `to_representation(instance)`: for each readable field, it resolves the
`source` path with `get_attribute()` (following dots and calling callables), then calls
the field's own `to_representation()` (a `Decimal` becomes a string, a related object
becomes its pk). `many=True` doesn't change the serializer: it wraps it in a
`ListSerializer` that iterates the QuerySet and calls the child once per item, which is
exactly why per-object database access multiplies.

Deserializing runs `to_internal_value(data)`: each writable field's
`run_validation()` (type conversion, field validators, `validate_<name>`), collecting
errors in a dict. Only if that dict is empty does `validate(attrs)` run, followed by any
`Meta.validators` such as uniqueness checks. `save()` then calls `create()` or
`update()` depending on whether `instance` was passed.

## Common mistakes

- **`fields = "__all__"`** exposing new model fields automatically (and accepting writes
  to them). List fields explicitly.
- **Indexing `attrs[...]` in `validate()`** on serializers used for PATCH.
- **`SerializerMethodField` doing queries** in list endpoints. Annotate or prefetch.
- **Nested writable serializers without `create()`/`update()`**: DRF raises an error
  telling you to write them, and they're easy to get subtly wrong. Prefer writing by ID.
- **Treating `price` as a JSON number on the client** and losing precision.
- **Putting authorization in serializers.** They validate shape and values; who may do
  what belongs in permissions (lesson 03).

## Exercise

1. Write `AuthorSerializer`, `BookSerializer` and `ReviewSerializer` for your catalogue
   and print each one's `repr()`.
2. Add a `validate_rating()` to `ReviewSerializer` and a `validate()` rule that a user's
   review body must be non-empty if the rating is 1.
3. Write a `BookDetailSerializer` with nested author and reviews, then get serializing
   all books to a constant query count, measured with `CaptureQueriesContext`.
4. Serialize a `Decimal` price and parse the JSON in JavaScript (`JSON.parse`). What type
   do you get, and what would happen if DRF had sent a number?
