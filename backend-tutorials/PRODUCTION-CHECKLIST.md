# Production-readiness checklist

[Tutorial index](README.md) · [Learning path](LEARNING-PATH.md)

Use this checklist for an actual deployment adaptation, not as a claim that the local capstone already implements every item. Record evidence and an owner for each applicable item. Mark an item not applicable only with a reason. Production requirements depend on the product, data, users, and availability goals.

## Application and Django configuration

- [ ] Demo mode, fixed demo identity, and demo CSRF exemptions are removed.
- [ ] `DEBUG=False` and `ALLOWED_HOSTS` reflect the actual deployment.
- [ ] Secrets come from a restricted deployment mechanism and are absent from source/images/logs.
- [ ] HTTPS is enforced with correctly trusted proxy headers.
- [ ] Cookie Secure, HttpOnly, SameSite, and CSRF settings fit the client architecture.
- [ ] A production WSGI/ASGI server replaces `runserver`.
- [ ] Runtime, framework, driver, and package versions are recorded and supported for the intended deployment.
- [ ] `manage.py check --deploy` runs against production settings and relevant findings are addressed.
- [ ] Static files and private user media have distinct serving and authorization policies.
- [ ] Graceful shutdown has a deadline and handles long-lived connections/workers.

See [deployment](07-operations/04-deployment-cloud-and-orchestration.md) and [Django assets](10-django/05-static-media-email-and-localization.md).

## Identity, permissions, and abuse controls

- [ ] User-model choices and authentication lifecycle are documented.
- [ ] Password storage, reset, verification, recovery, and session rotation use established framework/provider mechanisms.
- [ ] Tenant membership is checked independently of caller-supplied tenant IDs.
- [ ] List, retrieve, create, update, delete, export, admin, jobs, and subscriptions enforce permission.
- [ ] Revocation, disabled users, stale roles, and long-lived sessions are tested.
- [ ] OAuth/OIDC checks include state/nonce/PKCE and token validation as appropriate to the flow.
- [ ] Login, reset, expensive queries, and integrations have appropriate rate/resource controls.
- [ ] Security-relevant actions have access-controlled audit records and retention.
- [ ] Threat modeling covers the actual trust boundaries and high-value abuse paths.
- [ ] A secret/key rotation and suspected-compromise response is rehearsed.

See [security](03-security/01-security-foundations-and-threat-modeling.md).

## Data and concurrency

- [ ] Database credentials follow least privilege; runtime and migration authority are separated.
- [ ] Constraints enforce required uniqueness, references, states, and numeric ranges.
- [ ] Raw SQL is parameterized, and dynamic identifiers/operators are allowlisted.
- [ ] Money, timestamps, Unicode, identifiers, and null semantics preserve domain meaning.
- [ ] Transactions are short and stay on the intended connection.
- [ ] Concurrent writes have explicit version/lock/invariant behavior.
- [ ] Queries are tenant-scoped; RLS, if used, is tested with the actual non-bypass runtime role.
- [ ] Index choices are supported by representative plans and measured write/storage cost.
- [ ] Pool sizes, replica counts, workers, and database connection limits are budgeted together.
- [ ] Migrations use reviewed expand/backfill/contract steps where rolling compatibility requires them.
- [ ] Retention, deletion, archives, and backups agree on data lifecycle requirements.

See [data](02-data/01-relational-modeling-and-sql.md) and [ORM](10-django/01-models-orm-and-querysets.md).

## API and integrations

- [ ] Contracts match actual requests, responses, headers, errors, and media types.
- [ ] Body size, nesting, string length, page size, batch size, and query complexity are bounded.
- [ ] Pagination uses a deterministic order and preserves cursor precision/scope.
- [ ] Error responses expose useful codes without secrets or internals.
- [ ] Timeouts cover the intended operation phases; retries have an overall budget.
- [ ] Ambiguous writes use idempotency/reconciliation where duplicate effects matter.
- [ ] Webhook signatures verify the exact provider-defined bytes and replay policy.
- [ ] Outbound integration destinations are protected against SSRF and unsafe redirects.
- [ ] Realtime delivery has replay, gap, disconnect, and slow-consumer behavior.
- [ ] API evolution and deprecation have client compatibility evidence.

See [APIs](05-apis/01-rest-and-resource-design.md).

## Jobs, queues, caching, and files

- [ ] Job acceptance is durable before success is promised.
- [ ] Outbox/inbox transactions and deduplication scopes are correct.
- [ ] Consumers tolerate duplicate delivery and crashes between side effects and acknowledgements.
- [ ] Retry, backoff, jitter, poison-message handling, and dead-letter review have owners.
- [ ] Scheduler timezone and overlapping-execution policy are explicit.
- [ ] Cache freshness, invalidation, key scope, and outage behavior are documented.
- [ ] Locks have ownership checks and fencing where stale ownership could corrupt state.
- [ ] Uploads are bounded, authorized, quarantined when appropriate, and stored with server-controlled keys.
- [ ] File cleanup handles partial failures and database/object-store non-atomicity.
- [ ] Email and notification retries cannot silently duplicate high-impact actions.

See [async systems](04-async/01-jobs-and-scheduling.md).

## Quality and release evidence

- [ ] Pure rules, database semantics, HTTP behavior, and integration contracts have appropriate checks.
- [ ] Tests use disposable databases and nonproduction credentials.
- [ ] Permission and concurrency failures are tested, not just happy paths.
- [ ] Representative load testing measures throughput, tail latency, errors, and saturation together.
- [ ] Formatting/linting, dependency review, and reproducible installs run in CI.
- [ ] The artifact promoted to production is the one that passed verification.
- [ ] Rollback/roll-forward accounts for database and event-schema compatibility.
- [ ] Feature flags have owners, safe fallbacks, and cleanup dates.
- [ ] Infrastructure changes are reviewed and drift is observable.

See [quality](06-quality/01-testing-strategy-and-unit-tests.md) and [CI/CD](07-operations/03-ci-cd-and-release-strategies.md).

## Operations and recovery

- [ ] Logs avoid credentials and sensitive payloads and use consistent correlation IDs.
- [ ] Metrics use bounded labels and cover request rate, errors, latency, and saturation.
- [ ] Traces are sampled and redacted with clear ownership.
- [ ] SLOs describe a user-visible population, target, and time window.
- [ ] Alerts identify actionable symptoms and link to runbooks.
- [ ] Liveness, readiness, startup checks, and load-balancer draining have distinct purposes.
- [ ] Backup retention, encryption, access, and failure monitoring are configured.
- [ ] A restore into an isolated target has passed integrity and application checks.
- [ ] RPO/RTO are measured, not assumed from backup settings.
- [ ] An incident owner, escalation path, recovery procedure, and communication mechanism exist.
- [ ] Capacity and cost assumptions are reviewed against observed growth.

See [observability](07-operations/05-observability-and-slos.md), [recovery](07-operations/06-reliability-backups-and-disaster-recovery.md), and [incident response](07-operations/07-incident-response-and-production-runbooks.md).

## Advanced architecture, where applicable

- [ ] L4/L7 routing, endpoint discovery, draining, retries, and stale configuration have tested failure behavior.
- [ ] CDN cache keys preserve representation and authorization boundaries; revocation and origin-outage behavior are explicit.
- [ ] Shard ownership changes fence stale writers and define copy, catch-up, cutover, validation, and rollback.
- [ ] Consensus/lease assumptions include membership changes, stale leaders, restart behavior, and protected external resources.
- [ ] Conflict resolution preserves the business invariant; tombstone reclamation accounts for offline and restored replicas.
- [ ] Storage budgets include compaction, write/read amplification, free-space headroom, and maintenance load.
- [ ] Stream watermarks, late events, checkpoints, source retention, sink effects, and replay have compatible contracts.
- [ ] Regional promotion fences old writers, measures actual RPO/RTO, verifies dependent services, and plans failback.
- [ ] Distributed ID allocation handles worker reuse, process restart, clock rollback, overflow, and exposed ordering information.
- [ ] External-provider reconciliation accounts for deduplication expiry, aged replays, and ambiguous outcomes.

Use the [architecture study map](08-architecture/README.md) and [case-study review method](11-system-design-case-studies/README.md) to select relevant checks. These items do not require every application to adopt sharding, multiple regions, or a stream processor.

## Evidence record template

For each material requirement, record: owner, environment, date, exact check, observed outcome, known limitation, and next review trigger. An unchecked box is useful information. A checked box without evidence can hide unfinished work.
