# Modular monoliths and domain design

## Purpose and prerequisites

A modular monolith deploys as one application while separating business capabilities inside the codebase. It can use one process or several workers sharing the same release boundary. Its defining property is deliberate internal boundaries, not the number of folders.

Prerequisites: Django views, SQL transactions, and the [task-service capstone](../09-capstone/01-task-service.md). The Python examples are domain code; they require integration with your existing Django application rather than a new package installation.

## Design around business language

Start with what the product does: task management, tenant membership, notifications, and billing. A controller/service/repository arrangement separates technical concerns but does not by itself separate business capabilities. If every service imports every other service's tables, changing one rule still requires understanding the entire system.

Domain-driven design supplies a vocabulary:

- An entity has identity over time, such as a task whose title changes.
- A value object represents a meaningful value, such as a validated title or date range.
- An aggregate groups state governed by one consistency boundary.
- A bounded context defines where a particular model and vocabulary apply.
- A domain service represents a business operation that does not naturally belong to one entity.
- An application service coordinates use cases, authentication context, transactions, and external ports.

These are design tools, not a requirement to wrap every string in a class. Microsoft recommends evaluating service boundaries around business capabilities and cohesion; the same analysis is useful before any physical distribution. [Domain analysis guidance](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis).

In the tutorial application, “member” in tenant administration may mean a permission-bearing relationship. “Recipient” in notifications means someone eligible for a message. Keeping those concepts distinct prevents the notification module from becoming the authority for access control.

## Dependencies point toward rules

A useful structure is:

```text
project/
  tasks/
    domain.py
    services.py
    models.py
    selectors.py
    views.py
    api.py
  memberships/
    api.py
    ...
  notifications/
    api.py
    ...
  config/
    settings.py
```

Domain code should not need the HTTP request object, PostgreSQL connection class, email SDK, or global environment variables. An application service calls ports; adapters implement those ports. This makes business behavior easier to exercise without a running network.

For example, a domain transition can remain small and explicit:

```python
from dataclasses import dataclass, replace
from typing import Literal

Status = Literal["todo", "doing", "done"]

@dataclass(frozen=True)
class TaskState:
    id: str
    status: Status
    version: int

def transition(task: TaskState, next_status: Status) -> TaskState:
    if task.status == next_status:
        return task
    permitted = {
        "todo": {"doing"},
        "doing": {"todo", "done"},
        "done": {"doing"},
    }
    if next_status not in permitted[task.status]:
        raise ValueError("Invalid task transition")
    return replace(task, status=next_status, version=task.version + 1)
```

This is an example product rule, not a universal task-management requirement. It deliberately disallows `todo` directly to `done`; change it if your product permits that transition. Authorization happens before this rule, and persistence still needs a version predicate so concurrent requests cannot overwrite each other.

Django apps provide a natural packaging and migration boundary but do not enforce business isolation automatically. It is also reasonable for simple domain behavior to live on Django models. Extract pure objects when they make complex rules easier to understand, not merely to hide the ORM behind an identical wrapper. Signals are useful hooks but should not conceal an essential multi-step business workflow from the application service.

## Transactions and ownership

An aggregate boundary should align with invariants you need to preserve atomically. A task and its immediate status transition can fit in one transaction. A tenant containing millions of tasks should not become one giant in-memory aggregate merely because every task has a tenant ID.

Let each module own its tables and migrations. Other modules call its public interface or consume documented events. Within a monolith, a carefully chosen cross-module transaction can be useful; record the coupling it creates. Do not pretend a function call is a remote service simply to appear ready for hypothetical scale.

Repository interfaces should express actual operations, such as `find_task` or `save_if_version_matches`, rather than exposing every ORM feature. Avoid a universal generic repository that removes access to useful database capabilities while still leaking database details.

For the capstone, a persisted transition can use:

```sql
UPDATE tasks
SET status = $3, version = version + 1, updated_at = now()
WHERE tenant_id = $1 AND id = $2 AND version = $4
RETURNING id, status, version;
```

Zero affected rows requires an explicit conflict or missing-resource decision. When a domain rule depends on more state than one version, load and validate that state under a suitable transaction strategy. Do not assume an object method protects an invariant from concurrent writers.

## Lab: separate task rules from transport

1. Inventory the capstone's request handler responsibilities: parsing, authentication context, membership check, transition rule, query, response mapping, and logging.
2. Extract the status rule into a domain module without network or framework imports.
3. Define a repository port with `load(tenant_id, task_id)` and `save_if_version_matches(task, expected_version)` semantics. Document conflict behavior.
4. Write an application service that receives trusted tenant/user context, calls the membership policy, applies the transition, and persists it.
5. Keep the HTTP adapter responsible for status codes and JSON shape. Keep the PostgreSQL adapter responsible for SQL and row mapping.
6. Test the rule with a small in-memory example: `todo` to `doing` succeeds; `todo` to `done` throws; repeating `doing` leaves the version unchanged.
7. Test the real repository with two updates using the same expected version. Exactly one mutation should succeed.
8. Move notification delivery behind a port. Confirm a slow vendor call is not executed while holding a task-row lock; persist delivery intent if reliability requires it.

Expected result: a domain change can be understood without reading HTTP plumbing, and database concurrency behavior remains visible. A failure is a “clean” class design that accidentally removes tenant predicates or transactional guarantees.

## Architectural fitness and evolution

Use explicit module exports and import rules to discourage access to internals. A module dependency diagram should have a comprehensible direction. Cycles often indicate a missing abstraction, a misplaced rule, or two modules that should remain together.

Write short architecture decision records: context, decision, alternatives, consequences, and revisit triggers. For example, keep tasks and memberships in one deployment until independent scaling, ownership, release cadence, or fault isolation creates measured value from separation.

A modular monolith still needs observability, migrations, backups, security boundaries, and deployment discipline. It can also become too large. Conversely, splitting it into services does not automatically remove coupling; a distributed cycle is harder to debug than an in-process cycle.

## Pitfalls and exercises

Avoid a shared “common” module that accumulates every domain concept. Share stable primitives sparingly; duplicate a tiny translation when contexts intentionally differ. Do not publish internal database models as external API contracts. Avoid event-driven indirection for a simple synchronous rule that requires an immediate answer.

Exercise: add task assignment without importing the membership module's database table into the task domain. Decide which module verifies membership, what happens when the assignee leaves, and which invariant needs a transaction. Then document the tradeoff between a live membership lookup and an asynchronously maintained projection.

## Completion checklist

- [ ] Modules correspond to understandable business capabilities.
- [ ] Domain rules can execute without framework or vendor SDK objects.
- [ ] Public interfaces and table ownership are documented.
- [ ] Concurrency and tenant boundaries survive the refactor.
- [ ] Cross-module transactions and events have explicit semantics.
- [ ] Service extraction has a measured trigger and an owner.

Continue with [microservices](02-microservices-and-service-communication.md), [distributed consistency](03-distributed-systems-and-consistency.md), and [CQRS and sagas](04-cqrs-event-sourcing-and-sagas.md).
