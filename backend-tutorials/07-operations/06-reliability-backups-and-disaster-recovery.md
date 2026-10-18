# Reliability, backups, and disaster recovery

[Tutorial index](../README.md) · Related: [incident runbooks](07-incident-response-and-production-runbooks.md)

## Outcome and prerequisites

Design recovery around business needs and prove that stored backups can reconstruct a usable service. You need the capstone's local PostgreSQL 17 Compose service named `db`, synthetic data, and Docker Compose. Commands below run from the directory containing that Compose file. They are a local learning exercise and do not operate on production or delete an existing database.

Reliability includes continuing through faults, preventing data corruption, and recovering when prevention fails. Redundancy reduces some outages, while backups address restoration. A replica can rapidly copy an accidental deletion, so replication is not a substitute for historical recovery. A successful backup job proves that a job ran; a restore drill proves substantially more.

## Define recovery objectives and failure scope

Recovery point objective (RPO) is the tolerable amount of data loss measured in time or another explicit business unit. Recovery time objective (RTO) is the target duration to restore the required service after a defined disruption. Neither is a universal promise: specify incident start, recovery criteria, affected datasets, and measurement method.

If nightly backups are the only recovery source, the potential data-loss window can approach a day plus backup delay. If restoration requires six hours of manual steps, a 30-minute RTO is unsupported. Include detection, escalation, infrastructure provisioning, secret access, data restoration, validation, and traffic switching in recovery time.

Model process failure, host loss, zone loss, region loss, operator error, malicious modification, and corrupted credentials separately. Multiple copies in one account with identical delete permissions share a failure domain. Choose isolation, retention, encryption, and protected deletion controls according to the recovery requirements.

## Choose a backup method

A logical dump captures database objects and data in a portable representation suitable for many selective restores and migrations. A physical backup captures a database cluster's storage representation and has different version and infrastructure constraints. Continuous WAL archiving combined with an appropriate base backup supports point-in-time recovery; `pg_dump` alone cannot provide that capability. [PostgreSQL continuous archiving](https://www.postgresql.org/docs/17/continuous-archiving.html)

PostgreSQL custom-format dumps support flexible restoration with `pg_restore`. A dump is internally consistent for its database snapshot, but separate backups of a database and object store do not automatically represent one globally consistent application state. Plan reconciliation for cross-system references. [PostgreSQL pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html)

## Walkthrough: create and restore a logical backup

1. Seed a few synthetic tasks through the capstone. Record the expected tenant, task count, and selected task IDs. Verify `docker compose ps` shows the intended local `db` service before continuing.

2. Create a custom-format dump inside the container, then copy it to a local backup directory. PowerShell uses a unique filename so repeated drills do not overwrite earlier artifacts.

```powershell
$backupFile = 'taskapp-' + (Get-Date -Format 'yyyyMMdd-HHmmss') + '.dump'
New-Item -ItemType Directory -Path '.\backups' -Force | Out-Null
docker compose exec -T db pg_dump -U taskapp -d taskapp --format=custom --file="/tmp/$backupFile"
if ($LASTEXITCODE -ne 0) { throw 'Backup failed' }
docker compose cp "db:/tmp/$backupFile" ".\backups\$backupFile"
if ($LASTEXITCODE -ne 0) { throw 'Backup copy failed' }
Get-FileHash -Algorithm SHA256 -LiteralPath ".\backups\$backupFile"
```

Using an output file and `docker compose cp` avoids piping a binary archive through PowerShell's text redirection. The lab's local database authentication supplies access inside the container; use an approved credential mechanism for any other environment. Store backup metadata and checksums separately enough to detect accidental changes, and protect archives as sensitive data.

3. Inspect the archive contents without restoring it:

```powershell
docker compose exec -T db pg_restore --list "/tmp/$backupFile"
```

Expect entries for schemas, tables, data, indexes, constraints, and Django metadata. A readable archive list is useful but still does not prove complete recovery.

4. Create a separate empty database and restore there. The fixed lab name intentionally fails if already present; choose a new explicitly reviewed restore name on a later drill rather than dropping an existing database.

```powershell
docker compose exec -T db createdb -U taskapp taskapp_restore_lab
if ($LASTEXITCODE -ne 0) { throw 'Restore database was not created; stop here' }
docker compose exec -T db pg_restore -U taskapp --dbname=taskapp_restore_lab --no-owner --no-privileges --exit-on-error --single-transaction "/tmp/$backupFile"
if ($LASTEXITCODE -ne 0) { throw 'Restore failed' }
docker compose exec -T db psql -U taskapp -d taskapp_restore_lab -c 'SELECT tenant_id, count(*) FROM tasks GROUP BY tenant_id;'
```

The no-owner/no-privileges options suit this single-role lab. Real recovery must reconstruct intended owners, grants, and global roles rather than accidentally granting everything to an application role. `pg_restore` options control error handling and transactional restoration; choose them with archive size and operating constraints in mind. [PostgreSQL pg_restore](https://www.postgresql.org/docs/17/app-pgrestore.html)

5. In a separate shell with the capstone settings, change only `PGDATABASE` to `taskapp_restore_lab` and start Django on `127.0.0.1:3001`. Verify readiness and task reads against the restored database, then exercise a synthetic write. Record elapsed recovery time and the newest recovered task. Do not redirect the original application or real users during this drill.

## Validate more than row counts

Check schema/migration version, expected constraints and indexes, tenant isolation, relationships, permissions, and representative queries. Verify application compatibility with the restored schema. For a point-in-time drill, use known markers before and after the target time and show that recovery lands between them as intended.

Test missing archive segments, lost encryption-key access, insufficient disk space, and expired credentials in a controlled recovery environment. A backup that requires an unavailable key is operationally unusable. Monitor backup age, completion, archive lag, retention enforcement, and restore-test freshness. Set ownership for every failure signal.

Restore untrusted dumps only in an appropriately isolated environment: database restoration can execute SQL contained in the archive. Treat both the archive source and the target's privileges as part of the trust boundary.

## Resilience and exercises

Use bounded timeouts, load shedding, retry budgets, idempotent operations, and bulkheads to keep partial failures from becoming global ones. Failover needs fencing or another mechanism to avoid competing writers. Test the behavior of clients during DNS changes and database promotion, including uncertain outcomes for writes whose acknowledgments were lost.

Common mistakes are keeping the only backup on the database host, confusing replicas with backups, reporting RTO without a timed drill, and restoring data without restoring configuration or permissions. Never delete old recovery sources before verifying the new recovery chain.

Exercise: write a recovery plan for database corruption discovered four hours after it began. Identify the clean recovery point, expected lost writes, reconciliation approach, approval owner, communication checkpoints, and validation criteria. Then compare the measured local drill with the desired RPO/RTO and list the concrete capabilities still missing.
