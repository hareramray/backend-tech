# Microservices and service communication

## Purpose and prerequisites

Microservices introduce independently deployable boundaries around capabilities. Their value can include independent scaling, team ownership, release cadence, or fault isolation. Their cost includes network failures, distributed state, contract evolution, identity propagation, deployment coordination, and more operational work.

Prerequisites: [modular domain design](01-modular-monoliths-and-domain-design.md), HTTP, background jobs, and database transactions. This tutorial extends the [task service](../09-capstone/01-task-service.md) conceptually with a notification service. Code snippets are pseudocode contracts unless explicitly identified as executable.

## Identify a reason to split

A service boundary should own a coherent capability and its data writes. If deploying tasks requires simultaneous deployment of memberships and notifications every time, your services are not operationally independent. Shared database tables can be a migration step, but direct writes from several services weaken ownership and make schema changes hazardous.

Extract notifications only if its workload, reliability, ownership, or release needs justify it. Keep immediate task correctness in the task service. A task save should not depend on the email vendor responding successfully when notification can happen later.

Account for the whole dependency chain. If five mandatory independent dependencies each succeed with probability 0.999 for a request, their combined success is approximately `0.999^5 = 0.995`, under the simplifying independence assumption. Real failures are often correlated, making that estimate incomplete. More services do not automatically mean more availability.

## Choose communication by semantics

| Mechanism | Appropriate use | Failure concern |
|---|---|---|
| HTTP/JSON | Public or straightforward request/reply APIs | Timeouts, versioning, payload costs |
| gRPC | Typed internal request/reply or streaming | Deadlines, compatibility, proxy support |
| Queue | Durable work distribution | Redelivery, poison jobs, lag |
| Event stream | Ordered history within partitions, multiple consumers | Partition key, replay, retention |
| Webhook | Push to an external subscriber | Untrusted destination, retries, signing |

Synchronous communication gives the caller an immediate response but couples availability and latency. Asynchronous communication decouples completion time but introduces a visible pending state. A `202 Accepted` response means work has been accepted according to the API contract, not that the final business effect occurred.

Keep messages explicit. A command asks a particular owner to act; an event records something that happened. `SendTaskNotification` and `TaskCompleted` should not be treated as interchangeable merely because both are JSON.

## Timeouts, retries, and budgets

Set an end-to-end deadline and allocate time for connection setup, processing, downstream calls, and response delivery. A downstream timeout should leave enough time for the caller to return a useful result. Propagate cancellation where supported, while recognizing that a canceled request may already have committed remotely.

Retry only failures that may be transient and only operations whose retry semantics are safe. Use a bounded attempt count, exponential backoff with jitter, and a shared retry budget. Multiple retrying layers multiply traffic during an outage. A timeout does not prove that the remote side did nothing. [AWS guidance on timeouts and retries](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).

An idempotency key identifies one logical operation. Scope it to the authenticated caller or tenant and operation, bind it to a request payload digest, and durably retain the result for the promised retry period. The same key with a different payload must not silently reuse the old operation. Merely generating a different key on every retry defeats deduplication.

## Resilience mechanisms

Circuit breakers temporarily stop calls to an unhealthy dependency and probe recovery cautiously. Bulkheads isolate concurrency or resource pools so a failing email provider cannot consume every database worker. Bounded queues reject or defer excess work before memory exhaustion. These controls complement timeouts; none fixes incorrect business semantics.

Fallbacks require care. Serving slightly stale task counts can be acceptable; inventing a successful payment or permission grant is not. Define degradation by operation and data class. A cached authorization decision may have a revocation delay that is unacceptable for an administrative change.

Service discovery and load balancing decide where requests go; they do not establish authority. Authenticate services using workload identity, mTLS, or an appropriate token mechanism, and authorize the requested capability. If user identity is propagated, bind it to a trusted service request; never trust an arbitrary `X-User-Id` arriving from the internet.

## Reliable publication and delivery

The task service cannot atomically commit PostgreSQL data and an unrelated broker publish without additional coordination. Publishing after commit can lose an event on a crash. Publishing before commit can announce an operation that later rolls back.

A transactional outbox writes the task change and an event record in the same database transaction. A publisher later sends events and records progress. Publishing may happen more than once if it crashes after sending but before marking success; consumers must deduplicate or perform idempotent effects. The [events and sagas tutorial](04-cqrs-event-sourcing-and-sagas.md) supplies an outbox schema and failure exercise.

Avoid making payloads mirror every database column. Use a versioned integration contract containing necessary identifiers and meaning. Include event ID, tenant ID, occurrence time, schema version, and correlation information. Do not include passwords, session tokens, or unnecessary personal data.

## Lab: isolate notification failure

1. Start with task mutation and notification in one request. Use a local stub notification adapter that can succeed, delay, or fail. Do not use a real mail account.
2. Measure request behavior when the stub delays beyond the task endpoint's latency budget. Record whether a task was saved despite the timeout.
3. Change the design so task mutation and notification intent commit together through an outbox. Return the committed task result without waiting for email delivery.
4. Have a worker read the intent, call the stub with a stable operation ID, and mark progress. Bound concurrent deliveries and retry attempts.
5. Simulate a crash immediately after the stub accepted delivery but before progress is saved. Redeliver the job. Confirm your documented duplicate policy: provider idempotency if available, a reconciliation path, or explicitly accepted duplicate risk.
6. Leave the stub unavailable. Task creation should remain available within its independent dependencies; notification lag should rise and alert.
7. Recover the stub and drain backlog at a bounded rate. Avoid overwhelming the vendor or starving fresh high-priority work.
8. Deploy a consumer that understands event version one and then an additive version two. Verify old pending events remain readable.

Expected result: each failure has a visible state, deadline, owner, and recovery path. A failure is a silently dropped notification or a retry loop that consumes unbounded resources.

## Contract evolution and operations

Use additive changes when possible, tolerate unknown fields, and do not change the meaning of an existing field silently. Deploy consumers before producers when adding a required behavior. Version incompatible contracts deliberately and observe old-client usage before removal.

Propagate correlation IDs and tracing context without treating them as trusted identity. Measure request rate, error rate, latency distributions, saturation, queue age, retries, and dead-letter volume. Maintain service ownership, runbooks, and an inventory of mandatory dependencies.

## Exercises and completion checklist

Design a task export service. Choose which interactions are synchronous, which are asynchronous, and how the user observes completion. Then estimate the cost of a deployment, incident, and schema migration before deciding whether extraction is justified.

- [ ] Each service has coherent ownership and a reason for independence.
- [ ] Calls have deadlines, cancellation, and bounded retries.
- [ ] Duplicate requests and messages have explicit semantics.
- [ ] Service identity and user authorization remain distinct.
- [ ] Backlogs, outages, and contract migrations are observable.
- [ ] Recovery does not depend on an operator guessing which steps completed.

Related: [distributed consistency](03-distributed-systems-and-consistency.md) and [capacity planning](05-system-design-and-capacity-planning.md).
