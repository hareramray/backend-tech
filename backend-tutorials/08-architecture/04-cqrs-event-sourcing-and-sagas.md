# CQRS, event sourcing, outboxes, and sagas

## Purpose and prerequisites

These patterns solve different problems and can be adopted separately. CQRS separates command and query models. Event sourcing stores authoritative state changes as events. An outbox makes publication intent durable with a local transaction. A saga coordinates a business process across multiple local transactions.

Prerequisites: [distributed consistency](03-distributed-systems-and-consistency.md), SQL transactions, and the [Django capstone](../09-capstone/01-task-service.md). The schemas below are explicit optional extensions for a disposable database. Represent them with Django models and migrations when integrating; do not create competing unmanaged copies of production tables.

## CQRS does not require event sourcing

A command expresses an intent, such as complete a task. A query asks for a representation, such as a tenant dashboard. Separate models let the write path enforce invariants while the read path optimizes for retrieval. They can initially share one database and run synchronously. Separate stores introduce synchronization and consistency costs. [Microsoft CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs).

For the task service, a command might validate membership and expected version, then update a row. A query can return a denormalized dashboard. If an asynchronous projection serves that dashboard, expose freshness and define what the user sees immediately after a write. Avoid adopting two databases just because command and query functions have different names.

## Event sourcing changes the source of truth

In an event-sourced design, events such as `TaskCreated` and `TaskStatusChanged` form the authoritative history. Current state is obtained by folding events, optionally starting from a snapshot. A snapshot is an optimization, not an independent truth. Event schema evolution, replay, ordering, and data retention become central responsibilities. [Microsoft event-sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing).

An audit log alongside ordinary rows is not automatically event sourcing. If you cannot rebuild authoritative state from the events, you have a different architecture. Likewise, an integration event sent to another service need not expose every internal domain event.

Example event-store extension:

```sql
CREATE TABLE task_events (
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  aggregate_id uuid NOT NULL,
  sequence bigint NOT NULL CHECK (sequence > 0),
  event_id uuid NOT NULL UNIQUE,
  event_type text NOT NULL,
  schema_version integer NOT NULL CHECK (schema_version > 0),
  payload jsonb NOT NULL,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, aggregate_id, sequence)
);
```

Concurrent writers trying to append the same next sequence must conflict. Append all events for one command atomically; do not catch a sequence conflict and blindly replay a business action without reloading state and reevaluating it. The example intentionally does not require an existing `tasks` row: an event-sourced design might use `tasks` only as a projection.

Keep projectors deterministic with respect to event data and explicit dependencies. Replaying historical events must not resend email or charge a customer. Separate state reconstruction from external side effects. Store money, time zones, policy versions, and identifiers with sufficient precision to preserve intended meaning.

## Transactional outbox

For the ordinary row-based capstone, use an outbox before considering full event sourcing:

```sql
CREATE TABLE task_outbox (
  id uuid PRIMARY KEY,
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  aggregate_id uuid NOT NULL,
  aggregate_version integer NOT NULL,
  event_type text NOT NULL,
  schema_version integer NOT NULL DEFAULT 1,
  payload jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz
);
CREATE INDEX task_outbox_pending
  ON task_outbox(created_at, id) WHERE published_at IS NULL;

CREATE TABLE consumer_inbox (
  consumer_name text NOT NULL,
  event_id uuid NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer_name, event_id)
);
```

Within one database transaction, update the task with its expected version and insert the corresponding outbox row. Roll back both if either fails. In Django, `transaction.on_commit()` is useful for waking a dispatcher after commit, but it is not durable delivery: a process can die after the database commits and before a callback finishes. A persistent outbox closes that publication-intent gap.

A dispatcher reads unpublished records, publishes them with stable IDs, and records completion after broker acknowledgment. A crash after publish and before marking completion leads to redelivery. Multiple dispatchers need locking or a lease/claim design; keep transactions and network waits bounded. Preserve per-aggregate ordering where required rather than assuming `SKIP LOCKED` workers always publish in sequence.

Consumers can insert an inbox event ID and update their local projection in one transaction. A duplicate insert means the logical event was already handled for that consumer. This guarantees deduplication only for effects within the same transactional boundary. A remote email or payment API still needs provider idempotency, durable coordination, or reconciliation.

## Sagas and compensating actions

A saga coordinates local transactions for a multi-service workflow. Orchestration uses an explicit coordinator; choreography reacts to events across participants. Both need persistent progress, deadlines, retries, correlation, and operational visibility. Compensation is a business action that offsets an earlier action, not a magical rollback of all history. [Microsoft saga pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga).

Consider a paid task export:

| Step | Local result | Compensation or recovery |
|---|---|---|
| Reserve export quota | Reservation ID | Release unused reservation |
| Generate private file | Object ID | Delete abandoned object |
| Record charge | Payment operation ID | Refund according to policy |
| Publish download | Export available | Revoke access if workflow requires |

Some actions are irreversible or costly. A delivered email cannot be unsent; a refund can fail or settle later. Distinguish retryable, compensatable, irreversible, and manual-review states. A saga does not provide automatic isolation from concurrent workflows; reservations, version checks, and state transitions must preserve business rules.

## Lab: inject failures at the transaction boundary

1. Add the outbox extension to a disposable capstone copy and create a task-update application service.
2. In one `transaction.atomic()` block, update the task and insert an event with a generated UUID and committed task version.
3. Raise an exception before transaction completion. Verify neither the task change nor the outbox row survives.
4. Commit successfully, then stop before publication. Verify the pending row survives and a restarted dispatcher can find it.
5. Use a local fake broker that records accepted event IDs. Publish once, simulate a crash before setting `published_at`, and publish again after restart.
6. Feed both deliveries to a consumer. Insert into `consumer_inbox` and update the projection atomically; expect one logical projection update.
7. Deliver versions three and two out of order. Your projector must reject, buffer, or rebuild according to its contract rather than replace version three with older state.
8. Add a broken message. After bounded retries, isolate it for inspection and alert; do not silently discard it or block every unrelated tenant forever.

Expected result: all crash points have a recoverable durable state, and duplicate delivery is normal rather than exceptional. “Exactly once” must identify the specific boundary; it is not a universal property of a broker plus arbitrary external effects.

## Evolution, privacy, and exercises

Version event schemas and maintain readers for retained history. Upcasting translates old event representations into the current model without rewriting original facts. Rebuild projections into a new table, compare results, then switch readers deliberately. Protect replay tooling with authorization and rate limits.

Immutable history complicates deletion and correction. Avoid placing unnecessary personal content in permanent events; consider references, protected payload stores, or an explicit redaction design. A correction event can change derived business state but does not erase sensitive bytes already retained in older events.

Exercise: implement a persistent export saga state table with permitted transitions, deadlines, and a manual-review state. Simulate a failure during compensation and explain how an operator safely resumes it.

## Completion checklist

- [ ] CQRS, event sourcing, outbox, and saga are distinguished.
- [ ] Task mutation and outbox intent commit atomically.
- [ ] Producers and consumers tolerate duplicates and reordering.
- [ ] Replays cannot repeat uncontrolled external effects.
- [ ] Saga progress and compensation are durable and observable.
- [ ] Schema evolution, retention, and privacy have explicit policies.

Related: [microservices](02-microservices-and-service-communication.md), [audit trails](../03-security/07-abuse-prevention-and-audit.md), and [system design](05-system-design-and-capacity-planning.md).
