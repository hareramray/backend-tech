# Transactions and concurrency

Prerequisites: [relational modeling](01-relational-modeling-and-sql.md), including its fixed-ID seed step, the [course map](../README.md), and the initialized [capstone database](../09-capstone/01-task-service.md). Open two separate PostgreSQL sessions for the concurrency exercises.

## Objectives

Protect invariants under simultaneous requests, choose between atomic updates, optimistic concurrency and locks, and recover from retryable transaction failures without duplicating external effects.

## What a transaction promises

A transaction groups database work into a commit or rollback boundary. Atomicity prevents a partially committed group; consistency is the application's responsibility to express through constraints and correct transitions; isolation controls interference; durability concerns committed data surviving the failures covered by the database configuration.

A successful query is not necessarily a successful transaction. A network failure during COMMIT can leave the client uncertain whether the commit occurred. That uncertainty is why an operation identity and reconciliation are often necessary. Transactions cannot automatically include arbitrary HTTP requests, emails, or object storage writes.

PostgreSQL's default Read Committed isolation uses a new snapshot for each statement. Repeatable Read provides a stable transaction snapshot but can still allow serialization anomalies. Serializable detects outcomes that cannot correspond to an appropriate serial order and can require a transaction retry. [PostgreSQL isolation documentation](https://www.postgresql.org/docs/current/transaction-iso.html)

The practical questions are: which invariant must hold, which rows represent it, and which writes can race? A task's version is a single-row invariant. A quota spanning many task rows is a multi-row invariant. A condition checked before a later write is vulnerable unless the chosen isolation or locks protect the condition.

## Lab 1: demonstrate and prevent a stale update

Read the same task in both sessions:

~~~sql
SELECT id, title, status, version
FROM tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222';
~~~

Suppose both reads return version 1. In session A, run:

~~~sql
UPDATE tasks
SET status = 'doing', version = version + 1, updated_at = now()
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222'
  AND version = 1
RETURNING id, status, version;
~~~

In session B, execute the same statement with status `done` and the original version. It must return zero rows. The client can reload and reconcile rather than silently replacing a newer value. Use the actual initial version if it was not 1. An API can expose a version or ETag and map this condition to an explicit conflict response.

Optimistic concurrency is effective when conflicts are uncommon and user intent can be retried or reconciled. It does not make an arbitrary sequence of queries atomic. Every writer participating in the invariant must honor the version policy.

## Lab 2: inspect locking and bounded waiting

In session A:

~~~sql
BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '5s';
SELECT id FROM tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222'
FOR UPDATE;
~~~

Leave A open briefly. In session B:

~~~sql
BEGIN;
SET LOCAL lock_timeout = '1s';
UPDATE tasks SET title = 'A conflicting writer'
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222';
ROLLBACK;
~~~

B should time out while waiting. Roll back A promptly. A row lock coordinates writers; it does not prevent ordinary snapshot reads from observing a previously committed row. Long-lived transactions increase contention and can hold back cleanup, so never wait for user input or a remote service while a transaction remains open.

When locking several records, use a consistent ordering of IDs to reduce deadlocks. Ordering reduces a common cause but does not eliminate all deadlocks; retain a retry policy. SKIP LOCKED is useful for queue workers that can skip busy work, but its incomplete view is unsuitable for a normal report. [PostgreSQL SELECT locking clauses](https://www.postgresql.org/docs/current/sql-select.html)

## Lab 3: make a multi-statement operation atomic

Create this extension table once:

~~~sql
CREATE TABLE task_status_history (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id uuid NOT NULL,
  task_id uuid NOT NULL,
  new_status text NOT NULL CHECK (new_status IN ('todo', 'doing', 'done')),
  recorded_at timestamptz NOT NULL DEFAULT now(),
  FOREIGN KEY (tenant_id, task_id) REFERENCES tasks(tenant_id, id)
);
~~~

Use one acquired database connection for BEGIN, all statements, and COMMIT. The following is transaction pseudocode, not a runnable driver implementation:

~~~text
client = acquire connection
try:
  begin
  set local statement and lock timeouts
  update task with tenant, id, and expected version
  if affected rows != 1: raise conflict
  insert matching history row
  commit
catch error:
  rollback if connection is usable
  classify error
finally:
  release or discard connection according to driver guidance
~~~

In Django, use `transaction.atomic()` around the complete operation and use the same database alias for every query. With direct psycopg, use one connection and its transaction context. Savepoints can roll back a subsection, but they are not independent commits; an outer rollback still discards their changes. [Django transaction guidance](https://docs.djangoproject.com/en/5.2/topics/db/transactions/)

Retry the complete transaction for recognized serialization failures or deadlocks, using a small attempt limit, jitter and an overall deadline. Do not automatically retry permission errors, validation failures, or every connection error. A connection loss at commit needs idempotency or a status lookup because the result may be ambiguous.

## Choosing a technique

Use an atomic conditional UPDATE for counters, state transitions and stock checks that fit one statement. Use a version predicate when clients edit stale representations. Use row locks for short coordinated changes to existing rows. For predicates spanning rows, consider a locked parent or quota row, a constraint, or Serializable isolation with retries. Advisory locks can coordinate application-defined resources, but require an explicit ownership and release policy.

Distributed transactions and two-phase commit add operational coupling. Often a local transaction plus [outbox](../04-async/03-event-driven-systems-and-outbox.md) and compensation provides a better recovery story. Compensation is a new business operation, not a magical rollback of history.

## Failure tests, mistakes and exercises

Deliberately fail the history INSERT after updating the task, then roll back: neither change should persist. Kill a worker after commit but before replying and observe the ambiguous client result. Run concurrent requests until a version conflict appears. Track lock wait time, deadlocks, transaction duration and retry counts.

Common mistakes include locking too much data, retrying only the last statement, retrying forever, trusting a read-before-write uniqueness check, and calling a payment provider inside a retried transaction.

Implement a task limit using a tenant quota row; compare its behavior with a count-then-insert implementation under concurrency. Describe what happens when two writers both see one remaining slot.

- [ ] The invariant and transaction boundary are written down.
- [ ] All transaction statements share one connection.
- [ ] Locks and queries have bounded deadlines.
- [ ] Retryable failures retry the whole operation.
- [ ] External effects are idempotent or deferred through an outbox.
- [ ] Concurrent and ambiguous-outcome cases are tested.
