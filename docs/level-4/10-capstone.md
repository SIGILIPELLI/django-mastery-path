---
description: "Capstone: a production-ready Django book club app with roles, per-user time zones, idempotent reminder tasks, CSP, health checks and tests."
---

# 10 · Capstone — A Production-Ready Book Club App

The capstone puts the whole course into one application that's ready to run for real
people: a book club organiser. Clubs have hosts and members; hosts schedule meetings;
members RSVP; everyone sees meeting times in their own time zone; reminders go out once,
and only to people who haven't declined. Around the features sits everything from Level
4: environment-driven settings, a content security policy, logging with request IDs,
health checks, a management command for scheduling, a clean `check --deploy`, and tests.

We built it on Django 6.1.1, ran its 10 tests, and served it with Gunicorn against
PostgreSQL 16. The results are at the end of each section. Treat the code here as the
reference solution, and build your own before reading it in full.

## Requirements

1. Custom user with a unique email and a time zone (Level 2 · 07, Level 4 · 08).
2. Clubs with memberships; roles **host** and **member**. Non-members get 404s.
3. Hosts schedule meetings (book, time, place). Times are entered and shown in each user's
   own zone.
4. Members RSVP yes / maybe / no, changeable until the meeting starts, one answer per
   person (enforced by the database).
5. A reminder for each meeting goes to every member who hasn't said no, **exactly once**,
   even if the job runs twice.
6. A scheduled command enqueues reminders for meetings in the next N hours, with a dry run.
7. Production settings from the environment; `check --deploy --fail-level WARNING` passes.
8. CSP, request IDs in logs, liveness and readiness endpoints.
9. The club page uses a constant number of queries.

## Project layout

```text
bookclub/
├── accounts/          # User model, time-zone middleware, admin
├── clubs/             # models, views, forms, tasks, command, templates, tests
│   ├── management/commands/send_meeting_reminders.py
│   ├── tasks.py
│   └── tests/test_clubs.py
├── config/
│   ├── settings.py        # production settings, all from the environment
│   ├── settings_test.py   # test overrides
│   └── request_id.py      # middleware + logging filter (Level 4 · 07)
└── templates/
```

## Users and time zones

```python title="accounts/models.py"
import zoneinfo

from django.contrib.auth.models import AbstractUser
from django.db import models

TIMEZONE_CHOICES = [(tz, tz) for tz in sorted(zoneinfo.available_timezones()) if "/" in tz] + [("UTC", "UTC")]


class User(AbstractUser):
    email = models.EmailField("email address", unique=True)
    timezone = models.CharField(max_length=64, choices=TIMEZONE_CHOICES, default="UTC")
```

```python title="accounts/middleware.py"
import zoneinfo

from django.utils import timezone


class UserTimezoneMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        user = getattr(request, "user", None)
        if user is not None and user.is_authenticated:
            timezone.activate(zoneinfo.ZoneInfo(user.timezone))
        else:
            timezone.deactivate()
        return self.get_response(request)
```

It sits right after `AuthenticationMiddleware`. Because `MeetingForm` uses Django's form
handling with time-zone support on, a host in Kolkata typing "19:00" stores 13:30 UTC,
and a member in New York sees 09:30.

## Models

```python title="clubs/models.py"
class ClubQuerySet(models.QuerySet):
    def for_user(self, user):
        return self.filter(memberships__user=user)


class Club(models.Model):
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)
    created = models.DateTimeField(auto_now_add=True)
    objects = ClubQuerySet.as_manager()
    ...


class Membership(models.Model):
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name="memberships")
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                             related_name="memberships")
    role = models.CharField(max_length=10, choices=Role, default=Role.MEMBER)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["club", "user"], name="one_membership")]


class MeetingQuerySet(models.QuerySet):
    def upcoming(self):
        return self.filter(starts_at__gte=timezone.now()).order_by("starts_at")


class Meeting(models.Model):
    club = models.ForeignKey(Club, on_delete=models.CASCADE, related_name="meetings")
    book_title = models.CharField(max_length=200)
    starts_at = models.DateTimeField()
    location = models.CharField(max_length=200)
    reminder_sent_at = models.DateTimeField(null=True, blank=True)
    objects = MeetingQuerySet.as_manager()

    class Meta:
        ordering = ["starts_at"]
        indexes = [models.Index(fields=["club", "starts_at"], name="meeting_club_starts")]


class RSVP(models.Model):
    class Answer(models.TextChoices):
        YES = "yes", "Going"
        MAYBE = "maybe", "Maybe"
        NO = "no", "Not going"

    meeting = models.ForeignKey(Meeting, on_delete=models.CASCADE, related_name="rsvps")
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                             related_name="rsvps")
    answer = models.CharField(max_length=5, choices=Answer)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["meeting", "user"], name="one_rsvp_per_member")]
```

`reminder_sent_at` is the idempotency marker for requirement 5. The `(club, starts_at)`
index serves the club page's "upcoming meetings for this club" query.

## Views

```python title="clubs/views.py (core)"
def get_club_and_membership(user, pk):
    club = get_object_or_404(Club.objects.for_user(user), pk=pk)
    return club, Membership.objects.get(club=club, user=user)


@login_required
def club_detail(request, pk):
    club, membership = get_club_and_membership(request.user, pk)
    meetings = (club.meetings.upcoming()
                .annotate(going=Count("rsvps", filter=Q(rsvps__answer=RSVP.Answer.YES)))
                .prefetch_related(Prefetch("rsvps", queryset=RSVP.objects.filter(user=request.user),
                                           to_attr="my_rsvp")))
    return render(request, "clubs/club_detail.html",
                  {"club": club, "membership": membership, "meetings": meetings})


@login_required
def meeting_create(request, pk):
    club, membership = get_club_and_membership(request.user, pk)
    if membership.role != Role.HOST:
        raise PermissionDenied("Only hosts can schedule meetings.")
    form = MeetingForm(request.POST or None)
    if request.method == "POST" and form.is_valid():
        meeting = form.save(commit=False)
        meeting.club = club
        meeting.save()
        messages.success(request, "Meeting scheduled.")
        return redirect(club)
    return render(request, "clubs/meeting_form.html", {"club": club, "form": form})


@login_required
@require_POST
def rsvp(request, pk, meeting_pk):
    club, membership = get_club_and_membership(request.user, pk)
    meeting = get_object_or_404(club.meetings.upcoming(), pk=meeting_pk)
    form = RSVPForm(request.POST)
    if form.is_valid():
        RSVP.objects.update_or_create(meeting=meeting, user=request.user,
                                      defaults={"answer": form.cleaned_data["answer"]})
        messages.success(request, "Thanks, your answer is saved.")
    return redirect(meeting)
```

The club page shows, per meeting, how many are going (an annotation) and the current
user's own answer (a filtered `Prefetch` with `to_attr`), so its query count doesn't grow
with the number of meetings. RSVPs are looked up through `club.meetings.upcoming()`: you
can't answer for another club's meeting or for one that has already started.
`update_or_create` plus the unique constraint makes changing your answer an update, never a
duplicate.

## The reminder task and the scheduling command

```python title="clubs/tasks.py"
@task
def send_meeting_reminder(meeting_id):
    """Email every member who hasn't declined. Safe to run more than once."""
    with transaction.atomic():
        meeting = (Meeting.objects.select_for_update()
                   .select_related("club").filter(pk=meeting_id).first())
        if meeting is None or meeting.reminder_sent_at is not None:
            return 0
        declined = meeting.rsvps.filter(answer="no").values("user_id")
        recipients = list(meeting.club.memberships.exclude(user_id__in=declined)
                          .values_list("user__email", flat=True))
        for email in recipients:
            send_mail(f"Reminder: {meeting.book_title}",
                      f"{meeting.club.name} meets at {meeting.location}.",
                      None, [email])
        meeting.reminder_sent_at = timezone.now()
        meeting.save(update_fields=["reminder_sent_at"])
    return len(recipients)
```

- **Idempotent**: it checks and sets `reminder_sent_at` inside one transaction.
- **Concurrency-safe on PostgreSQL**: `select_for_update()` means two workers handed the
  same meeting can't both pass the check (Level 3 · 05). (`select_related("club")` joins
  the club; to lock only the meeting row, add `of=("self",)`.)
- **Takes an ID**, re-fetches, and tolerates the meeting having been deleted.

```python title="clubs/management/commands/send_meeting_reminders.py"
class Command(BaseCommand):
    help = "Enqueue reminders for meetings starting within the next N hours."

    def add_arguments(self, parser):
        parser.add_argument("--hours", type=int, default=24)
        parser.add_argument("--dry-run", action="store_true")

    def handle(self, hours, dry_run, **options):
        now = timezone.now()
        due = Meeting.objects.filter(starts_at__gte=now, starts_at__lte=now + timedelta(hours=hours),
                                     reminder_sent_at__isnull=True)
        count = 0
        for meeting in due.iterator():
            count += 1
            if dry_run:
                self.stdout.write(f"would remind: {meeting} at {meeting.starts_at:%Y-%m-%d %H:%M} UTC")
            else:
                transaction.on_commit(partial(send_meeting_reminder.enqueue, meeting.pk))
        verb = "Would enqueue" if dry_run else "Enqueued"
        self.stdout.write(self.style.SUCCESS(f"{verb} {count} reminder(s)."))
```

Run it hourly from cron or your platform's scheduler. With the default
`ImmediateBackend` the reminder runs inside the command; with a worker-backed tasks
backend (set `DJANGO_TASKS_BACKEND`) it runs in a worker. Either way the code doesn't
change (Level 3 · 08).

## Settings

`config/settings.py` is the environment-driven file from Level 4 · 02, plus:

```python
MIDDLEWARE = [
    "config.request_id.RequestIDMiddleware",
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "accounts.middleware.UserTimezoneMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "django.middleware.csp.ContentSecurityPolicyMiddleware",
]

SECURE_REDIRECT_EXEMPT = [r"^healthz$", r"^readyz$"]
SECURE_CSP = {
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],
    "style-src": [CSP.SELF],
    "img-src": [CSP.SELF, "data:"],
    "frame-ancestors": [CSP.NONE],
    "form-action": [CSP.SELF],
}
AUTH_USER_MODEL = "accounts.User"
TASKS = {"default": {"BACKEND": env("DJANGO_TASKS_BACKEND",
                                    "django.tasks.backends.immediate.ImmediateBackend")}}
# Subdomains and preload stay off until the whole domain is HTTPS-only (see README).
SILENCED_SYSTEM_CHECKS = ["security.W005", "security.W021"]
```

plus the `LOGGING` config with request IDs from Level 4 · 07 (with
`django.security.DisallowedHost` routed nowhere, so host-header scanners don't flood the
logs). No templates contain inline styles or scripts, so the strict policy needs no
exceptions.

### Test settings, and a failure that justified them

Our first test run failed three tests with:

```text
ValueError: Missing staticfiles manifest entry for 'clubs/site.css'
```

The production settings use WhiteNoise's manifest storage, which needs `collectstatic`
to have run, and tests render templates without it (Level 1 · 09 predicted exactly this).
Rather than weakening production settings, a small test module overrides what tests need:

```python title="config/settings_test.py"
"""Test settings: production settings with test-friendly overrides."""
import os

os.environ.setdefault("DJANGO_SECRET_KEY", "test-only-not-secret")
os.environ.setdefault("DJANGO_ALLOWED_HOSTS", "testserver")

from .settings import *  # noqa: E402,F403

# The manifest storage needs collectstatic; tests render templates without it.
STORAGES = {**STORAGES, "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage"}}
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]
SECURE_SSL_REDIRECT = False
MAILERS = {"default": {"BACKEND": "django.core.mail.backends.locmem.EmailBackend"}}
TASKS = {"default": {"BACKEND": "django.tasks.backends.immediate.ImmediateBackend"}}
LOGGING["root"]["level"] = "WARNING"
MIDDLEWARE = [m for m in MIDDLEWARE if m != "whitenoise.middleware.WhiteNoiseMiddleware"]
LOGGING["loggers"]["django.request"]["level"] = "ERROR"
```

```bash
python manage.py test --settings=config.settings_test
```

Keep a CI step that runs `collectstatic` with the real settings as well, so the manifest
problem is still caught where it matters.

## Tests

Ten tests, one per requirement or risk:

| Test | Proves |
|------|--------|
| `test_outsider_gets_404` | non-members can't see clubs or meetings |
| `test_only_host_schedules` | member 403, host 200 on the schedule page |
| `test_rsvp_is_created_then_updated_not_duplicated` | answering twice updates one row |
| `test_rsvp_requires_post` | GET → 405 |
| `test_cannot_rsvp_to_past_meeting` | past meetings → 404 |
| `test_meeting_time_shown_in_each_users_zone` | same instant, each user's local clock time on the page |
| `test_reminder_skips_members_who_declined_and_is_idempotent` | one email, to the right person; second run sends none |
| `test_command_dry_run_and_real_run` | dry run sends nothing; real run (with `captureOnCommitCallbacks`) emails both members and sets `reminder_sent_at` |
| `test_club_page_query_count_does_not_grow_with_meetings` | 6 queries with 21 meetings |
| `test_health_endpoints` | `/healthz` and `/readyz` return ok |

The reminder test, for example:

```python
def test_reminder_skips_members_who_declined_and_is_idempotent(self):
    RSVP.objects.create(meeting=self.meeting, user=self.member, answer="no")
    self.assertEqual(send_meeting_reminder.call(self.meeting.pk), 1)
    self.assertEqual([m.to for m in mail.outbox], [["hana@example.com"]])
    self.assertEqual(send_meeting_reminder.call(self.meeting.pk), 0)
    self.assertEqual(len(mail.outbox), 1)
```

Result:

```text
Found 10 test(s).
System check identified no issues (0 silenced).
..........
Ran 10 tests in 0.068s

OK
```

## Production checks and a real run

With a generated `DJANGO_SECRET_KEY`, `DJANGO_ALLOWED_HOSTS=bookclub.example,127.0.0.1`,
a PostgreSQL `DATABASE_URL` and the SMTP email backend:

```text
$ python manage.py check --deploy --fail-level WARNING
System check identified no issues (2 silenced).
$ python manage.py makemigrations --check --dry-run
No changes detected
$ python manage.py collectstatic --noinput
131 static files copied to '.../staticfiles', 393 post-processed.
```

Served by Gunicorn (two workers) against PostgreSQL:

```text
GET /healthz  → {"status": "ok"}
GET /readyz   → {"status": "ok"}
GET /         → HTTP/1.1 302 Found
                Location: /accounts/login/?next=/
                Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self';
                  img-src 'self' data:; frame-ancestors 'none'; form-action 'self'
                X-Request-ID: ab7fc073f007
```

(The nonce only appears in the policy on pages that use one; this redirect didn't.) We
then seeded a club with host `hana` (Asia/Kolkata) and member `mo` (America/New_York) and
a meeting at 13:30 UTC on 4 October, logged in as each, and read the club page:

```text
hana 200 <time datetime="2026-10-04T19:00:00+05:30">Sun 4 Oct, 19:00 IST</time>
  queries 6
mo   200 <time datetime="2026-10-04T09:30:00-04:00">Sun 4 Oct, 09:30 EDT</time>
  queries 6
```

One meeting, two correct local times, the same six queries for each user.

## How It Actually Works

Follow a reminder from schedule to inbox. Cron runs `send_meeting_reminders` hourly. The
command selects meetings starting in the next 24 hours with no `reminder_sent_at`, and
for each registers an `on_commit` callback; `call_command` isn't in an atomic block, so on
autocommit those run immediately. `enqueue()` hands the meeting ID to the configured tasks
backend. The task opens a transaction, locks the meeting row, re-checks
`reminder_sent_at`, computes recipients with a single subquery (members minus those who
said no), sends the mails, stamps the row and commits, releasing the lock. If a second
worker received the same meeting, it waits on the lock, then sees the stamp and returns 0.
Every layer, from database constraints to the task's re-check, assumes that something
upstream might run twice.

## Where to take it next

1. **API**: add DRF endpoints for clubs, meetings and RSVPs, reusing `for_user()` and
   the role checks (Level 3 · 10 is the template).
2. **Invitations**: hosts invite by email with a signed, expiring link
   (`django.core.signing.TimestampSigner`).
3. **Calendar export**: an `.ics` download per meeting, generated in UTC with a `TZID`.
4. **Translations**: French or your own language, with plurals tested (Level 4 · 08).
5. **Deploy it** to a platform you have access to, with PostgreSQL, a worker-backed tasks
   backend and a scheduled job, and keep `check --deploy` in CI.

## Common mistakes

- **Treating "send once" as a hope** instead of a stamped, locked, re-checked fact.
- **Storing local times** instead of UTC instants plus each user's zone name.
- **Scoping only the club view** and forgetting meetings and RSVPs.
- **Weakening production settings to make tests pass**; override in test settings instead.
- **Inline scripts and styles** that force `'unsafe-inline'` into the CSP.

## Exercise

Build the capstone from the requirements, without copying the reference code first. Then:

1. Make all ten tests pass, plus three of your own for features you add.
2. Get `check --deploy --fail-level WARNING` clean with production-like environment
   variables, and record any silenced check with a reason.
3. Measure the club page with 100 meetings and 50 members and keep it at a constant query
   count.
4. Run the reminder command twice concurrently against PostgreSQL (two terminals) and
   confirm each member receives exactly one email.
