# Integration, contract, and end-to-end tests

[Tutorial index](../README.md) · Prerequisite: [unit testing](01-testing-strategy-and-unit-tests.md)

## Outcome and prerequisites

Verify the boundaries that isolated tests cannot prove: real SQL behavior, HTTP serialization, consumer compatibility, and complete workflows. You need the [Task Service capstone](../09-capstone/01-task-service.md), its PostgreSQL database, Python, and Django dependencies in an activated virtual environment. Use the disposable local PostgreSQL instance containing synthetic data. Django's test runner creates a separate test database; never point a custom test configuration at production.

Testing a boundary requires a clear statement of what is real. An HTTP test with a fake repository validates routing and serialization but not SQL. A repository test using PostgreSQL validates queries but not authentication middleware. A deployed journey validates the assembled path but makes individual failures harder to localize. Combine these views instead of expecting one suite to explain everything.

## Design isolated fixtures

Prefer a fresh database or schema per test worker when practical. This provides clear ownership and allows parallelism, at the cost of setup time. Transaction rollback is fast when every operation uses the same connection; it does not automatically isolate a web server that acquires its own pooled connections. Unique tenant IDs can partition fixtures, but they do not isolate global counters, queues, shared caches, or background jobs.

Apply the real migrations to an empty test database. A schema created by a separate test-only shortcut can silently diverge from production. Test upgrade paths from a representative previous schema as well. Include defaults, indexes, foreign keys, constraints, grants, and migration ordering in the verification scope.

Create builders that make relevant properties explicit: tenant, owner, status, and version. Keep timestamps fixed unless the test concerns time. Avoid relying on row order without `ORDER BY`. A test should fail because behavior changed, not because a database returned an equally valid order.

## Walkthrough: exercise optimistic concurrency in PostgreSQL

1. Start the capstone's disposable database and use its documented `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, and `PGPASSWORD` environment settings. The test database role must be able to create the test database. The local Compose role is intentionally powerful for learning; production application credentials should have narrower permissions.

2. Add `tasks/test_concurrency.py` to a learning copy of the capstone. This complete test requires the capstone's `Tenant` and `Task` models. It exercises actual PostgreSQL updates, not the HTTP route's exact response format.

```python
from concurrent.futures import ThreadPoolExecutor
from threading import Barrier

from django.db import close_old_connections, connections
from django.db.models import F
from django.test import TransactionTestCase
from django.utils import timezone

from tasks.models import Task, Tenant


class OptimisticConcurrencyTests(TransactionTestCase):
    def test_only_one_writer_replaces_the_same_version(self):
        tenant = Tenant.objects.create(name="concurrency fixture")
        task = Task.objects.create(tenant=tenant, title="original")
        barrier = Barrier(2, timeout=5)

        def write(title):
            close_old_connections()
            try:
                barrier.wait()
                return Task.objects.filter(
                    id=task.id, tenant_id=tenant.id, version=1
                ).update(
                    title=title,
                    version=F("version") + 1,
                    updated_at=timezone.now(),
                )
            finally:
                connections.close_all()

        with ThreadPoolExecutor(max_workers=2) as executor:
            results = list(executor.map(write, ["writer A", "writer B"]))
        self.assertEqual(sorted(results), [0, 1])
        task.refresh_from_db()
        self.assertEqual(task.version, 2)
```

Use `TransactionTestCase` here because separate worker connections must observe committed fixtures. Ordinary Django `TestCase` wraps tests in transactions and can hide commit/locking behavior. Database cleanup belongs to Django's disposable test database lifecycle, not a broad deletion against your development tables. [Django testing tools](https://docs.djangoproject.com/en/5.2/topics/testing/tools/)

3. Run `python manage.py test tasks.test_concurrency`. Expect one successful update, one update affecting zero rows, and a final version of two. Remove `version=1` temporarily: both updates should succeed and the test should fail. Restore the predicate. Run this against PostgreSQL; SQLite is not a substitute for this locking experiment.

4. Add a different-tenant update using the same task ID. Expect zero affected rows. This demonstrates isolation at the repository boundary, but you still need an HTTP test proving that tenant identity comes from trusted authentication.

## Contracts and compatibility

An OpenAPI document describes paths, parameters, request bodies, responses, and schemas. A response validator can detect missing fields or incompatible types, but a structurally valid response can still contain the wrong tenant's data. Treat schema validation as one layer. [OpenAPI 3.1.1 specification](https://spec.openapis.org/oas/v3.1.1.html)

Consumer-driven contracts record interactions a consumer depends on, then verify them against the provider. They are useful when services deploy independently. They do not replace provider business-rule tests or a small number of integrated journeys. [How Pact works](https://docs.pact.io/getting_started/how_pact_works)

For Task Service, record whether `version` is a number, whether timestamps include a timezone, whether missing tasks return a stable error shape, and whether a conflict is distinguishable from absence. Adding an optional field is often compatible; changing a field's meaning, removing an enum value, or newly requiring an input may break clients. Check real consumer assumptions, including clients that reject unknown fields.

Version contracts alongside code. Verify the candidate provider against currently deployed consumers and planned consumers before release. Contract publication alone is insufficient: the release process must actually check compatibility and identify the artifact versions involved.

## End-to-end walkthrough and failure evidence

Use a disposable environment to create a task, list it, update it using its observed version, retry the stale version, and verify persisted state. Record HTTP status, relevant body fields, and request ID for each step. Avoid asserting volatile timestamps byte-for-byte. Poll boundedly for asynchronous outcomes rather than sleeping a guessed number of seconds.

Add failure journeys for malformed JSON, unknown routes, database unavailability, and denied cross-tenant access once authentication exists. For queues, publish a uniquely identified event, verify its durable effect, and test duplicate delivery. Stub external email or payment systems at a controlled boundary; never let ordinary test runs create real charges or messages.

Common mistakes include using shared developer data, allowing tests to run against arbitrary environment URLs, cleaning every row instead of owned fixtures, and testing only HTTP 200. Other mistakes are requiring tests to run in one order, reusing stale tokens, and treating a mock provider as proof that the real provider is compatible.

Finish with an evidence table listing each boundary, its real dependencies, fixture ownership, timeout, cleanup, and expected failure behavior. Extend the concurrency test to ten writers and verify that exactly one succeeds. Then test a migration against a populated earlier schema and document what makes rollback safe or unsafe.
