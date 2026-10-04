---
description: "Handle user input safely with Django forms: the GET/POST view pattern, field and form-level validation, ModelForms, CSRF protection, and rendering errors."
---

# 08 · Forms, ModelForms & CSRF

Every piece of user input is untrusted: it might be missing, the wrong type, too long,
or deliberately malicious. Django's forms do the unglamorous work of turning raw
strings from `request.POST` into validated Python values, and re-displaying the form with
helpful errors when they're not valid. This lesson builds the "add a book" page and
explains each layer of validation, plus the CSRF protection that guards every POST.

## A plain form

A `Form` class declares fields; each field knows how to **clean** a raw string into a
Python value and which **widget** renders it as HTML.

```python
from django import forms

class ContactForm(forms.Form):
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea, max_length=500)
```

Bind it to data and validate:

```python
>>> cf = ContactForm({"email": "not-an-email", "message": ""})
>>> cf.is_valid()
False
>>> dict(cf.errors)
{'email': ['Enter a valid email address.'], 'message': ['This field is required.']}
```

Rendering it (`cf.as_div()`, or just `{{ form }}` in a template) produced, among other
markup, `<input type="email" name="email" maxlength="320" required id="id_email">` and a
`<textarea ... maxlength="500" required>`. The field definitions turned into browser-side
hints (`type="email"`, `required`, `maxlength`) as well as server-side rules. The
browser hints are a convenience; the server-side validation is the one you can rely on,
because anyone can send a POST without your HTML.

## ModelForm: forms generated from models

Most forms create or edit a model. A `ModelForm` builds its fields from the model, so
`max_length`, `choices`, `blank` and foreign keys become form rules automatically:

```python title="catalog/forms.py"
from django import forms
from .models import Book


class BookForm(forms.ModelForm):
    class Meta:
        model = Book
        fields = ["title", "author", "pages", "status", "published"]

    def clean_title(self):
        title = self.cleaned_data["title"].strip()
        if title.isupper():
            raise forms.ValidationError("Please don't shout: use normal capitalisation.")
        return title

    def clean(self):
        cleaned = super().clean()
        if cleaned.get("status") == Book.Status.DONE and not cleaned.get("pages"):
            raise forms.ValidationError("A finished book needs a page count.")
        return cleaned
```

Always list `fields` explicitly. `fields = "__all__"` is allowed, but it means any field
added to the model later, such as an `is_featured` flag, is instantly editable by anyone
who can submit the form. That's the **mass assignment** problem.

### What the generated validation catches

We submitted junk to a bound `BookForm`:

```python
>>> f = BookForm(data={"title": "x" * 201, "author": "9", "pages": "abc", "status": "maybe"})
>>> f.is_valid(); f.errors.get_json_data()
{'title': [{'message': 'Ensure this value has at most 200 characters (it has 201).', 'code': 'max_length'}],
 'author': [{'message': 'Select a valid choice. That choice is not one of the available choices.', 'code': 'invalid_choice'}],
 'pages': [{'message': 'Enter a whole number.', 'code': 'invalid'}],
 'status': [{'message': 'Select a valid choice. maybe is not one of the available choices.', 'code': 'invalid_choice'}]}
```

None of those rules were written by hand. The author ID 9 doesn't exist, so the
`ModelChoiceField` rejected it. That's a check against the database, not just the format.

And valid data comes back as **Python types**, not strings:

```python
>>> f = BookForm(data={"title": "  Lathe  ", "author": "1", "pages": "184", "status": "want"})
>>> f.is_valid(), f.cleaned_data["title"], f.cleaned_data["author"], type(f.cleaned_data["pages"])
(True, 'Lathe', <Author: Ursula K. Le Guin>, <class 'int'>)
```

The title was stripped (`CharField` strips whitespace by default), the author became an
`Author` instance, and pages became an `int`.

## The order validation runs in

For each field, in declaration order:

1. `field.clean(raw)` — type conversion and the field's built-in validators
   (`to_python`, `validate`, `run_validators`).
2. `form.clean_<fieldname>()` — your per-field method, if defined, only if step 1 passed.

Then, once for the whole form:

3. `form.clean()` — cross-field rules. It runs **even if some fields failed**, so use
   `cleaned_data.get()`, never `cleaned_data["x"]`.
4. For a `ModelForm`, `_post_clean()` then builds the model instance and calls the
   model's own validation (`full_clean()`), which checks `unique` fields and model-level
   constraints.

We saw step 3's behaviour directly. Posting `{"title": "DUNE", "author": "", "pages":
"-3", "status": "done"}` produced four errors at once:

```python
{'title': ["Please don't shout: use normal capitalisation."],
 'author': ['This field is required.'],
 'pages': ['Ensure this value is greater than or equal to 0.'],
 '__all__': ['A finished book needs a page count.']}
```

`pages` failed its own validation, so it wasn't in `cleaned_data`; `clean()` still ran,
`cleaned.get("pages")` returned `None`, and the form-wide rule fired too. Errors raised
from `clean()` are stored under `__all__` and shown by `{{ form.non_field_errors }}`.

## The view pattern

```python title="catalog/views.py"
from django.shortcuts import redirect, render
from .forms import BookForm

def book_create(request):
    if request.method == "POST":
        form = BookForm(request.POST)
        if form.is_valid():
            book = form.save()
            return redirect(book)
    else:
        form = BookForm()
    return render(request, "catalog/book_form.html", {"form": form})
```

Three paths: a GET shows an empty form; an invalid POST falls through and re-renders the
*bound* form with errors and the user's input preserved; a valid POST saves and
**redirects**. A valid POST through the test client returned `302` with
`Location: /books/6/`.

The redirect matters. If you rendered a success page directly from the POST, refreshing
the browser would resubmit the form and create a duplicate. Redirecting after a
successful POST (the Post/Redirect/Get pattern) makes refresh harmless.

To edit an existing object, pass `instance=`:
`BookForm(request.POST, instance=book)`; `save()` then updates instead of inserting.
`form.save(commit=False)` returns the unsaved instance so you can set fields the user
doesn't control (like `owner = request.user`) before saving.

## The template and CSRF

```django title="catalog/templates/catalog/book_form.html"
{% extends "base.html" %}
{% block content %}
<form method="post">
  {% csrf_token %}
  {{ form.as_div }}
  <button type="submit">Save</button>
</form>
{% endblock %}
```

`{% csrf_token %}` is not optional. With CSRF checks enforced (Django's test client
skips them unless you ask), posting without the token returned **403** with Django's
"CSRF verification failed" page. Fetching the form first set a `csrftoken` cookie and
embedded a `csrfmiddlewaretoken` hidden field; posting that back succeeded with a 302.

**Cross-site request forgery** is an attack where another website makes your browser
submit a form to a site where you're logged in: the browser attaches your session cookie
automatically, so without protection the request looks legitimate. The token proves the
form came from a page your site rendered.

For full control over markup, render fields individually:

```django
{% for field in form %}
  <div class="field{% if field.errors %} has-error{% endif %}">
    {{ field.label_tag }} {{ field }}
    {% if field.help_text %}<small>{{ field.help_text }}</small>{% endif %}
    {{ field.errors }}
  </div>
{% endfor %}
{{ form.non_field_errors }}
```

## How It Actually Works

A form is either **unbound** (no data; renders empty) or **bound** (`Form(data)`).
Calling `is_valid()` or touching `form.errors` triggers `full_clean()` once and caches
the result. `full_clean()` loops over `self.fields`, takes each raw value from the data
via the widget's `value_from_datadict()` (that's how a checkbox's absence becomes
`False`, and a multi-select collects a list), and runs the cleaning chain above. Each
`ValidationError` is caught and filed under the field name, so one bad field doesn't
stop the others from being checked: users see every problem at once.

CSRF protection is implemented by `CsrfViewMiddleware`. On a GET that renders
`{% csrf_token %}`, it ensures a secret is stored in the `csrftoken` cookie and puts a
**masked** version of it in the form (re-masked on each render, so the token in the page
changes even though the secret doesn't, which defends against compression-based attacks
like BREACH). On any unsafe method (POST, PUT, PATCH, DELETE), the middleware unmasks the
submitted token and compares it to the cookie secret in constant time. For HTTPS requests
it also checks the `Origin` header (or falls back to `Referer`) against the site's host
and `CSRF_TRUSTED_ORIGINS`. A cross-site attacker can make the browser send the cookie,
but can't read it, so can't produce a matching token.

## Common mistakes

- **Using `request.POST["field"]` directly** instead of a form, skipping validation.
- **`fields = "__all__"`** on user-facing ModelForms.
- **`cleaned_data["x"]` in `clean()`** raising `KeyError` when that field failed.
- **Rendering a page directly after a successful POST** instead of redirecting.
- **Adding `@csrf_exempt` to make an error go away.** Fix the template or the JavaScript
  header (`X-CSRFToken`) instead.
- **Trusting `required` and `maxlength` in HTML** as validation.

## Exercise

1. Build `BookForm` and the `book_create` view, and add a link to it from the list page.
2. Add a `clean_pages()` method that rejects books over 5,000 pages.
3. Add a form-level rule: a book can't be "finished" with a `published` date in the
   future. Test it with the shell, the way the examples above did.
4. Write `book_update(request, pk)` that reuses the same form and template with
   `instance=`.
5. Remove `{% csrf_token %}` from the template and submit the form in a browser. Read the
   403 page, then restore it.
