---
description: "Extend Django templates safely: custom filters, simple and inclusion tags, format_html escaping, a query-string helper, and Django 6 template partials."
---

# 06 · Custom Template Tags, Filters & Template Partials

Level 1 kept logic out of templates on purpose. But some presentation logic genuinely
belongs at the template layer and repeats across pages: rendering a rating as stars,
building pagination links that keep the current filters, rendering a book card the same
way everywhere. Custom tags and filters are the sanctioned way to add that, and Django
6.0's **template partials** remove a lot of tiny include files. This lesson writes each
kind, shows the output, and focuses on the part people get wrong: escaping.

## Where they live

```text
catalog/
└── templatetags/
    ├── __init__.py
    └── catalog_extras.py      # {% load catalog_extras %}
```

The module name is what templates `{% load %}`. Each template must load it; forgetting
produced, in our test:

```text
TemplateSyntaxError: Invalid filter: 'stars'
```

(After creating a new `templatetags` package, restart the development server; it's
registered when the app loads.)

## A filter

```python title="catalog/templatetags/catalog_extras.py"
from django import template

register = template.Library()


@register.filter
def stars(rating, out_of=5):
    """Render a 1–5 rating as filled and empty stars, e.g. ★★★☆☆."""
    try:
        rating = int(rating)
    except (TypeError, ValueError):
        return ""
    rating = max(0, min(rating, out_of))
    return "★" * rating + "☆" * (out_of - rating)
```

```django
{{ 4|stars }} | {{ 2|stars:10 }} | [{{ 'x'|stars }}]
```

```text
★★★★☆ | ★★☆☆☆☆☆☆☆☆ | []
```

Filters take one value and at most one argument. Make them **forgiving**: templates
render whatever data arrives, and a filter that raises turns a bad value into a 500 for
the whole page. Ours returns an empty string for non-numbers and clamps out-of-range
values.

## A simple tag with context: preserving the query string

Pagination and sorting links must keep the current filters (Level 2 · 04's warning). A
tag that edits the current query string:

```python
@register.simple_tag(takes_context=True)
def query_transform(context, **changes):
    """Return the current query string with some parameters changed."""
    params = context["request"].GET.copy()
    for key, value in changes.items():
        if value in (None, ""):
            params.pop(key, None)
        else:
            params[key] = value
    return params.urlencode()
```

On a request for `/books/?status=done&page=3&tag=sci-fi`:

```django
<a href="?{% query_transform page=4 %}">Next</a>
<a href="?{% query_transform page=None status='reading' %}">Reading</a>
```

rendered as:

```text
?status=done&amp;page=4&amp;tag=sci-fi
?status=reading&amp;tag=sci-fi
```

The `&amp;` is correct: `simple_tag` output is auto-escaped, and in an HTML attribute
`&amp;` is how a literal `&` should be written; the browser decodes it back. (It needs
the `request` context processor, which is on by default.)

## Building HTML safely: `format_html`

A tag that returns HTML must escape every piece of data in it. A first version of a
"tag links" helper:

```python
from urllib.parse import urlencode
from django.utils.html import format_html_join

@register.simple_tag
def tag_list(book):
    return format_html_join(", ", '<a href="?{}">{}</a>',
                            ((urlencode({"tag": t.name}), t.name) for t in book.tags.all()))
```

`format_html_join` escapes each argument and marks only the fixed markup safe. We tested
it with a hostile tag name, `<img src=x onerror=alert(1)>`:

```html
<a href="?tag=%3Cimg+src%3Dx+onerror%3Dalert%281%29%3E">&lt;img src=x onerror=alert(1)&gt;</a>
```

Harmless text. Our *first* draft built the link as `'?tag={}'` with the raw name; that
was still HTML-safe (escaped), but produced `href="?tag=short stories"` with an unencoded
space. HTML escaping and URL encoding are different jobs: `urlencode` for values in a
URL, `format_html` for values in HTML. Never build HTML with f-strings plus `mark_safe()`;
that's the XSS hole `format_html` exists to close.

## An inclusion tag: reusable components

```python
@register.inclusion_tag("catalog/_book_card.html")
def book_card(book, show_author=True):
    return {"book": book, "show_author": show_author}
```

```django title="catalog/templates/catalog/_book_card.html"
<article class="card"><h3>{{ book.title }}</h3>{% if show_author %}<p>{{ book.author.name }}</p>{% endif %}</article>
```

`{% book_card book show_author=False %}` rendered
`<article class="card"><h3>Exhalation</h3></article>`. An inclusion tag is a function
plus a template: it can compute or default values before rendering, which a plain
`{% include %}` can't. Watch for queries: `book.author.name` inside the card is a query
per card unless the caller used `select_related` (Level 2 · 03).

## Template partials (Django 6.0+)

Small fragments often get their own include file only so they can be reused or rendered
alone (for example, one table row returned to an htmx request). Django 6.0 builds in
**partials**, named fragments defined inside a template:

```django title="catalog/templates/catalog/partials_demo.html"
{% partialdef book-row %}<li id="book-{{ book.pk }}">{{ book.title }} <span class="status">{{ book.get_status_display }}</span></li>{% endpartialdef %}
<ul>
{% for book in books %}{% partial book-row %}
{% endfor %}
</ul>
```

Rendering the template with two books:

```html
<ul>
<li id="book-1">The Dispossessed <span class="status">Finished</span></li>
<li id="book-2">A Wizard of Earthsea <span class="status">Want to read</span></li>
</ul>
```

And the fragment **alone**, by appending `#name` to the template name:

```python
render_to_string("catalog/partials_demo.html#book-row", {"book": book})
# '<li id="book-1">The Dispossessed <span class="status">Finished</span></li>'
```

That works anywhere a template name is accepted, including `render(request,
"catalog/book_list.html#book-row", ...)` and a class-based view's `template_name`. A view
that updates one row with htmx or plain `fetch()` can return just that fragment, and the
full page and the fragment stay in one file, guaranteed to match. By default
`{% partialdef %}` only *defines* the fragment; add `inline` (`{% partialdef book-row
inline %}`) to also render it in place. On Django 5.x, the third-party
`django-template-partials` package (which this feature came from) provides the same tags.

## How It Actually Works

`{% load catalog_extras %}` makes the template engine import
`catalog.templatetags.catalog_extras` (it finds all `templatetags` packages of installed
apps at startup) and merge that module's `register` library into the parser for this
template. A filter is just a function the `FilterExpression` calls during rendering, with
flags (`is_safe`, `needs_autoescape`) that tell the engine whether the output may need
escaping. `simple_tag` generates a `Node` subclass that resolves its arguments against
the context at render time, calls your function, and **escapes the result unless it's a
`SafeString`**, which is why returning `format_html(...)` output keeps the markup while
returning a plain string gets escaped. `inclusion_tag` generates a node that renders the
named template with the dict your function returns.

Partials are parsed when the template is compiled: each `{% partialdef %}` body is stored
as a named nodelist on the template's origin. `{% partial name %}` renders that nodelist
with the current context. The loader understands `"name.html#fragment"` by loading
`name.html` and returning a template object wrapping just that nodelist.

## Common mistakes

- **Filters that raise on unexpected input**, turning bad data into a page-wide 500.
- **`mark_safe(f"<a href='{url}'>{name}</a>")`**: an XSS hole. Use `format_html`.
- **Confusing HTML escaping with URL encoding.**
- **Database queries inside tags** called in loops.
- **Business logic in template tags**; they're for presentation. Logic belongs in models,
  services or views.
- **Forgetting `{% load %}`** in each template (loads aren't inherited through `extends`).

## Exercise

1. Write the `stars` filter and use it on your reviews. Add `|stars` to a value of `None`
   and `"abc"` and confirm the page still renders.
2. Use `query_transform` to build pagination and sort links that preserve all other
   query parameters. Test with three parameters present.
3. Turn your book card into an inclusion tag with an optional `compact` argument.
4. Define the list row as a partial and add a view that returns only that row (using
   `"template.html#row"`) after a book's status changes.
