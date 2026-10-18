# Abuse prevention, rate limiting, and audit trails

## Purpose and prerequisites

An authenticated request can still be abusive. A user may scrape every record they can access, trigger expensive exports repeatedly, stuff stolen passwords into login, or monopolize workers with oversized jobs. Availability controls and business quotas therefore complement authentication and authorization.

Prerequisites: [web security](05-web-security-and-input-attacks.md), basic Redis concepts, and the [task-service capstone](../09-capstone/01-task-service.md). The Lua example is a small lab limiter, not a full fraud-detection service. Run it against a disposable Redis instance through your existing client or `redis-cli`.

## Pick a key and a budget

An IP limit alone penalizes shared networks and can be bypassed through distributed clients. A user limit alone does not protect login before identity is established. A tenant limit supports fairness between customers but does not stop one member from consuming the entire tenant allowance.

Combine controls at several scopes: coarse network protection, credential-endpoint limits, authenticated principal and tenant quotas, operation-specific cost budgets, and global concurrency ceilings. Trust forwarding headers only from explicitly trusted proxies; otherwise the caller can spoof its apparent address.

Count the resource you actually protect. Ten tiny reads and ten million-row exports are not equivalent. An export may consume a concurrency slot plus a row or byte quota. A password verification attempt consumes expensive hashing work even when authentication fails. Set body and response limits before debating requests per second.

## Algorithms and tradeoffs

| Algorithm | Behavior | Main tradeoff |
|---|---|---|
| Fixed window | Counts within discrete intervals | Boundary permits a concentrated burst |
| Sliding log | Stores recent request timestamps | Precise but memory grows with activity |
| Sliding counter | Approximates overlapping windows | Lower memory, approximate admission |
| Token bucket | Refills tokens, allows bounded burst | Needs atomic time/refill/update logic |
| Leaky bucket or paced queue | Smooths departure rate | Adds delay and requires bounded queueing |

Distributed checks must be atomic. Reading a counter, deciding, then writing it in separate commands lets concurrent callers exceed the budget. Redis scripts execute atomically on the server, but a long script blocks other work, so keep limiter operations bounded. [Redis scripting documentation](https://redis.io/docs/latest/develop/programmability/eval-intro/).

For clustered Redis, keys touched by one script must satisfy the cluster's key-slot rules. A global all-tenant limiter can become a hotspot; decide whether approximate local admission plus a global budget is sufficient for the protected operation.

## Lab: an atomic expiring counter

Save this as `limit.lua` in your disposable lab directory:

```lua
-- KEYS[1]: server-derived quota key
-- ARGV[1]: positive maximum count
-- ARGV[2]: positive window duration in milliseconds
local maximum = tonumber(ARGV[1])
local duration = tonumber(ARGV[2])
if not maximum or not duration or maximum < 1 or duration < 1 then
  return redis.error_reply('invalid limiter configuration')
end

local count = redis.call('INCR', KEYS[1])
local ttl = redis.call('PTTL', KEYS[1])
if count == 1 or ttl < 0 then
  redis.call('PEXPIRE', KEYS[1], duration)
  ttl = duration
end
local allowed = 0
if count <= maximum then allowed = 1 end
return {allowed, count, ttl}
```

This is a fixed-duration window starting with the first counted request. It counts rejected requests too and does not extend the expiration on every rejection. Configure the namespace and values on the server, never from an untrusted client's proposed allowance.

1. Execute the script with key `tutorial:limit:tenant-a:export`, maximum `3`, and duration `10000` using your Redis client. With `redis-cli`, the command form is `redis-cli --eval limit.lua tutorial:limit:tenant-a:export , 3 10000`.
2. Call it four times within ten seconds. The first array item should be `1`, `1`, `1`, then `0`. The second item should increase from one through four.
3. Wait for expiration and call again. Expect permission and a fresh count of one.
4. Send parallel calls. Exactly three may be allowed within a fresh window, regardless of request concurrency.
5. Use a separate tenant key. Its budget should be independent.
6. Stop the disposable Redis instance and verify the application's documented dependency-failure behavior.

Map a quota rejection to a controlled `429` response with a useful retry delay, using the returned TTL rounded appropriately. Do not automatically retry a non-idempotent operation. Add a concurrency limit because a rate limit can still admit enough simultaneous work to overload a dependency.

## Failure behavior and fairness

Fail-open versus fail-closed is a business decision per operation. An unavailable limiter might permit inexpensive authenticated reads using a small local emergency budget while denying expensive exports. Authentication and authorization themselves must not become optional because a limiter failed. Record degraded mode and alert on it.

Bound queues, request sizes, running jobs, retry counts, and per-tenant resource use. Shed work before memory exhaustion. Use backpressure so upstream producers learn that capacity is exhausted. CAPTCHA or additional verification can raise abuse cost but introduces accessibility and usability costs; it should not be the only protection.

## Audit events are a separate data product

Operational logs help debug performance and errors. Audit events answer who did what, to which resource, under what authority, and with what result. Capture important membership changes, credential changes, data exports, administrator actions, and permission denials without storing raw secrets.

Example event shape:

```json
{
  "event_id": "generated-unique-id",
  "occurred_at": "2026-10-18T10:00:00Z",
  "actor_id": "internal-user-id",
  "tenant_id": "11111111-1111-4111-8111-111111111111",
  "action": "task.updated",
  "resource_id": "22222222-2222-4222-8222-222222222222",
  "outcome": "allowed",
  "request_id": "correlation-id",
  "changed_fields": ["status"]
}
```

Use structured logging, control untrusted field lengths, and restrict read/write access to audit storage. Redact credentials and unnecessary personal data; plan retention, monitoring, and protection against tampering. [OWASP logging guidance](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html).

For an action that must never succeed without an audit record, persist the audit intent in the same database transaction, often via an outbox, then deliver it reliably. Merely calling a remote logger after committing leaves a crash gap. Hash chaining can make certain changes detectable, but a compromised service that controls both records and anchors can still rewrite history; independent protected storage matters.

## Detection and response lab

Generate fixture events for repeated login failures, a sudden export spike, and a privileged membership change. Write detection rules with a time window and owner. Confirm one harmless burst does not create an endless alert storm. Follow an event from request ID through the database mutation and worker activity without needing the raw session token.

Exercise: define what evidence an investigator needs after a suspected cross-tenant read. Decide which metadata is necessary, how long it remains useful, and who may inspect it. Then design a tenant-fair queue for exports.

## Completion checklist

- [ ] Limits cover identities, tenants, expensive operations, and concurrency.
- [ ] Counter admission is atomic and keys are server-derived.
- [ ] Dependency failure and retry behavior are documented.
- [ ] Audit records describe authority, action, resource, and outcome.
- [ ] Sensitive values are excluded from logs and traces.
- [ ] Detection rules have tested response procedures.

Continue with [distributed systems](../08-architecture/03-distributed-systems-and-consistency.md) and [CQRS, events, and sagas](../08-architecture/04-cqrs-event-sourcing-and-sagas.md).
