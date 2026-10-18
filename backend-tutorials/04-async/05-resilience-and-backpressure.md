# Resilience and backpressure

Prerequisites: [jobs](01-jobs-and-scheduling.md), [idempotency](04-idempotency-and-deduplication.md), the [course map](../README.md), and Python 3.12+ for the standalone lab.

## Objectives

Bound work and waiting, budget retries across a request path, protect dependencies from overload, and test degradation as a measurable behavior.

## Failure handling can amplify failure

A slow dependency occupies connections and workers longer. Queues grow, memory rises, callers time out and retry, and the extra traffic makes recovery harder. Adding workers or retries without understanding the bottleneck can intensify the overload.

Backpressure makes upstream producers respond to downstream capacity. A bounded queue can block briefly or reject new work. A worker can limit concurrent calls. A server can shed optional work or return an explicit retryable overload response. An unbounded queue only postpones the decision while increasing latency and memory.

Use Little's Law as an intuition for stable workloads: average in-flight work equals throughput multiplied by average time in the system. If throughput stays fixed while latency grows tenfold, the required in-flight capacity grows too. This relationship is not a license to size a system from averages alone; bursts and tail latency matter.

## Time budgets and retries

Set an overall operation deadline, then allocate shorter budgets for connection acquisition, connection establishment, requests and retries. A timeout on only one socket phase may leave other phases unbounded. Propagate cancellation where clients and protocols support it, and release resources reliably.

Retry transient failures only when the operation is safe to repeat or protected by idempotency. Bound attempts and total elapsed time. Use exponential backoff with random jitter to avoid synchronized retry waves. Avoid retries independently multiplying at every layer; three attempts at three layers can create many downstream attempts. [AWS guidance on timeouts, retries and jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)

Honor a valid Retry-After response when it fits the remaining deadline, and cap any server-provided delay. Treat validation, permission and unsupported-operation errors as permanent unless the API contract says otherwise.

## Lab: bounded Python concurrency and retries

Save this standard-library-only code as `resilience_demo.py` and run `python resilience_demo.py`. It simulates transient failures and measures maximum concurrent operations:

~~~python
import asyncio
import random

active = 0
peak = 0
slots = asyncio.Semaphore(3)

async def dependency(item: int, attempt: int) -> str:
    global active, peak
    active += 1
    peak = max(peak, active)
    try:
        await asyncio.sleep(0.03)
        if item % 4 == 0 and attempt == 0:
            raise TimeoutError("simulated transient failure")
        return f"ok:{item}"
    finally:
        active -= 1

async def call(item: int) -> str:
    async with asyncio.timeout(1.0):
        for attempt in range(3):
            try:
                async with slots:
                    return await dependency(item, attempt)
            except TimeoutError:
                if attempt == 2:
                    raise
                delay = random.uniform(0, min(0.2, 0.02 * (2 ** attempt)))
                await asyncio.sleep(delay)
    raise RuntimeError("unreachable")

async def main() -> None:
    results = await asyncio.gather(*(call(i) for i in range(12)))
    print(results)
    print("peak dependency concurrency:", peak)
    assert peak <= 3

asyncio.run(main())
~~~

The semaphore covers dependency work, not the backoff sleep, so waiting retries do not reserve scarce execution slots. The outer timeout also bounds waiting for a slot. Python's asyncio timeout context cancels overdue work and presents a timeout outside the context; actual blocking libraries must support cooperative cancellation or their own bounded operations. [Python asyncio task documentation](https://docs.python.org/3/library/asyncio-task.html)

This example creates only twelve tasks. For an unbounded input stream, creating millions of tasks before they reach a semaphore still uses unbounded memory. Use an asyncio.Queue with maxsize and a fixed worker count, or a durable broker with bounded consumer prefetch.

## Circuit breakers and bulkheads

A circuit breaker stops repeatedly calling a failing dependency after a threshold. It opens for a period, then permits limited probes before closing. Define the observation window, minimum sample size, counted failure types and recovery probes. A breaker is a state machine, not a replacement for timeouts.

A bulkhead isolates capacity: separate pools for interactive reads and long exports, or separate worker queues for distinct dependencies. Without isolation, a failing optional report service can consume every database connection needed for sign-in.

Rate limiting controls arrival rate; concurrency limiting controls active work. Both can be necessary. Token buckets allow bounded bursts; fixed windows can create boundary spikes; sliding windows offer different cost and precision tradeoffs. A per-user limiter without a global limit can still overload the whole system.

For multi-tenant services, add fairness: per-tenant quotas, weighted scheduling or capped concurrency. Otherwise a single tenant can consume all queue slots. Quota failures should provide a clear response and a useful retry policy.

## Fallback and graceful degradation

A fallback must remain truthful and safe. Serving a slightly stale public catalog may be acceptable; returning cached authorization after revocation may not be. Skipping optional recommendations is different from claiming an unperformed write succeeded.

If the cache fails and every request falls back to PostgreSQL, bound database concurrency and reject excess load. If an upstream API fails, a circuit breaker should not redirect all traffic to an equally constrained fallback without capacity planning.

Maintain a retry budget such as a limited fraction of successful request volume, or a bounded number of retries per process and interval. This constrains amplification during widespread failure better than a per-request attempt count alone.

## Failure tests and measurement

Increase the dependency delay above the deadline and verify calls terminate with no leaked active count. Make every call fail and measure total attempts. Feed work faster than the consumer capacity and verify queue size remains bounded and the rejection policy is visible.

Monitor p50/p95/p99 latency, active work, queue age, rejected requests, timeouts, retries, breaker state and dependency saturation. Separate time spent waiting for a pool from time executing the downstream call.

Common mistakes include catching every exception as retryable, no total deadline, sleeping while holding a scarce resource, unlimited task creation, breakers that never probe recovery, and success-shaped fallback responses that conceal lost writes.

## Exercises and completion checklist

Replace the fixed task list with a bounded queue and three workers. Add a retry budget and demonstrate that a full outage cannot exceed it. Design different overload behavior for interactive task reads and nightly exports.

- [ ] Every external operation has a bounded deadline.
- [ ] Retries are safe, limited and jittered.
- [ ] Queue size and concurrency are bounded independently.
- [ ] Critical workloads have protected capacity.
- [ ] Degradation behavior preserves truthful outcomes.
- [ ] Tests cover slowdowns, total failure, overload and recovery.
