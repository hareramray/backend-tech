# Indexing and query performance

Prerequisites: [SQL modeling](01-relational-modeling-and-sql.md), [transactions](02-transactions-and-concurrency.md), and the [course map](../README.md). The lab creates a separate table, so it does not require a populated capstone.

## Objectives

Read an execution plan, design an index around a real access pattern, implement deterministic keyset pagination, and measure improvement without confusing a small demonstration with a production benchmark.

## Performance starts with a workload

Specify the query, its frequency, result size, data distribution and latency target. A once-a-day scan and a search executed thousands of times per second need different treatment. Include the application path: connection-pool waits, network round trips, JSON serialization and frontend pagination can dominate a fast SQL statement.

An index is a second representation of data that accelerates some access paths while consuming storage and write work. B-tree indexes support common equality, range and ordered access. GIN is useful for supported composite values such as full-text vectors or JSONB; BRIN can help large physically correlated data; GiST supports specialized operator classes. Index type alone does not determine usefulness: predicates, sort order, operator classes, statistics and selectivity all matter. [PostgreSQL index overview](https://www.postgresql.org/docs/current/indexes.html)

Avoid blanket rules such as “index every foreign key” or “put the most selective column first.” Foreign-key access often benefits from indexing, but workload and maintenance costs matter. For a query with tenant equality followed by time ordering, an index beginning with tenant_id and continuing with the time and ID ordering is a useful hypothesis to test.

## Lab: measure a tenant feed

Run these SQL statements once in a learning database:

~~~sql
CREATE TABLE perf_tasks (
  id bigint PRIMARY KEY,
  tenant_id integer NOT NULL,
  title text NOT NULL,
  status text NOT NULL,
  created_at timestamptz NOT NULL
);
INSERT INTO perf_tasks
SELECT n, n % 100, 'Task ' || n,
       CASE WHEN n % 4 = 0 THEN 'done' ELSE 'todo' END,
       timestamptz '2025-01-01 00:00:00+00'
         + (n / 10) * interval '1 second'
FROM generate_series(1, 100000) AS series(n);
ANALYZE perf_tasks;
~~~

The repeated timestamps deliberately create ties. Synthetic data is useful for a controlled experiment but does not reproduce all production skew or correlations.

Inspect the baseline:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM perf_tasks
WHERE tenant_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

EXPLAIN reports the chosen plan; ANALYZE executes it and records observed work. That execution matters for modifying statements: EXPLAIN ANALYZE UPDATE really updates rows. Plan estimates are not milliseconds; compare estimated and actual row counts, loops, scan types, sorts and buffers. [PostgreSQL EXPLAIN guide](https://www.postgresql.org/docs/current/using-explain.html)

Add the candidate index and repeat:

~~~sql
CREATE INDEX perf_tasks_tenant_feed_idx
ON perf_tasks(tenant_id, created_at DESC, id DESC);
ANALYZE perf_tasks;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM perf_tasks
WHERE tenant_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

Expect a plan that can locate the tenant's ordered rows with less scanning or sorting, although the optimizer chooses based on actual circumstances. Record the plan and buffer counts; do not assert that every environment must produce the same node or speedup.

## Lab: implement keyset pagination

Run the first-page query and copy its last row's created_at and id. Use those exact values for the next-page predicate. This example uses fixed values only to demonstrate valid SQL:

~~~sql
SELECT id, title, created_at
FROM perf_tasks
WHERE tenant_id = 42
  AND (created_at, id) <
      (timestamptz '2025-01-01 02:45:54+00', 99542::bigint)
ORDER BY created_at DESC, id DESC
LIMIT 20;
~~~

The cursor compares the complete ordering tuple. Comparing only created_at can skip rows sharing a timestamp. Ascending order needs the corresponding greater-than predicate. Nullable sort fields require an explicit null-ordering strategy; use non-null sort values when possible.

In an API, encode the last tuple together with relevant filter and sort context. Validate cursor structure and types, cap the page size, and reapply the authenticated tenant predicate on every request. A signed cursor can detect tampering but is not authorization. Use a page-size-plus-one fetch to determine whether a next page exists without an expensive total count.

Keyset pagination handles deep traversal efficiently, but it does not provide a fixed snapshot across requests. Inserts, deletes and changes to the sort key can affect what subsequent pages contain. For a consistent export, use a defined snapshot or export job with a stable cutoff and documented semantics.

## Additional techniques and tradeoffs

Partial indexes store rows matching a predicate, such as active records, and are useful when the query can use that predicate. Expression indexes support expressions such as normalized text. Covering indexes can include projected fields, but wider indexes increase storage and write cost; an index-only scan also depends on visibility information.

Do not wrap an indexed timestamp in a date function when a half-open timestamp range expresses the same filter. Parameterize query values and watch generic versus parameter-sensitive plans when data is highly skewed. Large IN lists, accidental many-to-many joins and per-row ORM calls may need query redesign rather than another index.

Keep statistics current, investigate table and index growth, and understand vacuum before blaming every slowdown on a missing index. Partitioning can help lifecycle operations and partition pruning, but is not a universal replacement for indexes. Read replicas can offload eligible reads, at the cost of possible replication lag and read-after-write surprises.

On a busy table, PostgreSQL offers concurrent index creation with operational caveats, including prohibition inside a transaction block and possible invalid indexes after failure. Check the operation's progress and outcome instead of assuming a deployment succeeded. [CREATE INDEX reference](https://www.postgresql.org/docs/current/sql-createindex.html)

## Failure tests and common mistakes

Insert several rows with the same timestamp and traverse them using a cursor; verify no duplicates or omissions in a static dataset. Compare a deep OFFSET page with the equivalent keyset page. Test tenants with very different data volumes. Add writes to a load test and observe index maintenance cost.

Common mistakes include timing a single cached query, forcing index use to satisfy a tutorial expectation, benchmarking empty tables, adding redundant indexes, returning SELECT *, and treating average latency as sufficient while tail latency worsens.

## Exercises and completion checklist

Add a status filter and compare two index candidates. Explain why an index optimized for status-specific feeds may not serve an all-status feed equally well. Measure one query before and after replacing N+1 lookups with a bounded join or batch.

- [ ] A reproducible workload and baseline are recorded.
- [ ] Estimated and actual row counts were compared.
- [ ] Index benefits and write/storage costs are understood.
- [ ] Pagination includes a unique tie-breaker and tenant filter.
- [ ] Large, skewed and concurrent workloads were considered.

Apply the same process to the [task-service capstone](../09-capstone/01-task-service.md).
