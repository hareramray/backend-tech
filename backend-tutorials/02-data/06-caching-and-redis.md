# Caching and Redis

Prerequisites: [SQL modeling](01-relational-modeling-and-sql.md), [concurrency](02-transactions-and-concurrency.md), the [course map](../README.md), and a disposable Redis instance if running the command lab. Keep Redis private and use credentials appropriate to your environment.

## Objectives

Design a cache contract, implement cache-aside with tenant-safe keys, understand invalidation races and stampedes, and keep a cache outage from becoming a database outage.

## A cache is a consistency decision

Caching trades freshness and complexity for reduced latency or load. Decide the source of truth, acceptable staleness, maximum entry size, key lifetime and failure behavior before writing GET and SET calls. A cache hit can still be wrong because the value belongs to another tenant, reflects outdated permissions, or was created with an incompatible schema.

Cache-aside reads the cache first, loads from the authoritative store on a miss, and then stores the result. Write-through updates through a coordinated layer. Write-behind acknowledges before durable storage catches up and needs a strong loss/recovery design. An HTTP cache, an application cache, a database buffer pool and a CDN solve different parts of the request path.

Use keys that encode all response-shaping context. A typical resource key includes environment, schema version, tenant and resource ID. A user-specific response may also require user or permission context. Do not cache a privileged representation under a key used by ordinary readers.

## Lab: observe expiration and conditional writes

Redis is optional and is not started by the base capstone. With Docker already installed, these PowerShell commands create a separate loopback-only development cache and install the Python binding into your activated virtual environment:

~~~powershell
docker run --detach --name backend-cache-lab --publish 127.0.0.1:6379:6379 redis:7-alpine
python -m pip install redis
docker exec -it backend-cache-lab redis-cli
~~~

The container name must be unused. This disposable local cache has no configured durable volume or production authentication. For a shared deployment, use a secured Redis endpoint and store its connection string in an environment variable.

The following commands run in redis-cli, not in SQL or PowerShell. Replace the environment prefix if sharing a learning instance.

~~~text
SET tutorial:v1:tenant:11111111-1111-4111-8111-111111111111:task:22222222-2222-4222-8222-222222222222 "{\"title\":\"Prepare launch\",\"version\":1}" EX 60
GET tutorial:v1:tenant:11111111-1111-4111-8111-111111111111:task:22222222-2222-4222-8222-222222222222
TTL tutorial:v1:tenant:11111111-1111-4111-8111-111111111111:task:22222222-2222-4222-8222-222222222222
SET tutorial:singleflight:demo owner-a NX PX 5000
SET tutorial:singleflight:demo owner-b NX PX 5000
~~~

The first conditional SET succeeds; the second should not replace the unexpired key. SET can combine conditional creation and expiration in one operation. Using separate SETNX and EXPIRE operations introduces a crash window between them. [Redis SET reference](https://redis.io/docs/latest/commands/set/)

The lock-like example is only a coordination demonstration. A lease expiring does not stop its former owner from running. A distributed correctness lock needs an ownership model and often fencing at the protected resource; do not treat a cache lock as a substitute for a database constraint.

## Django cache configuration

Add this fragment to the capstone's settings after installing redis. Django's built-in RedisCache backend uses redis-py; the standard cache API exposes get, set and delete operations. [Django cache documentation](https://docs.djangoproject.com/en/5.2/topics/cache/)

~~~python
import os

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": os.environ.get("REDIS_URL", "redis://127.0.0.1:6379/1"),
        "TIMEOUT": 60,
        "KEY_PREFIX": "backend-tutorial",
        "OPTIONS": {
            "socket_connect_timeout": 0.2,
            "socket_timeout": 0.2,
        },
    }
}
~~~

Run this smoke check inside `python manage.py shell`:

~~~python
from django.core.cache import cache

key = "v1:tenant:11111111-1111-4111-8111-111111111111:demo"
cache.set(key, {"title": "Prepare launch", "version": 1}, timeout=30)
assert cache.get(key)["version"] == 1
cache.delete(key)
assert cache.get(key) is None
~~~

This verifies connectivity and expiration configuration, not the full cache-aside algorithm. Django cache calls may raise backend errors; implement the chosen fallback policy explicitly. Configure dependency versions in the project's requirements and lock process when adopting this extension.

## Lab: design a bounded cache-aside read

The following is language-neutral pseudocode. Supply a Redis client with a short timeout, a database repository that enforces tenant ownership, request cancellation, serialization limits and a local single-flight implementation before translating it into application code.

~~~text
readTask(authenticatedTenant, taskId, deadline):
  key = versioned key(authenticatedTenant, taskId)
  cached = bounded cache GET; on cache failure treat as miss
  if cached has valid schema:
    return cached

  return local singleFlight(key):
    recheck cache
    row = bounded database query using tenant and taskId
    if row absent:
      optionally cache absence briefly
      return not found
    payload = serialize bounded public representation(row)
    best-effort SET key payload with TTL plus random jitter
    return payload
~~~

Single-flight combines identical concurrent misses inside a process. Across many instances, a short lease can reduce duplicate work, but callers still need a bounded wait or fallback. Add TTL jitter so a whole collection does not expire simultaneously. Negative caching suppresses repeated misses but must expire quickly enough that newly created records become visible according to the contract.

## Invalidation is harder than deleting a key

After a committed database update, deleting the corresponding cache entry is a common starting point. However, this race remains:

1. Reader loads an old database value.
2. Writer commits a new value and deletes the cache.
3. Reader fills the now-empty cache with the old value.

A TTL bounds this stale period only if the configured freshness contract permits it. Stronger approaches include version-aware cache writes, generation keys, change-data-driven invalidation, or avoiding shared caches for correctness-critical reads. Version checks must be atomic where concurrent cache updates occur.

Invalidating a single resource does not invalidate every list, count, search result or permission-filtered representation that contains it. Versioned namespaces can simplify broad invalidation at the cost of extra reads or abandoned entries waiting to expire. Event-driven invalidation adds lag and duplicates, so consumers need idempotency and monitoring.

A lock release must verify ownership atomically; a plain DEL can remove a replacement owner's lease after the original expires. Redis documents token comparison and deletion as part of its distributed-lock discussion. This is a narrow release safety property, not a universal guarantee for all distributed work. [Redis distributed locking guidance](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

## Capacity, eviction and failure behavior

Measure hit rate together with miss cost, memory, evictions, entry size, command latency and backend load. A 99% hit rate can hide a disastrous 1% miss path. Hot keys can concentrate traffic; large values increase network and serialization costs. A cache should have a documented memory limit and eviction policy appropriate to its role.

Do not mix disposable cache entries and irreplaceable job or session state under one undifferentiated eviction assumption. Persistence and replication need separate decisions. A cache-aside database fallback is useful, but a sudden full cache outage can send every request to the database. Apply request limits, bounded concurrency, load shedding and circuit breaking so fallback stays within capacity.

For rate counters, use an atomic operation or script that combines the relevant changes. A GET followed by SET loses increments under concurrency. For sessions, consider logout, account suspension and permission changes; a long TTL can extend unauthorized access if revocation is not consulted.

## Expected outcomes and failure tests

Repeat a resource read and verify database calls decrease. Change the record and observe the chosen invalidation behavior. Expire many keys together to compare jitter and single-flight. Stop Redis and verify requests remain bounded rather than hanging until the API deadline.

Request the same resource ID under another tenant and confirm keys and authorization remain isolated. Send malformed cached JSON and verify it is treated as a miss and repaired safely. Do not log complete sensitive cache values during debugging.

## Exercises and completion checklist

Design caching for a public catalog, a private task feed and an authorization decision. Explain why their acceptable TTLs differ. Add a metric for coalesced requests and demonstrate a bounded miss storm.

Common mistakes include no TTL, cache keys missing tenant scope, unlimited fallback, stale permissions, delete-before-commit invalidation, and using Redis locks to assert exactly-once business effects.

- [ ] Freshness, ownership and failure contracts are explicit.
- [ ] Keys contain all relevant isolation and representation context.
- [ ] Misses, stampedes and cache failure are bounded.
- [ ] Invalidation races are understood and tested.
- [ ] Memory, evictions, latency and backend miss load are monitored.

Connect this work to [resilience and backpressure](../04-async/05-resilience-and-backpressure.md).
