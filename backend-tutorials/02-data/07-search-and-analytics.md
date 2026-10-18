# Search and analytics

Prerequisites: [SQL modeling](01-relational-modeling-and-sql.md), [indexing](03-indexing-and-query-performance.md), the [course map](../README.md), and the initialized [capstone schema](../09-capstone/01-task-service.md).

## Objectives

Distinguish transactional lookup, full-text search and analytics; build a tenant-safe text query; and design derived indexes and analytical datasets with clear freshness and reconciliation contracts.

## Three different questions

A transactional lookup asks for a specific authoritative fact, such as a task's current status. Search asks which documents are relevant to a phrase, often with ranking, language analysis, typo tolerance or facets. Analytics asks how many events or entities satisfy a definition over a period. These workloads overlap but have different latency, consistency and correctness needs.

An indexed exact match is not full-text search. Text search may tokenize words, normalize forms, remove stop words and rank matches. Configuration matters: an English analyzer may be unsuitable for multilingual text, identifiers or product codes. Always test a representative relevance dataset rather than judging by one successful query.

PostgreSQL provides tsvector document representations, tsquery queries and text-search configurations, allowing many applications to start inside their existing database. [PostgreSQL text-search introduction](https://www.postgresql.org/docs/current/textsearch-intro.html)

## Lab: add text search to task titles

This lab adds an expression index to the existing tasks table. Run the DDL once:

~~~sql
CREATE INDEX tasks_title_search_idx
ON tasks USING gin(to_tsvector('english', title));
~~~

Run the following executable SQL. The CTE simply makes the query expression readable:

~~~sql
WITH search AS (
  SELECT websearch_to_tsquery('english', 'launch checklist') AS query
)
SELECT t.id, t.title,
       ts_rank(to_tsvector('english', t.title), search.query) AS rank
FROM tasks AS t
CROSS JOIN search
WHERE t.tenant_id = '11111111-1111-4111-8111-111111111111'
  AND to_tsvector('english', t.title) @@ search.query
ORDER BY rank DESC, t.id
LIMIT 20;
~~~

Seed titles containing the search terms if the current dataset has none. The same language configuration appears in index and query expressions. In application code, pass the user's search string as a SQL value parameter, limit its length, cap the result count and set a query timeout.

The tenant predicate is mandatory even when an external search engine is used. A global index containing private documents needs authorization-aware filtering, and a stale index must not expose a document after its access rights are revoked. For sensitive results, revalidate authorization against authoritative state before returning content.

Rank is not a fixed business score. A tie-breaker makes a single result ordering deterministic, but pagination can still shift when the corpus or scoring changes. For large search sessions, use the engine's supported stable pagination or point-in-time approach and document its lifetime.

## Relevance and feature decisions

Build a small evaluation set: query, expected useful results, unacceptable results and rationale. Include synonyms, misspellings, punctuation, short queries, multiple languages and private records. Measure precision for the first few results, recall for important known matches and latency under realistic filters.

Boosting recent or popular content can help some tasks and harm others. Facets require counts within the authorized result universe, not global counts that leak private existence. Highlighting must escape untrusted content before embedding it in HTML. Search logs can contain secrets or personal data, so apply retention and access controls.

A dedicated search engine can offer richer analysis, distributed indexing and relevance tools, but creates a second data system to operate. Specify maximum index lag and user behavior after a new record is written. A user may need to retrieve the newly created record directly while search catches up.

## Derived indexes and rebuilds

Treat a search index as a projection of committed source data. Use an outbox or change-data-capture feed to update it; avoid independently writing database and search state without recovery. Include entity version in events so older deliveries cannot replace newer documents. Deletions need tombstones or an equivalent mechanism.

To rebuild, create a new index, record a source cutoff, backfill the source, apply changes that occurred during the backfill, verify counts and sample content, then switch readers. Keep the previous index available during a deliberate rollback window. The exact checkpoint method depends on the source and engine; a timestamp alone may not capture every concurrent change safely.

Monitor event age, failed documents, authorization discrepancies, delete lag and reconciliation mismatches. “No indexing exceptions” does not prove the projection is complete.

## Lab: a simple analytical query

The following SQL counts task creation by UTC day. It uses a half-open time range, so adjacent periods do not double-count the boundary:

~~~sql
SELECT date_trunc('day', created_at AT TIME ZONE 'UTC') AS utc_day,
       count(*) AS tasks_created
FROM tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
  AND created_at >= timestamptz '2025-01-01 00:00:00+00'
  AND created_at < timestamptz '2026-01-01 00:00:00+00'
GROUP BY 1
ORDER BY 1;
~~~

This measures created tasks, not completed tasks, active users or historical task status. The current tasks table cannot reconstruct every past status transition; that requires events or history records. Define a metric's grain, time zone, filters, deduplication rule and late-arrival policy before building a dashboard.

For larger analytical systems, distinguish facts from dimensions. A fact table might have one row per status transition, with event time and ingestion time. Slowly changing dimensions preserve historical attribute changes when analysis requires “what was true then.” Incremental transformations need watermarks, replay windows, unique event identities and reconciliation.

OLTP databases favor small operational transactions; warehouses and column-oriented stores often favor broad scans and aggregates. Moving analytics off the primary can protect request latency, but introduces freshness and pipeline failure concerns. Materialized aggregates offer an intermediate step if their refresh behavior fits the workload.

## Failure tests and common mistakes

Search for the same title under two tenants and verify isolation. Change a title and verify when the result updates. Replay an older event after a newer event and ensure the newer version survives. Delete a task and check database, search, cache and analytical handling.

Common mistakes include confusing substring matching with relevance, trusting stale search authorization, rebuilding without concurrent-change capture, counting retries as new events, mixing local and UTC boundaries, and presenting a current-state count as a historical metric.

## Exercises and completion checklist

Create twenty representative search queries and score results before and after one relevance change. Design a daily completion metric that handles duplicate and late events. Write a rebuild runbook with a measurable success condition.

- [ ] Search behavior and relevance are evaluated explicitly.
- [ ] Tenant and permission filters apply to results and facets.
- [ ] Index freshness, deletion and rebuild procedures are defined.
- [ ] Analytical metrics have precise grain and time semantics.
- [ ] Duplicate, late and missing events have a repair policy.

Continue with [event-driven systems and outbox](../04-async/03-event-driven-systems-and-outbox.md).
