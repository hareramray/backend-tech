# Validation record

[Tutorial index](README.md)

Reviewed on **18 October 2026**. This record distinguishes documentation checks, local framework checks, and infrastructure exercises that still require their stated prerequisites.

## Documentation checks

- The collection contains **74 topic tutorials** across eleven subject folders, plus the index, learning path, glossary, production checklist, this record, and two folder study guides: **81 Markdown files** inside `backend-tutorials`. The workspace README is the 82nd Markdown file and points to the collection.
- Local Markdown file links were checked for target existence.
- Backtick and tilde code fences were checked for matching closure.
- All **84 Python code blocks** were parsed for syntax; this includes fragments whose runtime dependencies are described in their tutorials.
- The collection was checked for stale Node.js/TypeScript/npm examples after the requested Python/Django/PostgreSQL selection. JavaScript remains where k6 requires its script format and where browser numeric/string behavior is relevant.
- Shared models, route slashes, task/tenant fixtures, membership fields, migration order, and Python-versus-database defaults received a consistency review.

Syntax and link checks do not prove that every integration works. Official source links were consulted for the technical material; there was no exhaustive automated availability check of every external URL.

## Capstone checks performed

The seven complete Python files embedded in the capstone were extracted into a separate temporary practice project. Verification used **Python 3.13.15, Django 5.2.18, and psycopg 3.3.6**. Dependencies were installed into a temporary virtual environment; no backend application or infrastructure was deployed by creating this tutorial collection.

| Check | Result | What it establishes |
| --- | --- | --- |
| Django system check | Passed | Project/model/URL configuration is accepted |
| Seven pure validation tests | Passed | Title normalization, version type, unknown fields, duplicate JSON keys, invalid database text, deep nesting, and page limits |
| Generate model migration | Passed in isolated validation project | Model definitions produce a migration |
| Eleven capstone tests using temporary SQLite | Passed | Validation plus create/update/conflict, tenant filtering, cursor traversal/tampering, and Origin restriction behavior |

The SQLite smoke run used a temporary settings override that is not part of the PostgreSQL tutorial. It does **not** establish PostgreSQL locking, query plans, RLS, replication, driver behavior, or production concurrency guarantees. The capstone instructs readers to run its actual test suite on a disposable PostgreSQL database.

## Additional examples exercised

- Standalone duplicate-delivery and deduplication behavior.
- Bounded concurrency with observed peak activity limited to the configured value.
- Upload size limits, collision preservation, checksum-failure cleanup, and file lifecycle handling.
- Pure domain-policy examples and the quality chapter's isolated unit tests.
- Django 5.2 framework checks in a separate SQLite project for session login, password-reset templates, verification expiry/tampering/replay, membership permission/revocation, and FileField save/cleanup.
- Syntax parsing of selected YAML examples in the operations section.

The initial quality-chapter unit examples were also run using the host's Django 6.1.1. The principal capstone and the additional authentication/media framework checks used Django 5.2.18, matching the tutorial's selected framework series.

## Advanced system-design additions

The expansion adds **ten advanced architecture chapters and six worked designs**. Independent content reviews checked distributed guarantees, cache races, authorization during content changes, regional restoration timing, notification ambiguity, and capacity arithmetic. Review findings were corrected, including delayed stale cache fills, authorizing the actual hydrated post version, and checkpoint replay distance measured from captured source positions.

The **eleven complete Python labs** in these additions were extracted and run separately with **Python 3.13.15**, without third-party dependencies. All passed after replacing an exact floating-point equality in the balancing example with a tolerant comparison.

| Lab | Observed result |
| --- | --- |
| Consensus | Stale candidate and repeat vote after modeled restart rejected; old-term replication alone does not advance the demonstrated commit calculation |
| Shard placement | Placement/movement assertions passed; stale owner rejected after cutover |
| Load balancing | Fourth-request latency is 9.7 seconds with round robin and 1.0 second with the modeled least-outstanding policy |
| CDN cache keys | Deliberately unsafe key leaks across tenants; scoped key isolates them and excludes irrelevant tracking |
| Service discovery | Expiry, replacement generation, stale updates, and draining assertions passed |
| Regional failover | Missing acknowledged IDs 97–100 identified; stale writer rejected; illustrative recovery phases sum to 240 seconds |
| Logical clocks and CRDTs | Causal comparison and convergence assertions passed; concurrent add survives observed removal |
| Bloom filter | 9,586 bits, seven hashes, 1,199 bytes; 93 false positives in 10,000 tested absent keys and no false negatives for inserted keys |
| Stream recovery | Revised window total is 15 at revision 3; replay leaves two visible sink changes; too-late event `d` is classified separately |
| Distributed IDs | Local uniqueness, overflow, and clock checks passed; unsafe restart collision deliberately reproduced |
| Notifications | One local delivery record; one external effect with fake-provider deduplication versus two without it |

These are deterministic, local teaching models. Their output does not establish a real cluster's consensus safety, network behavior, failover RTO, CDN policy, stream-engine guarantees, or provider delivery. The case-study calculations are planning assumptions, not measured infrastructure benchmarks. The earlier capstone/framework checks above are retained as their own validation record; the system-design expansion did not modify or rerun that application.

## Checks not performed here

PostgreSQL/Docker services were not available in this environment. PostgreSQL integration and concurrency tests, SQL execution plans, migrations against a live PostgreSQL instance, RLS policies, backup/restore drills, and database failure experiments remain reader-run labs.

The new reservation tutorial's two-session PostgreSQL race exercise and the distributed-ID sequence exercise were not executed. Neither were real load-balancer configurations, CDN policies, discovery clusters, Raft/Paxos deployments, resharding, regional promotion/failback, LSM-engine benchmarks, stream-processor checkpoints, video processing/delivery, or notification-provider integrations. The relevant tutorials describe the prerequisites and expected observations; no external services or real messages were created by the standalone simulations.

Redis, Celery/brokers, GraphQL integrations, gRPC servers, WebSocket/ASGI deployments, external identity providers, object stores, SMTP delivery, k6 load tests, CI providers, Kubernetes, cloud infrastructure, and production rollout procedures were not provisioned or exercised. Their chapters state prerequisites and expected verification steps. Local mock/framework checks are not substitutes for testing those systems.

The unauthenticated capstone is an intentionally constrained local demo. The [production checklist](PRODUCTION-CHECKLIST.md) and security chapters explain the changes needed before an adapted service can be exposed.
