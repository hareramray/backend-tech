# Worked system-design tutorials

[Complete curriculum](../README.md) · [Architecture study map](../08-architecture/README.md) · [Capacity-planning method](../08-architecture/05-system-design-and-capacity-planning.md)

These six fictional designs turn requirements into interfaces, data models, critical flows, capacity estimates, and failure policies. Each includes a diagram, implementation guidance for the Python/Django/PostgreSQL stack, exercises, and verification criteria. Their traffic numbers are explicit planning assumptions, not benchmarks or claims about a real company.

## Choose a design

| Case study | Main design decisions | Correctness property to defend |
| --- | --- | --- |
| [01. URL shortener](01-url-shortener.md) | Short-code allocation, redirects, read caching, expiry, abuse, and analytics | A code resolves only to its authorized current mapping; creation retries cannot silently create conflicting mappings |
| [02. Chat and messaging](02-chat-and-messaging.md) | Durable history, conversation ordering, WebSockets, reconnect/replay, receipts, and presence | Acknowledged messages survive the promised failures; clients detect gaps and deduplicate replay |
| [03. News feed](03-news-feed.md) | Push/pull fanout, celebrity skew, ranking, pagination, deletion, and freshness | Cached or materialized feed entries do not bypass current access rules |
| [04. Video streaming](04-video-streaming.md) | Resumable upload, processing, rendition publication, manifests, adaptive playback, and CDN delivery | Playback references a complete authorized asset version; partial processing cannot publish broken media |
| [05. Inventory and reservations](05-inventory-and-reservations.md) | Stock invariants, row locks, expiry, payment ambiguity, idempotency, and reconciliation | Concurrent reservations cannot oversell; uncertain payment outcomes do not release stock prematurely |
| [06. Notification system](06-notification-system.md) | Intent acceptance, preferences, channel fanout, provider limits, retries, receipts, and suppression | Accepted work is recoverable; duplicate attempts and unknown external outcomes have explicit policies |

Start with URL shortening for a compact read-heavy design. Follow with reservations to practice strong invariants, then chat and feeds for ordering and fanout. Video and notifications emphasize external systems, asynchronous pipelines, and substantial bandwidth or provider constraints.

## A reusable review method

1. **Define the contract.** List actors, permitted operations, invariants, latency/availability targets, freshness, retention, and excluded features. Explain what each success response promises.
2. **Estimate with units.** Derive average and peak rates, concurrent work, payload sizes, retention volume, replication overhead, and bandwidth. Give ranges for uncertain inputs.
3. **Write APIs and ownership.** Specify request identity, idempotency scope, authorization, pagination, error behavior, and the component authoritative for each fact.
4. **Choose the data model.** Name keys, constraints, indexes, transaction boundaries, partition keys, derived state, and deletion semantics. Include the busiest query and write paths.
5. **Trace critical flows.** Walk through a normal write/read and every durable handoff. Mark the moment after which a retry might repeat an already-completed effect.
6. **Test failure reasoning.** Consider process crash, timeout after commit, duplicate/out-of-order delivery, cache staleness, slow consumers, unavailable dependencies, and regional loss where required.
7. **Prove capacity and protection.** Identify the first bottleneck, queue growth, hot keys, retry amplification, access checks, quotas, and overload behavior.
8. **Plan operations and evolution.** Specify metrics, alerts, rollout, backfill, recovery, reconciliation, cost, and the evidence that would justify a more complex architecture.
9. **Run the exercise and record limits.** Keep the observed results, assumptions not yet tested, and a concrete next experiment. A plausible diagram is not runtime evidence.

## How these relate to the Django capstone

The [Task Service](../09-capstone/01-task-service.md) is the collection's complete local application walkthrough. These case studies are independent designs, not six applications already installed in that project. Create separate practice apps or deliberately adapt its services and data model. Do not paste incompatible schemas, routes, or identity policies together.

The notification tutorial contains an executable stdlib simulation using an in-memory database and a fake provider. The reservation tutorial contains a PostgreSQL concurrency exercise requiring separate sessions and a disposable database. Other exercises guide implementation and fault injection with stated prerequisites. Consult the [validation record](../VALIDATION.md) for what was actually executed while authoring.

## Stretch assignments

For every case, increase traffic tenfold, place half the traffic on one key or tenant, and remove one important dependency. Recalculate the bottleneck and recovery time before adding components. Then change one correctness requirement: immediate revocation, a stricter durability target, an offline client, or a shorter retention window. Explain which architecture decisions must change and why.

Finish with a short design document containing assumptions, a diagram, APIs, schema, calculations, two rejected alternatives, failure scenarios, and measured lab results. Use the [production checklist](../PRODUCTION-CHECKLIST.md) to identify the work a real deployment would still need.
