---
description: "Serve CSS, JS and images with staticfiles, collectstatic and hashed names, and accept user uploads safely with FileField, MEDIA_ROOT and validators."
---

# 09 · Static Files & User Uploads

Django draws a hard line between two kinds of files:

- **Static files** are part of your code: CSS, JavaScript, logos, fonts. You write them,
  they're versioned in git, and they change only when you deploy.
- **Media files** are uploaded by users at runtime: avatars, book covers, attachments.
  They're data, they're never in git, and you can't trust them.

They have separate settings, separate storage and separate security concerns. Mixing
them up is one of the classic Django deployment mistakes.

## Static files in development

`django.contrib.staticfiles` (installed by default) collects static files from each app's
`static/` folder. As with templates, namespace them by app name:

```text
catalog/
└── static/
    └── catalog/
        └── site.css
```

In a template:

```django
{% load static %}
<link rel="stylesheet" href="{% static 'catalog/site.css' %}">
```

With `DEBUG = True`, `runserver` serves these automatically. To check where Django finds
a file, use `findstatic`:

```text
$ python manage.py findstatic catalog/site.css
Found 'catalog/site.css' here:
  /…/shelf/catalog/static/catalog/site.css
```

Files that don't belong to any one app (a site-wide logo) go in a project folder listed
in `STATICFILES_DIRS = [BASE_DIR / "static"]`.

## Static files in production

In production you don't want Python serving every CSS request out of a dozen app
folders. Instead, `collectstatic` copies every static file from every app (including the
admin's) into one directory, `STATIC_ROOT`, which a web server, a CDN or WhiteNoise
serves directly:

```python title="config/settings.py"
STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
```

```text
$ python manage.py collectstatic --noinput
131 static files copied to '/…/shelf/staticfiles'.
```

131, not 1: almost all of them belong to the admin.

### Cache-busting with hashed filenames

Browsers cache CSS aggressively. If `site.css` changes but keeps its name, returning
visitors can see stale styles for days. The fix is to put a content hash in the file
name, and Django's `ManifestStaticFilesStorage` does exactly that:

```python title="config/settings.py"
STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.ManifestStaticFilesStorage"},
}
```

After `collectstatic` with that setting and `DEBUG = False`, our template tag
`{% static 'catalog/site.css' %}` rendered:

```text
/static/catalog/site.c2819c21bcc3.css
```

and `staticfiles/catalog/` contained both `site.css` and `site.c2819c21bcc3.css`. The
hash changes whenever the file's content changes, so you can tell browsers and CDNs to
cache these URLs forever. The storage also rewrites `url(...)` references inside CSS to
the hashed names.

The strictness has a cost worth knowing about before your first deploy. We referenced a
file that didn't exist:

```text
ValueError: Missing staticfiles manifest entry for 'catalog/missing.css'
```

With the manifest storage, a typo in a `{% static %}` path is a **500 error** in
production, not a broken link. Run `collectstatic` in CI and render your main pages in a
test with `DEBUG = False` to catch it early. Level 4 · 03 adds WhiteNoise, which serves
these files efficiently from the Django process itself.

## Media: user uploads

```python title="config/settings.py"
MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"
```

A model field stores the **path** in the database and the file in storage:

```python title="catalog/models.py"
from django.core.validators import FileExtensionValidator

class Cover(models.Model):
    book = models.OneToOneField(Book, on_delete=models.CASCADE)
    image = models.FileField(
        upload_to="covers/%Y/",
        validators=[FileExtensionValidator(["jpg", "png"])],
    )
```

We saved two covers, both uploaded as `front.png`:

```python
>>> c1.image.name, c1.image.url, c1.image.size
('covers/2026/front.png', '/media/covers/2026/front.png', 9)
>>> c2.image.name
'covers/2026/front_J8OaAe3.png'
```

`upload_to` supports `strftime` codes, which keeps any one directory from holding
millions of files. The second upload didn't overwrite the first: the storage noticed the
name was taken and appended a random suffix.

`ImageField` works the same way but also verifies the upload is a real image and can
record its dimensions. It requires the **Pillow** library (`pip install pillow`); without
it, Django's system checks stop the project as soon as a model uses `ImageField`:

```text
catalog.Cover.photo: (fields.E210) Cannot use ImageField because Pillow is not installed.
	HINT: Get Pillow at https://pypi.org/project/Pillow/ or run command "python -m pip install Pillow".
```

### Upload forms

A form that accepts files needs two things, or `request.FILES` will be empty:

```django
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {{ form.as_div }}
  <button>Upload</button>
</form>
```

```python
form = CoverForm(request.POST, request.FILES)
```

The validator ran when we tried to upload `evil.html`:

```text
File extension “html” is not allowed. Allowed extensions are: jpg, png.
```

### Serving media during development

`runserver` does **not** serve media automatically. Add this to the root URLconf, which
only takes effect when `DEBUG = True`:

```python title="config/urls.py"
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

In production, media should be served by the web server or, more commonly, stored in
object storage (S3, GCS, Azure Blob) through a storage backend such as the
`django-storages` package. Configuring a cloud bucket can't be demonstrated without an
account, so this course doesn't show output for it; the pattern is changing
`STORAGES["default"]` to the provider's backend class.

## Upload security

User uploads are one of the riskiest features you can add:

- **Extension checks are a first filter, not a guarantee.** A file named `cover.png` can
  contain anything. `ImageField` (via Pillow) at least checks that it parses as an image.
- **Never serve uploads from the same origin with a type the browser will execute.** An
  uploaded `.html` or `.svg` served from your domain can run JavaScript with your users'
  cookies. Serving media from a separate domain (or object storage) removes that risk.
- **Limit size.** `DATA_UPLOAD_MAX_MEMORY_SIZE` limits non-file request bodies, and
  `FILE_UPLOAD_MAX_MEMORY_SIZE` only controls when uploads spill from memory to a temp
  file. Neither caps file size. Enforce a size limit in your form's `clean_<field>()`
  and in your web server (`client_max_body_size` in Nginx).
- **Don't trust the uploaded file name** for anything other than display. Django's
  storage already sanitises it; don't build paths from it yourself.

## How It Actually Works

Both kinds of files go through **storage backends**, configured in `STORAGES`. A storage
is an object with methods like `save(name, content)`, `url(name)`, `exists(name)` and
`delete(name)`. `FileSystemStorage` writes under `MEDIA_ROOT`; cloud backends implement
the same methods against an API. A `FileField` stores only the name string; accessing
`book.cover.image.url` asks the storage to build a URL from that name. That indirection
is why moving media to S3 is a settings change, not a code change.

On save, the storage calls `get_available_name()`, which checks `exists()` and appends
a random seven-character suffix until the name is free. That's the `_J8OaAe3` we saw.

Deleting a model instance does **not** delete its file. We deleted `c1` and the file was
still on disk afterwards. Django stopped doing it automatically years ago because a
rolled-back transaction could otherwise delete a file whose row still exists. Clean up
orphaned files with a periodic job (Level 4 · 05 shows a management command for this
kind of task).

For static files, `{% static %}` asks the `staticfiles` storage for a URL. The manifest
storage looks the name up in `staticfiles.json`, a mapping written by `collectstatic`,
and raises if it's missing; that's the error we saw.

## Common mistakes

- **Putting uploads under `STATIC_ROOT`**, or static files under `MEDIA_ROOT`.
  `collectstatic` may overwrite or ignore them.
- **Forgetting `enctype="multipart/form-data"`** and wondering why `request.FILES` is
  empty.
- **Forgetting to pass `request.FILES`** to the form.
- **Running without `collectstatic` in production** and getting an unstyled admin.
- **Committing `media/` to git**, including users' uploads.
- **Assuming deleting the row deletes the file.**

## Exercise

1. Add `catalog/static/catalog/site.css`, link it from `base.html` with `{% static %}`,
   and confirm with `findstatic`.
2. Configure `ManifestStaticFilesStorage`, run `collectstatic`, set `DEBUG = False`
   (and `ALLOWED_HOSTS = ["127.0.0.1"]`), and view the page source to find the hashed
   CSS name. Change the CSS and run `collectstatic` again: what happens to the hash?
3. Add a `Cover` model with a `FileField`, an upload form, and a page that shows the
   cover. Add a `clean_image()` that rejects files over 2 MB.
4. Upload two files with the same name and inspect the stored names.
