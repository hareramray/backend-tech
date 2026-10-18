# Event-driven systems and the transactional outbox

Prerequisites: [transactions](../02-data/02-transactions-and-concurrency.md), [queues and streams](02-message-queues-and-streams.md), the [course map](../README.md), and the initialized [task-service capstone](../09-capstone/01-task-service.md). Run the fixed-ID seed step in [relational modeling](../02-data/01-relational-modeling-and-sql.md) before the lab.

## Objectives

Publish events without losing committed changes, understand the remaining duplicate window, evolve event contracts, and distinguish event-driven integration from event sourcing.

## The dual-write failure

Suppose an API updates a task and then publishes TaskStatusChanged. If the database commit succeeds but publication fails, subscribers never learn about the change. Publishing first reverses the problem: subscribers can act on a change whose database transaction later rolls back. Retrying a request does not automatically repair either case.

A transactional outbox stores the business mutation and an event record in the same local database transaction. A separate relay publishes committed outbox records and records progress. The relay can deliver duplicates, so subscribers remain idempotent. The pattern addresses the local database-plus-publication boundary; it does not make all downstream systems one transaction. [AWS transactional outbox guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)

## Lab: create an outbox

Run this extension DDL once:

~~~sql
CREATE TABLE tutorial_outbox (
  event_id uuid PRIMARY KEY,
  tenant_id uuid NOT NULL,
  aggregate_id uuid NOT NULL,
  aggregate_version integer NOT NULL,
  event_type text NOT NULL,
  schema_version integer NOT NULL,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  payload jsonb NOT NULL CHECK (jsonb_typeof(payload) = 'object'),
  published_at timestamptz,
  UNIQUE (tenant_id, aggregate_id, aggregate_version, event_type)
);
CREATE INDEX tutorial_outbox_pending_idx
  ON tutorial_outbox(occurred_at, event_id)
  WHERE published_at IS NULL;
~~~

The aggregate is the task whose state changes. The event ID identifies this fact across publication retries; generate it once per actual state change, not on every relay attempt. Event envelopes should also carry a correlation ID and causation ID when those relationships help tracing.

Run the following statement inside a transaction. It updates a task and derives the event from the changed row:

~~~sql
BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '5s';
WITH changed AS (
  UPDATE tasks
  SET status = 'doing', version = version + 1, updated_at = now()
  WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
    AND id = '22222222-2222-4222-8222-222222222222'
    AND status <> 'doing'
  RETURNING id, tenant_id, status, version
)
INSERT INTO tutorial_outbox(
  event_id, tenant_id, aggregate_id, aggregate_version,
  event_type, schema_version, payload
)
SELECT '55555555-5555-4555-8555-555555555555',
       tenant_id, id, version, 'TaskStatusChanged', 1,
       jsonb_build_object('taskId', id, 'status', status, 'version', version)
FROM changed
RETURNING event_id, payload;
COMMIT;
~~~

If the task is already doing, no event is inserted because no state change occurs. For repeated experiments, use a fresh event UUID for each new change. A real API should additionally apply its expected-version or transition policy.

Fail the INSERT intentionally in a separate experiment by reusing an existing event ID for a genuinely new update, then roll back. Both the state update and attempted event must be absent. This demonstrates atomicity across the local tables.

## Relay algorithm and failure windows

A minimal single-relay demonstration can read pending rows, publish each with publisher confirmation, and mark published_at. Production relays need claiming or partition ownership, bounded concurrency, retries and recovery. The following is design pseudocode:

~~~text
claim a bounded batch with lease tokens in a short transaction
for each claimed event:
  publish stable event ID and payload
  wait for the broker's configured durable confirmation
  mark published only if the relay still owns the claim
on failure:
  leave recoverable work; retry with a bounded backoff policy
~~~

The schema above demonstrates atomic event creation; implementing this leased relay requires adding lease_token, lease_until and attempt fields or another ownership mechanism. Do not paste the pseudocode as if those fields already exist.

Never mark published before broker acceptance: a crash would lose the event. If publication succeeds but the relay crashes before marking, the event is sent again. Keep the same event_id so consumers can recognize the duplicate. A broker confirmation is not proof that every subscriber has processed the event.

Polling is straightforward and observable. Change data capture can read committed database changes with lower polling overhead, but adds connector operation, log retention, snapshot and schema-evolution concerns. Both require reconciliation and lag monitoring.

## Ordering and contracts

Database insertion order, commit order and relay publication order can differ. Multiple relays using SKIP LOCKED can publish later work while earlier work is busy. If strict per-task order matters, assign that task to one ordered publishing lane or have consumers enforce aggregate versions and recover gaps.

Use event names that describe facts, not a subscriber's implementation step. Publish only the fields consumers need, including tenant identity, entity identity and a schema version. Decide whether the payload is a complete after-state or a delta; deltas may require every preceding event, while after-state events can simplify rebuilding certain projections.

Prefer additive, backward-compatible changes during rolling deployments. Consumers should tolerate unknown fields but reject unsupported meanings. A field's semantic change can be breaking even if its JSON type stays the same. Store contract examples and validate producer/consumer compatibility.

Do not include credentials, access tokens or unnecessary personal data in events. Events can live in logs, dead-letter queues and backups longer than the source record.

## Related patterns

Event-driven integration uses events to connect components. Event sourcing makes an event log the authoritative history from which state is derived. CQRS separates command and query models; it does not require event sourcing or separate services. A saga coordinates a multi-step business workflow with explicit recovery or compensation.

For a saga, define each step's precondition, durable outcome, retry identity and compensating action. Reserving inventory can be released; an already sent email cannot be unsent. Compensation must follow business semantics and may itself fail and need retries.

## Failure tests and operations

Stop the broker, perform a task change, and verify the outbox retains the event. Restart the broker and ensure delivery catches up. Crash the relay after confirmation and verify a duplicate has the same ID. Deliver version 5 after version 6 and check the consumer's version policy.

Monitor oldest unpublished event age, publish failure rate, pending count, relay ownership and consumer lag. Retain enough outbox history for investigation and reconciliation, then archive or delete according to policy. Never delete pending rows merely because they are old.

## Exercises and completion checklist

Build a task-search projection fed by outbox events. Add a reconciliation job comparing authoritative task versions with indexed versions. Write a compatibility plan for a new event field.

Common mistakes include independent database and broker writes, regenerated IDs on retries, treating an outbox as exactly-once delivery, and omitting deletion events.

- [ ] Business mutation and event creation commit together.
- [ ] Relay progress is recoverable and observable.
- [ ] Consumers deduplicate and enforce required version rules.
- [ ] Contract evolution and sensitive-data rules are documented.
- [ ] Replay, reconciliation and deletion are supported.
