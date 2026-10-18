# Django models, ORM and QuerySets

Prerequisites: [course map](../README.md), [relational modeling](../02-data/01-relational-modeling-and-sql.md), [transactions](../02-data/02-transactions-and-concurrency.md), and the initialized Django 5.2 [capstone](../09-capstone/01-task-service.md). Run Python examples inside `python manage.py shell` from the task-service directory unless another destination is stated.

## Objectives

Map domain rules to models and constraints, understand when QuerySets execute, eliminate N+1 queries, express database-side calculations, and retain explicit transaction and tenant boundaries.

## Models describe more than Python objects

A Django model connects Python fields, validation metadata, relationships and database schema. The capstone's Task has a UUID identity, a tenant ForeignKey, title, constrained status, version and timestamps. Its Meta class defines table name, indexes and database constraints.

Separate `null`, which concerns database NULL, from `blank`, which concerns validation. A Python `default` is evaluated while constructing model instances; `db_default` is a database-side default. Callable defaults such as uuid.uuid4 or timezone.now must be passed as callables, not evaluated once while importing the module. [Django model fields and options](https://docs.djangoproject.com/en/5.2/topics/db/models/)

Field choices improve forms and validation but do not automatically enforce every domain invariant in SQL. Model save() also does not automatically call full_clean(). Validate at the application boundary and retain database checks for critical invariants. [Model instance validation](https://docs.djangoproject.com/en/5.2/ref/models/instances/)

ForeignKey stores the related key in a column such as tenant_id; reading task.tenant can issue a query while reading task.tenant_id does not need to load the Tenant object. OneToOneField represents a unique relation. ManyToManyField uses a join table; an explicit through model is useful when the relationship itself has attributes.

Choose on_delete behavior deliberately. Django's ORM deletion policies and database foreign-key actions are distinct layers; do not assume a Python cascade policy makes an arbitrary raw SQL delete cascade in the same way. Multi-table inheritance creates joins; abstract base classes share field definitions without a parent table; proxy models change Python behavior over the existing table.

## Lab 1: seed deterministic records

~~~python
from uuid import NAMESPACE_URL, uuid5
from tasks.models import Tenant, Task

tenant, _ = Tenant.objects.get_or_create(
    id="11111111-1111-4111-8111-111111111111",
    defaults={"name": "Local demo"},
)
ids = [uuid5(NAMESPACE_URL, f"backend-orm-lab/{i}") for i in range(4)]
for i, task_id in enumerate(ids):
    Task.objects.get_or_create(
        id=task_id,
        defaults={
            "tenant": tenant,
            "title": f"ORM exercise {i}",
            "status": "todo",
        },
    )
~~~

This uses a genuinely unique ID for safe get_or_create arbitration. A lookup on non-unique title alone would not guarantee duplicate prevention under concurrency.

## QuerySets are query descriptions

Chaining filter(), exclude() and order_by() generally constructs a query without fetching rows. Iteration, list(), len() and other evaluation operations execute it. An evaluated QuerySet can cache its results, while a newly constructed QuerySet is a separate query. Slicing without a step usually expresses database limits; avoid materializing every row and then slicing in Python. [Django query construction](https://docs.djangoproject.com/en/5.2/topics/db/queries/)

~~~python
query = Task.objects.filter(tenant=tenant, id__in=ids)
query = query.exclude(status="done").order_by("-created_at", "-id")
page = list(query[:2])
print([(item.title, item.tenant_id) for item in page])
~~~

Use exists() for an existence question and count() for a count when rows are not already needed. If you will immediately consume all rows anyway, an extra existence/count query may add unnecessary work. values() and values_list() return projected data without constructing full model instances.

Keep tenant filtering visible. A custom QuerySet method such as `for_tenant(tenant_id)` can reduce repetition, but it is not automatic authorization. Related managers, admin paths, scripts and raw SQL still need deliberate scope. Do not hide a request-specific tenant in global mutable manager state.

## Lab 2: measure and remove N+1 access

Run both blocks with fresh QuerySets:

~~~python
from django.db import connection
from django.test.utils import CaptureQueriesContext

with CaptureQueriesContext(connection) as baseline:
    names = [
        (item.title, item.tenant.name)
        for item in Task.objects.filter(tenant=tenant, id__in=ids)
    ]
with CaptureQueriesContext(connection) as optimized:
    names_again = [
        (item.title, item.tenant.name)
        for item in Task.objects.filter(tenant=tenant, id__in=ids)
                               .select_related("tenant")
    ]
print("baseline:", len(baseline), "optimized:", len(optimized))
assert sorted(names) == sorted(names_again)
assert len(optimized) == 1
~~~

With four records, the baseline normally performs one task query plus four tenant fetches. select_related uses SQL joins for single-valued relations. prefetch_related performs separate queries and connects results in Python, supporting reverse and many-to-many relationships. [QuerySet relation-loading reference](https://docs.djangoproject.com/en/5.2/ref/models/querysets/)

~~~python
from django.db.models import Prefetch

parents = Tenant.objects.filter(pk=tenant.pk).prefetch_related(
    Prefetch(
        "task_set",
        queryset=Task.objects.filter(status="todo"),
        to_attr="todo_tasks",
    )
)
for parent in parents:
    print(parent.name, [item.title for item in parent.todo_tasks])
~~~

Prefetching an unbounded relation can consume considerable memory. Page the parent or child workload and avoid eager-loading everything by habit. Accessing deferred fields from only()/defer() can trigger additional queries, so measure the complete rendering path.

## Lab 3: expressions, aggregates and subqueries

Q objects combine AND, OR and negation conditions. F expressions reference stored fields for database-side computation, avoiding a read-modify-write race for simple increments.

~~~python
from django.db.models import Q, F, Count, OuterRef, Subquery
from django.utils import timezone

open_tasks = Task.objects.filter(
    Q(status="todo") | Q(status="doing"),
    tenant=tenant,
)
print(list(open_tasks.values("status").annotate(total=Count("id"))
                     .order_by("status")))

current = Task.objects.get(tenant=tenant, pk=ids[0])
changed = Task.objects.filter(
    tenant=tenant, pk=current.pk, version=current.version
).update(
    status="doing",
    version=F("version") + 1,
    updated_at=timezone.now(),
)
assert changed == 1
current.refresh_from_db()

latest = Task.objects.filter(tenant_id=OuterRef("pk")).order_by(
    "-created_at", "-id"
)
summary = Tenant.objects.filter(pk=tenant.pk).annotate(
    latest_title=Subquery(latest.values("title")[:1]),
    todo_count=Count("task", filter=Q(task__status="todo")),
)
print(list(summary.values("name", "latest_title", "todo_count")))
~~~

A scalar Subquery must return the right shape: here one selected column and at most one row. Exists is useful for correlated existence checks. Conditional Case/When expressions, database functions and window expressions support more complex computations when their SQL semantics fit the requirement. [Django expression reference](https://docs.djangoproject.com/en/5.2/ref/models/expressions/)

Annotation and filtering order can change meaning. Joining multiple many-valued relations can multiply rows and inflate aggregates; inspect the SQL and use suitable subqueries or deliberately justified distinct counts.

## Writes, transaction boundaries and profiling

QuerySet.update(), bulk_create() and bulk_update() have behavior different from saving each instance. In particular, do not rely on per-instance save overrides or save signals for invariants on bulk paths. Supply timestamps and other derived values intentionally, bound batch size, and keep critical rules in constraints or explicit service operations.

Use transaction.atomic() for multi-statement units and select_for_update() inside a transaction for appropriate locking. In-memory instances do not automatically revert their field values after rollback; reload or discard them. on_commit callbacks are useful for work that should happen after success, but are not durable substitutes for an outbox.

Inspect `str(query.query)` for debugging and `query.explain()` for the database plan. Do not execute the printed debug SQL as a supposedly safe replacement for parameterized execution. Use query capture in tests and profiling tools in development; avoid unrestricted SQL-value logging in production. Optimize based on actual query count, plans and latency. [Django database optimization](https://docs.djangoproject.com/en/5.2/topics/db/optimization/)

For raw SQL, use connection.cursor() with unquoted %s placeholders and a separate values sequence. Keep ownership predicates and result limits. RawQuerySet mappings have their own requirements, including model identity; use raw SQL because the query warrants it, not to bypass validation or tenant scope.

## Failure tests, exercises and completion checklist

Repeat an optimistic update with the old version and verify zero changed rows. Attempt an invalid status through update() and confirm the database constraint rejects it. Demonstrate that save() alone is not full model validation. Profile a serializer that accesses related fields and ensure the query count stays bounded as page size grows.

Add a custom TaskQuerySet with for_tenant() and open() methods, then use as_manager() inside the Task model. Write a test showing it cannot replace authorization checks. Compare an aggregate join with a correlated subquery on realistic data.

- [ ] Field validation and database constraints have distinct roles.
- [ ] Query evaluation and result caching are understood.
- [ ] Tenant scope survives every ORM and raw-SQL path.
- [ ] Eager loading fixes measured N+1 queries without unbounded loading.
- [ ] Bulk writes preserve explicit invariants and timestamps.
- [ ] Transaction, conflict and rollback behavior are tested.
