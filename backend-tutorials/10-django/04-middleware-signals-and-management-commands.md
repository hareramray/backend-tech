# Django middleware, signals, app initialization, and management commands

[Curriculum](../README.md) · Prerequisites: request lifecycle, Django apps, and transactions

## Place behavior where its lifetime belongs

Middleware wraps HTTP requests. Views translate requests to application operations. Services coordinate business rules and transactions. Signals notify receivers about events inside the process. Management commands run explicit operational or administrative tasks. A background worker owns durable deferred work.

These mechanisms are not interchangeable. Putting a payment call in `post_save` hides a critical effect from the service that controls the transaction. Putting a large import in a request makes its lifetime depend on HTTP timeouts. Putting database queries in module imports can break startup, migrations, and tests before their environment is ready.

## Middleware order and scope

Middleware runs around the view in a configured order, with responses unwinding through the chain. Session middleware must make sessions available before authentication middleware uses them. A component may return a response early, so not every request reaches the view. Async compatibility also affects the actual execution stack. [Django middleware](https://docs.djangoproject.com/en/5.2/topics/http/middleware/).

Example synchronous middleware for a new app, or a refactored capstone after removing duplicate request-ID creation from views:

```python
import logging
import time
import uuid

logger = logging.getLogger("requests")

class RequestContextMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request.request_id = str(uuid.uuid4())
        started = time.perf_counter()
        response = self.get_response(request)
        response["X-Request-Id"] = request.request_id
        match = request.resolver_match
        route = match.route if match is not None else "unmatched"
        logger.info("request id=%s method=%s route=%s status=%s duration_ms=%.2f",
                    request.request_id, request.method, route,
                    response.status_code, (time.perf_counter() - started) * 1000)
        return response
```

Save as `tasks/middleware.py` and add `tasks.middleware.RequestContextMiddleware` to `MIDDLEWARE` where it should wrap the routes you want measured. The log uses the route pattern, not arbitrary URL query strings or bodies. Configure the logger/handler before expecting INFO output. This measures time to produce the response object; it does not include complete streaming-response transmission.

Generate an ID unless you have a trusted upstream policy for accepting one. Avoid module globals for request identity. For context spanning async tasks, use context-local mechanisms carefully and clear their lifetime; a global variable can leak identities between simultaneous requests.

## Signals and their limits

Django model signals such as `post_save` run as part of the process's call flow. They are not a durable message broker. Receivers can increase save latency, raise exceptions, create recursion, or obscure ordering. Bulk operations and queryset updates do not follow every per-instance save/signal path. [Django signals](https://docs.djangoproject.com/en/5.2/topics/signals/).

If a signal is appropriate, make registration explicit and idempotent. Example diagnostic receiver, not an audit system:

```python
# tasks/signals.py
import logging
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Task

logger = logging.getLogger(__name__)

@receiver(post_save, sender=Task, dispatch_uid="tasks.diagnostic.created")
def task_created(sender, instance, created, **kwargs):
    if created:
        logger.debug("Task instance saved; transaction may still roll back")
```

Import the module in `TasksConfig.ready()` in `tasks/apps.py`:

```python
from django.apps import AppConfig

class TasksConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "tasks"

    def ready(self):
        from . import signals  # noqa: F401
```

Do not execute queries in `ready()`. Initialization occurs in management commands and can occur more than once in tests. For a required business effect, call an explicit service and write an outbox row inside the transaction. `transaction.on_commit()` can delay a callback until successful commit, but a process crash after commit can still prevent delivery.

## Custom management command lab

Create `tasks/management/__init__.py` and `tasks/management/commands/__init__.py` as empty files. Create `tasks/management/commands/task_counts.py`:

```python
import uuid
from django.core.management.base import BaseCommand, CommandError
from django.db.models import Count
from tasks.models import Task

class Command(BaseCommand):
    help = "Report task counts for exactly one tenant"

    def add_arguments(self, parser):
        parser.add_argument("--tenant", required=True)

    def handle(self, *args, **options):
        try:
            tenant_id = uuid.UUID(options["tenant"])
        except ValueError as exc:
            raise CommandError("--tenant must be a UUID") from exc
        rows = (Task.objects.filter(tenant_id=tenant_id).values("status")
                .annotate(total=Count("id")).order_by("status"))
        for row in rows:
            self.stdout.write(f"{row['status']}: {row['total']}")
```

Run it from the configured capstone environment:

```powershell
python manage.py task_counts --tenant 11111111-1111-4111-8111-111111111111
```

Expected output lists present status groups and their counts. An empty tenant returns no rows. This command uses database credentials directly and assumes an authorized operator; it does not magically inherit browser permissions.

## Safe maintenance jobs

A mutating maintenance command needs explicit scope, dry-run support, bounded batches, progress/checkpoints, idempotent reruns, and a verification query. Avoid holding one transaction over millions of rows. Estimate row count and lock impact before updating. Separate the operator's authority from the application's least-privilege runtime credentials.

Scheduled execution must prevent accidental overlap where overlap is unsafe. A process-local flag does not coordinate multiple schedulers. Use the job/locking model from the async chapters and define behavior after an interrupted run.

## Exercises and checklist

Test middleware on success, view error, 404, and a streaming response. Verify the log includes no body, token, or arbitrary query. Test command output with Django's `call_command` and an in-memory output stream. Compare `Task.save()` with `QuerySet.update()` to observe signal differences.

- [ ] Cross-cutting HTTP behavior belongs in a compatible middleware stack.
- [ ] Signals are optional in-process notifications, not durability guarantees.
- [ ] Initialization is free of database side effects.
- [ ] Commands declare scope and can be tested without parsing terminal output manually.
- [ ] Maintenance work has interruption and rerun semantics.
