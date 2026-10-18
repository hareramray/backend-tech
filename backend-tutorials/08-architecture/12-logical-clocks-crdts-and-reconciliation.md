# Logical clocks, CRDTs, and reconciliation

## Purpose and prerequisites

Replicas can disagree because updates arrive late, arrive in a different order, or occur independently during a partition. A reconciliation design must identify which differences represent newer information, which represent concurrent choices, and which require a business decision. A timestamp by itself cannot answer all three questions.

Read [distributed systems and consistency](03-distributed-systems-and-consistency.md), [capacity planning](05-system-design-and-capacity-planning.md), and [multi-region failover](11-multi-region-and-failover.md). This chapter develops small, executable models using Python's standard library. These models explain merge rules; they do not implement durable replication, authorization, or a production synchronization service.

## Wall-clock order is not causal order

An event happens before another when they occur in that order in one process, when one sends a message that the other receives, or when a transitive chain of those relations connects them. If neither direction holds, they are concurrent in this model. Concurrent does not mean the same physical instant: it means the observed communication history does not order them.

A laptop can create edit A at its displayed 14:02 and another create edit B at displayed 14:01. Clock skew can make B physically later. Clock correction can even move a displayed time backward. Use wall-clock timestamps for human context and audited time requirements, with known uncertainty where necessary; do not treat arbitrary client timestamps as authority over protected state.

Lamport clocks increment a local scalar for events. Receiving a message advances to `max(local, received) + 1`. If A happens before B, A's Lamport time is smaller; the reverse implication does not hold. Adding a replica identifier gives a deterministic total-order tie breaker, but that total order does not discover causality between otherwise concurrent events. [Lamport's original clock paper](https://www.microsoft.com/en-us/research/publication/time-clocks-ordering-events-distributed-system/).

For a concrete example, process A performs ten unrelated local actions while B performs one. B's timestamp 1 being smaller than A's timestamp 10 does not mean A observed B. A physical-clock-assisted logical scheme can keep timestamps near wall time, but its bounded metadata still does not encode every concurrency relationship. Pick clock metadata for the question it must answer.

## Vector clocks expose concurrency

A vector records one counter per participating actor. A local event increments that actor's component. Receiving information takes componentwise maxima and then increments the receiver's component for the receive event. Vector X precedes Y if every component of X is at most Y and at least one is smaller. Unequal vectors are concurrent when neither precedes the other; equal vectors represent equal recorded causal context. Missing components can represent zero when the actor universe and representation allow that convention. [Fidge's original vector-timestamp paper](https://fileadmin.cs.lth.se/cs/Personal/Amr_Ergawy/dist-algos-papers/4.pdf).

Consider two task-title edits starting from vector `{A: 2, B: 1}`:

| Action | New title | Vector |
|---|---|---|
| A edits locally | Prepare release | `{A: 3, B: 1}` |
| B edits without seeing A | Delay release | `{A: 2, B: 2}` |
| A receives B and explicitly resolves | Prepare release after review | `{A: 4, B: 2}` |

The first two edits are concurrent, so neither should silently be described as the other's replacement. The resolution observes both, so it dominates both vectors. A user interface could retain both titles until a person chooses; a structured task description might support an automatic merge for independent fields.

Vector metadata grows with actors. A system with millions of intermittently connected clients needs a carefully designed actor and compaction policy. Reusing a replica ID after losing its counter can make new operations look old or duplicate. Version vectors often summarize object-update histories rather than every process event; specify what each component means. Equal vector metadata does not excuse inconsistent payloads caused by corruption or an application bug.

## Convergence requires a merge algebra

A state-based CRDT sends mergeable state. Its merge is associative, commutative, and idempotent, while legal local updates advance through the type's state order. Consequently, duplicate and reordered state deliveries can converge once replicas receive the needed information. Operation-based CRDTs have different delivery obligations; do not assume every operation can be replayed arbitrarily without deduplication or causal delivery. [Preguiça's CRDT overview](https://arxiv.org/abs/1805.06358).

The three algebraic checks have operational meanings:

- Associative: regrouping batches does not change the merged result.
- Commutative: receiving A then B or B then A produces the same result.
- Idempotent: receiving the same state twice does not count it twice.

A grow-only counter assigns each actor its own monotonically increasing component. The visible value is the sum; merge takes componentwise maximum. If A reports 3 twice, its contribution remains 3. A positive-negative counter uses two such structures, one for increments and one for decrements; its value is `sum(positive) - sum(negative)`.

Only the owning actor may advance its component under this simple design. Two disconnected processes concurrently updating the same component from 3 to 4 would merge to 4 and lose an increment. Give independent writers independent durable identities. Decrementing a grow-only component directly similarly violates the model; use the PN structure instead.

## Sets, tombstones, and the cost of forgetting

An observed-remove set gives every add a unique tag. Removing an element records the tags observed for that element. Merging unions adds and removed tags. An add concurrent with the removal survives because the remover never observed its tag: this is an add-wins policy. The policy is semantic, not proof that the add was physically later. More sophisticated observed-remove sets compact causal metadata instead of retaining every tombstone indefinitely. [Observed-remove set research](https://arxiv.org/abs/1210.3368).

Imagine tag `(A, 7)` adds label `urgent`. B sees that tag and removes it. Meanwhile C independently adds `urgent` with tag `(C, 3)`. The merged label is present because C's add remains live. If the business requirement is that an administrator's ban defeats all concurrent grants, an add-wins set is the wrong primitive without additional policy.

Tombstones cannot be deleted merely because they are old. An offline replica can later resend the corresponding add and resurrect deleted content. Safe reclamation needs evidence that relevant participants have observed the deletion, a controlled membership/epoch scheme, or a documented forced-resynchronization rule for obsolete replicas. Reclaim the corresponding removed add-tags along with their tombstones under that protocol: clearing only tombstones in the model below immediately exposes the retained add-tags again. Backups and restored clients count as potential sources of old state. Data retention and deletion requirements therefore influence the replication protocol itself.

A last-write-wins register instead chooses a maximum timestamp and deterministic tie breaker. It can converge while discarding valuable concurrent information. If a fast client clock selects a mistaken title for hours, convergence faithfully reproduces the mistake. LWW is often reasonable for replaceable preferences; it needs much stronger justification for permissions, balances, and collaborative content.

## Runnable lab: compare clocks and test merge laws

Save this block as `replica_merge.py` and run `python replica_merge.py`. It prints three confirmation lines after its assertions. Actor IDs and operation tags are assigned explicitly so the causal examples are repeatable. The set model deliberately retains all tombstones.

```python
from dataclasses import dataclass
from itertools import permutations


def join(left, right):
    return {k: max(left.get(k, 0), right.get(k, 0)) for k in left.keys() | right.keys()}


def compare(left, right):
    keys = left.keys() | right.keys()
    less = any(left.get(k, 0) < right.get(k, 0) for k in keys)
    greater = any(left.get(k, 0) > right.get(k, 0) for k in keys)
    if less and greater:
        return "concurrent"
    if less:
        return "before"
    if greater:
        return "after"
    return "equal"


def event(clock, actor, received=None):
    result = join(clock, received or {})
    result[actor] = result.get(actor, 0) + 1
    return result


base = {"A": 2, "B": 1}
edit_a = event(base, "A")
edit_b = event(base, "B")
resolved = event(edit_a, "A", edit_b)
assert compare(edit_a, edit_b) == "concurrent"
assert compare(edit_a, resolved) == "before"
assert compare(edit_b, resolved) == "before"
assert compare({}, {"A": 0}) == "equal"
print("vector concurrency and causal resolution passed")

states = [{"A": 3}, {"B": 2}, {"A": 4, "C": 1}]
expected = {"A": 4, "B": 2, "C": 1}
for ordering in permutations(states):
    result = {}
    for state in ordering + ordering:  # Duplicate all deliveries.
        result = join(result, state)
    assert result == expected
assert join(states[0], states[0]) == states[0]
assert join(join(states[0], states[1]), states[2]) == join(states[0], join(states[1], states[2]))
assert sum(expected.values()) == 7

positive = join({"A": 3}, {"B": 2})
negative = join({"A": 1}, {"B": 1})
assert sum(positive.values()) - sum(negative.values()) == 3
print("G-counter value: 7; PN-counter value: 3")


@dataclass(frozen=True)
class ORSet:
    adds: frozenset = frozenset()       # (element, unique_tag)
    removed: frozenset = frozenset()    # unique_tag

    def add(self, element, tag):
        # A real writer must allocate durable unique tags, never reuse them.
        return ORSet(self.adds | {(element, tag)}, self.removed)

    def remove(self, element):
        observed = {tag for value, tag in self.adds if value == element}
        return ORSet(self.adds, self.removed | observed)

    def merge(self, other):
        return ORSet(self.adds | other.adds, self.removed | other.removed)

    def values(self):
        return {value for value, tag in self.adds if tag not in self.removed}


original = ORSet().add("urgent", ("A", 7))
removed = original.remove("urgent")
concurrent_add = original.add("urgent", ("C", 3))
merged = removed.merge(concurrent_add)
assert merged.values() == {"urgent"}
assert removed.merge(original).values() == set()  # Old add stays removed.
assert merged.merge(merged) == merged
assert removed.merge(concurrent_add) == concurrent_add.merge(removed)
for ordering in permutations([original, removed, concurrent_add]):
    result = ORSet()
    for state in ordering + ordering:
        result = result.merge(state)
    assert result == merged
print("OR-set converges; a concurrent add survives an observed remove")
```

The expected output is the three literal strings supplied to `print`. The permutation checks exercise duplicate and reordered delivery for these finite examples; they do not prove the merge implementation for every possible history. Extend the examples with a fourth actor, a remove after observing both adds, and restoration of an older replica. The post-observation remove should remove both tags, while replaying the old state should not resurrect them.

## Anti-entropy finds missing information

Good merge rules do not deliver messages. A replication protocol also needs retry, discovery of missing data, membership handling, and bounded resource usage. Anti-entropy periodically compares replicas and exchanges differences. A Merkle tree hashes deterministic leaves and then groups their hashes hierarchically; unequal parent hashes identify subtrees worth comparing. The technique reduces comparison work when large datasets differ in a small fraction of their ranges. [Dynamo's anti-entropy design](https://www.amazon.science/publications/dynamo-amazons-highly-available-key-value-store).

For a task dataset, hash a canonical representation containing the task ID, payload version, and deletion metadata. Hashing only live payloads omits evidence required to prevent resurrection. Compare compatible ownership ranges at an appropriate snapshot/version boundary; otherwise concurrent writes can make every comparison appear unstable. Tree equality is a digest-based equivalence check under the hash assumptions, not a business validation or proof that authorized updates occurred.

Read repair reconciles versions encountered while serving a read. It improves visited keys but leaves cold keys untouched, so a periodic anti-entropy path is still needed. Repair must use the documented version/merge rules. Selecting whichever response arrived first can propagate an obsolete value faster instead of correcting it.

## Semantic reconciliation and PostgreSQL integration

Classify fields before selecting a merge rule. Independent task labels may suit an observed-remove set. Concurrent free-text titles may need retained alternatives. A workflow transition from approved to rejected needs a business rule about authority and observed state, not alphabetical ordering. A maximum timestamp cannot enforce an approval policy.

An unconstrained PN-counter can become negative: two regions can each spend a locally observed final unit. If nonnegative inventory is mandatory, coordinate the decrement or allocate bounded spending rights through an escrow-style design. Bounded-counter research explains how distributing rights can preserve numeric invariants while allowing some local progress. [Bounded counter paper](https://arxiv.org/abs/1503.09052).

In a Django service, keep received operation IDs, actor metadata, and resulting state in the same local PostgreSQL transaction. Validate tenant, actor permissions, payload shape, and supported schema before merging. An idempotent merge does not make a duplicate notification harmless; write notification intent through an outbox with a stable deduplication key. Avoid updating a JSON state with an unprotected read-modify-write loop: concurrent local workers can lose changes before replication even begins.

Provide operational evidence: count unresolved conflicts, oldest unsynchronized changes, tombstone volume, repair traffic, rejected obsolete actors, and divergence discovered during comparison. A replication queue becoming empty does not establish that every disconnected replica has caught up. Rejoining replicas need an explicit protocol, especially after metadata compaction or restoration from backup.

## Exercises and completion checklist

Write three concurrent histories for task labels, task titles, and inventory reservations. Select a merge policy for each and explain which information survives. Then remove the OR-set tombstones before merging an old snapshot and show the resurrection bug. Finally, describe how a deleted actor identity is replaced without reusing its sequence numbers.

- [ ] Wall-clock display, logical order, and causality have separate meanings.
- [ ] Concurrent versions are distinguished from dominated versions.
- [ ] Merge rules satisfy the required algebra and delivery assumptions.
- [ ] Actor identities and operation tags remain unique across failures.
- [ ] Tombstone reclamation accounts for offline replicas and old backups.
- [ ] Repair exchanges the metadata needed to preserve deletions and causality.
- [ ] Business invariants are checked independently of convergence.

Return to [consensus](06-consensus-raft-and-paxos.md) to compare coordination with reconciliation, or continue with [storage engines](13-storage-engines-lsm-trees-and-bloom-filters.md).
