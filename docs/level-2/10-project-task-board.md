---
description: "Level 2 project: a multi-user Django task board with a custom user, role-based memberships, scoped querysets, a query budget and a test suite."
---

# 10 · Project — A Multi-User Task Board

This project combines Level 2 into one application that several people use at once: a
Trello-style task board. Users belong to boards with a role (owner, editor or viewer);
tasks move between columns; nobody can see or touch a board they don't belong to. The
hard parts aren't the screens, they're the rules: who may do what, and making sure every
view enforces them. That's where this project spends its effort.

It was built and tested on Django 6.1.1. The ten tests at the end all passed; they're
part of the deliverable, not an afterthought. Level 3 · 10 adds a REST API to this same
project, so keep your code.

## Requirements

1. Custom user model from the start (lesson 07).
2. Boards with members via a `Membership` model holding a role.
3. Tasks with status (To do / In progress / Done), an optional assignee and due date.
4. **Visibility:** users see only boards they belong to. Anyone else gets a 404, not a
   403: we don't confirm that a board exists.
5. **Roles:** viewers can view; editors can create, edit and move tasks; only owners can
   add members.
6. Assignees must be members of that board.
7. The board page uses a fixed number of queries regardless of task count.

## Models

```python title="accounts/models.py"
from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
    pass
```

(`AUTH_USER_MODEL = "accounts.User"` in settings, before the first `migrate`.)

```python title="boards/models.py"
from django.conf import settings
from django.db import models
from django.urls import reverse


class Role(models.TextChoices):
    VIEWER = "viewer", "Viewer"
    EDITOR = "editor", "Editor"
    OWNER = "owner", "Owner"


class BoardQuerySet(models.QuerySet):
    def for_user(self, user):
        return self.filter(memberships__user=user)


class Board(models.Model):
    name = models.CharField(max_length=100)
    members = models.ManyToManyField(settings.AUTH_USER_MODEL, through="Membership",
                                     related_name="boards")
    created = models.DateTimeField(auto_now_add=True)

    objects = BoardQuerySet.as_manager()

    class Meta:
        ordering = ["name"]

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse("boards:detail", args=[self.pk])


class Membership(models.Model):
    board = models.ForeignKey(Board, on_delete=models.CASCADE, related_name="memberships")
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                             related_name="memberships")
    role = models.CharField(max_length=10, choices=Role, default=Role.VIEWER)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["board", "user"],
                                               name="one_membership_per_user")]

    def can_edit(self):
        return self.role in (Role.EDITOR, Role.OWNER)


class Task(models.Model):
    class Status(models.TextChoices):
        TODO = "todo", "To do"
        DOING = "doing", "In progress"
        DONE = "done", "Done"

    board = models.ForeignKey(Board, on_delete=models.CASCADE, related_name="tasks")
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    status = models.CharField(max_length=10, choices=Status, default=Status.TODO)
    assignee = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, blank=True,
                                 on_delete=models.SET_NULL, related_name="assigned_tasks")
    created_by = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT,
                                   related_name="created_tasks")
    due = models.DateField(null=True, blank=True)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["due", "created"]
        indexes = [models.Index(fields=["board", "status"])]

    def __str__(self):
        return self.title
```

Decisions worth noticing:

- **`BoardQuerySet.for_user()`** is the single definition of "boards this user may see".
  Every view goes through it. If the rule changes (say, public boards), it changes in one
  place.
- **`assignee` uses `SET_NULL`**: removing a user shouldn't delete their tasks.
  `created_by` uses `PROTECT` so history can't vanish silently.
- **The composite index** on `(board, status)` matches the board page's access pattern.
  On a small table it makes no measurable difference; Level 4 · 04 shows how to check
  whether an index is used.
- The **unique constraint** on membership makes "add the same person twice" impossible
  at the database level, not just in the form.

## Forms

```python title="boards/forms.py"
from django import forms
from django.contrib.auth import get_user_model

from .models import Membership, Task


class TaskForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ["title", "description", "status", "assignee", "due"]
        widgets = {"due": forms.DateInput(attrs={"type": "date"})}

    def __init__(self, *args, board, **kwargs):
        super().__init__(*args, **kwargs)
        # Only members of this board can be assigned.
        self.fields["assignee"].queryset = get_user_model().objects.filter(memberships__board=board)


class AddMemberForm(forms.Form):
    username = forms.CharField()
    role = forms.ChoiceField(choices=Membership._meta.get_field("role").choices)

    def __init__(self, *args, board, **kwargs):
        super().__init__(*args, **kwargs)
        self.board = board

    def clean_username(self):
        User = get_user_model()
        try:
            user = User.objects.get(username=self.cleaned_data["username"])
        except User.DoesNotExist:
            raise forms.ValidationError("No user with that username.")
        if Membership.objects.filter(board=self.board, user=user).exists():
            raise forms.ValidationError("That user is already a member.")
        return user
```

Restricting `assignee`'s **queryset** does two jobs: the dropdown only lists members, and
a forged POST with an outsider's ID fails validation (requirement 6). `board`, `created_by`
and the role checks are never form fields: the user can't set what they shouldn't
control.

## Views

```python title="boards/views.py"
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.core.exceptions import PermissionDenied
from django.db import transaction
from django.db.models import Count, Q
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST

from .forms import AddMemberForm, TaskForm
from .models import Board, Membership, Role, Task


def get_board_and_membership(request, pk):
    """404 unless the current user belongs to the board."""
    board = get_object_or_404(Board.objects.for_user(request.user), pk=pk)
    membership = Membership.objects.get(board=board, user=request.user)
    return board, membership


@login_required
def board_list(request):
    boards = (Board.objects.for_user(request.user)
              .annotate(open_tasks=Count("tasks", filter=~Q(tasks__status=Task.Status.DONE))))
    return render(request, "boards/board_list.html", {"boards": boards})


@login_required
def board_create(request):
    if request.method == "POST":
        name = request.POST.get("name", "").strip()
        if name:
            with transaction.atomic():
                board = Board.objects.create(name=name)
                Membership.objects.create(board=board, user=request.user, role=Role.OWNER)
            messages.success(request, f"Board “{board}” created.")
            return redirect(board)
        messages.error(request, "A board needs a name.")
    return redirect("boards:list")


@login_required
def board_detail(request, pk):
    board, membership = get_board_and_membership(request, pk)
    tasks = board.tasks.select_related("assignee")
    columns = [(value, label, [t for t in tasks if t.status == value])
               for value, label in Task.Status.choices]
    return render(request, "boards/board_detail.html",
                  {"board": board, "membership": membership, "columns": columns})


@login_required
def task_form(request, pk, task_pk=None):
    board, membership = get_board_and_membership(request, pk)
    if not membership.can_edit():
        raise PermissionDenied("Viewers can't change tasks.")
    task = get_object_or_404(board.tasks, pk=task_pk) if task_pk else None
    form = TaskForm(request.POST or None, instance=task, board=board)
    if request.method == "POST" and form.is_valid():
        task = form.save(commit=False)
        task.board = board
        if task.created_by_id is None:
            task.created_by = request.user
        task.save()
        return redirect(board)
    return render(request, "boards/task_form.html", {"board": board, "form": form, "task": task})


@login_required
@require_POST
def task_move(request, pk, task_pk):
    board, membership = get_board_and_membership(request, pk)
    if not membership.can_edit():
        raise PermissionDenied
    status = request.POST.get("status")
    if status not in Task.Status.values:
        messages.error(request, "Unknown status.")
        return redirect(board)
    board.tasks.filter(pk=task_pk).update(status=status)
    return redirect(board)


@login_required
def member_add(request, pk):
    board, membership = get_board_and_membership(request, pk)
    if membership.role != Role.OWNER:
        raise PermissionDenied("Only the owner can manage members.")
    form = AddMemberForm(request.POST or None, board=board)
    if request.method == "POST" and form.is_valid():
        Membership.objects.create(board=board, user=form.cleaned_data["username"],
                                  role=form.cleaned_data["role"])
        messages.success(request, "Member added.")
        return redirect(board)
    return render(request, "boards/member_form.html", {"board": board, "form": form})
```

The security-relevant patterns:

- **`get_board_and_membership()`** is the gate every board view calls first: board
  lookup scoped to the user (outsiders get 404), plus the membership for role checks.
- **Tasks are looked up through the board** (`board.tasks`), never
  `Task.objects.get(pk=...)`. A task ID from a *different* board, even one the user also
  belongs to, is a 404, so URLs can't be mixed to edit across boards.
- **`transaction.atomic()`** around creating a board and its owner membership: if the
  second insert failed, we'd otherwise have a board nobody can see.
- **One task query** for the board page: the three columns are built in Python from a
  single `select_related("assignee")` QuerySet, evaluated once.

## URLs and templates

```python title="boards/urls.py"
app_name = "boards"
urlpatterns = [
    path("", views.board_list, name="list"),
    path("new/", views.board_create, name="create"),
    path("<int:pk>/", views.board_detail, name="detail"),
    path("<int:pk>/tasks/new/", views.task_form, name="task-create"),
    path("<int:pk>/tasks/<int:task_pk>/edit/", views.task_form, name="task-update"),
    path("<int:pk>/tasks/<int:task_pk>/move/", views.task_move, name="task-move"),
    path("<int:pk>/members/new/", views.member_add, name="member-add"),
]
```

The root URLconf includes `django.contrib.auth.urls` under `accounts/` and the boards
app at the root. The heart of the board template:

```django title="boards/templates/boards/board_detail.html"
<h1>{{ board.name }} <small>({{ membership.get_role_display }})</small></h1>
{% if membership.can_edit %}<a href="{% url 'boards:task-create' board.pk %}">Add task</a>{% endif %}
{% if membership.role == "owner" %} · <a href="{% url 'boards:member-add' board.pk %}">Add member</a>{% endif %}
<div class="columns">
{% for value, label, tasks in columns %}
  <section aria-labelledby="col-{{ value }}">
    <h2 id="col-{{ value }}">{{ label }} ({{ tasks|length }})</h2>
    <ul>
    {% for task in tasks %}
      <li>
        {{ task.title }}{% if task.assignee %} · {{ task.assignee.username }}{% endif %}
        {% if membership.can_edit %}
          <a href="{% url 'boards:task-update' board.pk task.pk %}">Edit</a>
          <form method="post" action="{% url 'boards:task-move' board.pk task.pk %}">{% csrf_token %}
            <select name="status" aria-label="Move {{ task.title }}">
              {% for v, l, _ in columns %}<option value="{{ v }}"{% if v == task.status %} selected{% endif %}>{{ l }}</option>{% endfor %}
            </select><button>Move</button>
          </form>
        {% endif %}
      </li>
    {% empty %}<li>Nothing here.</li>{% endfor %}
    </ul>
  </section>
{% endfor %}
</div>
```

`base.html` renders flash messages and a POST logout button (lesson 05 and 08). Moving a
task is a small form with a `<select>`: it works without JavaScript and is
keyboard-accessible; you could enhance it with drag-and-drop later.

## The test suite

The tests encode the requirements. Abbreviated (the full file has ten tests):

```python title="boards/tests/test_access.py"
class BoardAccessTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        User = get_user_model()
        cls.owner = User.objects.create_user("olivia")
        cls.editor = User.objects.create_user("ed")
        cls.viewer = User.objects.create_user("vic")
        cls.outsider = User.objects.create_user("oscar")
        cls.board = Board.objects.create(name="Launch")
        for user, role in [(cls.owner, Role.OWNER), (cls.editor, Role.EDITOR), (cls.viewer, Role.VIEWER)]:
            Membership.objects.create(board=cls.board, user=user, role=role)
        cls.task = Task.objects.create(board=cls.board, title="Write docs", created_by=cls.owner)

    def test_outsider_gets_404_not_403(self):
        self.client.force_login(self.outsider)
        self.assertEqual(self.client.get(self.board.get_absolute_url()).status_code, 404)

    def test_viewer_can_view_but_not_edit(self):
        self.client.force_login(self.viewer)
        self.assertEqual(self.client.get(self.board.get_absolute_url()).status_code, 200)
        url = reverse("boards:task-create", args=[self.board.pk])
        self.assertEqual(self.client.get(url).status_code, 403)

    def test_cannot_assign_non_member(self):
        self.client.force_login(self.editor)
        url = reverse("boards:task-create", args=[self.board.pk])
        response = self.client.post(url, {"title": "Sneaky", "status": "todo",
                                          "assignee": self.outsider.pk})
        self.assertFormError(response.context["form"], "assignee",
            "Select a valid choice. That choice is not one of the available choices.")

    def test_task_from_other_board_is_404(self):
        other = Board.objects.create(name="Other")
        Membership.objects.create(board=other, user=self.editor, role=Role.EDITOR)
        self.client.force_login(self.editor)
        url = reverse("boards:task-update", args=[other.pk, self.task.pk])
        self.assertEqual(self.client.get(url).status_code, 404)


class BoardQueryTests(TestCase):
    def test_board_detail_query_count_is_constant(self):
        ...  # one owner, one board, 30 tasks spread across three statuses
        self.client.force_login(owner)
        with self.assertNumQueries(5):
            response = self.client.get(board.get_absolute_url())
        self.assertContains(response, "In progress (10)")
```

The other tests cover: anonymous users redirected to login; an editor can move a task;
an unknown status is rejected without changing the task; only the owner can add members
(editor gets 403, owner succeeds); the board list shows only the user's boards. Result:

```text
Found 10 test(s).
System check identified no issues (0 silenced).
..........
Ran 10 tests in 0.059s

OK
```

Five queries for a board with 30 tasks: session, user, board (scoped), membership, and
tasks with assignees joined. The count stays at five for 300 tasks.

## How It Actually Works

Follow an editor moving a task. `POST /1/tasks/1/move/` passes CSRF and auth middleware;
`login_required` confirms `request.user.is_authenticated`; `require_POST` confirms the
method. `get_board_and_membership()` runs `SELECT ... FROM boards_board INNER JOIN
boards_membership ... WHERE membership.user_id = <me> AND board.id = 1`: if no row, 404.
Then the role check. Then `board.tasks.filter(pk=1).update(status="doing")`, which
compiles to `UPDATE boards_task SET status = 'doing' WHERE board_id = 1 AND id = 1`. The
`board_id` condition is the cross-board protection: a task ID from another board matches
zero rows. Every layer narrows what the next can touch, which is the general shape of
authorization in a multi-user app.

## Common mistakes

- **One view forgetting the scoped lookup.** Centralise it (`for_user()`,
  `get_board_and_membership()`), and test every view with an outsider.
- **Returning 403 for boards the user can't see**, which confirms they exist.
- **Accepting `board` or `created_by` from the form.**
- **Not limiting the assignee queryset**, letting a forged POST assign outsiders.
- **Creating related rows without a transaction**, leaving orphans on failure.

## Exercise

1. Build the project and make all ten tests pass.
2. Add "remove member" for owners. What should happen to tasks assigned to the removed
   user? Write the test first.
3. Add a `?mine=1` filter on the board page that shows only tasks assigned to the current
   user, without increasing the query count.
4. Add an `activity` log: a model recording "ed moved 'Write docs' to In progress", shown
   at the bottom of the board. Keep the board page's query count fixed.
5. Prevent the last owner from being demoted or removed.
