# Architecture and system-design study map

[Complete curriculum](../README.md) · [Learning path](../LEARNING-PATH.md) · [Worked system designs](../11-system-design-case-studies/README.md)

This folder contains **15 architecture tutorials**: five foundations and ten advanced chapters. Six separate case studies show how to combine those techniques for concrete products. The implementation context is Python, Django, and PostgreSQL; distributed algorithms are explained independently of any particular framework.

Start with requirements and invariants: what must remain true, what may be delayed, and what failure the product must survive. Then choose the storage, communication, and operational mechanisms that satisfy those promises. A diagram is only part of a design; capacities, ownership, authorization, recovery, and verification belong beside it.

## Foundations

| Tutorial | Questions it answers |
| --- | --- |
| [01. Modular monoliths and domain design](01-modular-monoliths-and-domain-design.md) | Where do business rules and ownership boundaries belong? |
| [02. Microservices and service communication](02-microservices-and-service-communication.md) | When is independent deployment useful, and what does it cost? |
| [03. Distributed systems and consistency](03-distributed-systems-and-consistency.md) | What do replication, quorums, partitions, and consistency promises mean? |
| [04. CQRS, event sourcing, and sagas](04-cqrs-event-sourcing-and-sagas.md) | How do write models, projections, histories, and compensating workflows differ? |
| [05. System design and capacity planning](05-system-design-and-capacity-planning.md) | How do requirements become workload, latency, storage, recovery, and cost budgets? |

## Advanced chapters

| Tutorial | Detailed coverage |
| --- | --- |
| [06. Consensus, Raft, and Paxos](06-consensus-raft-and-paxos.md) | Failure assumptions, elections, terms/ballots, replicated logs, quorum intersection, commit rules, reads, membership, and fencing |
| [07. Sharding and consistent hashing](07-sharding-and-consistent-hashing.md) | Range/hash/tenant partitioning, virtual nodes, rendezvous hashing, skew, cross-shard work, online movement, cutover, and rollback |
| [08. Load balancing and traffic management](08-load-balancing-and-traffic-management.md) | L4/L7, assignment units, selection algorithms, health checks, draining, retries, overload, and failover capacity |
| [09. CDNs and edge caching](09-cdns-and-edge-caching.md) | Cache keys, freshness, validators, authorization, signed access, immutable assets, invalidation, shielding, and origin protection |
| [10. Service discovery and routing](10-service-discovery-and-routing.md) | DNS, registries, watches, control/data planes, endpoint leases, generations, stale state, and rolling changes |
| [11. Multiple regions and failover](11-multi-region-and-failover.md) | Active/passive and active/active choices, write ownership, latency, replication lag, RPO/RTO, fencing, promotion, failback, and reconciliation |
| [12. Logical clocks, CRDTs, and reconciliation](12-logical-clocks-crdts-and-reconciliation.md) | Happens-before, Lamport/vector clocks, hybrid clocks, conflict policies, counters/sets, tombstones, and anti-entropy |
| [13. Storage engines, LSM trees, and Bloom filters](13-storage-engines-lsm-trees-and-bloom-filters.md) | B-tree versus LSM tradeoffs, WAL, memtables, SSTables, compaction, amplification, filters, deletions, and recovery |
| [14. Stream processing, windows, and checkpoints](14-stream-processing-windows-and-checkpoints.md) | Event/processing time, window types, watermarks, lateness, state, checkpoints, replay, sink contracts, and backlog recovery |
| [15. Distributed identifiers](15-distributed-identifiers.md) | Sequences, ranges, UUIDs, time/worker/counter IDs, clock rollback, allocation, index locality, and ordering limits |

Each advanced chapter includes a small executable Python model, its limitations, failure exercises, and primary references. The models expose a mechanism; they do not implement production consensus, a real CDN, a replicated database, or a distributed stream processor. PostgreSQL and infrastructure labs state their additional prerequisites.

## How to study the material

1. Read chapters 01–05 and write a one-page requirement and capacity brief for the Task Service.
2. Read 08–10 to trace a request through edge caching, routing, discovery, and application instances. Identify every authorization and timeout boundary.
3. Read 07, 13, and 15 to choose data ownership, storage behavior, and identifiers. Estimate read, write, and maintenance cost.
4. Read 06 and 12 to explain coordination and concurrency. Separate ordering, conflict detection, conflict resolution, and uniqueness.
5. Read 11 and 14 to plan regional recovery and stateful replay. Include both restoration time and backlog catch-up.
6. Complete one [worked design](../11-system-design-case-studies/README.md), then change its workload or failure requirement and revise the design with calculations.

## Related subjects elsewhere in the curriculum

System design also depends on [transactions and locks](../02-data/02-transactions-and-concurrency.md), [indexes and plans](../02-data/03-indexing-and-query-performance.md), [NoSQL models](../02-data/05-nosql-and-data-models.md), [Redis caching](../02-data/06-caching-and-redis.md), and [search and analytics](../02-data/07-search-and-analytics.md). For asynchronous correctness, study [queues and streams](../04-async/02-message-queues-and-streams.md), [outbox](../04-async/03-event-driven-systems-and-outbox.md), [idempotency](../04-async/04-idempotency-and-deduplication.md), and [backpressure](../04-async/05-resilience-and-backpressure.md).

Use [threat modeling](../03-security/01-security-foundations-and-threat-modeling.md), [multitenancy](../03-security/04-authorization-and-multitenancy.md), and [rate limiting](../03-security/07-abuse-prevention-and-audit.md) to protect the design. Use [load testing](../06-quality/04-performance-and-load-testing.md), [SLOs](../07-operations/05-observability-and-slos.md), and [disaster recovery](../07-operations/06-reliability-backups-and-disaster-recovery.md) to verify it.

This map makes the covered subjects explicit. System design remains an open-ended discipline: specialized products and deployment platforms introduce further requirements. Record unresolved assumptions and consult the linked specifications when extending a teaching example.
