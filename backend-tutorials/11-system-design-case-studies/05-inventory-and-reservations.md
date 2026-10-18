# Case study: inventory reservations without overselling

## Requirements and the invariant

Design a fictional store that offers a short checkout hold on a limited quantity of an item. Buyers reserve units, complete a mock payment, cancel, or let an unused hold expire. The service must not sell more units than exist, even when requests overlap, time out, or retry. This chapter uses Django/PostgreSQL and a simulated payment adapter; it requires no real charges or external payment account. It is an independent design, not an implemented capstone feature.

Read [distributed consistency](../08-architecture/03-distributed-systems-and-consistency.md), [consensus](../08-architecture/06-consensus-raft-and-paxos.md), and [multi-region failover](../08-architecture/11-multi-region-and-failover.md). The inventory invariant for one SKU and location is:

```text
total = available + reserved + sold
available >= 0, reserved >= 0, sold >= 0
```

An inventory adjustment changes `total` through an audited operation. A hold transfers units from available to reserved; confirmation transfers reserved to sold; release transfers reserved to available. A reservation reaches at most one terminal state. Every retry with the same operation key refers to the same logical attempt.

Fictional targets: reservation p99 below 300 ms outside advertised flash-sale overload; 99.9% eligible reservation availability monthly; held inventory expires after two minutes according to database time. Correct allocation takes priority over accepting writes when the inventory authority is unavailable. Browsing can show an approximate stock count; checkout must make an authoritative decision.

## Workload and contention

Assume 500,000 reservation attempts/day, an eightfold normal peak, and 10% of attempts concentrated on one SKU during a promotion.

```text
Average = 500,000 / 86,400 = 5.79 attempts/s
Normal peak = 5.79 * 8 = 46.3 attempts/s
If 70% succeed: 350,000 reservation rows/day
At 500 logical bytes/row: 175 MB/day before indexes and history
At 40 accepted holds/s lasting 120 s: about 4,800 simultaneous holds
```

The last line is a steady-state Little's Law estimate, not the number of available units required for every SKU. A launch can produce 20,000 attempts in two seconds, or 10,000 attempts/s. Most will fail if stock is small; queuing all of them behind one lock wastes connections and adds latency.

If a hot inventory row is held for an average 5 ms per accepted transaction, a simplified serial ceiling is about `1 / 0.005 = 200 transactions/s` for that row before other overhead. Adding application replicas does not remove that serialization. A waiting room, bounded admission queue, or preallocated stock buckets can help, but each changes fairness or allocation semantics.

## Architecture and contracts

```text
Buyer --> edge admission/rate limit --> Django reservation API
                                             |
                                  PostgreSQL inventory + holds
                                             |
                                           outbox
                                             v
                                      payment-mock worker
                                             |
                                  durable mock payment ledger
                                             |
                                     reconcile/confirm worker

Expiry sweeper --> same inventory/reservation transactions
Read cache --> browsing only; never the allocation authority
```

`POST /v1/reservations` accepts SKU, location, quantity, and a tenant-scoped idempotency key. The server chooses the price snapshot and buyer from authenticated state. Return `201` with reservation ID, quantity, state, and expiry; return `409` for insufficient stock or conflicting key reuse. Replaying a completed operation returns its existing resource and current state, not a new hold. Authenticate ownership on `GET /v1/reservations/{id}`.

`POST /v1/reservations/{id}/payment-attempts` creates one mock payment attempt after validating a live hold. Return `202` while the result is pending and expose a status endpoint. A timeout is not proof of failed payment. `DELETE` requests cancellation through the state machine rather than deleting audit history.

```text
inventory(tenant_id, sku, location_id, total, available, reserved, sold, version)
  PRIMARY KEY (tenant_id, sku, location_id)
  CHECK nonnegative counters AND total = available + reserved + sold
reservation(tenant_id, id, buyer_id, sku, location_id, quantity, state,
            expires_at, price_minor, currency, version, created_at)
  CHECK quantity > 0
request_result(tenant_id, buyer_id, operation, idempotency_key,
               request_hash, reservation_id)
  UNIQUE (tenant_id, buyer_id, operation, idempotency_key)
payment_attempt(id, reservation_id, operation_key, state, result_reference)
  UNIQUE (reservation_id), UNIQUE (operation_key)
outbox(event_id, aggregate_id, kind, payload, published_at)
```

Use composite foreign keys for tenant boundaries. Database check and uniqueness constraints supplement application validation. They cannot by themselves assert an arbitrary sum across reservation rows; maintain counter transfers in transactions and audit their agreement with reservation state. [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html).

## Allocation and expiry flows

1. Authenticate, validate a positive bounded quantity, and claim the idempotency key atomically. If an existing key has a different canonical request hash, reject it.
2. In one short transaction, decrement available and increment reserved only where available is at least the requested quantity. Require exactly one returned row.
3. Insert the hold with an expiry based on database time, plus an outbox event and operation result. Roll back the counter update if any dependent insert fails.
4. Commit, then acknowledge. A response lost after commit is recovered through the operation key.
5. The sweeper locks candidate holds, rechecks state and expiry, moves only eligible `HELD` reservations to `EXPIRED`, and returns their units exactly once in the same transaction.

The allocation statement is a parameterized sketch, with values bound by the database driver:

```sql
UPDATE inventory
SET available = available - %(quantity)s,
    reserved = reserved + %(quantity)s,
    version = version + 1
WHERE tenant_id = %(tenant_id)s AND sku = %(sku)s
  AND location_id = %(location_id)s AND available >= %(quantity)s
RETURNING available, reserved, version;
```

Under PostgreSQL Read Committed, a competing updater can wait and then re-evaluate its condition against the updated row. This supports the guarded decrement; an earlier unlocked stock read followed by an unconditional write does not. [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html).

For operations involving existing holds, lock the reservation before its inventory row consistently across confirmation, cancellation, and expiry. For multi-item orders, acquire resources in a deterministic order or retry deadlocks. Use small batches; `SKIP LOCKED` can distribute sweeper work, but a skipped hold must be revisited. Django's `select_for_update()` must run within a transaction on PostgreSQL. [Django row-locking API](https://docs.djangoproject.com/en/5.2/ref/models/querysets/#select-for-update).

Database `now()` is the transaction-start timestamp. For a decision made after potentially waiting for a lock, obtain database `clock_timestamp()` after the lock and compare expiry against that value; otherwise an old transaction timestamp could accept an already expired hold. Application wall clocks do not decide this invariant. [PostgreSQL date/time functions](https://www.postgresql.org/docs/current/functions-datetime.html).

## Mock payment state machine and ambiguous outcomes

| Reservation state | Allowed transition | Inventory effect |
|---|---|---|
| HELD | PAYMENT_PENDING before expiry | None; units stay reserved |
| HELD | EXPIRED or CANCELLED | Reserved decreases; available increases |
| PAYMENT_PENDING | CONFIRMED after definitive mock success | Reserved decreases; sold increases |
| PAYMENT_PENDING | CANCELLED after definitive failure/cancellation | Reserved decreases; available increases |
| Terminal state | Duplicate delivery of same result | No additional effect |

Starting payment locks the hold, compares expiry using current database time, and commits `PAYMENT_PENDING`, the payment operation key, and an outbox event together. It then releases all database locks. The worker calls a local mock adapter outside the transaction. The mock ledger stores a stable result by operation key and can simulate success, rejection, or timeout after recording success.

If the mock reports success, a transaction locks the pending reservation, checks the attempt identity, transfers reserved units to sold, and records confirmation plus an outbox event. If that transaction fails, reconciliation queries the mock ledger using the original operation key and retries finalization. Duplicate callbacks do not transfer units twice. Treat a mismatched result reference as an incident rather than applying it to a different attempt.

An ordinary hold TTL does **not** release `PAYMENT_PENDING` inventory. Otherwise a delayed successful result could arrive after those units were resold. Give pending payments a resolution deadline for retries and alerts, not an automatic stock-release deadline. Release only after a definitive failure or confirmed cancellation; unresolved cases remain reserved and require reconciliation. This can reduce availability of stock, but preserves the invariant under ambiguity.

If the product requires forced release after a maximum time, it also needs an explicit late-success compensation policy and a mock refund/cancel operation with its own idempotency. The user must not receive both a confirmed reservation and returned stock. No database transaction can atomically commit with an arbitrary remote payment service; the state machine handles that boundary.

## Scaling, safety, and recovery

Keep one write authority per SKU/location allocation pool. Sending the same pool to two independently writable regions without coordination can oversell. Geographic stock buckets can trade utilization for local independence if their total allocation never exceeds real stock. Failover must fence the old writer and respect acknowledged reservation durability before opening the new writer.

Enforce per-buyer quotas and short holds to reduce inventory hoarding. Scope operation keys and status reads to the authenticated tenant/buyer, keep price snapshots immutable, and avoid logging personal details or payment-like secrets. This lab stores only mock payment references. Limit retries and admission queues so exhausted stock does not overload the database.

Observe successful allocations, out-of-stock responses, lock waits, contention by SKU, expiration delay, pending-payment age, idempotency replays, and reconciliation discrepancies. Do not classify every legitimate out-of-stock response as an infrastructure failure. Regularly compare inventory counters with active and confirmed reservation totals using a consistent snapshot.

Recovery restores inventory, reservations, operation results, outbox, and payment-attempt records together. Replay mock results and unresolved outbox events after restoration. A backup that predates acknowledged confirmations requires reconciliation before selling restored stock. Audit trails explain which transaction moved each unit; a cache rebuild cannot establish that history.

## Local lab: two buyers, one remaining unit

Use an existing disposable PostgreSQL database and two SQL sessions. This reduced lab tests only atomic allocation; it is not a complete checkout implementation. Run setup once:

```sql
CREATE TABLE tutorial_inventory_race (
    sku text PRIMARY KEY,
    available integer NOT NULL CHECK (available >= 0)
);
INSERT INTO tutorial_inventory_race VALUES ('one-seat', 1);
```

In session A, start the transaction and leave it open after the update:

```sql
BEGIN;
UPDATE tutorial_inventory_race
SET available = available - 1
WHERE sku = 'one-seat' AND available >= 1
RETURNING available;
```

In session B, run the same `BEGIN` and `UPDATE`. B waits. Commit A, then observe B return zero rows; commit B. A received `0` as the remaining quantity, B did not allocate anything, and final stock is zero. Repeat from a fresh one-unit fixture but roll A back; B should then allocate the released opportunity. Set a bounded lock timeout if desired so a forgotten open transaction does not leave the lab waiting indefinitely.

## Tabletop lab: payment and expiry races

Start with total 1, available 0, reserved 1, sold 0, and hold H expiring at database time 120. Independently execute each schedule and record all four counters.

| Schedule | Expected outcome |
|---|---|
| Payment locks H at 119; expiry waits | Payment enters pending; sweeper sees pending and releases nothing |
| Expiry locks H at 121; payment waits | H expires; payment observes terminal state and cannot start |
| Transaction begins at 110, obtains hold lock at 121 | Current database time rejects starting payment despite old transaction timestamp |
| Mock records success, response times out | H remains pending; reconcile by operation key, then confirm once |
| Success callback delivered three times | Exactly one reserved-to-sold transfer |
| Worker crashes after commit but before outbox acknowledgment | Replay is safe; one logical reservation/confirmation |
| Pending payment cannot be resolved at its deadline | Alert and retain reserved stock; no silent release/resale |

## Exercises and completion checklist

Extend the model to a cart with two SKUs and define all-or-nothing holds. Then model assigned seats: use a unique seat allocation authority rather than relying only on aggregate counters. Explain how a seat change differs from increasing a quantity.

- [ ] Counter invariants and terminal transitions are enforced transactionally.
- [ ] Concurrent allocation and lost-response retries have different defenses.
- [ ] Expiry uses database time after lock acquisition.
- [ ] Ambiguous payment results retain inventory until resolved.
- [ ] Failover, restoration, and reconciliation preserve acknowledged allocations.

Return to the [case-study index](README.md) or compare this coordination requirement with the [news feed](03-news-feed.md).
