# Backend engineering: Python, Django, and PostgreSQL

This is a comprehensive core and advanced backend curriculum with a separate Markdown tutorial for every topic in the index below. It starts with request handling and builds toward reliable, secure, operable systems. The main implementation stack is **Python + Django + PostgreSQL**, as requested. Redis, Celery, DRF, GraphQL, gRPC, containers, and cloud tools are introduced where their techniques apply.

Backend engineering is open-ended: no finite collection can contain every vendor feature, framework, or future technique. This collection makes its coverage explicit instead of claiming literal completeness. Every indexed tutorial contains explanations, mechanisms or tradeoffs, practical steps/examples, failure considerations, exercises, and verification guidance. Official references provide deeper specification detail.

## Start here

- [Learning path and project milestones](LEARNING-PATH.md)
- [Complete local Django/PostgreSQL Task Service](09-capstone/01-task-service.md)
- [System-design study map: foundations and advanced chapters](08-architecture/README.md)
- [Six worked system-design case studies](11-system-design-case-studies/README.md)
- [Glossary](GLOSSARY.md)
- [Production-readiness checklist](PRODUCTION-CHECKLIST.md)
- [Validation performed and remaining runtime checks](VALIDATION.md)

The numbered folders organize subjects, not a strict prerequisite order. Read core foundations first, build the capstone, then study security, data, and Django together. Advanced architecture comes after you can test and operate a simple service.

## 01. Foundations

1. [Client-server systems and request lifecycle](01-foundations/01-client-server-and-request-lifecycle.md)
2. [Networking, DNS, TLS, and HTTP](01-foundations/02-networking-dns-tls-and-http.md)
3. [Python, Django, synchronous and asynchronous execution](01-foundations/03-python-django-and-async.md)
4. [Project structure, configuration, and dependency boundaries](01-foundations/04-project-structure-and-configuration.md)
5. [Data formats, identifiers, time, numbers, and Unicode](01-foundations/05-data-formats-time-and-numbers.md)
6. [Documentation, Git, and engineering workflow](01-foundations/06-documentation-and-engineering-workflow.md)

## 02. Data and persistence

1. [Relational modeling and SQL](02-data/01-relational-modeling-and-sql.md)
2. [Transactions, isolation, locks, and concurrency](02-data/02-transactions-and-concurrency.md)
3. [Indexes, query plans, and performance](02-data/03-indexing-and-query-performance.md)
4. [Migrations, backfills, retention, and data lifecycle](02-data/04-migrations-and-data-lifecycle.md)
5. [NoSQL, document, key-value, graph, and time-series models](02-data/05-nosql-and-data-models.md)
6. [Caching, Redis, invalidation, and stampedes](02-data/06-caching-and-redis.md)
7. [Search, full-text retrieval, vectors, and analytics](02-data/07-search-and-analytics.md)

## 03. Security and identity

1. [Security foundations and threat modeling](03-security/01-security-foundations-and-threat-modeling.md)
2. [Authentication, passwords, sessions, and recovery](03-security/02-authentication-passwords-and-sessions.md)
3. [OAuth, OpenID Connect, and tokens](03-security/03-oauth-oidc-and-tokens.md)
4. [Authorization, roles, object permissions, and multitenancy](03-security/04-authorization-and-multitenancy.md)
5. [Web security, injection, XSS, CSRF, CORS, and SSRF](03-security/05-web-security-and-input-attacks.md)
6. [Secrets, cryptography, and privacy](03-security/06-secrets-cryptography-and-privacy.md)
7. [Abuse prevention, rate limits, and audit trails](03-security/07-abuse-prevention-and-audit.md)

## 04. Background and asynchronous systems

1. [Jobs, Celery, scheduling, and worker lifecycles](04-async/01-jobs-and-scheduling.md)
2. [Message queues, streams, acknowledgements, and delivery](04-async/02-message-queues-and-streams.md)
3. [Event-driven systems and transactional outbox](04-async/03-event-driven-systems-and-outbox.md)
4. [Idempotency and deduplication](04-async/04-idempotency-and-deduplication.md)
5. [Retries, timeouts, circuit breakers, and backpressure](04-async/05-resilience-and-backpressure.md)
6. [Files, uploads, and media pipelines](04-async/06-files-and-media-pipelines.md)

## 05. API techniques and integrations

1. [REST, resources, methods, and status contracts](05-apis/01-rest-and-resource-design.md)
2. [Validation, serialization, and error handling](05-apis/02-validation-and-error-handling.md)
3. [Pagination, filtering, and sorting](05-apis/03-pagination-filtering-and-sorting.md)
4. [GraphQL schemas, resolvers, and query cost](05-apis/04-graphql.md)
5. [gRPC and Protocol Buffers](05-apis/05-grpc-and-protocol-buffers.md)
6. [WebSockets, SSE, and realtime delivery](05-apis/06-websockets-and-server-sent-events.md)
7. [Webhooks and external integrations](05-apis/07-webhooks-and-external-integrations.md)
8. [OpenAPI, contracts, documentation, and evolution](05-apis/08-openapi-contracts-and-evolution.md)

## 06. Testing and engineering quality

1. [Testing strategy and unit tests](06-quality/01-testing-strategy-and-unit-tests.md)
2. [Integration, concurrency, contract, and end-to-end tests](06-quality/02-integration-contract-and-end-to-end-tests.md)
3. [Debugging, profiling, and memory analysis](06-quality/03-debugging-and-profiling.md)
4. [Performance measurement and load testing](06-quality/04-performance-and-load-testing.md)
5. [Code quality, dependencies, and reproducible builds](06-quality/05-code-quality-and-dependency-management.md)

## 07. Deployment and operations

1. [Linux, processes, sockets, and networking](07-operations/01-linux-processes-and-networking.md)
2. [Containers, Docker, and local environments](07-operations/02-containers-and-local-environments.md)
3. [CI/CD, releases, feature flags, and rollback](07-operations/03-ci-cd-and-release-strategies.md)
4. [Cloud deployment, orchestration, and infrastructure](07-operations/04-deployment-cloud-and-orchestration.md)
5. [Logs, metrics, tracing, SLOs, and alerts](07-operations/05-observability-and-slos.md)
6. [Reliability, backups, restore, and disaster recovery](07-operations/06-reliability-backups-and-disaster-recovery.md)
7. [Incident response and production runbooks](07-operations/07-incident-response-and-production-runbooks.md)

## 08. Architecture and system design

Use the [system-design study map](08-architecture/README.md) for coverage, prerequisites, and a suggested reading order.

1. [Modular monoliths, domain design, and boundaries](08-architecture/01-modular-monoliths-and-domain-design.md)
2. [Microservices, gateways, and service communication](08-architecture/02-microservices-and-service-communication.md)
3. [Distributed systems, replication, and consistency](08-architecture/03-distributed-systems-and-consistency.md)
4. [CQRS, event sourcing, and sagas](08-architecture/04-cqrs-event-sourcing-and-sagas.md)
5. [System design, capacity, scaling, and cost](08-architecture/05-system-design-and-capacity-planning.md)
6. [Consensus, Raft, Paxos, elections, and replicated logs](08-architecture/06-consensus-raft-and-paxos.md)
7. [Sharding, consistent hashing, and online resharding](08-architecture/07-sharding-and-consistent-hashing.md)
8. [Load balancing, traffic management, and overload](08-architecture/08-load-balancing-and-traffic-management.md)
9. [CDNs, edge caching, cache keys, and invalidation](08-architecture/09-cdns-and-edge-caching.md)
10. [Service discovery, routing, registries, and endpoint lifecycles](08-architecture/10-service-discovery-and-routing.md)
11. [Multiple regions, failover, write ownership, and failback](08-architecture/11-multi-region-and-failover.md)
12. [Logical clocks, CRDTs, conflicts, and reconciliation](08-architecture/12-logical-clocks-crdts-and-reconciliation.md)
13. [Storage engines, LSM trees, compaction, and Bloom filters](08-architecture/13-storage-engines-lsm-trees-and-bloom-filters.md)
14. [Stream processing, windows, watermarks, and checkpoints](08-architecture/14-stream-processing-windows-and-checkpoints.md)
15. [Distributed identifiers, UUIDs, and allocation](08-architecture/15-distributed-identifiers.md)

## 09. Integrated project

1. [Task Service: complete Django/PostgreSQL walkthrough](09-capstone/01-task-service.md)

## 10. Django in depth

1. [Models, ORM, QuerySets, relationships, and query optimization](10-django/01-models-orm-and-querysets.md)
2. [Forms, ModelForms, templates, and admin](10-django/02-forms-templates-and-admin.md)
3. [Django REST Framework: serializers, views, permissions, and testing](10-django/03-django-rest-framework.md)
4. [Middleware, signals, initialization, and management commands](10-django/04-middleware-signals-and-management-commands.md)
5. [Static files, media, email, and localization](10-django/05-static-media-email-and-localization.md)
6. [Custom users, permissions, and authentication workflows](10-django/06-custom-users-permissions-and-auth-workflows.md)

## 11. Worked system-design case studies

The [case-study guide](11-system-design-case-studies/README.md) includes a shared review method and practice assignments. Each design develops requirements, capacity calculations, APIs, data models, architecture, critical flows, tradeoffs, failure handling, security, and verification.

1. [URL shortener](11-system-design-case-studies/01-url-shortener.md)
2. [Chat and messaging](11-system-design-case-studies/02-chat-and-messaging.md)
3. [News feed](11-system-design-case-studies/03-news-feed.md)
4. [Video streaming](11-system-design-case-studies/04-video-streaming.md)
5. [Inventory and reservations](11-system-design-case-studies/05-inventory-and-reservations.md)
6. [Multichannel notification system](11-system-design-case-studies/06-notification-system.md)

## Shared conventions and prerequisites

The capstone is the source of truth for its runnable baseline: project package `config`, app `tasks`, PostgreSQL database `taskapp`, Compose service `db`, and API on `http://127.0.0.1:3000`. URLs use trailing slashes. The principal tables are `tenants` and `tasks`; examples add extension tables explicitly. Canonical tenant ID is `11111111-1111-4111-8111-111111111111`. SQL concurrency labs use task ID `22222222-2222-4222-8222-222222222222`; the relational-modeling tutorial includes its fixture setup.

The capstone's Django defaults execute in Python. Raw SQL labs must supply required fields themselves or define explicit database defaults. Do not assume independent tutorial extensions can all be pasted into the same app unchanged; choose one canonical user/membership model and integrate related policies deliberately.

PowerShell blocks are Windows host commands. Bash blocks belong in Linux, WSL, or the specified container. SQL blocks run against a disposable PostgreSQL database. Python fragments name their required surrounding setup. Pseudocode is labeled and is not presented as a complete application. JavaScript appears only where an external tool such as k6 requires it or to explain browser interoperability.

The core lab requires Python and PostgreSQL; Docker is a documented optional way to provide PostgreSQL. Redis, Celery, DRF, GraphQL integrations, gRPC tooling, k6, Kubernetes, and cloud services are additional prerequisites in their respective labs, not silently installed baseline features.

## How to use this collection well

For each topic: explain the mechanism, run its local lab where prerequisites are available, trigger a failure, observe the result, complete an exercise, and record what remains unverified. A copied code fragment is not evidence of a working system. The production checklist collects the cross-cutting requirements that individual demos intentionally leave for later.

The material is original tutorial prose with links to official documentation and standards. Version-specific examples should be checked against the documentation for the versions you actually deploy. Review date: 18 October 2026.
