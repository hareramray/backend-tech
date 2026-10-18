# Sharding, consistent hashing, and online resharding

## Purpose and prerequisites

Sharding divides a dataset into ownership units that can live on different database servers. It can increase aggregate storage and write capacity, but it also creates routing, migration, and transaction boundaries. A successful design explains where every request goes during both normal operation and a partially completed migration.

Read [distributed systems and consistency](03-distributed-systems-and-consistency.md), [capacity planning](05-system-design-and-capacity-planning.md), and [consensus](06-consensus-raft-and-paxos.md). This chapter uses a Django task service with PostgreSQL shards as its running example. Its Python lab is a placement simulation, not a database router suitable for deployment.

## Choose an ownership boundary before an algorithm

Suppose every task belongs to one tenant. Co-locating a tenant's tasks, comments, assignments, and quota row preserves ordinary local transactions for most application requests. Sharding by task ID instead might balance task counts more evenly but scatter the tenant's dashboard and quota checks across machines.

Distinguish a logical shard from a physical database. A logical shard can be a stable hash bucket or an explicit group of tenants; several logical shards can share a database. Moving a logical shard changes placement without changing every application's understanding of the data model. PostgreSQL table partitioning within one instance does not, by itself, distribute writes over independent database servers.

| Placement scheme | Helpful property | Main cost to examine |
|---|---|---|
| Key ranges | Local range scans and understandable boundaries | Sequential inserts or popular ranges can concentrate load |
| Hash buckets | Broad spread for high-cardinality keys | Range scans fan out; skewed demand still causes hotspots |
| Tenant directory | Explicit placement, isolation, and movement | Directory availability, caching, and versioning become dependencies |
| Composite tenant/sub-bucket | Splits very large tenants | Tenant-wide joins and invariants cross sub-buckets |

Use estimated bytes, read/write demand, and transaction locality together. Equal row counts do not imply equal CPU, I/O, or lock pressure. One tenant performing a million writes to one counter remains hot even if its other rows are evenly distributed.

## Why modulo hashing creates movement

The simple rule `hash(key) % N` changes many assignments when `N` changes. With ideal uniformly distributed hashes, comparing modulo 3 with modulo 4 preserves the same numbered destination for only three of twelve equally likely residues. About 75% move in this example, even though the new server should ultimately own only about 25% of keys.

One solution is a fixed number of logical buckets, such as 4,096, plus a versioned bucket-to-database map. Adding a server changes selected bucket ownership, not the key-to-bucket function. Another is consistent hashing: place server tokens around a hash ring and assign a key to the first token clockwise. Adding a token claims only the interval between its predecessor and itself. Virtual nodes give one physical server multiple tokens, improving distribution and providing smaller movement units. Dynamo is a primary example of combining this approach with replication and version reconciliation. [Dynamo paper](https://www.amazon.science/publications/dynamo-amazons-highly-available-key-value-store).

Virtual nodes are not replicas. If ten consecutive tokens belong to the same host, selecting the next three tokens does not provide three independent failure domains. Replica selection must skip repeated owners and enforce the intended zone or region placement. More virtual nodes increase metadata and management work; choose their count from imbalance and migration measurements rather than an arbitrary folklore number.

## Rendezvous hashing and deterministic placement

Rendezvous hashing scores each `(key, node)` pair and chooses the highest score. Adding a node changes ownership only for keys that the new node wins; removing a node changes only its previous winners. The straightforward evaluation costs one score per candidate node, whereas a sorted ring supports a logarithmic successor lookup after construction. Weighted variants need a properly derived scoring rule; simply multiplying an arbitrary integer hash by capacity is not a generally correct weighting method. [Thaler and Ravishankar's original highest-random-weight paper](https://www.cs.ucr.edu/~ravi/Papers/Jrnl/HRW98.pdf).

All routers must agree on the hash algorithm, byte encoding, node identities, membership, and tie breaking. Python's built-in `hash()` is unsuitable for persistent cross-process routing because hashes of strings and bytes are randomized between interpreter processes by default. Use a specified stable digest or a maintained placement library and version the complete policy. [Python data model: object hashes](https://docs.python.org/3/reference/datamodel.html#object.__hash__).

A directory can sit above either algorithm: ordinary tenants use the default map, while tenant 900 is explicitly assigned to a dedicated shard. Authorization and routing remain separate. A client-supplied shard name must never grant access; resolve the authenticated tenant, authorize the operation, then select its trusted placement.

## Runnable lab: measure movement and reject stale routing

Save this block as `shard_placement.py`, then run `python shard_placement.py`. It uses only Python's standard library and operates on synthetic keys. It checks movement invariants rather than asserting that every finite sample has perfectly balanced load.

```python
from bisect import bisect_left
from collections import Counter
from hashlib import sha256


def digest(value):
    return int.from_bytes(sha256(value.encode("utf-8")).digest(), "big")


def make_ring(nodes, virtual_nodes=64):
    tokens = sorted(
        (digest(f"node:{node}:token:{i}"), node)
        for node in nodes for i in range(virtual_nodes)
    )
    return [position for position, _ in tokens], tokens


def ring_owner(key, ring):
    positions, tokens = ring
    index = bisect_left(positions, digest(f"key:{key}"))
    return tokens[index % len(tokens)][1]


def rendezvous_owner(key, nodes):
    # Separator and fixed node/key naming are part of this toy encoding.
    return max(nodes, key=lambda node: (digest(f"{key}|{node}"), node))


keys = [f"tenant-{i}" for i in range(12000)]
old_nodes = ["A", "B", "C"]
new_nodes = ["A", "B", "C", "D"]
old_ring, new_ring = make_ring(old_nodes), make_ring(new_nodes)
old_rv = {key: rendezvous_owner(key, old_nodes) for key in keys}
new_rv = {key: rendezvous_owner(key, new_nodes) for key in keys}

ring_moved = [k for k in keys if ring_owner(k, old_ring) != ring_owner(k, new_ring)]
rv_moved = [k for k in keys if old_rv[k] != new_rv[k]]
assert all(ring_owner(k, new_ring) == "D" for k in ring_moved)
assert all(new_rv[k] == "D" for k in rv_moved)
assert all(
    rendezvous_owner(k, ["A", "C", "D"]) == new_rv[k]
    for k in keys if new_rv[k] != "B"
)

mod_moved = sum(digest(k) % 3 != digest(k) % 4 for k in keys)
assert 0 < len(rv_moved) < mod_moved
assert 0 < len(ring_moved) < mod_moved
print("movement properties passed")
print("rendezvous owners:", dict(sorted(Counter(new_rv.values()).items())))
print("moved:", {"modulo": mod_moved, "ring": len(ring_moved), "rendezvous": len(rv_moved)})

# Model the receiver's ownership gate, not merely the router's cache.
directory = {"tenant-42": (8, "A")}


def accept_write(tenant, request_epoch, destination):
    epoch, owner = directory[tenant]
    if (request_epoch, destination) != (epoch, owner):
        raise ValueError("stale routing or wrong owner")
    return "accepted"


assert accept_write("tenant-42", 8, "A") == "accepted"
directory["tenant-42"] = (9, "D")
try:
    accept_write("tenant-42", 8, "A")
except ValueError:
    print("old owner rejected the stale request")
else:
    raise AssertionError("stale owner accepted a write")
assert accept_write("tenant-42", 9, "D") == "accepted"
```

Expected results: both assertion groups pass; the first and last printed lines are `movement properties passed` and `old owner rejected the stale request`. The middle lines report 2,947 / 2,959 / 3,033 / 3,061 rendezvous keys on A / B / C / D. Modulo moves 9,038 keys, the ring moves 2,789, and rendezvous moves 3,061. These finite-sample counts are less important than the property that an added node receives every moved key. A routing formula does not copy any data: switching from `old_ring` to `new_ring` in a real application before migration would route valid requests to empty storage.

The ownership gate is intentionally centralized in this toy model. A real old owner isolated from the directory must still lose write authority through a fencing, lease, or coordinated drain protocol. A stale cached epoch on both router and receiver is not protection. Replica availability, authentication, persistence, concurrency, and migration are outside this simulation.

## Online resharding: copy, catch up, fence, cut over

Treat moving tenant 42 from source A to target D as a state machine with a durable migration ID and ownership epoch. Retrying any step should be safe. Define what is authoritative at every stage; an ambiguous two-writer interval is the failure to prevent.

1. **Plan:** verify capacity, schema compatibility, keys, indexes, and dependencies. Record source ownership at epoch 8. Reserve target space for the copy plus ongoing change traffic.
2. **Establish capture:** obtain a consistent snapshot and its corresponding change-log boundary. Starting change capture after a long snapshot without linking their boundaries can lose intervening writes.
3. **Copy:** transfer the snapshot in resumable chunks. Source A remains authoritative. Record progress, row versions, deletes, and checksums; compare equivalent logical positions, not two tables still changing independently.
4. **Catch up:** replay committed changes after the snapshot boundary in order, with duplicate-safe application. Include deletes and transaction boundaries. Monitor lag in bytes and time; a target applying slower than the incoming write rate will never finish.
5. **Fence and drain:** stop source acceptance for this tenant, finish or reject in-flight work, and record a final source position. Drain the target through that position. Background jobs and delayed queue messages must obey the same ownership protocol as HTTP requests.
6. **Publish:** atomically advance placement to epoch 9 and owner D using the control plane's supported consistency mechanism. D accepts epoch-9 work only after the cutover preconditions hold. A rejects or redirects stale work.
7. **Verify and retire:** check business invariants, row counts, sampled content, replication position, and request errors. Retain a bounded recovery window before deleting old storage.

This sequence is a design model, not instructions for hand-building a live database migrator. Citus documents online shard movement based on PostgreSQL logical replication and the replica-identity requirements for relevant changes. Use the supported migration mechanism for the actual product and version. [Citus cluster management](https://docs.citusdata.com/en/latest/admin_guide/cluster_management.html).

Before D accepts new writes, rollback may mean discarding the copy and retaining A. After D accepts writes, pointing the directory back to stale A loses acknowledged work. Returning ownership now requires reverse catch-up or a new controlled migration, plus another fencing transition. Keep the migration journal distinct from an operator's memory of which command last returned successfully.

## Hotspots, queries, and cross-shard invariants

Separate a storage hotspot from a read hotspot and a serialized-write hotspot. A large tenant can move to dedicated capacity; popular immutable reads can use a cache; a high-rate counter may need partitioned aggregation. Splitting one strict account balance over independent writable shards does not preserve its invariant just because the hash distribution improves.

For cross-shard reports, choose an explicit path: bounded scatter/gather, a read projection fed by events, or an analytical store. Fan-out latency follows the slow shards required by the answer, and partial failures need a visible response contract. A global sort needs a stable ordering and cursor strategy across shards; offset pagination independently on each shard is not a complete solution.

Global uniqueness is also a coordination problem. A UUID reduces accidental identifier collision but does not enforce that two users cannot register the same normalized username. A global username registry, deterministic single-owner routing for that namespace, or a database with supported distributed constraints can provide the required boundary. A saga can compensate business operations, but it does not make a temporarily violated hard uniqueness constraint disappear.

Django supports database aliases and explicit `.using(alias)` selection, but a router does not automatically interpret every arbitrary queryset filter to discover the tenant. Carry trusted tenant context into a repository/service boundary and pass the alias explicitly where clarity matters. `transaction.atomic(using=alias)` protects that database; nesting blocks for different aliases is not an atomic distributed commit. Django does not support foreign-key relationships spanning separate databases. [Django multiple-database documentation](https://docs.djangoproject.com/en/5.2/topics/db/multi-db/).

## Exercises and completion checklist

Give 1% of the lab's tenants 90% of simulated requests and compare request load with key counts. Increase virtual nodes and explain why that cannot split one hot tenant. Then write a migration incident timeline in which cutover succeeds but the operator's connection drops before receiving confirmation; specify how recovery reads durable state before retrying.

- [ ] Shard keys match the application's transaction and query boundaries.
- [ ] Logical ownership is separate from physical server placement.
- [ ] Hash encoding and routing versions are deterministic and documented.
- [ ] Stale writers are rejected at an enforceable ownership boundary.
- [ ] Migration captures concurrent updates, deletes, and its final drain position.
- [ ] Rollback after new writes preserves the new authoritative history.
- [ ] Cross-shard reports, uniqueness, backups, and restoration have explicit designs.

Continue with [traffic management](08-load-balancing-and-traffic-management.md) or [multi-region failover](11-multi-region-and-failover.md).
