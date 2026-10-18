# Debugging and profiling Python/Django services

[Tutorial index](../README.md) · Related: [observability](../07-operations/05-observability-and-slos.md)

## Outcome and prerequisites

Turn an ambiguous symptom into a reproducible explanation and collect the right evidence before changing code. You need the Django capstone, its activated Python virtual environment, and basic knowledge of exceptions and SQL. Diagnostic commands run locally with the capstone environment settings. They are not instructions to attach a debugger to production or expose diagnostic endpoints publicly.

Debugging asks why behavior is incorrect. Profiling asks where resources or elapsed time are spent. A slow request can be correct but expensive; an extremely fast response can be incorrect. Define the symptom precisely: a particular input returns the wrong status, p95 latency increases after a release, or memory keeps rising after repeated identical work.

## Build a falsifiable hypothesis

Capture the source revision, configuration revision, request shape, timestamp, request or trace ID, and expected result. Redact cookies, authentication headers, personal content, database credentials, and secrets. Compare a failing case with a working case that differs in one meaningful dimension, such as tenant size or pagination depth.

Write a hypothesis that predicts evidence. “The database is slow” is too broad. “Listing tasks performs one extra tenant query per row” predicts a query count that increases with the number of returned tasks. “The sort index is unsuitable” predicts a particular query plan and data-volume sensitivity. Seek evidence that could disprove the hypothesis.

Reduce the reproduction while preserving the failure condition. A small fixture is useful unless the issue depends on concurrency, millions of rows, Unicode, or a cold cache. Preserve those dimensions. Once the cause is understood, add a regression test that fails for the original reason.

## Walkthrough: reproduce without the development reloader

1. Run the capstone locally with `python manage.py runserver 127.0.0.1:3000 --noreload`. The reloader can create extra processes and repeated initialization that complicate an experiment. This is a local debugging setup; Django's development server is not the production server.

2. In a second PowerShell terminal, measure a connectivity baseline:

```powershell
curl.exe --silent --show-error --output NUL --write-out "status=%{http_code} total=%{time_total}s connect=%{time_connect}s\n" http://127.0.0.1:3000/health/live/
```

Expect a successful status while the process runs. The health route does not represent database-heavy behavior. Measure `/v1/tasks/` with a representative synthetic dataset and the capstone's local demo setup next.

3. Follow the request through middleware, view validation, business behavior, ORM evaluation, serialization, and response. Use `time.perf_counter()` for elapsed durations. A queryset is lazy: constructing it and evaluating it are different events. Timing only `Task.objects.filter(...)` without iteration may omit the actual query.

4. Use a local debugger breakpoint at the failing decision. Inspect the smallest relevant values; do not dump the full request or settings object into logs. Never enable Django `DEBUG` in a publicly accessible production environment because diagnostic output can expose sensitive details. [Django deployment checklist](https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/)

5. Repeat the same case several times, separating warm and cold measurements. Startup, disk cache, connection establishment, and instrumentation overhead can distort a single sample.

## Profile a bounded workload

A bounded workload can be a better profiling target than an indefinitely running server. Save this complete `profile_read.py` beside `manage.py`, with the capstone's local database settings already configured. It reads synthetic fixtures and performs no writes:

```python
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
import django
django.setup()

from tasks.models import Task

for _ in range(100):
    list(Task.objects.filter(
        tenant_id="11111111-1111-4111-8111-111111111111"
    ).order_by("-created_at", "-id").values(
        "id", "title", "status", "version"
    )[:50])
```

Run it through Python's deterministic profiler:

```powershell
python -m cProfile -o task-profile.prof profile_read.py
python -m pstats task-profile.prof
```

In the interactive `pstats` session, use `sort cumulative`, then `stats 30`, then `quit`. Startup may dominate a tiny dataset, so distinguish initialization from repeated work. This script measures the query workload without HTTP middleware or serialization; profile those separately when they are implicated. A main-thread profile does not automatically cover independently running worker threads or processes. Python documents `cProfile` and the difference between total and cumulative function time. [Python profilers](https://docs.python.org/3.12/library/profile.html)

High elapsed time with low CPU use suggests waiting on SQL, locks, external calls, or worker queues. High CPU may point to serialization, expensive transforms, regex behavior, or compression. Increasing worker counts can improve some workloads but also multiplies memory and database connections. With standard CPython, threads are useful for many waiting workloads but are not a universal way to parallelize Python CPU work; measure the actual runtime and worker model.

## Query diagnosis and N+1 behavior

Use Django's `assertNumQueries` around a representative operation to prevent accidental query-count growth. A test that lists one task may miss an N+1 problem; use several tasks with distinct related objects. `select_related()` can join appropriate single-valued relations, while `prefetch_related()` performs separate queries and assembles relationships. Both have memory and query-shape tradeoffs. [Django database optimization](https://docs.djangoproject.com/en/5.2/topics/db/optimization/)

For a read-only query on synthetic data, inspect the actual plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, status, version
FROM tasks
WHERE tenant_id = '11111111-1111-4111-8111-111111111111'
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

`ANALYZE` executes the query. Do not casually apply it to a data-changing statement. Compare estimated and actual rows, scan type, loops, sorts, and buffer activity. A sequential scan can be correct for a small table. [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/17/using-explain.html)

## Memory investigation

Track process resident memory, request concurrency, worker count, result size, and cache size together. A large queryset converted to a list can retain many model objects. A global cache without bounds can grow indefinitely. Multiple workers may each hold their own copy of an expensive cache.

The following is a diagnostic fragment, not a complete application module. Place it around a bounded local workload that you supply:

```python
import tracemalloc

tracemalloc.start(10)
before = tracemalloc.take_snapshot()
# Run the same bounded application workload repeatedly here.
after = tracemalloc.take_snapshot()
for difference in after.compare_to(before, "lineno")[:10]:
    print(difference)
tracemalloc.stop()
```

It compares traced Python allocation sites. It does not account for every native allocation or prove a leak merely because some allocations remain. Treat diagnostic artifacts as sensitive and measure their overhead. [Python tracemalloc](https://docs.python.org/3.12/library/tracemalloc.html)

## Verification and practice

After a fix, rerun the original reproduction, regression tests, and the same measured workload. Report absolute values, sample count, and environment. Confirm the improvement did not increase memory, error rate, or downstream load unacceptably.

Common mistakes are exposing a debugger, using `DEBUG=True` in production, timing an unevaluated queryset, adding indexes without inspecting plans, and changing several variables at once. Increasing timeouts may hide symptoms while queues continue growing.

Practice reproducing an N+1 query, adding a query-count regression test, applying the appropriate ORM loading strategy, and measuring again. Then compare bounded iteration with eagerly loading a large result set. Keep a diagnostic record containing symptom, hypothesis, experiment, result, fix, and regression evidence.
