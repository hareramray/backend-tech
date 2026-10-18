# Client-server systems and the request lifecycle

[Curriculum](../README.md) · [Next: networking and HTTP](02-networking-dns-tls-and-http.md)

## Prerequisites and goals

You need basic programming knowledge and a terminal. By the end, you should be able to trace a request, place responsibilities in the right layer, and explain what happens when only part of an operation succeeds. Use the [Task Service](../09-capstone/01-task-service.md) for the practical steps.

## What a backend does

A backend accepts requests from clients, checks whether the caller may perform an operation, applies business rules, reads or changes durable state, and returns a result. A frontend is one client; other clients include mobile apps, scheduled workers, partner services, command-line tools, and webhook senders. None of their input is automatically trustworthy.

Consider a task update: `PATCH /v1/tasks/{id}` with a new status and expected version. The browser can disable an invalid button, but the backend must still validate the request. A caller can skip the browser entirely. The database adds another boundary: a status constraint protects the data even if an application path has a bug.

Typical components have different jobs:

| Component | Responsibility | Failure to consider |
| --- | --- | --- |
| DNS resolver | Locate the public service endpoint | Stale answer or resolution failure |
| Load balancer/reverse proxy | Route connections, often terminate TLS | Timeout or unhealthy upstream |
| Application process | Enforce policy and business rules | Crash or event-loop blockage |
| Database | Durable state, constraints, transactions | Lock waits or unavailable primary |
| Cache | Reuse expensive results | Stale or evicted entry |
| Queue and worker | Defer work and absorb bursts | Duplicate delivery or growing backlog |
| Object storage | Store large blobs | Upload succeeds but metadata write fails |

These can initially run on one machine. Separating responsibilities does not require separate services.

## Trace one request

1. The client resolves the hostname and opens or reuses a connection.
2. HTTPS authenticates the server and protects transport data. It does not establish application permission.
3. The proxy accepts the request, applies size/time limits, and selects an application instance.
4. Middleware assigns a request identifier, parses a bounded body, and authenticates the caller.
5. Validation converts untrusted values into accepted domain inputs.
6. Authorization checks the operation, tenant, object, and any field restrictions.
7. A service executes the business operation; a repository performs parameterized database queries.
8. A transaction commits the durable change. If an external notification is needed, a durable outbox entry can be committed with it.
9. The handler serializes an allowed response representation and sets a meaningful status.
10. Logs and metrics describe the outcome without exposing credentials or sensitive payloads.

The actual ordering is policy-dependent. For example, checking authentication before detailed body validation avoids exposing information to anonymous callers. Body size limits belong at the earliest feasible point. Streaming endpoints may never buffer a whole request.

## State, persistence, and failure boundaries

An in-memory array belongs to one process. Restarting that process loses the array; adding a second process creates a second independent copy. Use memory for disposable caches or bounded local coordination, and durable storage for state users expect to survive.

A successful database commit and a successful HTTP response are separate events. If the connection breaks after the commit, the client cannot infer whether the write happened. Repeating a non-idempotent operation can create a second record. This is why [idempotency](../04-async/04-idempotency-and-deduplication.md) matters even in a single-server application.

Define the durability contract explicitly. A `202 Accepted` response means work has been accepted according to your contract; it should not imply completion. If you acknowledge a job before placing it in durable storage, a crash can silently lose accepted work.

## Practical lab: follow a task

1. Complete the capstone setup and start its server.
2. Send `GET /health/live/` and `GET /health/ready/`. Observe their different dependency checks.
3. Create a task and record its ID, response status, and `X-Request-Id` header.
4. Read the task back. Map the route to validation, SQL, and response serialization in the code.
5. Stop the local database with `docker compose stop db`; this is a controlled local outage.
6. Repeat the health calls. Liveness should still return success; readiness should report unavailable. Database-backed requests must not pretend to succeed.
7. Restart the database with `docker compose start db` and verify recovery and persistence.

Expected observation: the API process and its dependency can have different health states. Restarting the application should not erase committed tasks. This lab is about ownership of state, not merely memorizing status codes.

## Decompose an operation before coding

Write an operation table for “mark task done”:

| Question | Example decision |
| --- | --- |
| Input | Task ID, desired status, expected version |
| Identity | Verified user and tenant membership |
| Invariant | Only a member with write permission can change this task |
| Concurrency | Conditional update using the expected version |
| Durable result | Updated status and incremented version |
| Secondary effect | Activity notification through an outbox |
| Failure contract | Invalid input, absent task, forbidden action, conflict, dependency failure |

This exercise exposes missing decisions before framework code obscures them. A controller should not independently invent business rules already owned by a service.

## Common mistakes

- Trusting a tenant ID or role supplied by the client without checking membership.
- Sending success before the promised durability boundary.
- Treating every exception as a client error, or returning database stack traces.
- Adding services before there is a clear ownership or scaling reason.
- Assuming one successful development request proves restart safety or concurrency correctness.
- Treating a timeout as proof that the remote action did not happen.

## Exercises and completion checklist

Draw the path of a file upload, a login, and a scheduled email. Mark where each operation becomes durable. For each path, interrupt one component and describe the observable result and recovery action.

- [ ] I can distinguish process memory from durable state.
- [ ] I can locate authentication, authorization, validation, and transaction boundaries.
- [ ] I can explain an ambiguous write outcome.
- [ ] I can trace a request using an identifier without logging its secrets.
- [ ] I can explain why a worker is a backend client too.

## Further reading

HTTP defines request/response semantics independently of your application architecture; use the standard to resolve protocol questions. [HTTP Semantics, RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html). Continue with [modular architecture](../08-architecture/01-modular-monoliths-and-domain-design.md).
