# Migrations and the data lifecycle

Prerequisites: [relational modeling](01-relational-modeling-and-sql.md), [transactions](02-transactions-and-concurrency.md), [query performance](03-indexing-and-query-performance.md), and the [course map](../README.md). Initialize the [capstone schema](../09-capstone/01-task-service.md) before running the lab.

## Objectives

Change a schema while multiple application versions coexist, backfill without overwhelming the database, define retention and restore procedures, and separate code rollback from data recovery.

## A migration is an operational change

Schema migration files record ordered transitions such as adding a column, index or constraint. A migration runner should maintain an applied-version ledger, prevent concurrent runners, and fail visibly. Once a migration has been shared or deployed, add a new corrective migration instead of rewriting history.

Not every migration fits safely in one transaction. Some operations acquire strong locks, scan large tables or rewrite stored data. A syntactically valid migration can still cause an outage if it queues behind a long transaction and blocks new traffic. Test on representative data, inspect the operation's lock behavior, set a bounded lock timeout, and know how to cancel and resume.

Separate schema changes, data backfills, and application behavior changes. They have different rollback and observability needs. A deployment tool reporting “success” does not prove the new invariant holds or the old application still works.

## Expand, migrate, contract

For a breaking change, first expand the schema so old and new code can coexist. Deploy compatible code, backfill data incrementally, verify consistency, switch reads, and only later remove old behavior. The overlap must have a clear source of truth: casual dual writing can create divergence if one path succeeds and the other fails.

For example, replacing a free-text status with a new representation cannot safely mean renaming a column while old servers are still running. Introduce the new representation, define conversion and compatibility rules, and wait until every old reader and writer is retired before removing the original.

PostgreSQL supports adding certain constraints as NOT VALID and validating existing rows later; new changes still face the constraint. Lock levels and rewrite behavior depend on the exact ALTER TABLE operation. [ALTER TABLE reference](https://www.postgresql.org/docs/current/sql-altertable.html)

## Lab: introduce task priority

This lab adds priority to the capstone tasks table. Run each phase separately and check its result.

1. Expand with a nullable column and a default for new writes:

~~~sql
BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '10s';
ALTER TABLE tasks ADD COLUMN priority integer;
ALTER TABLE tasks ALTER COLUMN priority SET DEFAULT 0;
ALTER TABLE tasks ADD CONSTRAINT tasks_priority_range
  CHECK (priority BETWEEN 0 AND 3) NOT VALID;
COMMIT;
~~~

This sequence deliberately leaves old rows NULL so the backfill is visible. A default affects future rows that omit the column; explicit NULL still needs a separate policy. Deploy writers that send a valid priority, and readers that use COALESCE(priority, 0) during the transition.

2. Backfill a bounded batch. Repeat the entire transaction until UPDATE reports zero rows:

~~~sql
BEGIN;
SET LOCAL lock_timeout = '1s';
SET LOCAL statement_timeout = '5s';
WITH batch AS (
  SELECT id
  FROM tasks
  WHERE priority IS NULL
  ORDER BY id
  FOR UPDATE SKIP LOCKED
  LIMIT 500
)
UPDATE tasks AS t
SET priority = 0
FROM batch
WHERE t.id = batch.id;
COMMIT;
~~~

The update is idempotent because it only fills missing values. The capstone uses globally unique task IDs; a schema with tenant-local IDs must include tenant_id in the batch join. Pause between batches in the worker, not while holding a transaction. Track rows remaining, batch duration, lock waits, replication lag and database load.

3. Check and enforce the invariant after all writers are compatible:

~~~sql
SELECT count(*) AS remaining FROM tasks WHERE priority IS NULL;

BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '30s';
ALTER TABLE tasks VALIDATE CONSTRAINT tasks_priority_range;
ALTER TABLE tasks ADD CONSTRAINT tasks_priority_present
  CHECK (priority IS NOT NULL) NOT VALID;
COMMIT;

ALTER TABLE tasks VALIDATE CONSTRAINT tasks_priority_present;

BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '30s';
ALTER TABLE tasks ALTER COLUMN priority SET NOT NULL;
COMMIT;
~~~

The standalone validation statement needs an appropriate session-level statement timeout in a real migration runner. Use a deliberate timeout for the production table size; the tutorial values are teaching defaults, not universal operational settings. If validation fails, investigate new NULL writes and backfill again instead of dropping the invariant silently.

4. Verify that an insert omitting priority receives zero, explicit NULL fails, and a value outside 0–3 fails. Only after the compatibility window closes should readers stop using the temporary fallback.

## Keep Django's migration state aligned

The SQL lab exposes database mechanics directly. A deployed Django change must also update model definitions and migration state. Do not apply the same change once by hand and again through a generated migration in the same database.

In a fresh practice database, add the expanded field to the Task model, run `python manage.py makemigrations tasks`, inspect the generated file and its `sqlmigrate` output, then run `python manage.py migrate`. Use separate migrations for expansion, data movement and final constraints. Historical data migrations obtain models with `apps.get_model` from the migration context, rather than importing today's model class; otherwise replaying an old migration after future model changes can break.

Large backfills often belong in a resumable management command between schema deployments. If using RunSQL or SeparateDatabaseAndState for specialized operations, make the resulting database and Django state describe the same schema. Test both a fresh installation and an upgrade from the previous release. [Django migration documentation](https://docs.djangoproject.com/en/5.2/topics/migrations/)

## Rollback, recovery and backups

A code rollback is safe only if the older code understands the current schema and data. Dropping a new nullable column may be reversible in an empty lab, but deleting populated columns loses information. Prefer forward fixes when reversing a migration would destroy newly written data.

Backups protect against a different failure class than replicas. Replication can faithfully copy an accidental deletion. Define recovery point objective, the acceptable data loss window, and recovery time objective, the acceptable restoration time. Choose logical backups, physical backups and WAL-based point-in-time recovery according to those requirements. PostgreSQL documents these as distinct backup approaches. [Backup and restore overview](https://www.postgresql.org/docs/current/backup.html)

A backup is only evidence of recoverability after a restore rehearsal. Restore into an isolated environment, verify schema versions and representative records, check application startup, measure elapsed time, and record missing dependencies such as encryption keys, extensions and object storage data. Protect backup credentials separately and restrict access to restored sensitive data.

## Retention, deletion and archival

Define data categories, ownership, retention periods and deletion dependencies. Soft deletion is a visibility state; it does not remove the data. It affects unique constraints, joins, cache keys, search indexes and authorization. Hard deletion may need staged work across database rows, blobs, search documents and derived analytics.

Keep a deletion workflow state so retries can complete partial cleanup. A tombstone or deletion ledger can prevent an old event replay from recreating removed data. Backups may retain deleted records until their retention expires; restoration procedures must reapply applicable deletion records before reopening access.

Archive infrequently accessed data only when you can still locate and restore it when needed. Partition dropping can simplify time-based lifecycle operations, but partition boundaries, foreign keys and retention must be designed together. Never equate “old timestamp” with “safe to delete” without business ownership.

## Failure tests, exercises and checklist

Hold a lock in another session and verify the migration fails within its lock budget. Stop the backfill halfway, restart it, and verify already migrated rows remain correct. Run the old and new application versions against the expanded schema. Restore a backup and compare a known sample with the source.

Common mistakes include one giant backfill transaction, migrations on every application replica at startup, unlimited lock waits, untested down migrations, storing backups only beside the primary, and forgetting blob or search cleanup.

Write a rollout plan for renaming a task field without downtime. Include compatibility, monitoring, rollback boundaries and the earliest safe removal date.

- [ ] Migration ownership and ordering are explicit.
- [ ] Old and new application versions coexist safely.
- [ ] Backfills are bounded, observable and resumable.
- [ ] Constraints are validated before cleanup.
- [ ] Restore rehearsal meets the documented recovery objectives.
- [ ] Retention covers databases, caches, indexes, blobs and backups.
