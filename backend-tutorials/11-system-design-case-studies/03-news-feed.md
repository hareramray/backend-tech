# Case study: a news feed with hybrid fanout

## Requirements before ranking

Design a following feed for a fictional community product. Users publish text and image references, follow accounts, edit or delete posts, block users, and page through recent updates. Start with reverse chronological order. A later ranked feed is a separate ordering contract, not a silent change to the same cursor. The design uses Django and PostgreSQL as its initial authority and is independent of the task-service capstone.

Read [capacity planning](../08-architecture/05-system-design-and-capacity-planning.md), [sharding](../08-architecture/07-sharding-and-consistent-hashing.md), and [stream processing](../08-architecture/14-stream-processing-windows-and-checkpoints.md). Feed generation is a derived read model. Posts, follow relationships, blocks, visibility, and moderation decisions remain authoritative elsewhere.

Fictional targets: first-page p99 below 250 ms; 99% of ordinary-author posts appear for active followers within five seconds; celebrity posts appear on the next eligible feed refresh. A block or visibility revocation committed before a new read begins must affect that read. Already rendered or downloaded content cannot be recalled. Deletes can take longer to disappear from internal inbox indexes, but those indexes must not authorize returning deleted content.

Do not promise exactly once display across every device and refresh. The useful contract is no duplicate post ID within one response, deterministic pagination within a defined feed session, and clear refresh behavior when new data or ranking changes.

## Workload with skew

Assume one million daily active users, 20 feed-page reads per user per day, 200,000 posts/day, 300 eligible followers per ordinary post on average, and a tenfold peak factor. The model excludes media transfer.

```text
Feed reads/day = 1,000,000 * 20 = 20,000,000
Read average  = 20,000,000 / 86,400 = 231.5 pages/s
Read peak     = 231.5 * 10 = 2,315 pages/s
Post average  = 200,000 / 86,400 = 2.31 posts/s
Push entries/day = 200,000 * 300 = 60,000,000
Push average     = 694.4 entries/s
At 40 bytes/entry = 2.4 GB/day logical before indexes and overhead
```

Keeping 500 inbox references for each active user requires 500 million references, approximately 20 GB at the fictional 40-byte logical estimate. The real store needs keys, indexes, allocator overhead, replicas, and safety headroom. Returning 30 posts averaging 1 KiB produces about 600 GB/day of logical page payload at 20 million pages/day, before protocol overhead and images.

A single author with five million followers publishing once creates five million push writes. At a dedicated 20,000 entries/s, that one fanout takes 250 seconds. An average of 300 followers therefore cannot determine architecture alone. Follow-count distribution and audience activity matter more than a single mean.

## Push, pull, and hybrid choices

| Strategy | Work at publication | Work at read | Suitable pressure |
|---|---|---|---|
| Push/fanout on write | Add references to follower inboxes | Read one inbox | Ordinary authors, frequent readers |
| Pull/fanout on read | Store post in author timeline | Merge followed timelines | Inactive readers or huge audiences |
| Hybrid | Push ordinary authors; retain pull sources | Merge inbox and selected timelines | Skewed social graphs |

For the fictional workload, push to recently active followers of ordinary authors and pull celebrity timelines. Dormant users get a rebuild or pull path when they return. Classification uses measured fanout cost and feed-read cost, not a magical universal follower threshold. An author can change classification; migration uses an epoch and deduplication so in-flight old fanout jobs do not create visible duplicates or gaps.

```text
Author --> Django write API --> posts/follows/blocks in PostgreSQL
                                     |
                                   outbox
                                     v
                                fanout workers --> inbox references
                                     |
                               author timelines

Reader --> feed API --> merge inbox + pull sources --> current policy filter
                                                        |
                                                  hydrate posts
                                                        |
                                                 cursor response
```

Keep inbox entries as post IDs plus immutable ordering keys. Copying full post bodies into millions of inboxes makes edits and deletion harder and can leave sensitive content in derived storage.

## APIs, schema, and access paths

`POST /v1/posts` accepts text, approved media IDs, and visibility with an idempotency key. `POST /v1/follows/{author}` and `DELETE` are idempotent relationship changes. `POST /v1/blocks/{user}` commits a policy change. `GET /v1/feed?cursor=...&limit=30` returns posts, the next opaque cursor, and a freshness timestamp. A stale cursor can return an explicit restart response rather than silently changing its meaning.

```json
{
  "items": [{"post_id": "p91", "version": 3, "text": "Updated schedule"}],
  "next_cursor": "signed-opaque-session-and-position",
  "generated_at": "2026-10-18T09:00:00Z",
  "ordering": "chronological-v1"
}
```

Logical schema:

```text
post(tenant_id, id, author_id, published_at, body, visibility,
     version, deleted_at, moderation_state)
  PRIMARY KEY (tenant_id, id)
  INDEX (tenant_id, author_id, published_at DESC, id DESC)
follow(tenant_id, follower_id, author_id, state, relationship_version)
  PRIMARY KEY (tenant_id, follower_id, author_id)
  INDEX (tenant_id, author_id, follower_id)
block(tenant_id, blocker_id, blocked_id, created_at)
  PRIMARY KEY (tenant_id, blocker_id, blocked_id)
inbox(tenant_id, owner_id, post_id, published_at, source_epoch)
  UNIQUE (tenant_id, owner_id, post_id)
  INDEX (tenant_id, owner_id, published_at DESC, post_id DESC)
outbox(event_id, post_id, post_version, kind, payload, published_at)
```

The forward follow index serves one reader's subscriptions; the reverse index enumerates an author's audience. Choose multicolumn index order from equality predicates and ordering needs, then verify representative queries with execution plans. [PostgreSQL multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html).

## Publish and read flows

1. Authenticate the author and validate tenant, media ownership, text limits, and posting quotas.
2. Commit the post and `post.published` outbox event together. Return the created post immediately; follower fanout is asynchronous.
3. A fanout coordinator records a job keyed by post ID and publication version, reads follower IDs in bounded batches, and saves progress. Retries upsert inbox references under their unique key.
4. Workers recheck whether the post is still eligible before processing a batch. A stale publication event must not resurrect a deleted post. Regardless, read-time policy filtering remains required.
5. On feed read, obtain bounded candidates from the reader inbox and followed pull-source timelines. Merge by `(published_at, id)` and deduplicate by post ID. Candidates are references, not permission to return content.
6. Fetch each batch's post payload and relevant current visibility, blocks, follow status, and moderation under a coherent authoritative snapshot, then return only the authorized payload versions. A cached body may be reused only if its version matches the version authorized by that check. Continue scanning within a fixed work budget until 30 visible posts are collected or candidates are exhausted. Return a short page rather than exceeding the latency budget indefinitely.

To satisfy the strong policy target, authorization reads must use an authority that includes committed policy changes relevant to the read. A stale read replica or cache without a reliable version barrier cannot satisfy it. Availability may need to degrade if current policy is unavailable; serving cached private content is not an acceptable fallback.

Do not authorize public version 1 and then independently hydrate current version 2: an edit may have made version 2 private in between. In the initial PostgreSQL design, fetch the content and policy through a consistent query or an appropriate transaction snapshot established after the request begins. If they live in separate services, define a version/authorization protocol and reject a result whose payload version was never authorized. This also preserves the stated distinction between policy changes committed before a read begins and changes racing with an already-authorized read.

## Pagination, refresh, edits, and deletes

For chronological pagination, use an opaque signed cursor containing user/tenant binding, ordering version, an upper publication boundary, the last examined `(published_at, id)` pair, and an expiry. The query uses a strict tuple comparison below that pair with descending order. Do not use only a timestamp because many posts can share it. Large OFFSET values make the database compute and skip earlier rows and are unstable under insertions. [PostgreSQL LIMIT and OFFSET](https://www.postgresql.org/docs/current/queries-limit.html).

The cursor advances over examined candidates, including filtered entries, so a page of blocked posts does not trap the client in a loop. If the work budget ends early, return a next cursor even with fewer than 30 results. Bind cursor state to the authenticated viewer; a signature prevents tampering but does not replace access checks.

In this chronological design, a feed session is not a database snapshot held open across HTTP requests. New posts above its upper boundary wait for refresh. Late fanout of an older post can be omitted from the current session if it arrives behind the cursor; a refresh rebuilds the view. This is an explicit freshness compromise. If strict snapshot membership is required, materialize a bounded ordered ID list for the session and paginate that, still applying current safety filters.

Ranked feeds similarly need a model/version and stable session order or they will duplicate and skip items as scores change. A stored candidate list solves ordering stability at a storage cost; current blocks and deletions can remove entries without reranking everything. Ranking must never reinstate an item removed by safety filtering.

Editing changes the source post version, not its publication position. Delete marks the source immediately and asynchronously removes derived references. Tombstones or version checks prevent old events from recreating content. Media cleanup follows retention rules and checks whether another allowed post references the same object.

## Reliability, security, and scaling evolution

Fanout workers use fair scheduling across tenants and authors. A celebrity event must not occupy every worker and delay ordinary-user updates. Track queue age, not just queue depth: 100 jobs of five million recipients are different from 100 jobs of 20 recipients. Batch inserts, bound retries, and send repeatedly failing work to a reviewable quarantine with its event identity intact.

Start with SQL pull queries while the graph is small. Add inbox materialization when repeated merging becomes expensive. Add hybrid fanout when skew overwhelms write budgets. Shard inboxes by recipient ID when necessary, while routing author timelines by author ID. These different access patterns justify separate derived stores; they do not automatically require separate microservices on day one.

Rate-limit follow churn and posting, protect private-account approval workflows, and prevent cross-tenant media or post references. Personalized responses use private/no-store policies unless a carefully isolated per-user cache is implemented. Avoid embedding sensitive post text in queue payloads or metrics labels. Audit access-control failures without storing entire feeds in logs.

Monitor page latency, visible candidates per examined candidate, fanout age, rebuild frequency, source-version lag, duplicate candidate rate, and policy-filter denials. Excessive filtering can indicate delayed cleanup or an inefficient candidate strategy. Recovery restores source posts and relationships first; inboxes can be rebuilt with checkpoints and current policies. Preserve deletion information across restore so replay does not revive removed posts.

## Tabletop lab with a deterministic fixture

Use reader R following ordinary author A and celebrity C. A's posts have IDs 10 and 12; C's post is 11. Their publication keys are ordered `12 > 11 > 10`. R's inbox initially contains 12 and 10; C's pull timeline contains 11. Set page size to two and scan budget to four candidates.

| Action or failure | Expected state and response |
|---|---|
| Read first page | Merge produces 12, 11; cursor is below the last examined key |
| Fanout retries A's post 12 three times | Unique reference remains one; response contains 12 once |
| Block C before a fresh read begins | 11 is filtered even if a stale candidate cache still contains it |
| Delete 12 while its fanout job is paused, then resume | Source remains deleted; stale work cannot make 12 visible |
| Edit 10 from version 1 to version 2 | Hydration returns version 2 without changing its ordering key |
| New post 13 arrives after first-page boundary | It waits for refresh; older-page traversal remains below the boundary |
| Late post 9 is inserted behind an already advanced cursor | Current session may omit it; documented refresh behavior recovers it |
| All four examined candidates are blocked | Return an empty/short page and advancing cursor, not an infinite scan |

Write every candidate list, filtered result, and cursor on paper. Then change C to five million followers and calculate the push cost versus merging one celebrity timeline for each active reader. Include the readers who never open the application: push work for them may be wasted.

## Exercises and completion checklist

Design follow/unfollow behavior during a fanout run. Decide whether a new follower sees historical posts and how much history is rebuilt. Then specify a ranked-feed cursor with a 15-minute session lifetime, including the storage cost of 300 candidate IDs per session.

- [ ] Source truth and derived inbox references have separate responsibilities.
- [ ] Fanout choices include celebrity skew and inactive audiences.
- [ ] Current authorization overrides stale candidates and old events.
- [ ] Pagination defines ties, late arrivals, refreshes, and bounded filtering.
- [ ] Rebuilds preserve deletion and tenant-isolation guarantees.

Continue with [video streaming](04-video-streaming.md) or return to the [case-study index](README.md).
