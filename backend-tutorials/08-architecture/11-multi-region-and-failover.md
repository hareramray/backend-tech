# Multi-region systems, failover, and failback

## Purpose and prerequisites

A second region can reduce a regional outage's impact, bring reads closer to users, or separate tenant data geographically. It also creates more failure states: a network partition can leave both regions healthy while they disagree about who may write. This tutorial designs recovery for the Django task service and rehearses ownership changes locally.

Read [capacity planning](05-system-design-and-capacity-planning.md), [distributed consistency](03-distributed-systems-and-consistency.md), and [consensus](06-consensus-raft-and-paxos.md). All budgets and thresholds below are fictional design inputs. No commands here operate a live database or change actual routing.

## Define the contract per user operation

Recovery time objective, RTO, is the target time to restore an explicitly defined service level. Recovery point objective, RPO, is the tolerated loss of recently committed data, often expressed as a time window. State the failure class: losing an instance, a region, credentials, or all online replicas requires different recovery paths. A target is not evidence that the implementation meets it.

For this task service, suppose task creation requires an RPO of 30 seconds and an RTO of five minutes after a regional outage. Attachment downloads can tolerate ten minutes of disruption. An authorization revocation must not become silently undone by failover. Separate these promises rather than applying one database lag number to every dependency.

Backups, a minimal recovery deployment, a warm standby, and continuously serving regions have different operating costs and recovery work. A warm region still needs capacity, configuration, and dependency readiness. [AWS disaster-recovery strategy comparison](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html).

## Select a write-ownership model

| Model | Normal behavior | Outage behavior | Principal cost |
|---|---|---|---|
| Active-passive | Region A owns writes; B follows | Fence A, recover B, then route writes | Spare capacity and replication lag |
| Active-active application, single writer | Both regions run Django; writes reach A | Applications survive while writer changes | Remote write latency remains |
| Active-active by ownership | Each tenant or shard has one home region | Transfer ownership for affected shards | Routing and transfer protocol |
| Multi-writer for the same data | Both regions accept conflicting mutations | Reconcile or coordinate across regions | Conflict semantics or WAN coordination |

“Active-active” alone says little about consistency. Two Django deployments writing to one PostgreSQL primary are not a multi-writer database. Ordinary PostgreSQL physical standby replication does not provide two independently writable primaries with automatic merge semantics.

A practical intermediate design assigns each tenant a home region and includes an ownership generation in routing metadata. A request for tenant 42 reaches its current writer even when the closest frontend is elsewhere. Move the tenant through a controlled write pause, replication catch-up, ownership transfer, and route update. Route caches may be stale; the destination must reject an old generation. See [sharding](07-sharding-and-consistent-hashing.md).

If the same record truly accepts concurrent regional writes, define its merge policy first. Set membership may support a suitable CRDT; a unique username or the last available seat cannot be made safe merely by selecting the latest wall-clock timestamp. Conflicting edits may require a user-visible reconciliation workflow. See [logical clocks and reconciliation](12-logical-clocks-crdts-and-reconciliation.md).

## Replication determines the loss boundary

Asynchronous PostgreSQL replication can acknowledge a primary commit before a standby has received it. A promotion can therefore lose acknowledged transactions. Synchronous replication can wait for a selected standby's durability acknowledgement, with stronger visibility behavior available through `remote_apply`; remote acknowledgement adds network waiting and can constrain availability. Inspect the actual configuration and failure assumptions. [PostgreSQL standby and synchronous replication](https://www.postgresql.org/docs/current/warm-standby.html).

Worked example: A acknowledges synthetic transactions 1 through 10,000. B has durably received and replayed only through 9,940 when A becomes unreachable. Sixty acknowledged transactions are absent from B. If they occurred uniformly at 20 transactions per second, the example loss window is three seconds. Real traffic is not uniform: retain commit timestamps and business identifiers to investigate loss instead of converting every byte or transaction difference directly into seconds.

Distinguish WAL generated, sent, received, flushed, and replayed. Received-but-unflushed bytes may not survive another crash; replay lag affects query freshness. A last-known lag observation is stale after monitoring becomes disconnected. “Lag was two seconds a minute ago” cannot certify the current loss window. If the old primary is inaccessible, the exact missing tail may be unknowable until it is recovered.

Cross-region synchronous confirmation with a hypothetical 90 ms round trip can add roughly a round-trip wait to commits that need that acknowledgement; it does not impose a universal 11-transactions-per-second limit because independent transactions can overlap and commits can be grouped. Benchmark concurrent workload latency and throughput separately.

## Fence the former writer before admitting a new one

A health check demonstrates what its observer can reach. It does not demonstrate that the unreachable primary has stopped accepting writes from every client. Removing A from DNS still leaves cached routes, existing connections, jobs, and internal callers.

Fencing makes A unable to commit authoritative writes: for example, verified power isolation, an enforceable storage/network boundary, or a write path that validates a monotonically increasing ownership token at the authoritative resource. A lease checked only in application memory is insufficient when the process can pause and resume after expiration. The enforcement point must reject the stale holder. PostgreSQL explicitly identifies preventing the previous primary from returning as a primary as a failover requirement. [PostgreSQL failover](https://www.postgresql.org/docs/current/warm-standby-failover.html).

Use a quorum or an independently reliable authority for ownership decisions; a two-region pair cannot always distinguish a lost peer from a broken link. An isolated minority should lose write permission. A third voting failure domain can help a particular consensus topology, but does not remove data placement or durability requirements.

## Recover the entire request path

For a task mutation, inspect the complete chain:

- **Routing:** Global traffic steering must lead to a ready frontend. DNS TTLs, resolver caches, connection reuse, and client retry behavior affect convergence. Health checks can inform DNS responses, but do not move existing connections. [Route 53 health-check behavior](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html).
- **Authentication:** The destination needs signing or verification keys, session data or a compatible session strategy, identity-provider connectivity, and the current revocation policy. Test a previously revoked credential after recovery.
- **Authorization and tenancy:** Never infer tenant ownership solely from client-supplied region hints. Check server-side tenant scope and the current owner generation.
- **Objects:** A committed database row may reference an attachment not yet replicated. Track object availability and replication independently; return a defined temporary status rather than falsely claiming the file never existed.
- **Caches:** Start with bounded warmup and load shedding. Old permission entries or tenant-location entries need expiry or version invalidation. A cold cache can overload a correctly recovered database.
- **Queues and workers:** Fence schedulers and consumers as needed. Replay outbox events with stable event identifiers, deduplicate effects, and inspect queue age. A replicated database does not imply a replicated broker.
- **Secrets and operations:** Verify key access, certificates, images, schema compatibility, logging, alerts, quotas, and on-call access from the destination region.

Data-residency requirements also apply to replicas, backups, logs, analytics exports, and operator access. Document the approved placement boundary with the responsible policy owner; do not assume a home-region flag controls every copy.

## A measurable recovery runbook

For a **planned switchover**, stop admitting new mutations and pause writers, drain in-flight transactions, record the final primary WAL position, and wait until the candidate has durably replayed that position. Fence the old writer, transfer ownership, promote through the supported operational mechanism, validate, then restore admission. Pausing admission without draining background writers is incomplete.

For an **unplanned failover**, follow this sequence:

1. Record incident start and affected operations. Correlate application, database, and independent regional evidence; avoid promotion from a single failed probe.
2. Freeze automated changes that could conflict with the recovery. Verify the selected candidate, replication history, available WAL, and the last reliable observations. State any unknown loss window.
3. Obtain enforceable fencing of the previous writer. If fencing or ownership cannot be established, preserve the write stop and investigate rather than create two authorities.
4. Recover the eligible candidate and establish the next ownership generation. Record the timeline, recovery position, and the incident's accepted loss assessment.
5. Test synthetic task creation, read-after-write, authorization denial, attachment access, and one idempotent worker effect. Confirm only the new writer accepts writes.
6. Shift traffic gradually, renew database connections, and watch the dependency chain. Validate the promised service level during the validation and routing phases, using synthetic checks plus enough representative live requests to evaluate the defined restoration criteria. Do not declare recovery from a green health endpoint alone.
7. Declare restored service only at the promised level, then reconcile missing mutations and external effects using audit records and stable request identifiers. Keep the incident open for a separate five-minute post-restoration observation window; sample thresholds are error rate below 0.5%, p99 below 500 ms, no stale-owner acceptance, and queue age below two minutes. Define the request population and minimum sample size for those measurements.

An example critical-path budget is 30 seconds detection + 60 fencing + 45 recovery/promotion + 45 validation + 60 routing convergence = 240 seconds. This leaves one minute of a five-minute RTO for variance, assuming restoration criteria are satisfied within those phases. The five-minute observation window follows restoration and does not delay its recorded time. If the product requires five stable minutes before service counts as restored, include those minutes in the RTO budget instead. These phases may overlap in a real implementation; measure elapsed critical-path time instead of blindly summing overlapping tasks.

For **failback**, keep the recovered old primary isolated. Preserve evidence of divergent writes, rebuild it as a follower of the new authority, and confirm catch-up before a later planned switchover. PostgreSQL's `pg_rewind` can assist with some diverged clusters when its prerequisites hold; otherwise a new base backup may be required. It is not a business-level merge of transactions. [PostgreSQL pg_rewind](https://www.postgresql.org/docs/current/app-pgrewind.html).

## Local lab: rehearse loss accounting and fencing

This deterministic model uses lists and an integer authority generation. It models an ideal enforcement point, not PostgreSQL promotion, real networking, or a distributed consensus implementation.

1. Save the block as `region_drill.py` in a disposable local directory.
2. Run `python region_drill.py` using Python 3.10 or later.
3. Read the assertions as recovery invariants. The model deliberately exposes the unreplayed tail and rejects a stale owner.

```python
from dataclasses import dataclass


@dataclass
class Authority:
    owner: str
    epoch: int


def commit(authority, region, epoch, log, transaction_id):
    if (region, epoch) != (authority.owner, authority.epoch):
        raise PermissionError("stale writer fenced")
    if transaction_id not in log:
        log.append(transaction_id)


authority = Authority("A", 7)
a_log = []
for transaction_id in range(1, 101):
    commit(authority, "A", 7, a_log, transaction_id)
b_log = a_log[:96]  # Only this prefix survived in B.
missing = sorted(set(a_log) - set(b_log))
assert missing == [97, 98, 99, 100]

# In this model, changing this object is an atomic enforced fence.
# A real deployment must establish equivalent authority externally.
authority.owner, authority.epoch = "B", 8
try:
    commit(authority, "A", 7, a_log, 101)
except PermissionError:
    fenced = True
else:
    fenced = False
assert fenced

commit(authority, "B", 8, b_log, 101)
commit(authority, "B", 8, b_log, 101)  # Idempotent retry.
assert b_log.count(101) == 1
assert 97 not in b_log  # Promotion does not reconstruct lost commits.

phase_seconds = [30, 60, 45, 45, 60]
assert sum(phase_seconds) == 240
print("Missing acknowledged IDs:", missing)
print("Stale writer rejected:", fenced)
print("Recovery budget seconds:", sum(phase_seconds))
```

Expected output lists IDs 97 through 100, reports `True`, and prints 240 seconds. The known missing set is available only because the simulation retains A's log for inspection. A real incident may not have that evidence initially.

Extend the tabletop with a lost fencing service, a missing encryption key, an unreplicated attachment, and a delayed route cache. For each injection, identify the gate that must remain closed. Do not execute promotion or firewall commands against a live environment as part of this exercise.

## Exercises and completion checklist

Recalculate the recovery budget when DNS convergence takes four minutes. Decide which promise or routing architecture must change. Then design a tenant move while old workers retain stale routing metadata, and specify exactly where their writes are rejected.

- [ ] RPO and RTO name the operation and failure scenario.
- [ ] Every record has an explicit authoritative writer or merge rule.
- [ ] Fencing is enforced independently of a stale application's beliefs.
- [ ] Database, object, queue, identity, and cache recovery are exercised.
- [ ] Unknown replication loss is recorded instead of guessed away.
- [ ] Failback preserves evidence and rebuilds a follower safely.
- [ ] A rehearsal measures client-visible recovery and dependency readiness.
