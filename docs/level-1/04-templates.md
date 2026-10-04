---
description: "Django templates in depth: where Django finds them, inheritance with blocks, tags and filters, auto-escaping, and how silent variable failures work."
---

# 04 · Templates: Inheritance, Tags, Filters & Autoescaping

Django's template language is intentionally limited. You can loop, branch, call methods
that take no arguments and apply filters, but you can't write arbitrary Python. That
limit is a design decision: it keeps logic in views, and it means a designer can edit a
template without being able to break the database. This lesson covers where templates
live, how inheritance works, the tags and filters you'll use daily, and the one feature
that quietly protects every page you render: auto-escaping.

## Where Django looks for templates

The `TEMPLATES` setting configures the engine. Two keys decide the search path:

```python title="config/settings.py"
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],   # project-wide templates, searched first
        "APP_DIRS": True,                    # then <app>/templates/ for each app
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

When a view asks for `"catalog/book_list.html"`, Django tries
`templates/catalog/book_list.html`, then `catalog/templates/catalog/book_list.html`, then
the same path under every other app in `INSTALLED_APPS` order. The first file found wins.

That's why app templates live in a folder named after the app
(`catalog/templates/catalog/`). Without that extra level, two apps that both ship
`list.html` would collide, and the one earlier in `INSTALLED_APPS` would silently win.
The same rule lets you **override** a third-party app's template by putting a file with
the same relative path in your project `templates/` directory; that's how people
customise the admin.

## Inheritance: one layout, many pages

A base template defines the page skeleton and named **blocks**:

```django title="templates/base.html"
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>{% block title %}Shelf{% endblock %}</title></head>
<body>
  <nav><a href="{% url 'catalog:book-list' %}">All books</a></nav>
  <main>{% block content %}{% endblock %}</main>
</body>
</html>
```

A child template `extends` it and fills in blocks. Anything outside a block in a child is
ignored.

```django title="catalog/templates/catalog/book_list.html"
{% extends "base.html" %}
{% block title %}Books{% endblock %}
{% block content %}
<h1>Books{% if status %} ({{ status }}){% endif %}</h1>
<ul>
{% for book in books %}
  <li><a href="{{ book.get_absolute_url }}">{{ book.title }}</a>
      by {{ book.author.name }} · {{ book.pages }} pages · {{ book.get_status_display }}</li>
{% empty %}
  <li>No books yet.</li>
{% endfor %}
</ul>
{% endblock %}
```

Rendering `/?status=reading` against our sample data produced:

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>Books</title></head>
<body>
  <nav><a href="/">All books</a></nav>
  <main>
<h1>Books (reading)</h1>
<ul>
  <li><a href="/books/3/">Exhalation</a> by Ted Chiang · 350 pages · Reading</li>
</ul>
</main>
</body>
</html>
```

(Blank lines trimmed.) Inheritance can go several levels deep: `base.html` →
`catalog/base.html` (adds a sidebar) → `catalog/book_list.html`. Inside a block,
`{{ block.super }}` inserts the parent's content for that block, which is handy for
appending to a `<head>` block rather than replacing it.

For fragments reused across pages (a book card, pagination controls) use
`{% include "catalog/_book_card.html" with book=b %}`. Django 6.0 added **template
partials** (`{% partialdef %}`), which let you define and reuse fragments inside one
file; they're covered in Level 4 · 06.

## Variables and the dot lookup

`{{ book.author.name }}` looks simple, but each dot tries several things in order:

1. dictionary lookup: `book["author"]`
2. attribute lookup: `book.author`
3. list-index lookup: `book[0]` style, for numeric parts like `items.0`

If the result is callable it's called with no arguments. That's how
`{{ book.get_status_display }}` and `{{ book.get_absolute_url }}` work: they're methods.
You can't pass arguments; if you need to, compute the value in the view or write a
custom filter (Level 4 · 06).

### Missing values fail silently

```django
{{ missing }}|{{ book.nope }}|{{ n|default:'none' }}
```

With `n = 0` and nothing else defined, this rendered `||none`. A misspelled variable
produces an empty string, not an error. That's convenient for optional values and
dangerous for typos. During development you can set `"string_if_invalid": "MISSING(%s)"`
in `OPTIONS` to make mistakes visible, but don't leave it on: the admin and some
third-party templates rely on the silent behaviour.

## Tags and filters you'll use daily

Tags (`{% %}`) control logic; filters (`|`) transform a value.

| Tags | Purpose |
|------|---------|
| `{% if %} {% elif %} {% else %}` | branching; supports `and`, `or`, `not`, `in`, comparisons |
| `{% for x in items %} ... {% empty %}` | loops; `forloop.counter`, `forloop.first`, `forloop.last` |
| `{% url 'name' arg %}` | reverse a URL |
| `{% csrf_token %}` | hidden CSRF field in POST forms (lesson 08) |
| `{% with total=items|length %}` | name an intermediate value |
| `{% load static %}` then `{% static 'path' %}` | static file URLs (lesson 09) |
| `{% comment %}` / `{# ... #}` | comments that never reach the browser |

We rendered a handful of filters in one template to see real output:

```django
{{ items|join:', ' }}|{{ text|truncatewords:3 }}|{{ 1234567|filesizeformat }}
```

with `items = ["a", "b"]` and `text = "one two three four five"` produced:

```text
a, b|one two three …|1.2 MB
```

Others worth knowing: `date:"Y-m-d"`, `pluralize`, `length`, `lower`/`upper`/`title`,
`linebreaksbr`, `urlize`, `default`, `yesno:"Yes,No"`, `floatformat:2`. The detail page
uses two of them:

```django
<p>{{ book.author }} · published {{ book.published|date:"Y" }} ·
   {{ book.pages }} page{{ book.pages|pluralize }}</p>
```

## Auto-escaping: the feature that prevents XSS

Every variable is HTML-escaped by default. We rendered the same malicious-looking string
three ways:

```django
{{ x }} | {{ x|safe }} | {% autoescape off %}{{ x }}{% endautoescape %}
```

with `x = "<b>bold</b> & 'quoted'"`:

```html
&lt;b&gt;bold&lt;/b&gt; &amp; &#x27;quoted&#x27; | <b>bold</b> & 'quoted' | <b>bold</b> & 'quoted'
```

The first is what the browser shows as literal text. The other two inject real HTML. If
`x` came from a user (a book review, a display name), the second and third forms are a
cross-site scripting hole: a review containing `<script>` would run in every reader's
browser.

Rules of thumb:

- Never use `|safe` or `autoescape off` on anything a user can influence.
- If you must produce HTML in Python, use `django.utils.html.format_html()`, which
  escapes the arguments and marks only your fixed markup as safe.
- Auto-escaping is for HTML **text and attribute** contexts. It does not make a value
  safe inside a `<script>` block or a JavaScript string. To pass data to JavaScript, use
  the `json_script` filter: `{{ data|json_script:"page-data" }}` renders a
  `<script type="application/json">` tag that's safe to parse.

## Context processors

Some variables appear in every template without the view passing them: `request`,
`user`, `perms` and `messages`. They come from the **context processors** listed in
`OPTIONS`. Each is a function that takes the request and returns a dict to merge into the
context. They only run when you render with a request, which `render(request, ...)` does.
You can write your own (for example, to put a site name into every page), but keep them
cheap: they run on every render.

## How It Actually Works

Rendering happens in two phases. First, `get_template()` finds the file and **compiles**
it: a lexer splits the text into tokens (text, `{{ var }}`, `{% tag %}`), and a parser
turns them into a tree of `Node` objects. `{% for %}` becomes a `ForNode` holding its
child nodes; `{{ book.title|upper }}` becomes a `VariableNode` holding a
`FilterExpression`. By default Django wraps its loaders in a **cached loader**, so each
template is compiled once per process (since Django 4.1 this is true even with `DEBUG`
on; the development server clears the cache when a template file changes).

Second, `render(context)` walks the tree. Each `VariableNode` resolves its dotted path
against the context (a stack of dicts, so `{% with %}` and `{% for %}` can push and pop
names), applies filters, and finally **escapes** the result unless it is a `SafeString`.
`|safe`, `mark_safe()` and `format_html()` all work by returning a `SafeString`, a `str`
subclass that the renderer trusts. Escaping isn't a separate pass over the HTML; it
happens per variable at output time, which is why literal markup in the template is
untouched and only interpolated values are escaped.

For inheritance, `{% extends %}` must be the first tag. The child's blocks are collected
into a `BlockContext`, the parent is rendered instead, and each `{% block %}` in the
parent asks the context for the most-derived override. `{{ block.super }}` renders the
next one up the chain.

## Common mistakes

- **Putting app templates directly in `<app>/templates/`** without the app-name folder,
  so they collide across apps.
- **Using `|safe` on user content** to "fix" escaped HTML. Store and render plain text,
  or sanitise with a dedicated library if users genuinely need formatting.
- **Querying in templates**: `{% for r in book.reviews.all %}` inside a loop over books
  runs one query per book. Fix it in the view with `prefetch_related` (Level 2 · 03).
- **Expecting an error for a typo'd variable.** It renders as nothing.
- **Writing content outside `{% block %}` in a child template**; it's silently dropped.

## Exercise

1. Create `templates/base.html` and `catalog/templates/catalog/book_list.html` as above,
   and make your list view render it.
2. Add a `{% block extra_head %}{% endblock %}` to the base and use `{{ block.super }}` in
   a child to append a stylesheet link.
3. Render `{{ x }}` and `{{ x|safe }}` with `x = "<script>alert(1)</script>"` in a test
   page. View the page source and explain the difference.
4. Use `json_script` to pass a list of book titles to a small inline script that logs
   them to the console.
