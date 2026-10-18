# Idempotency and deduplication

Prerequisites: [transactions](../02-data/02-transactions-and-concurrency.md), [message delivery](02-message-queues-and-streams.md), the [course map](../README.md), and PostgreSQL. The SQL lab is standalone.

## Objectives

Give retries a stable operation identity, atomically prevent duplicate database effects, distinguish duplicate requests from conflicting reuse, and manage the lifecycle of deduplication records.

## Retry safety is a business property

An operation is idempotent when repeating it has the same intended effect as executing it once. Setting a task status to done can be idempotent; appending a comment or incrementing a balance usually is not unless protected by a stable identity.

HTTP method semantics do not automatically make an implementation safe. A PUT handler that sends a new email on every retry can duplicate a side effect even if the resource ends in the same state. Similarly, a broker's message ID is useful only if it remains stable across producer retries for the same logical event.

Distinguish three identities: a request attempt, a logical operation, and an entity. Each network attempt can have a new trace/request ID while sharing the logical idempotency key. An entity can receive many legitimate operations; deduplicating solely by entity ID would incorrectly suppress later work.

## Design the key contract

Scope a key by authenticated tenant, actor where relevant, endpoint or operation type, and a client-supplied random value. Define a maximum length, retention duration and reuse policy. Associate the key with a fingerprint of the meaningful validated request fields.

The same key with a different fingerprint should produce a clear conflict, not silently return an unrelated result. Canonicalization matters: hashing raw JSON bytes treats whitespace and property order as differences. A stable canonical representation can normalize irrelevant formatting while preserving meaningful values.

Perform authentication and authorization before returning a cached response. Idempotency storage is not a way to bypass current access checks or fetch another tenant's historical result. Avoid storing sensitive response fields unnecessarily.

## Lab: atomically deduplicate a database effect

Create two standalone tables:

~~~sql
CREATE TABLE tutorial_processed_events (
  consumer_name text NOT NULL,
  tenant_id uuid NOT NULL,
  event_id uuid NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer_name, tenant_id, event_id)
);
CREATE TABLE tutorial_daily_counts (
  tenant_id uuid NOT NULL,
  metric_date date NOT NULL,
  count bigint NOT NULL DEFAULT 0 CHECK (count >= 0),
  PRIMARY KEY (tenant_id, metric_date)
);
~~~

Run the following complete transaction twice:

~~~sql
BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '5s';
WITH first_delivery AS (
  INSERT INTO tutorial_processed_events(consumer_name, tenant_id, event_id)
  VALUES (
    'daily-task-counter',
    '11111111-1111-4111-8111-111111111111',
    '66666666-6666-4666-8666-666666666666'
  )
  ON CONFLICT DO NOTHING
  RETURNING tenant_id
)
INSERT INTO tutorial_daily_counts(tenant_id, metric_date, count)
SELECT tenant_id, DATE '2026-01-01', 1 FROM first_delivery
ON CONFLICT (tenant_id, metric_date)
DO UPDATE SET count = tutorial_daily_counts.count + EXCLUDED.count;
COMMIT;

SELECT * FROM tutorial_daily_counts
WHERE tenant_id = '11111111-1111-4111-8111-111111111111';
~~~

Expected count is one after both executions. The uniqueness constraint arbitrates concurrent delivery; a prior SELECT followed by INSERT would leave a race. The deduplication insert and business effect share a transaction, so a failure cannot leave “processed” recorded without the corresponding count. PostgreSQL constraints enforce uniqueness across concurrent writers. [Constraint documentation](https://www.postgresql.org/docs/current/ddl-constraints.html)

Use a different consumer_name for an independent projection that should also process the event. That is why consumer identity belongs in the key.

## Django request handling pattern

For an API, use a database model with a unique key on tenant, operation and idempotency_key, plus fingerprint, state, response status/body and timestamps. This is design pseudocode; those model fields are not part of the capstone automatically:

~~~text
authenticate and authorize
validate payload and compute canonical fingerprint
transaction.atomic():
  try to insert idempotency record with unique scoped key
  if conflict:
    load the committed record
    reject mismatched fingerprint
    return stored completed response or bounded in-progress response
  perform the database business change
  store the stable response and completed state
commit
return response
~~~

For concrete Django code, handle IntegrityError using a nested atomic block/savepoint or an atomic INSERT ON CONFLICT strategy; catching a database error inside a broken transaction and immediately querying can fail. Keep the transaction short and do not include slow remote calls.

If the work is asynchronous, persist the idempotency record and job together, return the same operation identifier on retries, and let callers poll status. If concurrent attempts encounter in-progress work, define whether they wait briefly or receive a conflict/accepted response. Never wait indefinitely while holding an application worker.

## External effects and ambiguous outcomes

A local processed_events table cannot atomically cover a remote payment, email or storage service. Forward the same stable operation key to a provider that supports idempotency, or record intent and reconcile the provider's state. A timeout after sending does not prove the operation failed.

For a webhook, verify the signature over the correct original payload, validate timestamp/replay policy, identify the event, and persist it before processing asynchronously. Deduplication and signature verification solve different problems: one prevents repeated effects, the other establishes authenticity under the provider's protocol.

For an outbound webhook, persist attempts, sign each delivery as specified, keep a stable event ID, and allow receivers to deduplicate. Retry transient errors with backoff; permanently invalid destinations need review rather than an endless retry loop.

## Retention and reconciliation

Deduplication records consume space. Retain them longer than every relevant retry and replay window, or accept and document the consequences after expiry. Deleting an event's deduplication marker while the event can still replay re-enables the effect.

Some operations need effectively permanent business uniqueness, such as an external provider reference stored uniquely on the resulting record. A short-lived API response cache and a permanent business constraint serve different purposes.

Track duplicate suppression, fingerprint conflicts, in-progress age and ambiguous provider outcomes. A reconciliation job should compare intent with authoritative external results and repair missing local state without creating a second effect.

## Failure tests, mistakes and exercises

Run the SQL transaction concurrently in two sessions and verify count one. Force the count update to fail and verify the processed marker rolls back. Crash after commit but before HTTP response and retry with the same key. Reuse that key with another payload and verify rejection.

Common mistakes include storing keys only in process memory, using a check-then-write race, acknowledging before committing, expiring keys too early, and logging full request bodies while computing fingerprints.

Design an idempotent task-creation endpoint and a webhook receiver. Specify the uniqueness scope, replay window, in-progress response and recovery for a provider timeout.

- [ ] Attempts share a stable logical operation identity.
- [ ] Key scope includes tenant and operation.
- [ ] Conflicting payload reuse is rejected.
- [ ] Deduplication and local effects commit atomically.
- [ ] External ambiguous outcomes have reconciliation.
- [ ] Retention covers the supported retry and replay windows.
