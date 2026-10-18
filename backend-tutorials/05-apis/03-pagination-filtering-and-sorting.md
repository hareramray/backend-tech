# Pagination, filtering, sorting, and bounded queries

[Curriculum](../README.md) · Prerequisites: [SQL](../02-data/01-relational-modeling-and-sql.md) and [indexes](../02-data/03-indexing-and-query-performance.md)

## Why list endpoints need a contract

An unbounded list turns growth into memory, latency, and database-load failures. A robust collection endpoint specifies a default page size, maximum size, stable ordering, allowed filters, cursor lifetime, and behavior during concurrent changes.

Offset pagination uses `LIMIT` and `OFFSET`. It supports familiar page numbers but the database may scan or discard a growing number of earlier rows. Concurrent insertion/deletion can shift page boundaries. Keyset pagination asks for rows after a specific ordered position and is usually better for sequential traversal of large changing datasets.

A limit without deterministic ordering is not a reliable page definition. PostgreSQL explicitly documents that LIMIT/OFFSET need predictable ordering to select a predictable subset. [PostgreSQL LIMIT/OFFSET](https://www.postgresql.org/docs/17/queries-limit.html).

## Build a total order

Order tasks by `created_at DESC, id DESC`. The ID breaks timestamp ties. For the next page, use the same direction and comparison:

```sql
-- Parameterized query fragment: $1 tenant, $2 timestamp, $3 ID, $4 page size + 1.
SELECT id, title, status, created_at
FROM tasks
WHERE tenant_id = $1
  AND (created_at, id) < ($2::timestamptz, $3::uuid)
ORDER BY created_at DESC, id DESC
LIMIT $4;
```

The capstone expresses this condition with Django `Q` objects. It requests one extra row to determine whether a next page exists. The cursor contains the final returned row, not the extra lookahead row.

Preserve timestamp precision. Python datetime supports the PostgreSQL microseconds used here; a JavaScript client should transmit the opaque cursor unchanged. Do not rebuild it from a rounded display timestamp.

## Cursor scope and security

A cursor is a continuation token, not a credential. The server still applies authorization and tenant filtering. Sign it if tampering should be rejected; encrypt only if its contents must be confidential. Bound its size and validate its structure even when signatures are used.

Include a format version, sort position, tenant/query scope, and expiry policy. If a client changes filters between pages, reject a cursor from the old query or restart traversal. Canonicalize the filter representation before hashing or signing it; equivalent requests should not differ merely because JSON keys arrived in another order.

## Safe filters and sorts

Build an allowlist of public query parameters and translate them into controlled ORM expressions. Never pass arbitrary user-supplied ORM field paths directly into `filter` or `order_by`. Such input can expose unintended relations, create expensive joins, or bypass business-level restrictions.

```python
# Design fragment after authentication and input parsing.
SORTS = {"newest": ("-created_at", "-id"), "oldest": ("created_at", "id")}
sort_fields = SORTS[validated_sort]
query = Task.objects.filter(tenant_id=authorized_tenant_id)
if validated_status is not None:
    query = query.filter(status=validated_status)
query = query.order_by(*sort_fields)
```

Each allowed sort needs a matching cursor comparison; switching the order without switching its inequality is incorrect. A composite index starting with tenant and filter columns may help one workload and add write overhead to every update. Measure actual plans and cardinality.

Exact total counts can cost more than fetching one page. Offer counts only when their accuracy and cost are justified; approximate counts must be labeled. Large exports often belong in a background job with snapshot or watermark semantics.

## Practical lab

1. Create several tasks, including multiple tasks with exactly the same timestamp in a test fixture.
2. Traverse with page size two and collect IDs.
3. Verify every stable fixture ID appears exactly once.
4. Insert a new newest row between page requests. Explain why it appears on a fresh traversal rather than a later page of the old traversal.
5. Tamper with a cursor and try one from a different tenant.
6. Test size zero, negative, above 100, repeated parameters, and a malformed cursor.
7. Run EXPLAIN on a realistic dataset and compare the first page with a deep offset page.

Do not mistake keyset traversal for a snapshot. Mutable sort keys and changes to filter membership can still move records. For an exact export, define a transaction snapshot, immutable sequence watermark, or materialized export set with explicit resource costs.

## Checklist and extensions

- [ ] Every order has a unique tie-breaker and compatible cursor predicate.
- [ ] Page sizes and filter complexity are bounded.
- [ ] Cursors preserve precision and query scope.
- [ ] Tenant checks occur independently of cursor validation.
- [ ] Concurrent changes have documented traversal semantics.

Exercise: add status filtering to the capstone and bind it into the signed cursor. Add a regression test for using a `todo` cursor with a `done` query. Then compare an index with and without status as a leading equality column.
