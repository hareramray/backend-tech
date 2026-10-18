# Distributed systems and consistency

## Purpose and prerequisites

A system is distributed when correctness depends on communication between independently failing components. An application plus a database is already distributed. Adding replicas, queues, caches, and regions introduces additional histories that may disagree temporarily or permanently.

Prerequisites: transactions, [service communication](02-microservices-and-service-communication.md), and the [PostgreSQL capstone](../09-capstone/01-task-service.md). The SQL lab uses two interactive database sessions and the existing task fixture. Run it only on disposable data.

## Begin with ambiguity

A timeout means the caller did not obtain a timely answer. It does not establish whether the server received, executed, committed, or responded to the operation. A retry can therefore duplicate an effect. A process can pause rather than crash, and a network partition can resemble a slow dependency from one side.

Write guarantees in terms of observations. “The update is reliable” is vague. “After an acknowledged update, a read through the primary returns that version or a later version” is testable. “The dashboard may lag by up to ten seconds under normal operation, and displays freshness” is a different contract.

## Consistency models

| Model or guarantee | Meaning in a useful simplified form |
|---|---|
| Linearizability | Each operation appears to take effect at one point between call and response, respecting real-time order |
| Sequential consistency | Operations fit one order preserving each client's order, without the same real-time requirement |
| Causal consistency | Causally related operations remain ordered; unrelated operations may be seen differently |
| Read-your-writes | A session can observe its own completed writes |
| Monotonic reads | A session does not move backward to an older observed version |
| Eventual convergence | Replicas converge under specified conditions after new updates stop |

Eventual consistency is not a promise that arbitrary conflicting updates magically merge correctly. Specify conflict rules, replication assumptions, and how lag or divergence is detected. Last-write-wins based on wall clocks may discard meaningful work, especially when clocks differ.

Database isolation describes interactions among transactions. Serializable execution produces results equivalent to some serial order; it is not automatically the same guarantee as real-time linearizability. PostgreSQL's default Read Committed gives statements snapshots appropriate to that level, while stronger isolation can require transaction retries. [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html).

## CAP without the slogan

During a network partition, a distributed register cannot guarantee both linearizable consistency and availability to every nonfailed participant under the theorem's model. CAP availability is a theoretical response property, not simply your monthly uptime percentage. “Choose any two” obscures that partitions are a failure condition to handle, not a feature you can remove by preference. [Gilbert and Lynch, Perspectives on the CAP Theorem](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf).

Choose behavior per operation. A task-comment feed might accept locally available writes and reconcile. A scarce-resource allocation might refuse a write without sufficient coordination. A single application can have both choices for different data. Outside partitions, latency, throughput, durability, and operational complexity still matter.

## Replication, quorums, and leaders

A leader can order writes and replicate a log. Followers may lag and can return stale reads. Synchronous replication can improve acknowledged durability but adds latency and dependency on replicas. Asynchronous replication can acknowledge before replicas receive the write, leaving a data-loss window if the primary fails.

A common quorum intuition is that read and write sets overlap when `R + W > N`. Overlap alone does not prove linearizability: membership changes, concurrent writes, sloppy quorums, conflict handling, and incomplete propagation all matter. Use a storage system with documented semantics rather than inventing a replication protocol from this arithmetic.

Consensus protocols coordinate an agreed log despite specified failures; they do not eliminate the need for application transactions, idempotency, backups, or correct membership configuration. A three-voter majority configuration can tolerate one unavailable voter for progress under its assumptions, but correlated region or network failures may affect several voters at once.

## Concurrency controls and fencing

Optimistic concurrency uses an expected version. Pessimistic concurrency locks data while making a decision. Both work best when the protected invariant is explicit and transactions stay short. Distributed locks introduce lease expiry and paused-owner hazards.

Suppose worker A holds a lease, pauses, and loses it; worker B acquires the lease and writes; A resumes and writes stale state. The existence of a lock service did not stop A. A fencing token is a monotonically advancing generation that the protected resource checks, rejecting obsolete owners. If the resource cannot enforce fencing or idempotency, the lock alone cannot guarantee correctness.

Use monotonic clocks for elapsed-time measurements within a process. Wall clocks support timestamps but can jump or skew; they are not a universal ordering authority. Sequence numbers, database versions, or logical clocks can express relationships that timestamps cannot safely establish.

## Lab: observe a lost-update defense

First inspect the fixture and record its current version:

```sql
SELECT id, status, version
FROM tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222';
```

1. Open sessions A and B and read the same version, for example `7`. Use the actual value in your fixture rather than assuming seven.
2. In A, execute the following update with that version:

```sql
UPDATE tasks
SET status = 'doing', version = version + 1, updated_at = now()
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222'
  AND version = 7
RETURNING version;
```

3. In B, attempt a different update with the same expected version. Expect zero rows after A commits. Translate that into a conflict for an otherwise authorized existing resource.
4. Repeat with an explicit transaction in A and `SELECT ... FOR UPDATE`. Have B attempt to update the same row. Observe blocking until A commits or rolls back; set a short local lock timeout so the experiment cannot wait indefinitely.
5. Roll back the teaching transaction when appropriate and inspect final state. A timeout or canceled query must not be mistaken for proof that every prior operation rolled back.
6. Implement the version predicate in Django using `filter(..., version=expected).update(version=F("version") + 1, ...)`. Check the returned update count. Wrap multi-statement invariants in `transaction.atomic()`.

Expected result: conflicting writes produce explicit conflict or lock behavior, not silent overwrite. A single-row version check does not protect a multi-row invariant such as “at most ten active tasks”; that requires a separate coordinated design.

## Lab: reason about stale reads

Without needing a real replica, create a fake read adapter that returns the previous task version after a successful write. Observe the UI behavior. Then select a contract: route post-write reads to the primary, return the committed representation directly, or wait for a replica to reach a required version with a bounded timeout.

Test delayed, duplicated, and reordered event delivery to a projection. Store or compare per-aggregate versions so an old event cannot overwrite newer state. Define how missing sequence numbers are recovered rather than silently skipped.

## Exercises and completion checklist

Design a tenant quota invariant under concurrent task creation. Compare a locked quota row, serializable transactions with retries, and an asynchronous approximate quota. Specify which user-visible promises each can keep.

- [ ] Every important operation has an explicit consistency contract.
- [ ] Timeouts and retries account for uncertain completion.
- [ ] Replica lag and read-your-writes behavior are tested.
- [ ] Version checks or locks protect the actual invariant.
- [ ] Leases have a stale-owner defense where correctness requires it.
- [ ] Partition and recovery behavior are documented per operation.

Continue with [CQRS, event sourcing, and sagas](04-cqrs-event-sourcing-and-sagas.md) and [capacity planning](05-system-design-and-capacity-planning.md).
