# NoSQL and alternative data models

Prerequisites: [relational modeling](01-relational-modeling-and-sql.md), [transactions](02-transactions-and-concurrency.md), and the [course map](../README.md). The executable lab uses PostgreSQL JSONB, avoiding a requirement to install several database products.

## Objectives

Choose a data model from access patterns and invariants, understand embedding and partitioning, and evaluate consistency and operational costs before adding another datastore.

## “NoSQL” is a family, not a guarantee

Key-value systems address data through keys and often optimize simple point operations. Document databases store nested records and support document-oriented queries. Wide-column systems organize data around partition keys and clustering rules. Graph databases represent nodes and relationships for traversal. Time-series and search systems optimize other specialized workloads. Products can support several models, so labels are only a starting point.

Ask what the application must do: retrieve a session by token, show an order with its line items, traverse relationships, search text, or aggregate measurements by time. Then list the required invariants, maximum object sizes, update frequency, query fan-out, retention and availability needs. Do not choose a model simply because the current payload is JSON.

Embedding keeps related data together and can make a single-record update convenient. Referencing avoids duplicating independently changing entities and unbounded nested growth. MongoDB's modeling guidance frames this choice around application access patterns and the relationships between documents. [MongoDB data modeling](https://www.mongodb.com/docs/manual/data-modeling/)

A “schema flexible” store still has a schema in application code, validators, indexes and downstream consumers. Without explicit versioning, a field changing from a string to an object can break consumers just as surely as a SQL column change.

## Partitioning and consistency

A partition key distributes records and determines which operations stay local. A tenant ID may provide locality but create a hot partition for a large tenant. A random key distributes writes but makes time-range queries harder. A time bucket spreads retention work while potentially concentrating current writes. Estimate both the average and the largest partition.

Secondary indexes and cross-partition queries have costs that differ by system. A query that looks convenient in an SDK may require scatter/gather across many nodes. Denormalized read models can make important access paths efficient, but require coordinated updates, replay or reconciliation.

Consistency is an operation-level contract. Determine whether a read can miss a recent write, whether conditional writes are supported, what a quorum actually covers, and what happens during a partition or failover. Replication is not automatically a transaction across every related record. “Eventually consistent” needs a practical bound, stale-state UX and a repair process.

## Lab: use structured columns with flexible metadata

This standalone PostgreSQL table stores task-like documents. It is independent of the capstone tasks table:

~~~sql
CREATE TABLE document_tasks (
  tenant_id uuid NOT NULL,
  id uuid NOT NULL,
  schema_version integer NOT NULL DEFAULT 1 CHECK (schema_version >= 1),
  title text NOT NULL CHECK (length(btrim(title)) > 0),
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
  version integer NOT NULL DEFAULT 1,
  PRIMARY KEY (tenant_id, id),
  CHECK (jsonb_typeof(metadata) = 'object')
);

INSERT INTO document_tasks(tenant_id, id, title, metadata)
VALUES (
  '11111111-1111-4111-8111-111111111111',
  '22222222-2222-4222-8222-222222222222',
  'Prepare launch',
  '{"labels":["release","urgent"],"estimateMinutes":45}'::jsonb
);
~~~

Stable ownership and identity are relational columns. Optional, evolving metadata is JSONB. This hybrid can avoid an additional datastore while preserving relational constraints. JSONB supports containment queries and indexing, with tradeoffs different from JSON's textual representation. [PostgreSQL JSON types](https://www.postgresql.org/docs/current/datatype-json.html)

1. Query an access pattern:

~~~sql
SELECT id, title
FROM document_tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND metadata @> '{"labels":["urgent"]}'::jsonb;
~~~

2. Add an index only if this query matters at meaningful scale:

~~~sql
CREATE INDEX document_tasks_metadata_idx
ON document_tasks USING gin(metadata);
~~~

Inspect EXPLAIN before and after with representative data. A small table may still use a sequential scan. The primary key supports tenant-and-ID lookups; the JSON index does not remove the need for the tenant predicate.

3. Apply an optimistic update:

~~~sql
UPDATE document_tasks
SET metadata = jsonb_set(metadata, '{estimateMinutes}', '60'::jsonb),
    version = version + 1
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND id = '22222222-2222-4222-8222-222222222222'
  AND version = 1
RETURNING metadata, version;
~~~

The second execution should affect zero rows. If an API allows JSON patches, permit only recognized paths and validate type and range; accepting an arbitrary patch can let clients overwrite security-sensitive fields.

4. Compare missing and explicit JSON null:

~~~sql
SELECT metadata ? 'deadline' AS has_deadline,
       metadata -> 'deadline' AS json_deadline,
       metadata ->> 'deadline' AS text_deadline
FROM document_tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111';
~~~

Define whether absence means “unspecified,” “inherit,” or “clear.” PATCH APIs often need a distinction between omitted fields and explicit clearing.

## A decision worksheet

For a session store, evaluate key lookup latency, expiration, revocation and durability expectations. For a product catalog, evaluate nested variants, indexing, schema evolution and independent inventory updates. For relationship recommendations, evaluate traversal depth and whether a relational recursive query already suffices. For telemetry, evaluate ingestion rate, time windows, compression and retention.

Use polyglot persistence only when a specific requirement justifies the operational burden: backups, credentials, monitoring, expertise, migrations and consistency repair all multiply. A relational database plus JSON and appropriate indexes is often a good starting point, but cannot replace every specialized engine.

When introducing a second store, identify the authoritative system and rebuildable projections. Prefer one-way propagation from committed source data over uncontrolled dual writes. If both stores accept independent writes, define conflict ownership and reconciliation explicitly.

## Failure tests and common mistakes

Insert an array as metadata and verify the object constraint rejects it. Insert a string estimate and observe that the current database schema accepts it: then add application validation or a suitable database constraint. Simulate two clients replacing the entire metadata object and show how version checks prevent lost updates.

Common mistakes include unbounded embedded comment arrays, choosing a low-cardinality partition key, duplicating data without repair tooling, assuming every transaction is global, and treating JSON storage as permission to skip schema design.

## Exercises and completion checklist

Design an order record with bounded line items and separate shipment events. Identify which facts should be embedded, referenced or projected. Estimate the largest record and partition after five years. Specify how an old reader handles schema_version 2.

- [ ] Access patterns and invariants drive the model.
- [ ] Record growth and hot partitions have explicit limits.
- [ ] Missing, null and empty values have defined semantics.
- [ ] Consistency and conditional-write behavior are documented.
- [ ] Schema evolution and projection repair are designed.
- [ ] A second datastore has a measured reason to exist.

Continue with [caching](06-caching-and-redis.md) and [search and analytics](07-search-and-analytics.md).
