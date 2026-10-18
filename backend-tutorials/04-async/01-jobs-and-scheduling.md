# Background jobs and scheduling

Prerequisites: [course map](../README.md), [transactions](../02-data/02-transactions-and-concurrency.md), and a PostgreSQL learning database. This tutorial's queue table is standalone; use the [capstone](../09-capstone/01-task-service.md) when integrating it with the API.

## Objectives

Move slow work outside request lifetimes, build a durable job state machine, distinguish a lease from ownership forever, and define scheduling and recovery behavior.

## Decide what belongs in a job

Image transformations, exports, indexing and notifications often outlive an interactive request. An API can validate a request, persist a job, and return an operation identifier with a status URL. “Accepted” means durable responsibility for work, not completed work. A local promise, thread or timer can disappear when the process restarts and is not a durable queue.

A job needs an ID, tenant, type, versioned payload, state, availability time, attempt count and last error. Add progress only when it conveys meaningful completed work. Avoid putting secrets or complete database snapshots in payloads; store stable references and fetch authorized data at execution time when appropriate.

Typical states are queued, running, succeeded, failed and cancelled. Retries may move running back to queued. Cancellation requires cooperation: a running process might already have performed a side effect. Define when cancellation is too late and whether partial output needs cleanup.

## Lab: create a lease-based queue

Run this PostgreSQL DDL once:

~~~sql
CREATE TABLE tutorial_jobs (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id uuid NOT NULL,
  job_type text NOT NULL,
  payload jsonb NOT NULL CHECK (jsonb_typeof(payload) = 'object'),
  state text NOT NULL DEFAULT 'queued'
    CHECK (state IN ('queued','running','succeeded','failed','cancelled')),
  available_at timestamptz NOT NULL DEFAULT now(),
  attempts integer NOT NULL DEFAULT 0 CHECK (attempts >= 0),
  max_attempts integer NOT NULL DEFAULT 5 CHECK (max_attempts > 0),
  lease_token uuid,
  lease_until timestamptz,
  last_error text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX tutorial_jobs_ready_idx
  ON tutorial_jobs(available_at, id) WHERE state = 'queued';

INSERT INTO tutorial_jobs(tenant_id, job_type, payload)
VALUES (
  '11111111-1111-4111-8111-111111111111',
  'build-export',
  '{"schemaVersion":1,"format":"csv"}'
);
~~~

Claim one job in a short transaction. The fixed lease token is for this manual lab only; a real worker generates a fresh random UUID for every claim.

~~~sql
BEGIN;
SET LOCAL lock_timeout = '1s';
SET LOCAL statement_timeout = '3s';
WITH candidate AS (
  SELECT id FROM tutorial_jobs
  WHERE state = 'queued' AND available_at <= now()
    AND attempts < max_attempts
  ORDER BY available_at, id
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE tutorial_jobs AS j
SET state = 'running',
    attempts = attempts + 1,
    lease_token = '44444444-4444-4444-8444-444444444444',
    lease_until = now() + interval '30 seconds',
    updated_at = now()
FROM candidate
WHERE j.id = candidate.id
RETURNING j.*;
COMMIT;
~~~

SKIP LOCKED permits workers to skip jobs already being claimed, making it useful for queue-like consumption. It deliberately does not produce a complete consistent report of all eligible rows. [PostgreSQL locking clauses](https://www.postgresql.org/docs/current/sql-select.html)

Perform the work after committing the claim. Do not hold the row lock during an export or remote request. Replace job ID 1 below with the returned ID:

~~~sql
UPDATE tutorial_jobs
SET state = 'succeeded', lease_token = NULL, lease_until = NULL,
    updated_at = now()
WHERE id = 1 AND state = 'running'
  AND lease_token = '44444444-4444-4444-8444-444444444444'
  AND lease_until > now()
RETURNING id;
~~~

Zero rows means the worker no longer owns a valid lease. It must not announce success based solely on its local work.

## Heartbeats, recovery and duplicates

A long job can extend its lease periodically using a conditional update on ID, token, running state and an unexpired lease. Stop or fence further effects if renewal fails. A paused worker can resume after another worker has reclaimed the job; a lease does not physically stop it.

Recover expired jobs with a bounded reaper. This executable SQL handles a small learning table; add batching for a large queue:

~~~sql
UPDATE tutorial_jobs
SET state = CASE WHEN attempts >= max_attempts THEN 'failed' ELSE 'queued' END,
    available_at = now() + interval '5 seconds',
    lease_token = NULL, lease_until = NULL,
    last_error = 'lease expired', updated_at = now()
WHERE state = 'running' AND lease_until <= now();
~~~

A crash after creating an output but before marking success can produce a second attempt. Make output creation idempotent: use a stable operation identity, deterministic storage key with appropriate conditional writes, or an atomic database effect plus completion record. Side effects to external services need their own idempotency support or reconciliation.

Retries should distinguish transient errors from permanent validation or authorization failures. Use exponential backoff with jitter, an attempt cap and an overall age limit. Retain a redacted reason and a controlled retry/replay interface for failed jobs; endless retries are not recovery.

## Scheduling semantics

An interval means elapsed duration; a calendar schedule means a rule in a named time zone. “Every 24 hours” and “daily at 09:00 Asia/Kolkata” are different contracts. For daylight-saving zones, define what happens when a local time does not exist or occurs twice.

Persist schedule identity and planned occurrence time. A unique key on schedule ID plus occurrence can prevent two scheduler instances from enqueuing the same occurrence. Decide whether missed runs are skipped, caught up once or replayed individually. Decide whether overlapping runs are allowed. Store UTC instants for execution while retaining the original zone and rule for future calculation.

Do not rely on one application's in-memory cron loop without durable occurrence tracking. A managed scheduler, dedicated job library or workflow engine can provide useful infrastructure, but inspect its actual guarantees and failure behavior.

## Optional Django and Celery integration

The SQL queue above teaches the mechanism. Celery is a Python option for production-oriented task distribution, but it does not remove the need for idempotency and recovery design. Run this optional extension in Linux, a Linux container or WSL with reachable PostgreSQL and Redis; Celery does not officially support native Windows. [Celery platform guidance](https://docs.celeryq.dev/en/stable/getting-started/introduction.html)

Use the capstone project package `config` and app `tasks`. Install `python -m pip install "celery[redis]"` in that environment's activated virtual environment. Start a separate Redis service as described in [caching](../02-data/06-caching-and-redis.md), or provide an existing broker endpoint. Pin the chosen dependency through the project's normal dependency process.

Create `config/celery.py`:

~~~python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
app = Celery("task_service")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
~~~

Add to `config/__init__.py`:

~~~python
from .celery import app as celery_app

__all__ = ("celery_app",)
~~~

Add to settings, which already imports os in the capstone:

~~~python
CELERY_BROKER_URL = os.environ.get(
    "CELERY_BROKER_URL", "redis://127.0.0.1:6379/2"
)
CELERY_TASK_SERIALIZER = "json"
CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_IGNORE_RESULT = True
CELERY_WORKER_PREFETCH_MULTIPLIER = 1
~~~

Create `tasks/tasks.py` with a harmless read-only demonstration:

~~~python
from celery import shared_task
from celery.utils.log import get_task_logger
from .models import Task

logger = get_task_logger(__name__)

@shared_task(ignore_result=True)
def report_task_count(tenant_id):
    count = Task.objects.filter(tenant_id=tenant_id).count()
    logger.info("Task count for tenant %s: %s", tenant_id, count)
~~~

From the project directory in Linux/WSL, start `celery -A config worker --loglevel=INFO --concurrency=2`. In another terminal, run `python manage.py shell`, import `report_task_count` from `tasks.tasks`, then call `report_task_count.delay("11111111-1111-4111-8111-111111111111")`. Expect the count in worker logs; no result backend is configured.

When dispatch depends on newly committed rows, `transaction.on_commit` delays publication until the transaction succeeds. It still has a process-crash window after commit and before publication; use the [outbox](03-event-driven-systems-and-outbox.md) when durable dispatch must survive that gap. [Celery Django integration](https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html)

Celery Beat adds periodic scheduling; run one authoritative scheduler for a schedule set unless the chosen scheduler explicitly coordinates multiple instances. Late acknowledgements and retries require idempotent tasks and careful worker-loss settings; do not enable them blindly for irreversible side effects. [Celery task behavior](https://docs.celeryq.dev/en/stable/userguide/tasks.html)

## Expected outcomes and failure tests

Start two claim sessions and verify only one claims a particular queued row. Let its lease expire, run the reaper, and claim again with a different token. The original token must no longer complete the job. Crash after producing output and verify a retry does not create a second business effect.

Measure queue age, ready jobs, active workers, attempt distribution, execution duration and permanent failures. Queue age is often a better signal of user impact than queue length alone. Fairness matters: one tenant's large export workload should not starve everyone else.

## Exercises and completion checklist

Add job cancellation, a progress endpoint and a tenant-scoped status query. Add a schedule-occurrence uniqueness constraint. Implement a worker shutdown sequence that stops claiming, waits for bounded completion and safely leaves unfinished work recoverable.

Common mistakes include unbounded concurrency, in-memory queues for durable promises, updating job state without checking the lease token, and assuming “succeeded” proves an external effect happened exactly once.

- [ ] Acceptance persists durable work before replying.
- [ ] Claims are atomic and leases have unique ownership tokens.
- [ ] Expired work can be recovered without duplicate effects.
- [ ] Scheduling defines zone, overlap and missed-run behavior.
- [ ] Cancellation, retries, failure review and shutdown are explicit.

Continue with [message queues and streams](02-message-queues-and-streams.md).
