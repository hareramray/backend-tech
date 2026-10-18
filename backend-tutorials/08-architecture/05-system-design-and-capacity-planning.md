# System design and capacity planning

## Purpose and prerequisites

System design turns product requirements into data ownership, interfaces, dependencies, failure behavior, and operational budgets. Capacity planning asks how much load a design can support while meeting those promises. Begin with measured requirements and uncertainty, not a list of fashionable infrastructure products.

Prerequisites: [modular architecture](01-modular-monoliths-and-domain-design.md), [distributed consistency](03-distributed-systems-and-consistency.md), and the [Django task-service capstone](../09-capstone/01-task-service.md). All numbers below are fictional planning assumptions; replace them with evidence from your workload.

## System-design coverage beyond this chapter

This chapter establishes the requirements and capacity-planning method. Continue through the [system-design study map](README.md) for the full architecture sequence and the [six worked designs](../11-system-design-case-studies/README.md) for end-to-end applications of that method.

| Subject | Detailed tutorial |
| --- | --- |
| Agreement and leadership | [Consensus, Raft, and Paxos](06-consensus-raft-and-paxos.md) |
| Data distribution and movement | [Sharding, consistent hashing, and resharding](07-sharding-and-consistent-hashing.md) |
| Request distribution and overload | [L4/L7 load balancing and traffic management](08-load-balancing-and-traffic-management.md) |
| Global content delivery | [CDNs, edge caching, and cache-key isolation](09-cdns-and-edge-caching.md) |
| Finding and updating endpoints | [Service discovery and routing](10-service-discovery-and-routing.md) |
| Regional failure and recovery | [Multiple regions, fencing, failover, and failback](11-multi-region-and-failover.md) |
| Concurrent changes and convergence | [Logical clocks, CRDTs, and reconciliation](12-logical-clocks-crdts-and-reconciliation.md) |
| Physical storage tradeoffs | [LSM trees, compaction, and Bloom filters](13-storage-engines-lsm-trees-and-bloom-filters.md) |
| Stateful event processing | [Windows, watermarks, checkpoints, and replay](14-stream-processing-windows-and-checkpoints.md) |
| Identity across nodes | [Distributed identifiers and allocation](15-distributed-identifiers.md) |

Worked designs: [URL shortener](../11-system-design-case-studies/01-url-shortener.md), [chat](../11-system-design-case-studies/02-chat-and-messaging.md), [news feed](../11-system-design-case-studies/03-news-feed.md), [video streaming](../11-system-design-case-studies/04-video-streaming.md), [inventory reservations](../11-system-design-case-studies/05-inventory-and-reservations.md), and [notifications](../11-system-design-case-studies/06-notification-system.md). These separate tutorials include requirements, schemas, APIs, capacity estimates, critical flows, failure handling, and exercises.

## Specify behavior before topology

Describe users, critical journeys, data sensitivity, geographic needs, expected growth, and acceptable degradation. For a task application, distinguish interactive reads, task mutations, exports, notifications, and administrative changes. They have different latency, durability, and authorization requirements.

Define an SLI as a measured service property and an SLO as its target. An SLA adds an agreement and consequences. Select user-relevant measures, such as the fraction of valid requests completed successfully within a threshold, rather than only CPU utilization. [Google SRE service-level objectives](https://sre.google/sre-book/service-level-objectives/).

Example learning targets:

- At least 99.9% of eligible interactive requests succeed over a rolling 30-day window.
- At least 99% of successful interactive reads finish within 300 ms at the chosen measurement boundary.
- At least 99% of accepted exports complete within five minutes under the supported workload.
- An acknowledged task mutation survives the documented failure scenarios.

Define eligible requests and exclusions before measuring. Do not improve the metric by excluding failures users actually experience. If the first SLO receives one million eligible requests, its 0.1% error budget permits 1,000 unsuccessful requests; this is a request budget, not automatically a time budget.

## Estimate workload with units

Assume 20,000 daily active users, 30 API operations per user per day, and a peak factor of eight:

```text
Daily operations = 20,000 users/day * 30 operations/user
                 = 600,000 operations/day
Average rate     = 600,000 / 86,400 ~= 6.94 operations/second
Estimated peak   = 6.94 * 8 ~= 55.6 operations/second
```

The peak factor is an assumption, not a universal constant. Regional time zones, batch jobs, notifications, and launches can create different peaks. If 80% of operations are reads, peak reads are roughly 44.5/s and writes roughly 11.1/s under this model.

For storage, suppose 100,000 new tasks daily at 1 KiB of logical row data:

```text
Logical annual data = 100,000 * 365 * 1 KiB
                    ~= 34.8 GiB/year
```

Indexes, row overhead, dead tuples, replicas, WAL, backups, attachments, and retention multiply physical storage. Estimate them separately rather than calling logical payload bytes the database size. For network transfer, requests per second times average response bytes gives only a baseline; include headers, TLS, compression, and large-response tails.

## Concurrency and bottlenecks

Little's Law relates average in-flight work to arrival rate and average time in a stable system: `L = lambda * W`. At 60 requests/s with 0.2 seconds average duration, average in-flight requests are approximately 12. This does not mean a 12-connection database pool is automatically correct: only some of that duration holds a database connection, and bursts and variance matter.

With eight application processes and a pool limit of ten each, potential database connections can reach 80 before workers, administration, migrations, and replicas. Budget globally. Increasing application workers can worsen database saturation if every process adds more concurrent queries.

Measure CPU, memory, database wait time, query latency, I/O, lock contention, connection usage, queue age, and downstream latency. Tail latency often degrades sharply near saturation. Keep explicit headroom and shed load before failure cascades. Google SRE discusses bounded queues, load shedding, and overload behavior as availability mechanisms. [Handling overload](https://sre.google/sre-book/handling-overload/).

## Choose the simplest architecture that meets the budget

For the fictional initial workload, begin with a Django application, PostgreSQL, and a durable worker when asynchronous work is needed. Add Redis only for a defined cache, limiter, or coordination requirement. Index real access patterns before adding replicas. Paginate large reads before increasing server memory.

Each scaling step has a trigger and cost:

| Change | Possible trigger | Cost or limitation |
|---|---|---|
| Query/index optimization | Expensive known query | Write overhead and index maintenance |
| Vertical capacity | One instance is resource-bound | Hardware ceiling and failover sizing |
| More application instances | CPU or request concurrency | Shared dependency saturation |
| Cache | Repeated safe-to-cache reads | Invalidation and stale-data policy |
| Read replica | Primary read pressure | Lag and read consistency |
| Partitioning/sharding | Measured data or write boundary | Routing, transactions, balancing, operations |

Sharding by tenant can localize most queries, but a very large tenant becomes a hotspot. A cross-tenant report needs a different path. Rebalancing, backup, restoration, and tenant movement must be designed before a shard key becomes difficult to change.

## Lab: create and test a capacity hypothesis

1. Define one representative workload mix: task list, task detail, create, update with conflicts, and a small export fraction. Include realistic tenant sizes and query selectivity.
2. Write expected arrival rate, response-size distribution, latency target, and error budget. Record all unknowns.
3. Load only synthetic data into a disposable environment resembling deployment. A tiny empty database is not a representative index or cache benchmark.
4. Generate controlled load using an existing load tool. Increase arrival rate in steps, allow a steady interval, and record throughput, p50/p95/p99 latency, errors, database waits, and resource usage.
5. Find the first violated target and identify the limiting resource. Change one cause, such as a missing index or unbounded query, then repeat the same workload.
6. Test a burst and a sustained soak. Look for queue growth, memory leaks, connection leaks, and maintenance effects.
7. Remove one application instance or simulate a slow dependency in the disposable environment. Confirm remaining capacity and overload responses match the design.
8. Write a recommendation including measured safe throughput, tested dataset, headroom, limitations, and the next scaling trigger.

Do not run unrestricted load against production or third-party services. This lab is about a controlled local or owned test environment. An apparent throughput gain that increases errors or drops work does not satisfy the SLO.

## Recovery and cost are part of capacity

Specify recovery time objective, the time allowed to restore service, and recovery point objective, the acceptable data-loss window. Backups do not demonstrate recovery until restored and checked. Include storage restore speed, schema compatibility, encryption keys, DNS or traffic changes, and backlog catch-up.

Multi-region operation adds latency, conflict, failover, residency, and cost choices. A passive region must have enough tested capacity to take traffic; a diagram alone does not provide that capacity. Include cost per useful unit, such as per 1,000 successful requests or completed exports, along with compute, storage, egress, monitoring, and operator effort.

## Exercises and completion checklist

Recalculate the design for ten times the traffic and one tenant holding half the data. Identify which assumptions change and which do not. Then write a one-page decision record explaining why a cache or microservice is unnecessary until a stated trigger occurs.

- [ ] Workload assumptions have units and uncertainty ranges.
- [ ] SLOs describe actual user journeys and failure treatment.
- [ ] Database connections and concurrency are budgeted globally.
- [ ] Benchmarks measure realistic data, tails, and overload.
- [ ] Recovery, growth, and cost appear in the design.
- [ ] Every added component solves a measured or explicit requirement.

Return to [the curriculum](../README.md) and apply the design to [the capstone](../09-capstone/01-task-service.md).
