---
description: "Upgrade Django safely: the deprecation cycle, surfacing hidden RemovedInDjango warnings, rewrites with django-upgrade, and a step-by-step routine."
---

# 09 · Upgrading Django & Handling Deprecations

A Django project lives for years; any one Django version doesn't. Security fixes stop
arriving for old versions, Python versions age out, and third-party packages move on. The
projects that suffer are the ones that upgrade rarely and in big jumps. This lesson
explains Django's release and deprecation rules, shows how to see the warnings that
predict your next upgrade's breakages (they're hidden by default; we show why), and
walks through a routine that makes upgrades boring.

## How Django releases work

- **Feature releases** (x.y) arrive roughly every eight months: 5.1, 5.2, 6.0, 6.1...
- **Patch releases** (x.y.z) fix bugs and security issues. Always take these promptly;
  they're designed to be safe.
- Every **x.2** release is a **long-term support (LTS)** release with security fixes for
  about three years (5.2, then 6.2...). Other feature releases get support for a much
  shorter period.

Check the official "Supported versions" table on djangoproject.com for current dates.
Many teams move from LTS to LTS; others follow every feature release. Both are fine; what
isn't fine is running a version that no longer receives security fixes.

## The deprecation policy

Django rarely removes things abruptly. A feature is **deprecated** in one release, keeps
working with a warning for at least two feature releases, and is then **removed**. The
warning class names the removal version. In 6.1.1 we found:

```python
# django/utils/deprecation.py (6.1.1)
class RemovedInNextVersionWarning(DeprecationWarning): ...
class RemovedInDjango70Warning(PendingDeprecationWarning): ...
```

(`RemovedInNextVersionWarning` is a generic class for whatever the next version removes;
in 6.1.1 nothing in Django's own code was using it, since everything currently deprecated
targets 7.0.)

Two things follow:

1. If your code runs on the current version **without deprecation warnings**, it will very
   likely run on the next feature release.
2. These warnings are **hidden by default**. `PendingDeprecationWarning` and
   `DeprecationWarning` are ignored by Python unless triggered directly in `__main__`, so
   a project can accumulate dozens of them without anyone seeing.

## Seeing the warnings

We took the reading-list project and replaced 6.1's `MAILERS` block with the older
email settings (`EMAIL_HOST`, `EMAIL_PORT`), which 6.1 deprecates. A plain
`python manage.py check` printed only `System check identified no issues`. With warnings
enabled:

```text
$ python -W default manage.py check
manage.py:18: RemovedInDjango70Warning: The EMAIL_HOST setting is deprecated. Migrate to MAILERS before Django 7.0.
manage.py:18: RemovedInDjango70Warning: The EMAIL_PORT setting is deprecated. Migrate to MAILERS before Django 7.0.
System check identified no issues (0 silenced).
```

Code paths only warn when they run, so the test suite is the best place to look:

```bash
python -W error::DeprecationWarning -W error::PendingDeprecationWarning manage.py test
```

turns every deprecation into a failing test. `-W error` on its own also stopped `check`
at the first one:

```text
django.utils.deprecation.RemovedInDjango70Warning: The EMAIL_PORT setting is deprecated. ...
```

A call site we tested in a script, `send_mail("Hi", "Body", "a@example.com",
["b@example.com"], True)`, produced two warnings at once:

```text
RemovedInDjango70Warning: Passing positional argument(s) 'fail_silently' to send_mail() is
deprecated. Use keyword arguments instead.
RemovedInDjango70Warning: The 'fail_silently' argument is deprecated. See 'Migrating email
to mailers' in Django's documentation for recommended replacements.
```

Read deprecation messages carefully: they usually name the replacement.

## Automated rewrites: django-upgrade

Many deprecations are mechanical renames. The third-party **django-upgrade** tool (1.32.0
when we tested) rewrites source files for a target version:

```bash
python -m pip install django-upgrade
django-upgrade --target-version 6.1 $(git ls-files '*.py')
```

On a file written in older styles it made two changes:

```diff
-    send_mail("Hi", "Body", "from@example.com", [user_email], True)
+    send_mail("Hi", "Body", "from@example.com", [user_email], fail_silently=True)
...
-        constraints = [models.CheckConstraint(check=Q(stars__gte=1), name="stars_positive")]
+        constraints = [models.CheckConstraint(condition=Q(stars__gte=1), name="stars_positive")]
```

The first fixes the positional-argument warning (the `fail_silently` deprecation itself
still needs a human decision about mailers); the second applies 5.1's rename of
`check=` to `condition=`. It doesn't catch everything, and it can't make design
decisions, but it removes the tedious part. Review its diff like any other change, and
run it in a pre-commit hook to stop old patterns returning.

## A routine for each upgrade

1. **Read the release notes** for every version you're crossing, especially
   "Backwards incompatible changes" and "Features removed". Skipping versions doesn't skip
   their notes.
2. **Check third-party packages** support the target version (their changelogs, or try
   installing). An abandoned package is the most common upgrade blocker; find
   replacements early.
3. **Get to zero deprecation warnings on your current version** with the `-W error`
   test run above, plus `django-upgrade`.
4. **Upgrade one feature release at a time** when crossing several (5.2 → 6.0 → 6.1),
   running the tests at each step. Each step's warnings tell you about the next.
5. **Run `makemigrations --check`**: some releases change field defaults or
   introspection, producing new migrations for Django's own or your apps. Read them.
6. **Run `check --deploy`** with production settings; new checks appear in new releases
   (6.1 added the `MAILERS` development-backend error we met in lesson 01).
7. **Deploy to staging, then production**, with the usual rollback plan. A Django upgrade
   that includes migrations is a schema change: apply expand/contract thinking (Level 3 · 06).

Pin versions in requirements (`Django==6.1.1`, or `~=6.1.0` to accept patches only) so
upgrades happen when you choose. Let a dependency bot open pull requests for patch
releases so security fixes arrive within days.

## Python versions

Each Django release supports a range of Python versions; Django 6.x requires Python 3.12
or newer (the package metadata says `Requires-Python: >=3.12`). Upgrading Python is a
separate step with its own changes; avoid upgrading Python and Django in the same
deploy, so you know which one broke something.

## How It Actually Works

Python's `warnings` module decides what to show using filters. The defaults ignore
`DeprecationWarning` (except in `__main__`) and `PendingDeprecationWarning`, which is why
Django's `RemovedInDjango70Warning` is invisible in a normal run. `-W default` shows each
unique warning once per location; `-W error` raises it as an exception, which makes it
impossible to miss and gives you a traceback to the call site. As a removal approaches,
Django moves deprecations from the `PendingDeprecationWarning`-based class to the
`DeprecationWarning`-based one, which tools and test runners are more likely to surface.

Deprecated settings are detected when settings are loaded: Django compares the names
you defined against a list of deprecated settings and warns for each, which is why the
email warnings appeared before any command ran. Deprecated call signatures use helpers
like the `deprecate_posargs` decorator we saw in `django.core.mail`, which inspects the
arguments of each call and warns only for the old style.

## Common mistakes

- **Never looking at deprecation warnings** until the version that removes the feature.
- **Jumping several versions in one step** without reading each release's notes.
- **Upgrading Django and Python together.**
- **Unpinned dependencies**, so an upgrade happens by accident during an unrelated deploy.
- **Ignoring patch releases**, which carry security fixes.
- **Waiting for a blocking third-party package** instead of planning its replacement.

## Exercise

1. Run your test suite with `-W error::DeprecationWarning -W error::PendingDeprecationWarning`
   and fix every warning (or record why it can't be fixed yet).
2. Run `django-upgrade --target-version` for your current version over your project and
   review the diff.
3. Add a CI job that runs the tests with warnings as errors, so new deprecated usage
   can't be merged.
4. Write an upgrade checklist for your project's next Django release, including which
   third-party packages need checking and who reads which release notes.
