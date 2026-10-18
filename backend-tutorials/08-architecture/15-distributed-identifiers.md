# Distributed identifiers, ordering, and clock failures

[Curriculum](../README.md) · [System-design map](README.md) · Prerequisites: database constraints, [logical clocks](12-logical-clocks-crdts-and-reconciliation.md), [sharding](07-sharding-and-consistent-hashing.md)

## Start with the guarantees you need

An identifier can provide uniqueness, compactness, approximate time order, locality, opaque public representation, or offline generation. These properties are separate. A globally unique key does not prove authorization, hide creation time, guarantee gap-free numbering, or define the order in which transactions committed.

Write a contract before selecting a format:

- Scope: unique per table, tenant, region, or globally?
- Lifetime: can old data be imported or an old key ever be reused?
- Generation: must a client create an ID while disconnected?
- Ordering: approximate creation time, per-entity sequence, or committed global order?
- Security: may the ID reveal a timestamp, region, or business volume?
- Storage: what are the index, wire-format, and human-entry costs?

Use a database uniqueness constraint even if collision probability is tiny. That constraint converts an unexpected collision into a detectable event instead of silent data corruption. Make the retry policy explicit and distinguish a generated-ID collision from a client retry of an already completed operation.

## Compare common approaches

| Approach | Coordination | Useful property | Main limitation |
| --- | --- | --- | --- |
| Database sequence/identity | One database authority | Compact indexed key; simple generation | Allocation gaps; central dependency; no automatic cross-shard uniqueness |
| Allocated number blocks | Authority grants nonoverlapping ranges | Many local allocations between authority calls | Unused gaps; durable range ownership required |
| Random UUIDv4 | Suitable independent randomness | Decentralized generation; opaque-looking key | Larger key; random index locality; probabilistic uniqueness |
| UUIDv7 | Timestamp plus implementation-defined uniqueness strategy | Standardized time-oriented UUID layout | Clock handling and same-millisecond ordering vary by implementation |
| Worker/time/sequence integer | Unique worker assignment and clock policy | Compact, roughly time-sortable key | Worker reuse, restart, clock rollback, sequence overflow |
| Per-stream sequence | Stream/shard owner | Exact order within that stream | No total order across independent streams |

UUIDv7 places a 48-bit Unix-millisecond timestamp ahead of the remaining uniqueness fields; the standard permits random bits and defined monotonicity strategies. It does not promise one universal cross-host generation order. UUIDv4 uses 122 random bits after its version/variant fields. [UUID specification, RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html).

The Python 3.12/3.13 baseline can use `uuid.uuid4()`. Standard-library UUIDv7 generation was added in Python 3.14; do not paste `uuid.uuid7()` into an earlier-runtime tutorial without a compatible implementation and dependency decision. [Python UUID documentation](https://docs.python.org/3.14/library/uuid.html).

## Collision estimates and index consequences

For approximately uniform independent random choices from a space of size `N`, the small-probability birthday approximation is `p ~= n(n-1)/(2N)`. With `n = 10^12` UUIDv4 values and `N = 2^122`, this is roughly `9.4 * 10^-14`. The estimate assumes an appropriate generator and independence; a cloned random state or broken implementation can invalidate the assumption completely.

The same formula makes a short public alias very different from a UUID. An eight-character base-62 code has `62^8 ~= 2.18 * 10^14` possibilities. At 100 million generated aliases, collisions somewhere in the history are expected even though each individual draw has a low collision probability. Generate, attempt an atomic unique insertion, and retry on that specific conflict. Do not infer safety from a visually large alphabet.

Random keys spread B-tree insertion locations and may reduce cache locality. Increasing keys tend to concentrate insertion near the newest range, which improves locality but can create a hotspot in some distributed partition schemes. The actual outcome depends on engine, key layout, workload, and sharding strategy; benchmark the chosen system instead of declaring one ID universally faster.

Use the database's native UUID/binary/integer type rather than an unnecessarily expanded text representation when practical. Public JSON representations must preserve integers exactly across clients; JavaScript consumers cannot safely represent every 64-bit ID as a number. Encode large integer IDs as strings.

## Worker/time/sequence allocation

Consider an illustrative 63-bit nonnegative integer layout:

```text
41 bits: milliseconds since chosen epoch
10 bits: assigned worker number
12 bits: sequence within that worker's millisecond
```

The timestamp range lasts about 69.7 years from the epoch. There are 1,024 worker numbers and 4,096 sequence slots per worker per millisecond. The resulting theoretical namespace is not a throughput benchmark: physical clocks, synchronization, persistence, implementation cost, and downstream storage impose other limits.

The uniqueness argument requires that no two active generators share a worker number and timestamp/sequence range, including after restarts. A worker lease that expires during a pause does not stop the old process from generating IDs. A generation epoch, durable high-water mark, sufficiently conservative reuse protocol, or another design must prevent overlap. Clock rollback needs an explicit reject, wait, or logical-time policy.

## Lab: make hidden assumptions fail visibly

Save as `id_lab.py` and run with Python. This standalone **single-threaded simulation** injects time and has no durable state or worker coordinator. Its deliberate restart collision demonstrates why it is not a production ID service.

```python
class ClassroomIds:
    def __init__(self, worker):
        if type(worker) is not int or not 0 <= worker < 1024:
            raise ValueError("worker must fit 10 bits")
        self.worker = worker
        self.last_ms = -1
        self.sequence = -1

    def next_id(self, now_ms):
        if type(now_ms) is not int or not 0 <= now_ms < 2**41:
            raise ValueError("time must fit 41 bits")
        if now_ms < self.last_ms:
            raise RuntimeError("clock moved backward")
        candidate = self.sequence + 1 if now_ms == self.last_ms else 0
        if candidate >= 4096:
            raise OverflowError("wait for another millisecond")
        self.last_ms, self.sequence = now_ms, candidate
        return (now_ms << 22) | (self.worker << 12) | candidate


first = ClassroomIds(7)
ids = [first.next_id(1000) for _ in range(4096)]
assert len(set(ids)) == 4096
try:
    first.next_id(1000)
    raise AssertionError("overflow was not rejected")
except OverflowError:
    pass
try:
    first.next_id(999)
    raise AssertionError("rollback was not rejected")
except RuntimeError:
    pass
assert first.next_id(1001) > ids[-1]
other_worker = ClassroomIds(8)
assert other_worker.next_id(1000) not in ids
restarted_without_state = ClassroomIds(7)
assert restarted_without_state.next_id(1000) == ids[0]  # deliberate collision
print("local uniqueness and limits verified; unsafe restart collision reproduced")
```

Do not fix the collision by merely choosing a random worker ID from 1,024 slots. Independent random selection introduces its own collision risk. State exactly how worker ownership is assigned, renewed, retired, and recovered. Test a paused old owner resuming after reassignment, not just two clean startups.

## Sequences, allocation, and commit order

PostgreSQL sequences can allocate values that are not reclaimed after a transaction rollback. A database-generated ID therefore should not be advertised as a gap-free business counter. The sequence also does not define commit order: transaction A can allocate 100, transaction B allocate 101 and commit first, then A commit later. [PostgreSQL sequence functions](https://www.postgresql.org/docs/17/functions-sequence.html).

Optional disposable SQL lab, independent of the capstone tables:

```sql
CREATE TABLE tutorial_id_allocations (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    label text NOT NULL
);
BEGIN;
INSERT INTO tutorial_id_allocations(label) VALUES ('rolled back') RETURNING id;
ROLLBACK;
INSERT INTO tutorial_id_allocations(label) VALUES ('kept') RETURNING id;
SELECT * FROM tutorial_id_allocations;
```

On a fresh table with no concurrent callers, expect the first allocation to be 1 and the retained row's ID to be 2. This is expected sequence behavior, not lost business data. If the business needs a separate numbered document sequence, design that requirement independently with its own concurrency and cancellation rules.

For number blocks, grant ranges transactionally and never reissue a range that might have been used. A process can persist the range boundary and then allocate locally, accepting gaps after failure. After a restart, abandon the previous range's remaining values and acquire a fresh range, or durably track consumed positions with the atomicity required to prevent reuse. Persisting only the range boundary does not make restarting from its beginning safe. Range size trades authority load against wasted values and operational visibility.

## Django integration and migration

The capstone's `UUIDField(default=uuid.uuid4, primary_key=True)` is appropriate for its independent object identity. Keep `version` for optimistic concurrency and `created_at` plus ID as a deterministic pagination order. Do not replace those fields with assumptions about a UUID timestamp.

Changing an existing primary-key format affects foreign keys, indexes, caches, event payloads, URLs, exports, and clients. An expand-and-migrate approach can add a new identifier alongside the old one, populate a mapping, support both during transition, and retire the old representation only after consumers have migrated. A table rewrite alone is not the full migration.

Opaque IDs do not replace object authorization. Even an unguessable ID can leak through logs or links. Time-oriented IDs may reveal approximate creation times; expose only the metadata your product intends to disclose.

## Failure tests and completion checklist

Test clock rollback, same-tick overflow, process restart, cloned worker configuration, worker reassignment after a pause, imported legacy data, duplicate client requests, and database uniqueness violations. Record whether each case waits, rejects, retries, or reconciles.

- [ ] Uniqueness scope, lifetime, and ordering guarantees are written down.
- [ ] Generation independence and entropy/ownership assumptions are explicit.
- [ ] Restarts and clock changes cannot silently reuse an active range.
- [ ] IDs are not used as authorization or commit-order proof.
- [ ] Database constraints detect collisions and retries preserve business intent.
- [ ] Public serialization preserves all bits across supported clients.

Apply the decision to the [URL-shortener design](../11-system-design-case-studies/01-url-shortener.md) and [chat sequencing](../11-system-design-case-studies/02-chat-and-messaging.md).
