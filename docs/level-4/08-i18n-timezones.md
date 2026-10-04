---
description: "Time zones and translation in Django: aware datetimes, per-user zones, 'today' boundaries, gettext, plurals and language negotiation."
---

# 08 · Time Zones & Internationalization

Two features that look optional until the first user in another country signs up. Time
zones decide whether "due today" means the same day for a user in Kolkata and the server
in UTC. Internationalization decides whether the app can speak more than one language
without a rewrite. Both are cheap to get right at the start and painful to retrofit. This
lesson runs both on Django 6.1.1, including one translation bug that came from the
gettext tools themselves.

## Time zones: store UTC, display local

New projects have `USE_TZ = True` (the default since Django 5.0). With it:

- datetimes in the database are stored in **UTC**;
- Python datetimes are **aware** (they carry a `tzinfo`);
- conversion to the user's zone happens at the edges: templates, forms, and wherever you
  call `timezone.localtime()`.

```python
>>> from django.utils import timezone
>>> now = timezone.now(); now, now.tzinfo
(datetime.datetime(2026, 10, 2, 8, 58, 26, 965717, tzinfo=datetime.timezone.utc), datetime.timezone.utc)
>>> import datetime as dt; dt.datetime.now().tzinfo
None
```

`datetime.now()` is **naive**: a wall-clock time with no zone, meaning nothing once it
crosses a process or a border. Always use `timezone.now()`. Django warns when naive values
reach the database:

```python
Book.objects.filter(created__gte=dt.datetime(2026, 10, 1, 9, 0)).count()
```

```text
RuntimeWarning: DateTimeField Book.created received a naive datetime (2026-10-01 09:00:00)
while time zone support is active.
```

It still runs (interpreting the value in `TIME_ZONE`), which is how off-by-hours bugs get
in. Turn warnings into errors in tests so they can't slip through:
`python -W error::RuntimeWarning manage.py test`.

### Showing local time

Activate the user's zone (from their profile, typically in a middleware) and everything
renders in it:

```python
import zoneinfo
timezone.activate(zoneinfo.ZoneInfo("Asia/Kolkata"))
```

```text
stored:    2026-10-02 08:10:33.243137+00:00
localtime: 2026-10-02 13:40:33.243137+05:30
```

In a template, `{{ d|date:'Y-m-d H:i T' }}` rendered `2026-10-02 13:40 IST`, and with
`{% load tz %}`, `{{ d|utc|date:'H:i T' }}` rendered `08:10 UTC`. Same instant, two
displays. A small middleware completes it:

```python
class UserTimezoneMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tzname = getattr(request.user, "timezone", None) if request.user.is_authenticated else None
        if tzname:
            timezone.activate(zoneinfo.ZoneInfo(tzname))
        else:
            timezone.deactivate()
        return self.get_response(request)
```

Store zone **names** (`"Asia/Kolkata"`, `"America/New_York"`), never offsets like
`+05:30`: offsets change with daylight saving time, names don't.

### "Today" is a local concept

"Tasks due today" depends on where the user is. Midnight in Kolkata is:

```text
today in Kolkata starts at 2026-10-01 18:30:00+00:00
```

18:30 UTC *the previous day*. Use `timezone.localdate()` (today in the active zone) rather
than `date.today()` or `timezone.now().date()` (both effectively today in UTC on a UTC
server). Django's `__date`, `__year` and similar lookups convert to the **current active
time zone** in the database query, so with the user's zone activated,
`filter(created__date=timezone.localdate())` means their today.

### Daylight saving gaps

Some local times don't exist. On 8 March 2026, clocks in New York jump from 02:00 to 03:00.
Python accepted `02:30` there without complaint:

```text
nonexistent 02:30 NY -> 2026-03-08T02:30:00-05:00 -> 2026-03-08T03:30:00-04:00
```

and round-tripping through UTC turned it into 03:30. Scheduling features (recurring
meetings, reminders at "9am local") must decide what happens in gaps and repeated hours;
store the user's intended local time plus zone name, and compute the UTC instant when
needed.

## Internationalization (i18n)

Marking strings for translation, then translating them per language. In Python code:

```python title="books/i18n_views.py"
from django.http import HttpResponse
from django.utils.translation import gettext as _, ngettext

from .models import Entry


def summary(request):
    n = Entry.objects.count()
    text = ngettext("You have %(count)d book on your list.",
                    "You have %(count)d books on your list.", n) % {"count": n}
    return HttpResponse(f"{_('Reading list')}: {text}")
```

- `gettext` (conventionally `_`) for plain strings.
- `ngettext` for plurals: languages differ in how many plural forms they have.
- `gettext_lazy` for strings defined at import time (model field `verbose_name`,
  `help_text`, form labels, settings), so the translation happens when the string is
  used, in the active language.
- Use **named placeholders** (`%(count)d`) so translators can reorder words.

In templates: `{% load i18n %}`, then `{% translate "Reading list" %}` and
`{% blocktranslate count counter=n %}...{% plural %}...{% endblocktranslate %}`.

### Settings and middleware

```python title="config/settings.py"
from django.utils.translation import gettext_lazy as _

LANGUAGES = [("en", _("English")), ("fr", _("French"))]
LOCALE_PATHS = [BASE_DIR / "locale"]

MIDDLEWARE = [
    # ...
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.locale.LocaleMiddleware",   # after sessions, before CommonMiddleware
    "django.middleware.common.CommonMiddleware",
    # ...
]
```

### Extract, translate, compile

```bash
python manage.py makemessages -l fr      # needs GNU gettext installed
# edit locale/fr/LC_MESSAGES/django.po
python manage.py compilemessages
```

`makemessages` wrote entries like:

```po
msgid "Reading list"
msgstr ""

msgid "You have %(count)d book on your list."
msgid_plural "You have %(count)d books on your list."
msgstr[0] ""
msgstr[1] ""
```

We filled in `msgstr[0]` and `msgstr[1]`, compiled, and tested with different
`Accept-Language` headers:

```text
'en'             -> Reading list: You have 1 book on your list.            | Content-Language: en
'fr'             -> Liste de lecture: Vous avez 1 livre dans votre liste.  | Content-Language: fr
'fr-CA,fr;q=0.9' -> Liste de lecture: Vous avez 1 livre dans votre liste.  | Content-Language: fr
'de'             -> Reading list: You have 1 book on your list.            | Content-Language: en
```

French was chosen for French and Canadian French browsers; German, which we don't offer,
fell back to `LANGUAGE_CODE`. Then we added a second book and asked again in French:

```text
Liste de lecture: You have 2 books on your list.
```

**English, in the middle of a French page.** The `.po` header generated by the installed
GNU gettext (version 1.0) declared *three* plural forms for French:

```text
"Plural-Forms: nplurals=3; plural=(n == 0 || n == 1) ? 0 : n != 0 && n % "
"1000000 == 0 ? 1 : 2;\n"
```

(newer gettext data distinguishes French's form for large round numbers like "un million
de livres"). For `n = 2` the rule selects form 2, which we hadn't translated, so gettext
fell back to the English source. Adding `msgstr[2]` fixed it:

```text
0 Vous avez 0 livre dans votre liste.
1 Vous avez 1 livre dans votre liste.
2 Vous avez 2 livres dans votre liste.
1000000 Vous avez 1000000 livres dans votre liste.
```

Lessons: always fill **every** `msgstr[n]` the header declares, have translation tools
(or `msgfmt --check`) validate files in CI, and test plurals with 0, 1, 2 and a large
number. Older gettext versions declare two forms for French; the header in *your* file is
what counts.

### URLs and language choice

`i18n_patterns()` in the URLconf prefixes URLs with the language (`/fr/books/`), which is
better for search engines and sharing than relying on headers. `set_language` (a
built-in view) lets users switch explicitly; their choice is stored in a cookie and wins
over `Accept-Language`.

## How It Actually Works

With `USE_TZ`, the database layer converts aware datetimes to UTC on the way in and
returns aware UTC datetimes on the way out. The **active time zone** is stored in a
context variable (per thread or async task) by `timezone.activate()`. Template
rendering of datetimes calls `localtime()` with it, and the ORM uses it when compiling
lookups like `__date` (for example, PostgreSQL's `AT TIME ZONE`), which is why the active
zone changes query results.

Translation uses GNU gettext catalogs: `compilemessages` turns `.po` text files into
binary `.mo` files. At runtime, `gettext()` looks up the active language (also a context
variable, set by `LocaleMiddleware` from the URL prefix, the language cookie,
`Accept-Language` or `LANGUAGE_CODE`, in that order) and finds the message in that
language's catalogs, searching `LOCALE_PATHS`, then each app's `locale/` directory, then
Django's own. For `ngettext`, the catalog's `Plural-Forms` expression is evaluated with
`n` to pick the index; a missing or empty translation at that index falls back to the
source string.

## Common mistakes

- **`datetime.now()` or `date.today()`** in application code.
- **Storing offsets instead of zone names.**
- **Computing "today" in UTC** for user-facing features.
- **Building sentences by concatenating translated fragments**; translate whole sentences
  with placeholders.
- **`gettext` (non-lazy) at import time**, freezing strings in the server's default language.
- **Incomplete plural translations**, silently falling back to English.
- **Forgetting `compilemessages`** in the build, so translations never load.

## Exercise

1. Add a `timezone` field to your user model and the middleware above. Show each task's
   due time in the user's zone, and test with two users in different zones.
2. Write a "due today" query that's correct for a user in `Pacific/Auckland` and one in
   `America/Los_Angeles` at the same instant.
3. Run your tests with `-W error::RuntimeWarning` and fix any naive datetimes.
4. Translate your reading list into one language you know (or French with a dictionary),
   including a plural message. Test 0, 1, 2 and 1,000,000.
5. Add `i18n_patterns` and a language switcher using `set_language`.
