# Consensus, Raft, and Paxos

## Purpose and prerequisites

Consensus lets independently failing machines agree on decisions. A replicated log turns those decisions into an ordered history for a deterministic state machine. This chapter explains what the agreement protects, how Raft and Paxos establish it, and where application responsibilities begin.

Read [distributed systems and consistency](03-distributed-systems-and-consistency.md) and [capacity planning](05-system-design-and-capacity-planning.md) first. The lab uses only Python's standard library. It is a deliberately incomplete, in-memory simulation of voting and commit predicates, not a consensus implementation or a replacement for PostgreSQL replication management.

## Start with an invariant and a failure model

Imagine a task scheduler assigning ownership of partition 17. Its invariant is: two different ownership decisions cannot both be committed at the same log index. A secondary invariant is that every replica applies the same committed prefix in the same order. If the command contains an uncontrolled call to `random()` or the current time, identical logs can still produce different state; put the selected value in the command instead.

The model here includes crashes, restarts with durable state, delays, duplicate messages, and network partitions. It excludes malicious voters and arbitrary storage corruption. Byzantine consensus addresses a different threat model. A quorum is a set large enough to participate in a decision under the chosen protocol; here a majority of `N` voters is `floor(N / 2) + 1`.

For five voters, every pair of three-voter majorities intersects. Intersection supplies a witness to earlier decisions; protocol rules make that witness meaningful. Merely requiring three acknowledgements is insufficient if a voter forgets its promises after restarting. Safety means conflicting decisions never commit. Liveness means a decision eventually completes under the protocol's progress assumptions. Losing three of five voters should stop new decisions rather than invent another history.

## Raft: terms, votes, and replicated entries

Raft divides operation into numbered terms. A candidate advances its term, votes for itself, and requests votes. Each voter grants at most one vote per term, and only to a candidate with a sufficiently up-to-date log. Compare last-entry term first, then last-entry index. A higher term observed in a message causes a server to abandon an older leadership claim. Election timers are randomized to reduce repeated split votes. [Raft paper, sections 5.1, 5.2, and 5.4](https://raft.github.io/raft.pdf).

Consider voters A, B, and C with the same committed prefix through index 40:

1. A leads term 8 and appends `assign(17, worker_x)` at index 41.
2. A durably stores it and B durably acknowledges it. Two of three voters form a majority, so A can commit this current-term entry.
3. C remains at index 40. If A disappears, C cannot win B's vote with its stale log. B can campaign with the newer log.
4. B eventually leads term 9, establishes its new term, and brings C up to date. An isolated A cannot commit another command using only its own vote.

The vote record and current term must survive restart. Otherwise B could vote for A, crash, forget, and vote for C in the same term. Log acknowledgement also needs the required durable persistence boundary. A process having bytes in a userspace buffer is a different fact from recovery being able to reproduce them. The etcd Raft library explicitly separates consensus decisions from application-managed persistence and networking. [etcd Raft implementation contract](https://github.com/etcd-io/raft).

Append messages identify the preceding log index and term. A follower rejects an append when that predecessor does not match; the leader finds the shared prefix and replaces the conflicting uncommitted suffix. The matching invariant is stronger than equal lengths: equal index and term identify the same entry and preceding history in a valid Raft execution. A follower with 100 stale entries is not automatically a better candidate than one with 90 entries ending in a later term.

### Why old-term majority counting is dangerous

Suppose a term-12 leader inherits entry 51 from term 10. It observes that entry on a majority. It must not declare it newly committed solely by counting those copies: Raft's direct majority commit rule requires an entry from the leader's current term. Committing a term-12 entry at index 52 also commits the preceding prefix. The term restriction prevents a later election from selecting an incompatible history that still looks newer by last-entry term. The lab isolates this distinction without pretending to simulate every possible election. [Raft paper, section 5.4.2](https://raft.github.io/raft.pdf).

## Reads, retries, snapshots, and membership

An old leader can remain alive while a majority elects a successor. Reading its local dictionary is therefore not proof of a linearizable read. A safe read-index path establishes a current-term committed entry, obtains the protocol's quorum confirmation associated with the read, records a safe committed position, and waits until the local state machine has applied through that position. A lease shortcut introduces timing assumptions about clocks, pauses, and expiry that must be established by the implementation. [etcd Raft read-only request options](https://github.com/etcd-io/raft/blob/main/README.md).

For example, B commits ownership of partition 17 to worker Y while A is isolated. A's earlier ownership value for worker X cannot authorize fresh work. Even a correct ownership store needs the protected resource to enforce a fencing generation; otherwise a paused worker X can resume writing after losing ownership. Read guarantees also differ by API: etcd distinguishes its linearizable operations from cheaper reads that may return older state. [etcd API guarantees](https://etcd.io/docs/v3.6/learning/api_guarantees/).

A client timeout after commit does not undo the command. Attach a stable request identifier and persist its result with the state-machine update. Replaying `allocate(request_123)` can then return the original result. Sending an email inside every replica's apply function is still wrong: each replica might send it. Record an outbox intent and use an idempotent external delivery path.

Snapshots summarize an applied prefix and retain its last included index, term, and required configuration metadata. They reduce replay and log size; they do not excuse applying an incomplete snapshot. A practical restore also needs schema/version compatibility and application deduplication state. Test interruption between writing snapshot bytes, publishing metadata, and deleting obsolete log segments.

A membership change alters the definition of majority. Replacing `{A,B,C}` with `{D,E,F}` by editing configuration files can create two disjoint majorities. Joint consensus uses an intermediate configuration requiring a majority of both old and new voter sets, followed by the new configuration alone. Catch up new replicas before expecting them to sustain availability. Learners can receive data without immediately increasing the voting quorum. Membership and snapshot details are covered more extensively in [Ongaro's dissertation and errata](https://github.com/ongardie/dissertation).

## Paxos: preserve an already possible decision

Single-decree Paxos chooses one value. A proposer uses a unique, ordered ballot number. During prepare, acceptors promise not to accept lower ballots and return their highest previously accepted proposal. After a majority promises, the proposer must reuse the value attached to the highest reported accepted ballot, if any; otherwise it may choose its own value. During accept, a majority accepting that ballot and value chooses it. Promises and accepted proposals must survive relevant failures. [Lamport, Paxos Made Simple](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/paxos-simple-Copy.pdf).

An original worked sequence illustrates why recovery cannot just choose a new favorite:

| Step | Acceptor A | Acceptor B | Acceptor C |
|---|---|---|---|
| Ballot 7 prepares | Promise 7 | Promise 7 | Unreachable |
| Ballot 7 accepts blue | Accepted `(7, blue)` | Accepted `(7, blue)` | Unreachable |
| Proposer loses response | Blue remains chosen | Blue remains chosen | Still unaware |
| Ballot 9 prepares B and C | Unreachable | Reports `(7, blue)` | Reports no acceptance |
| Ballot 9 accepts | Unreachable | Accepts `(9, blue)` | Accepts `(9, blue)` |

The ballot-9 proposer cannot choose red because B provides evidence that blue might already be chosen. Even one reported acceptance can constrain the new value; the proposer cannot infer that no other acceptor also accepted it. A delayed ballot-7 accept arriving after a ballot-9 promise is rejected.

Competing proposers can continually preempt one another without violating safety. A stable leader and suitable communication conditions allow progress. Multi-Paxos extends agreement across log slots and can amortize prepare work under a stable leader; recovery still must preserve accepted values and fill gaps before ordered application. Raft emphasizes an explicit leader/log structure, while Paxos exposes the promise-and-accept reasoning directly. Neither eliminates the storage, retry, membership, and operational work needed by a production service.

## Runnable lab: inspect two Raft safety guards

Save this block as `consensus_guards.py` outside the tutorial tree if you want to experiment, then run `python consensus_guards.py`. No services or packages are required. The `persisted_*` fields merely model durable facts in memory: there is no actual disk or network simulation.

```python
from dataclasses import dataclass


@dataclass
class Voter:
    persisted_term: int
    persisted_vote: str | None
    last_log_term: int
    last_log_index: int

    def request_vote(self, term, candidate, log_term, log_index):
        if term < self.persisted_term:
            return False
        if term > self.persisted_term:
            self.persisted_term = term
            self.persisted_vote = None
        up_to_date = (log_term, log_index) >= (
            self.last_log_term, self.last_log_index
        )
        may_vote = self.persisted_vote in (None, candidate)
        if up_to_date and may_vote:
            self.persisted_vote = candidate
            return True
        return False


def advance_commit(current_term, log_terms, match_indexes, committed=0):
    # Index 0 is a sentinel. Caller supplies evidence from the active leader
    # for a fixed membership, with one match position per voting member.
    quorum = len(match_indexes) // 2 + 1
    for index in range(len(log_terms) - 1, committed, -1):
        replicated = sum(match >= index for match in match_indexes)
        if log_terms[index] == current_term and replicated >= quorum:
            return index
    return committed


voter = Voter(8, None, 8, 41)
assert not voter.request_vote(9, "stale", 8, 40)
assert voter.request_vote(9, "B", 8, 41)
assert not voter.request_vote(9, "C", 8, 41)
assert not voter.request_vote(8, "A", 8, 41)

restarted = Voter(**vars(voter))  # Restore the modeled persisted state.
assert not restarted.request_vote(9, "C", 8, 41)
assert restarted.request_vote(10, "C", 9, 42)

terms = [0, 10, 12]
assert advance_commit(12, terms, [2, 1, 1]) == 0
assert advance_commit(12, terms, [2, 2, 1]) == 2
assert advance_commit(12, terms, [2, 1, 1], committed=1) == 1

print("stale candidate rejected; second vote rejected after restart")
print("old-term majority alone: 0; current-term majority: 2")
```

Expected output is exactly the two printed lines. In the first commit test, two voters have index 1, but that entry belongs to term 10. The second test gives a majority index 2 in term 12, committing both entries. The third test proves an already known committed index does not move backward when a new leader has incomplete replication evidence.

The lab omits message ordering, actual stable storage, leader transitions, snapshot installation, membership changes, and state-machine execution. Passing it is evidence about these small predicates only. It is useful precisely because each assertion corresponds to a stated invariant rather than a claim of production correctness.

## Applying this to Django and PostgreSQL

Use PostgreSQL transactions for local business invariants. A separate consensus-backed coordination system may manage service membership, leases, or deployment ownership, but it does not make a Django transaction and an external payment atomically commit together. Keep transaction boundaries explicit and use outbox delivery or a documented distributed transaction mechanism when needed.

Budget coordination latency and quorum placement. Three voters placed in one failure domain tolerate one process loss, but not necessarily loss of that domain. Five voters cost more replication traffic and may have a slower quorum path. Measure disk sync and quorum round-trip tails, leadership churn, applied-log lag, snapshot duration, and pending proposals. A green process-health endpoint is weaker evidence than successful quorum-backed operations.

## Exercises and completion checklist

Change the lab so a restart forgets the vote while keeping the same term; demonstrate the forbidden second vote. Then design a five-voter placement across three regions and enumerate which individual region outages retain a majority. Finally, write the retry contract for a scheduler allocation that commits but loses its HTTP response.

- [ ] I can explain why intersecting quorums need durable protocol state.
- [ ] I distinguish appended, replicated, committed, and applied positions.
- [ ] I can explain the current-term commit restriction and stale-leader read hazard.
- [ ] Membership replacement preserves quorum intersection through the transition.
- [ ] Replayed requests and external side effects have explicit deduplication rules.
- [ ] My production design uses a maintained implementation with tested recovery.

Continue with [sharding and consistent hashing](07-sharding-and-consistent-hashing.md) and [logical clocks and reconciliation](12-logical-clocks-crdts-and-reconciliation.md).
