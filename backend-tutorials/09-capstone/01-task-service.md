# Capstone: build a Django and PostgreSQL Task Service

[Curriculum](../README.md) · Prerequisites: Python basics, HTTP, and [relational modeling](../02-data/01-relational-modeling-and-sql.md)

## What you will build

A local JSON API with PostgreSQL persistence, database constraints, migrations, bounded cursor pagination, tenant-scoped queries, optimistic concurrency, structured errors, health checks, and tests. The files below are complete for this lab; create them in a new `task-service` directory. Other tutorial fragments are extensions to this baseline.

This is an explicitly local learning application. It fixes one tenant on the server and does not authenticate callers. Its JSON write endpoints are exempt from CSRF only in this isolated demo. It refuses demo mode with `DEBUG=False`, rejects demo API requests carrying an Origin header, and must be bound to loopback. This Origin rule is an additional demo restriction, not authentication or a complete browser detector. Before a real deployment, implement [authentication](../03-security/02-authentication-passwords-and-sessions.md), [membership authorization](../03-security/04-authorization-and-multitenancy.md), restore CSRF protection for cookie-authenticated writes, and complete the production chapters. Never expose this demo through a public tunnel.

The chosen tutorial versions are Python 3.12/3.13, Django 5.2, psycopg 3, and PostgreSQL 17. They are a reproducible learning baseline, not a claim about the newest releases. Django separates project configuration from app code and provides development scaffolding. [Django tutorial](https://docs.djangoproject.com/en/5.2/intro/tutorial01/).

## 1. Prepare the directory and dependencies

Prerequisites: Python with the `py` launcher on Windows, and Docker with Compose for the local database. A native PostgreSQL 17 instance is an alternative if you create the same database and user. Docker installation is not part of this tutorial. Commands below are PowerShell and run from your chosen practice location.

```powershell
New-Item -ItemType Directory task-service
Set-Location task-service
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install "Django~=5.2.0" "psycopg[binary]>=3.2,<4"
.\.venv\Scripts\python.exe -m django startproject config .
.\.venv\Scripts\python.exe manage.py startapp tasks
.\.venv\Scripts\python.exe -m pip freeze | Set-Content requirements.lock.txt
```

Record direct dependencies in `requirements.in`:

```text
Django~=5.2.0
psycopg[binary]>=3.2,<4
```

`pip freeze` records this environment's resolved packages. The dependency-management chapter explains hash-locked builds and cross-platform lock considerations. In later commands, `python` means this virtual environment's interpreter. You can always replace it with `.\.venv\Scripts\python.exe` instead of activating the environment.

Create `.gitignore`:

```gitignore
.venv/
__pycache__/
*.py[cod]
.env
.coverage
htmlcov/
```

## 2. Start PostgreSQL

Create `compose.yaml`:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: taskapp
      POSTGRES_PASSWORD: taskapp_local_only
      POSTGRES_DB: taskapp
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - taskapp_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskapp -d taskapp"]
      interval: 5s
      timeout: 3s
      retries: 10
volumes:
  taskapp_data:
```

```powershell
docker compose up -d db
docker compose ps
```

Wait for healthy status. The password is a public local-lab example. The image's initial role has elevated privileges for setup; a production application must use a restricted role and separate migration credentials. Named volumes survive container replacement. `docker compose down` stops the project without deleting that volume; do not add a volume-deletion flag unless deliberately resetting disposable data.

## 3. Configure Django

Keep the generated `config/settings.py` and append the following block at its end, overriding the generated development values:

```python
import os
from django.core.exceptions import ImproperlyConfigured

SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DJANGO_DEBUG", "0") == "1"
DEMO_MODE = os.environ.get("DEMO_MODE", "0") == "1"
if DEMO_MODE and not DEBUG:
    raise ImproperlyConfigured("Demo mode is allowed only with DEBUG enabled")
DEMO_TENANT_ID = "11111111-1111-4111-8111-111111111111"
ALLOWED_HOSTS = ["127.0.0.1", "localhost", "[::1]"]
INSTALLED_APPS += ["tasks"]
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("PGDATABASE", "taskapp"),
        "USER": os.environ.get("PGUSER", "taskapp"),
        "PASSWORD": os.environ["PGPASSWORD"],
        "HOST": os.environ.get("PGHOST", "127.0.0.1"),
        "PORT": os.environ.get("PGPORT", "5432"),
        "CONN_MAX_AGE": 0,
        "OPTIONS": {
            "connect_timeout": 3,
            "options": "-c statement_timeout=3000 -c lock_timeout=1000",
        },
    }
}
USE_TZ = True
TIME_ZONE = "UTC"
DATA_UPLOAD_MAX_MEMORY_SIZE = 16 * 1024
```

Before any following management command, set these variables in the current PowerShell terminal:

```powershell
$env:DJANGO_SECRET_KEY = 'local-tutorial-only-change-before-real-use-please'
$env:DJANGO_DEBUG = '1'
$env:DEMO_MODE = '1'
$env:PGPASSWORD = 'taskapp_local_only'
```

These values are intentionally public development examples. A second terminal needs its own environment configuration. Environment variables are not loaded from `.env` automatically in this setup. Keep Django's generated security, session, authentication, and CSRF middleware enabled.

## 4. Define models and constraints

This disposable baseline uses Django's default user model. For a real new application, decide whether to use a custom user before the first `migrate` command. The [custom-user tutorial](../10-django/06-custom-users-permissions-and-auth-workflows.md) starts from a fresh project; switching an already-migrated auth model is a separate migration project.

Replace `tasks/models.py`:

```python
import uuid
from django.db import models
from django.db.models import Q
from django.utils import timezone


class Tenant(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name = models.CharField(max_length=200)

    class Meta:
        db_table = "tenants"


class Task(models.Model):
    class Status(models.TextChoices):
        TODO = "todo", "To do"
        DOING = "doing", "Doing"
        DONE = "done", "Done"

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    title = models.CharField(max_length=200)
    status = models.CharField(max_length=10, choices=Status, default=Status.TODO)
    version = models.PositiveIntegerField(default=1)
    created_at = models.DateTimeField(default=timezone.now)
    updated_at = models.DateTimeField(default=timezone.now)

    class Meta:
        db_table = "tasks"
        constraints = [
            models.UniqueConstraint(fields=["tenant", "id"], name="task_tenant_id_uq"),
            models.CheckConstraint(condition=Q(version__gte=1), name="task_version_positive"),
            models.CheckConstraint(
                condition=Q(status__in=["todo", "doing", "done"]),
                name="task_status_valid",
            ),
            models.CheckConstraint(condition=~Q(title=""), name="task_title_nonempty"),
        ]
        indexes = [models.Index(
            fields=["tenant", "-created_at", "-id"], name="task_tenant_page_idx",
        )]
```

Field choices help application validation but do not replace a database check. The redundant-looking tenant/ID uniqueness supports tenant-aware composite references in advanced SQL labs. Python defaults are not automatically SQL defaults; raw SQL inserts in later labs must supply omitted values or explicitly install database defaults.

## 5. Implement validation and cursors

Create `tasks/validation.py`:

```python
import json
import uuid
from datetime import datetime
from django.core import signing
from django.utils.timezone import is_aware


class ApiError(Exception):
    def __init__(self, status, code, detail):
        super().__init__(detail)
        self.status, self.code, self.detail = status, code, detail


def reject_constant(value):
    raise ValueError("Non-finite JSON number")


def unique_object(pairs):
    result = {}
    for key, value in pairs:
        if key in result:
            raise ValueError("Duplicate JSON property")
        result[key] = value
    return result


def parse_body(raw, *, patch=False):
    try:
        body = json.loads(raw.decode("utf-8"), parse_constant=reject_constant,
                          object_pairs_hook=unique_object)
    except (UnicodeDecodeError, ValueError, RecursionError) as exc:
        raise ApiError(400, "invalid_json", "Send a valid UTF-8 JSON object") from exc
    if not isinstance(body, dict):
        raise ApiError(400, "invalid_body", "Body must be an object")
    allowed = {"title", "status", "expected_version"} if patch else {"title", "status"}
    if set(body) - allowed:
        raise ApiError(400, "unknown_field", "Body contains an unsupported field")
    if (not patch and "title" not in body) or (patch and not {"title", "status"} & body.keys()):
        raise ApiError(400, "missing_field", "Supply the required task fields")
    result = {}
    if "title" in body:
        title = body["title"]
        if not isinstance(title, str) or not 1 <= len(title.strip()) <= 200:
            raise ApiError(400, "invalid_title", "Title must contain 1 to 200 characters")
        if "\x00" in title or any(0xD800 <= ord(char) <= 0xDFFF for char in title):
            raise ApiError(400, "invalid_title", "Title must contain valid Unicode text")
        result["title"] = title.strip()
    if "status" in body:
        if body["status"] not in ("todo", "doing", "done"):
            raise ApiError(400, "invalid_status", "Status must be todo, doing, or done")
        result["status"] = body["status"]
    if patch:
        version = body.get("expected_version")
        if type(version) is not int or not 1 <= version <= 2_147_483_646:
            raise ApiError(400, "invalid_version", "Supply a valid expected_version integer")
        result["expected_version"] = version
    return result


def parse_limit(raw):
    if raw is None:
        return 20
    if not raw.isascii() or not raw.isdigit() or len(raw) > 3:
        raise ApiError(400, "invalid_limit", "Limit must be an integer from 1 to 100")
    limit = int(raw)
    if not 1 <= limit <= 100:
        raise ApiError(400, "invalid_limit", "Limit must be an integer from 1 to 100")
    return limit


def encode_cursor(task):
    return signing.dumps({
        "v": 1, "tenant": str(task.tenant_id), "id": str(task.id),
        "created": task.created_at.isoformat(),
    }, salt="task-page-v1")


def decode_cursor(token, tenant_id):
    if not token or len(token) > 1024:
        raise ApiError(400, "invalid_cursor", "Cursor is invalid or expired")
    try:
        value = signing.loads(token, salt="task-page-v1", max_age=3600)
        if value["v"] != 1 or value["tenant"] != str(tenant_id):
            raise ValueError("Wrong cursor scope")
        created = datetime.fromisoformat(value["created"])
        task_id = uuid.UUID(value["id"])
        if not is_aware(created):
            raise ValueError("Timezone required")
        return created, task_id
    except (signing.BadSignature, ValueError, TypeError, KeyError) as exc:
        raise ApiError(400, "invalid_cursor", "Cursor is invalid or expired") from exc
```

Cursors preserve microseconds, expire after one hour, and are scoped to this tenant and format. Signing detects tampering; it does not encrypt the contents or authorize the caller. If adding filtering, bind the cursor to a canonical filter specification too.

## 6. Implement the API

Replace `tasks/views.py`:

```python
import logging
import uuid
from functools import wraps
from django.conf import settings
from django.core.exceptions import RequestDataTooBig
from django.db import DatabaseError, connection, transaction
from django.db.models import F, Q
from django.http import JsonResponse
from django.utils import timezone
from django.views.decorators.csrf import csrf_exempt
from .models import Task
from .validation import ApiError, parse_body, parse_limit, encode_cursor, decode_cursor

logger = logging.getLogger(__name__)


def api(methods, *, demo=True):
    def decorate(view):
        @wraps(view)
        def wrapped(request, *args, **kwargs):
            request_id = str(uuid.uuid4())
            try:
                if request.method not in methods:
                    raise ApiError(405, "method_not_allowed", "Method not allowed")
                if demo and not settings.DEMO_MODE:
                    raise ApiError(503, "demo_disabled", "Demo API is disabled")
                if demo and request.headers.get("Origin"):
                    raise ApiError(403, "browser_origin_denied", "Demo accepts local CLI requests only")
                response = view(request, *args, **kwargs)
            except ApiError as exc:
                response = JsonResponse({"error": {"code": exc.code, "detail": exc.detail},
                                         "request_id": request_id}, status=exc.status)
            except RequestDataTooBig:
                response = JsonResponse({"error": {"code": "body_too_large"},
                                         "request_id": request_id}, status=413)
            except DatabaseError:
                logger.error("database_failure request_id=%s", request_id)
                response = JsonResponse({"error": {"code": "database_unavailable"},
                                         "request_id": request_id}, status=503)
            except Exception:
                logger.error("unexpected_failure request_id=%s", request_id)
                response = JsonResponse({"error": {"code": "internal_error"},
                                         "request_id": request_id}, status=500)
            response["X-Request-Id"] = request_id
            response["Cache-Control"] = "no-store"
            if response.status_code == 405:
                response["Allow"] = ", ".join(methods)
            return response
        return wrapped
    return decorate


def representation(task):
    return {
        "id": str(task.id), "tenant_id": str(task.tenant_id),
        "title": task.title, "status": task.status, "version": task.version,
        "created_at": task.created_at.isoformat(),
        "updated_at": task.updated_at.isoformat(),
    }


def json_input(request, *, patch=False):
    if request.content_type != "application/json":
        raise ApiError(415, "unsupported_media_type", "Use application/json")
    return parse_body(request.body, patch=patch)


@api(["GET"], demo=False)
def live(request):
    return JsonResponse({"status": "live"})


@api(["GET"], demo=False)
def ready(request):
    with connection.cursor() as cursor:
        cursor.execute("SELECT 1")
    return JsonResponse({"status": "ready"})


@csrf_exempt  # LOCAL DEMO ONLY: replace with real auth + CSRF policy before deployment.
@api(["GET", "POST"])
def collection(request):
    tenant_id = settings.DEMO_TENANT_ID
    if request.method == "POST":
        values = json_input(request)
        task = Task.objects.create(tenant_id=tenant_id, **values)
        response = JsonResponse(representation(task), status=201)
        response["Location"] = f"/v1/tasks/{task.id}/"
        return response
    if set(request.GET) - {"limit", "cursor"}:
        raise ApiError(400, "unknown_query", "Unsupported query parameter")
    if any(len(request.GET.getlist(key)) != 1 for key in request.GET):
        raise ApiError(400, "duplicate_query", "Do not repeat query parameters")
    limit = parse_limit(request.GET.get("limit"))
    query = Task.objects.filter(tenant_id=tenant_id)
    if "cursor" in request.GET:
        created, task_id = decode_cursor(request.GET["cursor"], tenant_id)
        query = query.filter(Q(created_at__lt=created) |
                             Q(created_at=created, id__lt=task_id))
    rows = list(query.order_by("-created_at", "-id")[:limit + 1])
    page = rows[:limit]
    next_cursor = encode_cursor(page[-1]) if len(rows) > limit else None
    return JsonResponse({"items": [representation(task) for task in page],
                         "next_cursor": next_cursor})


@csrf_exempt  # LOCAL DEMO ONLY.
@api(["GET", "PATCH"])
def detail(request, task_id):
    query = Task.objects.filter(tenant_id=settings.DEMO_TENANT_ID, id=task_id)
    if request.method == "GET":
        task = query.first()
        if task is None:
            raise ApiError(404, "not_found", "Task not found")
        return JsonResponse(representation(task))
    values = json_input(request, patch=True)
    expected = values.pop("expected_version")
    with transaction.atomic():
        changed = query.filter(version=expected).update(
            **values, version=F("version") + 1, updated_at=timezone.now(),
        )
        if not changed:
            if not query.exists():
                raise ApiError(404, "not_found", "Task not found")
            raise ApiError(409, "version_conflict", "Refetch before applying a new update")
        task = query.get()
    return JsonResponse(representation(task))
```

The conditional SQL update prevents a stale writer from overwriting a committed update. Its row lock remains held until the short transaction commits, so the following read returns this update's result. No network calls occur inside the transaction. Django documents the atomic block's commit/rollback behavior. [Django transactions](https://docs.djangoproject.com/en/5.2/topics/db/transactions/).

The lab deliberately logs only an error category and request ID. A production service needs restricted, redacted exception diagnostics and more precise database error mapping; this simplified handler maps all database exceptions to 503. Readiness checks database reachability, not every table or migration. Limits at the proxy and application server are also required in production because an application body limit alone cannot protect every upload/framing path.

## 7. Add URLs and initialize data

Create `tasks/urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path("health/live/", views.live),
    path("health/ready/", views.ready),
    path("v1/tasks/", views.collection),
    path("v1/tasks/<uuid:task_id>/", views.detail),
]
```

Replace `config/urls.py`:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [path("admin/", admin.site.urls), path("", include("tasks.urls"))]
```

```powershell
.\.venv\Scripts\python.exe manage.py makemigrations tasks
.\.venv\Scripts\python.exe manage.py migrate
.\.venv\Scripts\python.exe manage.py shell -c "from tasks.models import Tenant; Tenant.objects.get_or_create(id='11111111-1111-4111-8111-111111111111', defaults={'name': 'Local demo'})"
.\.venv\Scripts\python.exe manage.py check
.\.venv\Scripts\python.exe manage.py runserver 127.0.0.1:3000
```

Keep the generated migration file in version control. Use trailing slashes consistently; do not rely on a redirect preserving an unsafe request body. The development server is for local practice, not production.

## 8. Exercise the API

Open a second PowerShell terminal:

```powershell
$base = 'http://127.0.0.1:3000'
Invoke-RestMethod "$base/health/ready/"
$created = Invoke-RestMethod -Method Post -Uri "$base/v1/tasks/" -ContentType 'application/json' -Body '{"title":"Learn Django","status":"todo"}'
$created
Invoke-RestMethod "$base/v1/tasks/$($created.id)/"
Invoke-RestMethod -Method Patch -Uri "$base/v1/tasks/$($created.id)/" -ContentType 'application/json' -Body '{"status":"done","expected_version":1}'
Invoke-RestMethod "$base/v1/tasks/?limit=1"
```

Expected: creation returns a UUID and version 1; the update returns status `done` and version 2. Repeating the version-1 update returns 409, not another success. PowerShell raises an HTTP error for that expected conflict. To inspect headers, use `curl.exe -i` for a GET.

Create at least three tasks. Request `limit=1`, copy `next_cursor`, URL-encode it with `[uri]::EscapeDataString(...)`, and request the next page. Each task should appear once during a stable traversal. This is not a snapshot across concurrent edits; the sort keys remain immutable, and later inserts ahead of the cursor appear on a fresh traversal.

## 9. Add tests

Replace `tasks/tests.py`:

```python
import json
import uuid
from django.conf import settings
from django.test import SimpleTestCase, TestCase, override_settings
from .models import Tenant, Task
from .validation import ApiError, parse_body, parse_limit


class ValidationTests(SimpleTestCase):
    def test_title_is_trimmed(self):
        self.assertEqual(parse_body(b'{"title":" hello "}'), {"title": "hello"})

    def test_boolean_version_rejected(self):
        with self.assertRaises(ApiError):
            parse_body(b'{"status":"done","expected_version":true}', patch=True)

    def test_unknown_field_rejected(self):
        with self.assertRaises(ApiError):
            parse_body(b'{"title":"hello","tenant_id":"other"}')

    def test_duplicate_json_field_rejected(self):
        with self.assertRaises(ApiError):
            parse_body(b'{"title":"a","title":"b"}')

    def test_invalid_database_text_rejected(self):
        for title in ("bad\x00text", "bad\ud800text"):
            with self.subTest(title=repr(title)), self.assertRaises(ApiError):
                parse_body(json.dumps({"title": title}).encode("ascii"))

    def test_excessive_nesting_rejected(self):
        with self.assertRaises(ApiError):
            parse_body(b"[" * 4000 + b"0" + b"]" * 4000)

    def test_limit_bounds(self):
        self.assertEqual(parse_limit(None), 20)
        for value in ("0", "101", "2.5", "-1", "1e2"):
            with self.subTest(value=value), self.assertRaises(ApiError):
                parse_limit(value)


@override_settings(DEMO_MODE=True)
class ApiTests(TestCase):
    def setUp(self):
        self.tenant = Tenant.objects.create(id=settings.DEMO_TENANT_ID, name="Test")

    def test_create_update_and_stale_conflict(self):
        response = self.client.post("/v1/tasks/", {"title": "Learn"},
                                    content_type="application/json")
        self.assertEqual(response.status_code, 201)
        task = response.json()
        path = f"/v1/tasks/{task['id']}/"
        body = json.dumps({"status": "done", "expected_version": 1})
        updated = self.client.patch(path, body, content_type="application/json")
        self.assertEqual(updated.status_code, 200)
        self.assertEqual(updated.json()["version"], 2)
        stale = self.client.patch(path, body, content_type="application/json")
        self.assertEqual(stale.status_code, 409)

    def test_other_tenant_is_not_visible(self):
        other = Tenant.objects.create(id=uuid.uuid4(), name="Other")
        task = Task.objects.create(tenant=other, title="Private")
        self.assertEqual(self.client.get(f"/v1/tasks/{task.id}/").status_code, 404)
        self.assertEqual(self.client.get("/v1/tasks/").json()["items"], [])

    def test_pagination_and_tampering(self):
        for title in ("One", "Two", "Three"):
            Task.objects.create(tenant=self.tenant, title=title)
        first = self.client.get("/v1/tasks/", {"limit": 2}).json()
        second = self.client.get("/v1/tasks/", {
            "limit": 2, "cursor": first["next_cursor"],
        }).json()
        ids = [item["id"] for item in first["items"] + second["items"]]
        self.assertEqual(len(set(ids)), 3)
        self.assertIsNone(second["next_cursor"])
        response = self.client.get("/v1/tasks/", {"cursor": first["next_cursor"] + "x"})
        self.assertEqual(response.status_code, 400)

    def test_browser_origin_is_rejected_in_demo(self):
        response = self.client.post("/v1/tasks/", {"title": "Unexpected"},
                                    content_type="application/json",
                                    HTTP_ORIGIN="https://untrusted.example")
        self.assertEqual(response.status_code, 403)
```

```powershell
.\.venv\Scripts\python.exe manage.py test tasks
```

Django creates a separate test database for `TestCase`; the local setup role can create it. Never point tests at a production database or reuse production credentials. Run pure parser tests without database creation using `manage.py test tasks.tests.ValidationTests`. PostgreSQL-specific constraints, locking, and SQL plans still require PostgreSQL tests.

## 10. Failure drills and extension order

| Drill | Expected outcome |
| --- | --- |
| Empty title / unknown field | 400 with stable code |
| Wrong media type | 415 |
| Unsupported method on a matched API route | 405 with Allow |
| Unknown task or another tenant's task | 404 |
| Stale expected version | 409; newer data remains intact |
| Tampered, expired, or wrong-tenant cursor | 400 |
| Stop database | Liveness 200, readiness 503, writes fail |
| Restart application | Previously committed tasks remain |

Unknown URL paths and middleware-level failures can still use Django's default HTML handlers in this baseline. Build JSON 404/500/CSRF handlers as an API consistency extension; do not claim every failure is already normalized.

Then add, in order: sessions and tenant memberships; object permissions; Django REST Framework serializers; a transactional outbox; a Celery worker; Redis cache invalidation; OpenAPI; integration and load tests; metrics; a nonroot production container; release migrations; restore drills. If choosing a custom user, begin a fresh project with that choice before its first migration as described above. Each subject has its own tutorial. Redis, Celery, DRF, and a production server are not dependencies of the initial lab.

## Completion checklist

- [ ] I can recreate the project from these instructions and a clean database.
- [ ] Validation, database constraints, and API responses agree.
- [ ] Pagination and stale-update tests pass on PostgreSQL.
- [ ] A process restart preserves tasks.
- [ ] I can explain why this local demo is not an authenticated production service.
- [ ] I have completed the production hardening chapters before exposing an adapted service.
