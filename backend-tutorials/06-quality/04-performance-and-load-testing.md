# Performance engineering and load testing

[Tutorial index](../README.md) · Related: [debugging and profiling](03-debugging-and-profiling.md)

## Outcome and prerequisites

Design a repeatable experiment that shows how the service behaves as demand increases. You need the Django capstone running on port 3000, synthetic task data, and a separately installed k6 CLI for the runnable load example. k6 is an optional load tool; its script language is JavaScript even though the application is Python. Run experiments only against an environment you control and have sized for them.

Performance is a workload-specific property. “Supports 1,000 requests per second” is incomplete without the route mix, payload distribution, data volume, hardware, cache state, latency objective, and error budget. A service that returns errors quickly can appear fast in a poorly designed benchmark.

## Establish the measurement model

Measure throughput, concurrency, latency distribution, errors, and saturation together. Throughput counts completed work per unit time. Concurrency is the number of operations in flight. Latency is elapsed time from a defined start to a defined end. Saturation describes how close a finite resource is to its practical limit, including CPU quota, database connections, queue capacity, or disk throughput.

Percentiles describe a population: p95 of 200 ms means approximately 95% of measured requests finished within that duration. Averages hide long tails. Do not average percentiles from multiple instances; aggregate distributions or histogram buckets. Ensure the sample count is large enough for the tail percentile you care about.

Little's Law relates average in-flight work, arrival rate, and average time in a stable system: `L = lambda × W`. At 100 requests/second and 0.2 seconds average residence time, expect roughly 20 concurrent requests. This is a reasoning aid, not a capacity guarantee; it does not apply cleanly to an indefinitely growing queue.

Choose an arrival model. A closed model has workers start new work after their previous work completes, so offered load can fall when the service slows. An open arrival model schedules starts independently of completion, making overload visible but requiring enough generator capacity. [k6 open and closed models](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/)

## Walkthrough: establish a modest local baseline

1. Record commit, runtime version, CPU/memory allocation, database row count, and whether the cache is enabled. Load a fixed synthetic dataset and keep the generator on a separate host for serious capacity measurements. For a laptop smoke exercise, note that generator and application compete for resources.

2. Save the following as `load-readiness.js` in a scratch directory. It is intentionally a low-rate readiness experiment. Readiness includes dependency checks in this curriculum but still does not represent the task-list workload.

```js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    baseline: {
      executor: 'constant-arrival-rate',
      rate: 5,
      timeUnit: '1s',
      duration: '30s',
      preAllocatedVUs: 5,
      maxVUs: 10,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
    checks: ['rate>0.99'],
    dropped_iterations: ['count==0'],
  },
};

export default function () {
  const base = __ENV.BASE_URL || 'http://127.0.0.1:3000';
  const response = http.get(`${base}/health/ready/`, {
    timeout: '2s',
    tags: { name: 'GET /health/ready/' },
  });
  check(response, { 'ready status is 200': r => r.status === 200 });
}
```

3. With k6 installed, run `k6 run load-readiness.js`. Expect roughly 150 scheduled iterations, successful status checks, and no dropped iterations on a healthy local setup. Actual timing and counts can vary at startup or shutdown. The thresholds are teaching values, not promised capstone performance.

4. Repeat with an invalid local port through `BASE_URL` and confirm the run fails. In PowerShell, set `$env:BASE_URL = 'http://127.0.0.1:3999'`, rerun, then remove that variable with `Remove-Item Env:BASE_URL`. A threshold makes performance expectations affect the run's exit status; individual checks alone do not necessarily fail the process. [k6 thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)

5. Adapt the script to `GET /v1/tasks/` using the capstone's documented local demo-tenant setup or production authentication once implemented. Add checks for the response's meaningful fields. Preserve a stable route label instead of labeling each unique task ID. For writes, use a disposable database, unique input IDs, and explicit cleanup or a fresh environment per run.

## Expand the experiment deliberately

A smoke test validates the script at low traffic. A load test measures expected demand. A stress test increases demand toward failure to discover the limiting resource. A spike test studies abrupt changes and recovery. A soak test runs long enough to reveal leaks, storage growth, token expiry, and accumulated retries. Each has a different question and should have a defined stopping condition.

Warm caches separately from cold-start tests. Model hot tenants and uneven key popularity instead of choosing every task uniformly. Include realistic pagination, payload size, read/write proportions, background jobs, and connection reuse. An all-cache-hit benchmark cannot estimate database demand after cache loss.

Observe queue length, active database connections, database locks, busy application workers, container throttling, memory, and downstream latency during the run. If throughput flattens while latency and queue depth rise, more callers are accumulating behind a bottleneck. Increasing every worker count and pool size can move the bottleneck into the database and make recovery worse. Django's development server is useful for this script smoke test; use the intended production server and worker model for capacity conclusions.

Tune one constraint at a time. Candidate changes include better indexes, bounded pagination, reduced serialization, caching, batching, bounded concurrency, backpressure, and avoiding duplicate work. Measure correctness and resource cost after each change. A cache may improve latency while introducing staleness; document that tradeoff.

## Failure tests and exercises

In an isolated environment, test a slow dependency, a full database connection pool, and a cold cache. Expect bounded request deadlines, controlled error responses, and recovery after the fault ends. Verify that retries do not multiply traffic without limit. Stop the experiment if agreed resource or error thresholds are exceeded.

Common mistakes include testing production accidentally, accepting HTTP 500 as a successful fast response, reporting only average latency, ignoring dropped generator iterations, and comparing runs with different data sizes. Another is claiming linear scaling from a tiny sample with no saturation evidence.

Produce a performance report containing the objective, workload, environment, dataset, duration, repeat count, results, bottleneck evidence, and uncertainty. Exercise: compare offset pagination with keyset pagination on a large synthetic tenant, then repeat after changing the sort index. Explain where the improvement appears and whether the API's pagination guarantees changed.
