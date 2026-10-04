---
description: "Write Django management commands safe for production: arguments, dry runs, transactions, form validation, exit codes, scheduling and tests."
---

# 05 · Custom Management Commands

Every real project accumulates jobs that aren't web requests: importing a spreadsheet,
backfilling a column, cleaning up expired data, sending a nightly digest, fixing a batch
of bad records. Running them as ad-hoc snippets in `manage.py shell` is how production
data gets damaged: no review, no tests, no dry run, no record. Management commands give
those jobs a home: versioned, testable, with arguments and help text, runnable by cron, a
scheduler or an operator.

This lesson builds an importer for the Level 1 reading-list app and runs it against good
and bad data. All output below is from those runs.

## Where commands live

```text
books/
└── management/
    ├── __init__.py
    └── commands/
        ├── __init__.py
        └── import_entries.py      # → python manage.py import_entries
```

The file name is the command name. Any installed app can contribute commands, which is
how `migrate` (from `django.core`) and `createsuperuser` (from `django.contrib.auth`)
appear.

## The command

```python title="books/management/commands/import_entries.py"
import csv
from pathlib import Path

from django.core.management.base import BaseCommand, CommandError
from django.db import transaction

from books.forms import EntryForm
from books.models import Entry


class Command(BaseCommand):
    help = "Import reading-list entries from a CSV file with title,author,status columns."

    def add_arguments(self, parser):
        parser.add_argument("csv_path", type=Path)
        parser.add_argument("--dry-run", action="store_true",
                            help="Validate and report without saving anything.")
        parser.add_argument("--skip-existing", action="store_true",
                            help="Skip rows whose title and author already exist.")

    def handle(self, csv_path, dry_run, skip_existing, **options):
        if not csv_path.exists():
            raise CommandError(f"No such file: {csv_path}")

        created = skipped = 0
        errors = []
        with csv_path.open(newline="", encoding="utf-8") as f, transaction.atomic():
            for line_no, row in enumerate(csv.DictReader(f), start=2):
                if skip_existing and Entry.objects.filter(
                        title=row.get("title"), author=row.get("author")).exists():
                    skipped += 1
                    continue
                form = EntryForm(data={**row, "status": row.get("status") or "want"})
                if not form.is_valid():
                    errors.append(f"line {line_no}: {form.errors.as_text()}")
                    continue
                form.save()
                created += 1
                if options["verbosity"] >= 2:
                    self.stdout.write(f"  + {form.instance}")
            if dry_run:
                transaction.set_rollback(True)

        for message in errors:
            self.stderr.write(self.style.ERROR(message))
        verb = "Would create" if dry_run else "Created"
        self.stdout.write(self.style.SUCCESS(f"{verb} {created}, skipped {skipped}, {len(errors)} error(s)."))
        if errors and not dry_run:
            raise CommandError("Some rows failed; the valid rows were imported.")
```

Design decisions:

- **Validation through the existing `EntryForm`.** The importer applies exactly the same
  rules as the web form (required fields, valid choices, the rating rule) instead of a
  second, drifting copy. Bulk imports are where invalid data usually sneaks in.
- **`--dry-run` that runs everything** and then rolls back with
  `transaction.set_rollback(True)`. It exercises the real code path, including database
  constraints, rather than a separate "pretend" branch.
- **`--skip-existing`** makes re-running safe (idempotent), which matters when a run is
  interrupted halfway.
- **`self.stdout` / `self.stderr`**, not `print()`: they respect `--no-color`, can be
  captured in tests, and `self.style.SUCCESS/ERROR` add colour on terminals.
- **`CommandError`** for failures: Django prints the message and exits with status 1, so
  cron, CI and schedulers can detect it.
- **Verbosity**: every command gets `-v 0..3` for free; read it from `options["verbosity"]`.
  (Our first version used `self.verbosity`, which doesn't exist, and crashed with
  `AttributeError: 'Command' object has no attribute 'verbosity'`.)

## Running it

The CSV:

```text
title,author,status
Piranesi,Susanna Clarke,done
Middlemarch,George Eliot,
The Overstory,Richard Powers,reading
,Nameless Author,want
Beloved,Toni Morrison,someday
```

`--help` comes from `argparse`:

```text
usage: manage.py import_entries [-h] [--dry-run] [--skip-existing] [--version]
                                [-v {0,1,2,3}] [--settings SETTINGS] ...
                                csv_path

Import reading-list entries from a CSV file with title,author,status columns.
```

A dry run:

```text
$ python manage.py import_entries entries.csv --dry-run
line 5: * title
  * This field is required.
line 6: * status
  * Select a valid choice. someday is not one of the available choices.
Would create 3, skipped 0, 2 error(s).
```

and afterwards the table still held **0** rows. The real run with `-v 2`:

```text
$ python manage.py import_entries entries.csv -v 2
  + Piranesi — Susanna Clarke
  + Middlemarch — George Eliot
  + The Overstory — Richard Powers
Created 3, skipped 0, 2 error(s).
line 5: ...
line 6: ...
CommandError: Some rows failed; the valid rows were imported.
```

(exit status 1; stdout and stderr interleave differently depending on your terminal).
Running it again with `--skip-existing` reported `Created 0, skipped 3, 2 error(s).`
And a missing file: `CommandError: No such file: missing.csv`, exit 1.

Whether partial success should commit is a policy choice. This command keeps the valid
rows; for financial or relational data you might prefer all-or-nothing (raise inside the
`atomic()` block on the first error).

## Testing commands

`call_command` runs a command in-process with the same argument handling:

```python title="books/tests/test_commands.py"
import tempfile
from io import StringIO
from pathlib import Path

from django.core.management import CommandError, call_command
from django.test import TestCase

from books.models import Entry


class ImportEntriesTests(TestCase):
    def write_csv(self, text):
        directory = self.enterContext(tempfile.TemporaryDirectory())
        path = Path(directory) / "in.csv"
        path.write_text(text, encoding="utf-8")
        return path

    def test_dry_run_saves_nothing(self):
        path = self.write_csv("title,author,status\nPiranesi,Susanna Clarke,done\n")
        out = StringIO()
        call_command("import_entries", path, dry_run=True, stdout=out)
        self.assertIn("Would create 1", out.getvalue())
        self.assertEqual(Entry.objects.count(), 0)

    def test_bad_rows_reported_and_good_rows_kept(self):
        path = self.write_csv("title,author,status\nPiranesi,Susanna Clarke,done\n,Nobody,want\n")
        err = StringIO()
        with self.assertRaisesMessage(CommandError, "Some rows failed"):
            call_command("import_entries", path, stdout=StringIO(), stderr=err)
        self.assertIn("line 3", err.getvalue())
        self.assertEqual(Entry.objects.count(), 1)
```

Both passed, and the reading-list app's full suite (now 12 tests) still passed.
`enterContext()` (Python 3.11+ `unittest`) cleans up the temporary directory after each
test. Note that `call_command` raises `CommandError` instead of exiting, so tests can
assert on it.

## Scheduling

Commands are the natural unit for scheduled jobs:

```text
# crontab: clean expired sessions nightly at 03:15, log output
15 3 * * *  cd /srv/app && .venv/bin/python manage.py clearsessions >> /var/log/app/cron.log 2>&1
```

On container platforms, use the platform's scheduled-job feature with the same image and
environment as the web app. Either way:

- **Only one instance should run** at a time; use a lock (a database advisory lock, or a
  row with `select_for_update(nowait=True)`) if overlap is possible.
- **Log start, end and counts**, and alert on non-zero exit codes.
- **Make it idempotent**, so a retry after a crash is safe.
- For long backfills, process in batches with `iterator(chunk_size=...)` and commit per
  batch, so progress survives interruption.

## How It Actually Works

`manage.py` calls `execute_from_command_line()`, which finds the command by scanning the
`management/commands` packages of every installed app (apps listed earlier win name
clashes, so a project app can override a built-in command). It imports the module,
instantiates its `Command` class and calls `run_from_argv()`. That builds an
`argparse` parser with the common options (`--settings`, `-v`, `--traceback`,
`--no-color`), calls your `add_arguments()`, parses `sys.argv`, and calls `execute()`,
which runs system checks (unless `requires_system_checks = []`) and then your
`handle(**options)`. A `CommandError` raised from `handle()` is caught by
`run_from_argv()`, printed to stderr and turned into `sys.exit(returncode)`, default 1.
`call_command()` skips the `sys.argv` and exit handling, which is why it re-raises.

## Common mistakes

- **One-off fixes typed into `manage.py shell` in production**, unreviewed and unrepeatable.
- **`print()` instead of `self.stdout.write()`**, making output untestable.
- **No dry run** for commands that change data.
- **Separate validation logic** from what the web app enforces.
- **Swallowing errors and exiting 0**, so failures go unnoticed by schedulers.
- **Loading a whole table into memory** (`list(Model.objects.all())`) for a backfill.

## Exercise

1. Build `import_entries` and its tests. Add a `--fail-fast` option that rolls back
   everything on the first invalid row.
2. Write `export_entries` that writes CSV to stdout (`self.stdout`) so it can be piped:
   `python manage.py export_entries > backup.csv`. Round-trip it through the importer.
3. Write `prune_orphan_covers` for Level 1 · 09's uploads: find files in `MEDIA_ROOT`
   with no matching database row, report them, and delete only with `--delete`.
4. Add a lock so two simultaneous runs of a command can't overlap, and test it.
