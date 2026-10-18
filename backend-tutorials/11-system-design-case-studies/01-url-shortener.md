# Case study: a URL shortener with safe redirects

## Problem, boundaries, and success criteria

Design a service where authenticated customers create short links and anonymous visitors follow them. Customers can choose an unused alias, set an expiry, change a destination, disable a link, and view approximate visit statistics. Custom domains are supported only after ownership verification. This is an independent design exercise using Django and PostgreSQL; the existing task-service capstone does not implement it.

Read [capacity planning](../08-architecture/05-system-design-and-capacity-planning.md) and [CDNs and edge caching](../08-architecture/09-cdns-and-edge-caching.md) first. The important invariant is that one normalized domain-and-alias pair identifies at most one link. An expired or disabled link must not redirect beyond its documented invalidation window. A random alias is an identifier, not permission to read private content.

Fictional targets: 99.95% of eligible redirect requests succeed monthly; regional p99 redirect latency is below 150 ms; creation p99 is below 400 ms. Destination edits propagate within 30 seconds under normal operation. Emergency abuse blocks require a separate edge denylist path with a measured five-second propagation target. If that path cannot establish a trustworthy decision, the protected redirect fails closed. Analytics may lag five minutes and lose a documented small fraction of events during overload.

## Workload and arithmetic

Assume 100,000 creations and 20 million redirects daily, with a tenfold peak factor. Decimal GB and MB below mean powers of 1,000.

```text
Creation average = 100,000 / 86,400 = 1.16 requests/s
Creation peak   = 1.16 * 10        = 11.6 requests/s
Redirect average = 20,000,000 / 86,400 = 231.5 requests/s
Redirect peak    = 231.5 * 10          = 2,315 requests/s
One year of rows = 100,000 * 365 * 600 bytes = 21.9 GB logical
Analytics input = 20,000,000 * 120 bytes = 2.4 GB/day logical
```

The 600-byte row estimate includes a typical destination and metadata, not indexes, WAL, row headers, replicas, or backups. Long destinations change it. A 95% redirect cache hit ratio leaves roughly 116 peak origin lookups/s, but a cold cache exposes all 2,315/s. The origin must handle a defined cold-start load or deliberately shed requests. One viral alias can dominate traffic even when average database utilization is small.

Ten base62 characters provide `62^10 = 839,299,365,868,340,224` possible aliases. With 100 million independently uniform generated aliases, the expected number of colliding pairs is approximately `n(n-1)/(2N) = 0.00596`. This is not a guarantee of zero collisions and does not cover human-selected names. Database uniqueness remains mandatory.

## Architecture and ownership

```text
Creator --> edge/WAF --> Django management API --> PostgreSQL
                                                    |
                                                 outbox
                                                    v
Visitor --> edge denylist/cache --> redirect API --> invalidation workers
                  |                    |
                  +--> events ---------+--> bounded analytics ingestion
                                             |
                                       daily aggregates
```

PostgreSQL owns links, tenants, domain ownership, management idempotency records, and durable change events. A cache owns expendable copies. Analytics owns visit aggregates, never the authoritative destination. A modular Django deployment can initially serve both APIs; reserve separate resource pools so analytics or management bursts cannot starve redirects.

Do not begin with globally distributed writes. Put each custom domain's management writes in one authority and route reads through caches. If creation availability across regions becomes necessary, define conflict-free namespace allocation or retain a globally coordinated uniqueness authority. [Sharding](../08-architecture/07-sharding-and-consistent-hashing.md) changes where uniqueness is enforced.

## API and data contract

```http
POST /v1/links
Authorization: Bearer <management-token>
Idempotency-Key: client-operation-123
Content-Type: application/json

{"domain":"go.example.test","alias":"launch","destination":"https://example.test/docs","expires_at":null}
```

Return `201` with an immutable link ID, effective alias, short URL, and version. Replaying the same idempotency key and request body returns the original result; reusing it for different content returns `409`. An occupied requested alias returns `409`; invalid schemes or excessive lengths return `400`. Store the idempotency record and creation result atomically, scoped to tenant and operation.

`PATCH /v1/links/{id}` requires an expected version, returning `409` on a concurrent edit. `GET /{alias}` accepts only GET and HEAD, returning `302` and `Location` for an active link, or `404` for unavailable links. HEAD performs the same eligibility decision without counting a visitor event. Never append an arbitrary incoming query string to the stored destination.

The following is a schema sketch; identity tables and migrations belong to the application:

```sql
CREATE TABLE short_link (
    id uuid PRIMARY KEY,
    tenant_id uuid NOT NULL,
    domain text NOT NULL,
    alias text COLLATE "C" NOT NULL,
    destination text NOT NULL,
    active boolean NOT NULL DEFAULT true,
    expires_at timestamptz,
    version bigint NOT NULL DEFAULT 1,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (domain, alias)
);
CREATE INDEX short_link_tenant_created
    ON short_link (tenant_id, created_at DESC, id DESC);
```

Normalize domain names consistently before insertion and lookup. Decide alias case sensitivity once; this design preserves case. Reserve application routes such as `health` and `v1`. Tenant ownership is checked on every management lookup, even if an attacker knows a UUID. Keep deleted aliases reserved with tombstones to prevent old printed links from silently becoming another customer's property.

## Critical flows and collision handling

1. Authenticate the creator, resolve verified domain ownership, validate the URL, and enforce tenant quotas.
2. For a generated alias, use a cryptographically strong random generator. Attempt insertion under the unique constraint. Do not perform an existence check followed by an unprotected insert.
3. On an alias conflict, generate a new candidate and retry a bounded number of times. A custom alias conflict is reported immediately. Insert the change outbox event in the same transaction.
4. Commit before acknowledging creation. The worker publishes invalidation and audit events, retrying with the event ID as the deduplication key.
5. For a redirect, construct the cache key from the verified request host and exact alias. Apply the abuse decision, load link metadata, check active status and absolute expiry, then return the stored destination.

`INSERT ... ON CONFLICT (domain, alias) DO NOTHING RETURNING id` allows the caller to distinguish a successful allocation from a conflict by whether a row is returned. It does not overwrite the existing owner's destination. [PostgreSQL INSERT](https://www.postgresql.org/docs/current/sql-insert.html).

In Django, catch an integrity error outside the `transaction.atomic()` block that failed, or use an inner savepoint around each attempt; continuing arbitrary queries inside a broken transaction is incorrect. A post-commit callback can wake a worker, but the committed outbox is the recovery record if the process dies before waking it. [Django transaction handling](https://docs.djangoproject.com/en/5.2/topics/db/transactions/).

## Caching and redirect semantics

This design uses temporary redirects because destinations are editable. `301` and `308` communicate permanent relocation, while `307` preserves the method and `302` allows historical POST-to-GET behavior. Restricting this endpoint to GET/HEAD avoids using redirect status as a substitute for an explicit API method policy. [HTTP redirect semantics](https://www.rfc-editor.org/rfc/rfc9110.html#name-redirection-3xx).

For ordinary editable public links, an example policy is `Cache-Control: public, max-age=0, s-maxage=30, must-revalidate`. The edge freshness lifetime is also capped by the remaining time until expiry. Near expiry, use `no-store`. Private links, if introduced, use authenticated resolution and `private, no-store`; they must never share the public cache path. Browser behavior, edge behavior, and application metadata caching need separate tests. [HTTP caching rules](https://www.rfc-editor.org/rfc/rfc9111.html).

Invalidating before committing an edit creates a race: a concurrent reader can refill the old value. Commit first, then invalidate through the outbox. A delayed reader can still refill an older version; include a version in cache values and have the fill/invalidation mechanism reject writes older than the recorded invalidation generation. The 30-second edit target requires a measured end-to-end invalidation budget covering the outbox, metadata cache, edge distribution, and in-flight fills. A version check confined to the metadata cache does not protect an edge that can still store an old response.

A 30-second TTL by itself bounds an entry's freshness lifetime, not staleness measured from edit commit: a delayed old read can refill it later, and cache layers can compound the delay. If the cache stack cannot enforce bounded version-aware invalidation, carry the age of the authoritative read through the layers and bound refill/network delays, reduce TTLs to fit the complete budget, or explicitly relax the edit target. Test the chosen mechanism with delayed fills. Never describe simple delete-on-write as instantaneous consistency.

Cache misses for nonexistent aliases may be cached briefly to resist scanning, but creation must invalidate a matching negative entry. Use request coalescing to prevent a hot expired cache key from triggering thousands of identical database queries. Cache TTL jitter distributes renewal times; it does not solve a single-key stampede on its own.

## Security, analytics, and operational recovery

Allow only `https` and, if explicitly supported, `http` destinations; reject credentials in URLs, control characters, excessive lengths, and unsupported schemes. Display the actual destination in abuse-review tools. A shortener intentionally redirects to user-supplied hosts, so phishing reporting, suspension, audit history, and creation rate limits are product requirements. If a preview or scanner fetches destinations, isolate that fetcher and defend against private-network access and redirect-based SSRF.

Never emit raw access tokens or full destination query strings into logs. Analytics can use coarse time buckets and short-retention pseudonymous visitor data. A best-effort event contains event ID, link ID, timestamp, and coarse attribution. Workers deduplicate redelivery, but a request that dies before publishing may be absent. If billing requires authoritative click accounting, redesign the acceptance boundary and budget the extra durable-write latency; approximate analytics cannot silently become a financial ledger.

Monitor redirect success and p99 separately from creation latency, edge hit ratio, invalidation age, collision retries, database lookup latency, and dropped analytics events. Use synthetic links to measure edit and disable propagation end to end. Restore link records, tombstones, ownership records, and signing configuration together; clear derived caches after restoration. A regional failover needs a declared RPO for newly created links, not an assumption that caches contain every record.

## Tabletop lab: exact failure schedule

Use paper or a local document; no service is needed. Start with alias `launch`, destination A, version 1, and an edge entry valid until time 30. Execute each row independently and record database, cache, and response state.

| Injected schedule | Expected outcome and evidence |
|---|---|
| Two creators insert `launch` concurrently | Exactly one insert succeeds; the other receives a conflict, never the winner's ownership |
| Creation commits, response disappears, same key retried | Same link ID and response; one link and one logical create event |
| Edit to B commits at time 5; invalidation delayed | A may appear only within the declared stale window; propagation alarm records delay |
| Old cache refill races with version 2 invalidation | Generation-aware fill rejects version 1 at each relevant layer; a TTL-only alternative measures total age and refill delay against its revised budget |
| Expiry is time 12 but normal TTL is 30 | Cache freshness cannot extend beyond 12; subsequent lookup does not redirect |
| Analytics ingestion is unavailable | Redirect still succeeds; loss/backlog metric increases according to the documented policy |
| Tenant B guesses tenant A's management ID | No data or mutation is authorized; public redirect behavior is unchanged |

Extend the exercise by cold-starting all caches and recalculating origin demand. Compare the cost of 30-second and five-minute TTLs, including abuse-response consequences. Implement the alias allocation transaction in a disposable PostgreSQL database and race two sessions as an optional next lab.

## Completion checklist

- [ ] Namespace normalization and uniqueness are enforced atomically.
- [ ] Retries cannot create multiple links for one management operation.
- [ ] Expiry, negative caches, edits, and abuse blocks have tested timing rules.
- [ ] Analytics durability matches its advertised purpose.
- [ ] Cold-cache load, hot aliases, and restore procedures have explicit budgets.

Continue with [chat and messaging](02-chat-and-messaging.md) or return to the [case-study index](README.md).
