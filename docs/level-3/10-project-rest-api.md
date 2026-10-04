---
description: "Level 3 project: a token-authenticated DRF API for the task board with nested routes, role-based access, a move action and 12 API tests."
---

# 10 · Project — A REST API for the Task Board

The Level 2 task board works in a browser. Now a mobile app and a command-line tool need
to use it. This project adds a JSON API to the **same project**, reusing its models and,
crucially, its access rules: the API must not become a side door that ignores the roles
the HTML views enforce. It brings together serializers, viewsets and routers (lessons
01–02), authentication, permissions and pagination (03), transactions (05) and the query
discipline from Level 2.

Built and tested on Django 6.1.1 and DRF 3.18.1. The full suite, the ten Level 2 tests plus
twelve new API tests, passes: `Ran 22 tests ... OK`.

## API design

| Method & path | Who | Does |
|---------------|-----|------|
| `GET /api/boards/` | any member | my boards, with my role and open-task count |
| `POST /api/boards/` | any user | create a board; creator becomes owner |
| `GET /api/boards/{id}/` | members | board with member list |
| `PATCH/DELETE /api/boards/{id}/` | owner | rename / delete |
| `GET /api/boards/{id}/tasks/?status=` | members | tasks, paginated |
| `POST /api/boards/{id}/tasks/` | editors, owner | create task |
| `GET/PATCH/DELETE /api/boards/{id}/tasks/{tid}/` | members read; editors+ write | task detail |
| `POST /api/boards/{id}/tasks/{tid}/move/` | editors, owner | change status |

Outsiders get **404** everywhere for boards they don't belong to, consistent with the HTML
views.

## Settings

```python title="config/settings.py"
INSTALLED_APPS += ["rest_framework", "rest_framework.authtoken"]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.TokenAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 50,
    "DEFAULT_THROTTLE_CLASSES": ["rest_framework.throttling.UserRateThrottle"],
    "DEFAULT_THROTTLE_RATES": {"user": "1000/hour"},
}
```

`IsAuthenticated` by default means a new endpoint is closed until someone opens it.
`TokenAuthentication` is listed **first**, so unauthenticated requests get a proper 401
with `WWW-Authenticate: Token` (lesson 03); we confirmed: `401 {'detail':
'Authentication credentials were not provided.'}` with header `Token`.

## Serializers

```python title="boards/serializers.py"
from django.contrib.auth import get_user_model
from rest_framework import serializers

from .models import Board, Membership, Task


class MemberSerializer(serializers.ModelSerializer):
    username = serializers.CharField(source="user.username", read_only=True)

    class Meta:
        model = Membership
        fields = ["username", "role"]


class BoardSerializer(serializers.ModelSerializer):
    my_role = serializers.SerializerMethodField()
    open_tasks = serializers.IntegerField(read_only=True)

    class Meta:
        model = Board
        fields = ["id", "name", "my_role", "open_tasks", "created"]
        read_only_fields = ["created"]

    def get_my_role(self, board):
        # Filled in by the viewset's annotation; no extra query.
        return getattr(board, "my_role", None)


class BoardDetailSerializer(BoardSerializer):
    members = MemberSerializer(source="memberships", many=True, read_only=True)

    class Meta(BoardSerializer.Meta):
        fields = BoardSerializer.Meta.fields + ["members"]


class TaskSerializer(serializers.ModelSerializer):
    assignee = serializers.SlugRelatedField(slug_field="username", allow_null=True, required=False,
                                            queryset=get_user_model().objects.all())
    created_by = serializers.CharField(source="created_by.username", read_only=True)

    class Meta:
        model = Task
        fields = ["id", "board", "title", "description", "status", "assignee", "created_by",
                  "due", "created", "updated"]
        read_only_fields = ["board", "created", "updated"]

    def validate_assignee(self, user):
        board = self.context["board"]
        if user is not None and not board.memberships.filter(user=user).exists():
            raise serializers.ValidationError("Assignee must be a member of this board.")
        return user


class MoveSerializer(serializers.Serializer):
    status = serializers.ChoiceField(choices=Task.Status.choices)
```

`board` and `created_by` are **read-only**: the server sets them from the URL and the
authenticated user. A client that sends `"board": <other id>` is simply ignored (there's a
test for that). The assignee is validated against *this* board's members, which the
serializer learns about through `context`.

## Viewsets

```python title="boards/api.py"
from django.db import transaction
from django.db.models import Count, OuterRef, Q, Subquery
from django.shortcuts import get_object_or_404
from rest_framework import mixins, permissions, viewsets
from rest_framework.decorators import action
from rest_framework.exceptions import PermissionDenied
from rest_framework.response import Response

from .models import Board, Membership, Role, Task
from .serializers import BoardDetailSerializer, BoardSerializer, MoveSerializer, TaskSerializer


def boards_for(user):
    my_role = Membership.objects.filter(board=OuterRef("pk"), user=user).values("role")[:1]
    return (Board.objects.for_user(user)
            .annotate(my_role=Subquery(my_role),
                      open_tasks=Count("tasks", filter=~Q(tasks__status=Task.Status.DONE)))
            .order_by("name"))


class BoardViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        qs = boards_for(self.request.user)
        if self.action == "retrieve":
            qs = qs.prefetch_related("memberships__user")
        return qs

    def get_serializer_class(self):
        return BoardDetailSerializer if self.action == "retrieve" else BoardSerializer

    def perform_create(self, serializer):
        with transaction.atomic():
            board = serializer.save()
            Membership.objects.create(board=board, user=self.request.user, role=Role.OWNER)
        board.my_role, board.open_tasks = Role.OWNER, 0

    def check_object_permissions(self, request, obj):
        super().check_object_permissions(request, obj)
        if request.method not in permissions.SAFE_METHODS and obj.my_role != Role.OWNER:
            raise PermissionDenied("Only the board owner can change or delete the board.")


class TaskViewSet(mixins.ListModelMixin, mixins.CreateModelMixin, mixins.RetrieveModelMixin,
                  mixins.UpdateModelMixin, mixins.DestroyModelMixin, viewsets.GenericViewSet):
    serializer_class = TaskSerializer

    def initial(self, request, *args, **kwargs):
        super().initial(request, *args, **kwargs)
        self.board = get_object_or_404(boards_for(request.user), pk=self.kwargs["board_pk"])
        if request.method not in permissions.SAFE_METHODS and self.board.my_role == Role.VIEWER:
            raise PermissionDenied("Viewers can't change tasks.")

    def get_queryset(self):
        qs = self.board.tasks.select_related("assignee", "created_by")
        if status := self.request.query_params.get("status"):
            qs = qs.filter(status=status)
        return qs

    def get_serializer_context(self):
        return {**super().get_serializer_context(), "board": getattr(self, "board", None)}

    def perform_create(self, serializer):
        serializer.save(board=self.board, created_by=self.request.user)

    @action(detail=True, methods=["post"])
    def move(self, request, board_pk=None, pk=None):
        task = self.get_object()
        data = MoveSerializer(data=request.data)
        data.is_valid(raise_exception=True)
        task.status = data.validated_data["status"]
        task.save(update_fields=["status", "updated"])
        return Response(self.get_serializer(task).data)
```

How the rules are enforced:

- **`boards_for(user)`** is the single scoped QuerySet, built on Level 2's
  `Board.objects.for_user()`. It also annotates the user's role (a `Subquery`) and the open
  task count, so lists need no per-board queries.
- **`TaskViewSet.initial()`** runs after DRF's authentication, permission and throttle
  checks and before any handler. It resolves the board from the URL (404 for outsiders)
  and blocks viewers from unsafe methods. Every task endpoint, including the custom
  `move` action, passes through it.
- **Tasks come from `self.board.tasks`**, so a task ID from another board is a 404.
- **`move` validates its input** with a serializer instead of trusting
  `request.data["status"]`, avoiding lesson 02's "actions bypass validation" trap.
- **The `order_by("name")`** is there because of a warning the tests surfaced. Our first
  version relied on `Board.Meta.ordering`, and DRF's paginator warned:
  `UnorderedObjectListWarning: Pagination may yield inconsistent results with an
  unordered object_list`. The `Count` annotation adds a `GROUP BY`, and Django drops
  `Meta.ordering` from grouped queries (Level 2 · 02). Unordered pagination can repeat or
  skip rows between pages.

## Nested routes

```python title="boards/api_urls.py"
from django.urls import include, path
from rest_framework.routers import DefaultRouter, SimpleRouter

from . import api

router = DefaultRouter()
router.register("boards", api.BoardViewSet, basename="board")

task_router = SimpleRouter()
task_router.register("tasks", api.TaskViewSet, basename="task")

urlpatterns = [
    path("", include(router.urls)),
    path("boards/<int:board_pk>/", include(task_router.urls)),
]
```

The resulting routes (format-suffix variants omitted):

```text
^boards/$                                          board-list
^boards/(?P<pk>[^/.]+)/$                           board-detail
boards/<int:board_pk>/^tasks/$                     task-list
boards/<int:board_pk>/^tasks/(?P<pk>[^/.]+)/$      task-detail
boards/<int:board_pk>/^tasks/(?P<pk>[^/.]+)/move/$ task-move
```

Nesting by `include()` with a captured `board_pk` keeps DRF's plain routers; packages
such as `drf-nested-routers` automate it if you have many nested resources.

## A session with the API

Requests made with DRF's `APIClient` as a fresh user:

```text
POST /api/boards/ {"name": "Demo board"}
201 {'id': 1, 'name': 'Demo board', 'my_role': 'owner', 'open_tasks': 0, 'created': '2026-10-02T08:49:07.365584Z'}

POST /api/boards/1/tasks/ {"title": "Write the API lesson", "due": "2026-10-09"}
201 {'id': 1, 'board': 1, 'title': 'Write the API lesson', 'description': '', 'status': 'todo',
     'assignee': None, 'created_by': 'demo', 'due': '2026-10-09', ...}

POST /api/boards/1/tasks/1/move/ {"status": "doing"}
200 ... "status": "doing"

GET /api/boards/1/
200 {'id': 1, 'name': 'Demo board', 'my_role': 'owner', 'open_tasks': 1, ...,
     'members': [{'username': 'demo', 'role': 'owner'}]}
```

From a terminal, with a token created by `python manage.py drf_create_token <username>`:

```bash
curl -H "Authorization: Token $TOKEN" http://127.0.0.1:8000/api/boards/
```

## Tests

`APITestCase` gives each test an `APIClient`. Twelve tests encode the table at the top
(abbreviated here):

```python title="boards/tests/test_api.py"
class TaskApiTests(APITestCase):
    # setUpTestData: users olivia (owner), ed (editor), vic (viewer), oscar (outsider),
    # one board "Launch" with one task.

    def test_requires_authentication(self):
        self.assertEqual(self.client.get(self.tasks_url).status_code, 401)

    def test_outsider_gets_404(self):
        self.client.force_authenticate(self.outsider)
        self.assertEqual(self.client.get(self.tasks_url).status_code, 404)

    def test_editor_creates_task_board_and_creator_set_by_server(self):
        self.client.force_authenticate(self.editor)
        other = Board.objects.create(name="Other")
        response = self.client.post(self.tasks_url, {"title": "Ship it", "board": other.pk, "assignee": "vic"})
        self.assertEqual(response.status_code, 201, response.data)
        self.assertEqual(response.data["board"], self.board.pk)
        self.assertEqual(response.data["created_by"], "ed")

    def test_task_ids_cannot_cross_boards(self):
        other = Board.objects.create(name="Other")
        Membership.objects.create(board=other, user=self.editor, role=Role.EDITOR)
        self.client.force_authenticate(self.editor)
        url = reverse("task-detail", args=[other.pk, self.task.pk])
        self.assertEqual(self.client.patch(url, {"title": "hijack"}).status_code, 404)

    def test_task_list_query_count_is_constant(self):
        for i in range(25):
            Task.objects.create(board=self.board, title=f"T{i}", created_by=self.owner, assignee=self.editor)
        self.client.force_authenticate(self.editor)
        with self.assertNumQueries(3):
            response = self.client.get(self.tasks_url)
        self.assertEqual(response.data["count"], 26)
```

The others cover token authentication, the role and open count in the board list, viewer
403 on create, assignee must be a member (400 with the custom message), `move` rejecting
an unknown status, only the owner renaming a board, and board creation making the
creator owner. Three queries for 26 tasks: the board lookup, the pagination `COUNT`, and
the page of tasks with both users joined. (`force_authenticate` skips the session and
user queries a real request would add.)

## How It Actually Works

Trace `POST /api/boards/1/tasks/1/move/` from an editor with a token. Django's resolver
matches the `include()` prefix, capturing `board_pk=1`, then the router's pattern for the
`move` action, capturing `pk=1`. DRF's `dispatch()` builds a `Request`, and `initial()`
authenticates (the `TokenAuthentication` class looks up the key, one query), runs
`IsAuthenticated`, checks the throttle, and then our override resolves the board through
`boards_for()` and the role. The `move` method calls `get_object()`, which filters
`get_queryset()` (that is, `board.tasks`) by `pk` and runs object permission checks. The
input passes through `MoveSerializer`, the save writes two columns, and the response is
serialized with the same `TaskSerializer` the other endpoints use. Every layer that the
HTML views relied on (scoping, roles, server-set fields) has an equivalent here, and the
tests prove each one.

## Common mistakes

- **An API that skips the web app's rules** because it was written separately. Reuse the
  scoped QuerySets.
- **Writable `board` / `created_by` fields** letting clients reassign data.
- **Grouped QuerySets without `order_by()`** under pagination.
- **Custom actions trusting `request.data`.**
- **`AllowAny` by default** and forgetting to lock down a new endpoint.

## Exercise

1. Add the API to your task board and get all 22 tests passing.
2. Add `GET /api/boards/{id}/members/` and `POST` (owner only) to add a member by
   username. Write tests for an editor (403) and an outsider (404).
3. Add `?assignee=me` filtering to the task list without increasing the query count.
4. Switch the task list to `CursorPagination` ordered by `-created` and update the tests.
5. Write a small Python script using `httpx` and a token that prints every open task
   assigned to you across all your boards.
