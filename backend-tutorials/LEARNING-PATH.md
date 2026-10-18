# Learning path and project milestones

[Complete index](README.md)

Learn by completing observable milestones. The order below is based on dependencies, not on a promised number of days. Spend more time on failed exercises than on rereading definitions. You can use one evolving practice project, but keep destructive database and fault-injection experiments in disposable copies.

## 1. Trace a request

Read foundations 01–03. Explain DNS, a TCP connection, TLS, HTTP, Django middleware, a view, a database query, and a response. Use a terminal to distinguish connection failure from HTTP 404. Write a small Python function with validation and a test.

Done when you can explain where a request waits, where state becomes durable, and why an HTTP timeout can have an ambiguous write outcome.

## 2. Build the local service

Follow the [capstone](09-capstone/01-task-service.md) exactly. Create a virtual environment, start PostgreSQL, configure Django, create migrations, and exercise the JSON endpoints. Read foundations 04–06 alongside it.

Deliverables: working local create/read/update/list operations, a committed migration, repeatable setup instructions, and a record of which checks actually passed. Do not expose the unauthenticated demo publicly.

## 3. Learn relational correctness

Read data 01–04 and Django ORM. Practice foreign keys, constraints, joins, aggregation, transactions, isolation, row locks, optimistic versions, indexes, and EXPLAIN. Create the fixed-ID SQL fixture from the relational tutorial.

Deliverables: two concurrent stale updates produce one accepted change; an invalid status cannot enter the database; a chosen query plan is explained using actual row counts. Know why a Python default differs from a SQL default.

## 4. Add real identity and tenant boundaries

Read security 01–05 and Django custom users. For a new real application, choose the custom user before initial migrations; do not change the capstone's already-migrated user model casually. Add sessions, memberships, object authorization, and protected writes.

Deliverables: anonymous access fails; a read-only member cannot write; users from another tenant cannot retrieve, list, update, export, or subscribe to private data; CSRF tests use real enforcement. Include disabled users and revoked memberships.

## 5. Build a maintainable Django API

Read REST, validation, pagination, OpenAPI, DRF, and Django middleware. Move shared business operations into explicit services where multiple adapters need them. Add a bounded list endpoint and preserve version checks on updates.

Deliverables: an API contract that matches successful and failed responses, tests for invalid input and stale versions, deterministic pagination with ties, and no tenant scope supplied by unchecked client input.

## 6. Learn Django's HTML and operational features

Read forms/templates/admin, static/media/email/localization, and management commands. Add a safe HTML create form, a scoped admin view, console email, and a read-only tenant report command.

Deliverables: CSRF-protected form, no unsafe template interpolation, no arbitrary ModelForm fields, no unintended admin bypass, separate static/media storage, and predictable timezone presentation.

## 7. Move work out of requests

Read all async chapters. Add an outbox event inside the task update transaction, a worker that processes it, and a consumer deduplication key. Install Redis/Celery only when beginning the relevant lab.

Deliverables: duplicate delivery does not duplicate a business effect; a worker crash leaves work recoverable; retries have a budget; a poison message has a visible failure path; overlapping schedules have a policy.

## 8. Measure before optimizing

Read caching, query performance, profiling, load testing, and observability. Measure database and request latency separately. Add one cache with an explicit freshness requirement and invalidation mechanism.

Deliverables: a before/after measurement, bounded cache keys and lifetimes, tenant-safe cache partitioning, a load test with a realistic workload, and evidence that errors did not increase while latency appeared to improve.

## 9. Add one integration style

Choose webhooks, GraphQL, gRPC, or realtime delivery based on a concrete requirement. You do not need every protocol in one service. Use the corresponding API tutorial and share the same authorization and domain operations.

Deliverables: integration contract, authentication boundary, deadline/retry rules, duplicate/reconnection behavior, and a failure test. A successful happy-path request alone is insufficient.

## 10. Deploy a hardened adaptation

Read operations 01–05. Replace demo identity, configure production secrets, turn off DEBUG, serve through a production WSGI/ASGI server, restrict database credentials, package the app, and create a release pipeline. Use staging before public traffic.

Deliverables: reproducible build, reviewed migrations, health/readiness policy, graceful shutdown, structured logs, bounded-cardinality metrics, an SLO, and a rollback/roll-forward plan. Run `manage.py check --deploy` using actual production settings and address relevant findings.

## 11. Prove recovery

Read operations 06–07. Back up a disposable database, restore into a separate target, verify application behavior, and time the procedure. Simulate database unavailability and a growing worker backlog.

Deliverables: measured RPO/RTO, verified restore, incident runbook with commands and expected observations, alert ownership, and a short incident review with concrete corrective actions.

## 12. Design for growth

Read architecture chapters 01–05, NoSQL, and search/analytics. Estimate data volume, peak requests, connection count, queue arrival/service rates, and retention. Consider a modular monolith before distributing ownership across services. The [architecture study map](08-architecture/README.md) connects this foundation to the advanced chapters below.

Deliverables: a capacity model with assumptions, an architecture decision record, a failure-domain map, and one justified scaling change. Explain why an alternative was rejected and what evidence would cause you to revisit the decision.

## 13. Study distributed mechanisms

Follow the advanced sequence in the [study map](08-architecture/README.md): traffic/discovery/caching, data placement/storage/identifiers, consensus/causality, then regions/stream processing. Run each standalone Python model and change one assumption. Keep the distinction between a simulated rule and the persistence, networking, recovery, and security required in a real implementation.

Deliverables: explain why a quorum alone does not prove safe leader election, why concurrent vectors differ from equal vectors, how a shard moves without two writers, and why checkpoints do not deduplicate arbitrary external effects. Calculate a Bloom-filter size, a failover budget, and a backlog drain time. Demonstrate stale discovery, a bad cache key, and a restarted ID allocator in the provided labs.

## 14. Complete worked system designs

Use the [six case studies](11-system-design-case-studies/README.md). Begin with URL shortening and reservations, then study chat, feeds, video, and notifications. For each, draw the critical flow from memory and annotate its authorization, durable commit, timeout, retry, and recovery boundaries.

Deliverables: one design brief per case with requirements, an invariant, API/schema choices, capacity arithmetic, failure scenarios, and one rejected alternative. Implement one bounded slice in a separate Django practice app; use the reservation concurrency lab or notification fake-provider lab to test a failure that matters to that slice.

## 15. Challenge and revise a design

Choose one completed design and increase its load tenfold. Then make half the traffic hit one tenant or key, remove a dependency, and introduce delayed or duplicated events. Use [load testing](06-quality/04-performance-and-load-testing.md), [regional recovery](08-architecture/11-multi-region-and-failover.md), and [stream recovery](08-architecture/14-stream-processing-windows-and-checkpoints.md) as applicable. Run experiments only in disposable environments you control.

Deliverables: a revised capacity budget, an explicit overload policy, a recovery procedure with measurable acceptance criteria, and evidence from the experiments you actually ran. Explain what remains unverified and what measurement would justify the next scaling change.

## Three further practice projects

**Team task platform:** extend the capstone with memberships, comments, attachments, reminders, activity feeds, and realtime updates. Focus on tenant isolation and concurrency.

**Inventory reservation system:** follow the [worked reservation design](11-system-design-case-studies/05-inventory-and-reservations.md) and add products, stock, reservations, expiry, idempotent checkout intents, and a mock payment provider. Focus on atomic invariants, sagas, timeout ambiguity, and reconciliation. Use no real payments in the exercise.

**Data ingestion service:** accept signed webhook events, store an inbox, process through workers, update a search index, and export analytics. Focus on deduplication, schema evolution, replay, backpressure, and retention.

## Self-assessment

You understand a technique when you can explain its mechanism, implement a bounded example, identify a failure it does not solve, verify its behavior, and choose when not to use it. Keep a study log with those five observations for each tutorial. Revisit the [production checklist](PRODUCTION-CHECKLIST.md) before calling any adaptation deployment-ready.
