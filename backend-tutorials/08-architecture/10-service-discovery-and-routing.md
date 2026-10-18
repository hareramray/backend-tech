# Service discovery, routing, and endpoint lifecycle

[Curriculum](../README.md) · [System-design map](README.md) · Prerequisites: DNS, [load balancing](08-load-balancing-and-traffic-management.md), [distributed consistency](03-distributed-systems-and-consistency.md)

## What discovery decides

Service discovery maps a logical service identity to candidate endpoints and associated metadata. A client wants `task-api`, while instances start, stop, change addresses, or move between zones. Discovery answers where the service might be reached; load balancing selects a destination; transport authentication verifies the peer; application authorization decides what the caller may do.

A registry entry is not evidence that the endpoint is currently reachable, healthy for this operation, or authorized to process a particular tenant. Those properties require separate checks and enforcement. This distinction matters when a stale address is reassigned to another workload.

```text
Deployment/controller -> service registry or DNS -> proxy/client endpoint cache
                                 ^                         |
                          health/lifecycle updates         v
                                                   authenticated service call
```

The registry/configuration distribution path is the control plane. The actual application request path is the data plane. A good design can often keep serving bounded existing traffic during a control-plane outage, but only under a deliberate stale-configuration policy.

## Compare discovery mechanisms

| Mechanism | How consumers learn endpoints | Strength | Important failure |
| --- | --- | --- | --- |
| Static configuration | Deployment injects addresses | Simple for a small fixed topology | Reconfiguration and stale addresses |
| DNS discovery | Resolver returns service records | Widely supported interface | Multiple cache layers and connection reuse |
| Registry polling | Client periodically queries catalogue | Rich metadata and explicit selection | Poll load, staleness, synchronized refresh |
| Registry watch/stream | Changes delivered to clients/proxies | Low-latency updates | Lost watches, ordering, reconnect snapshots |
| Stable virtual service address | Platform routes to changing endpoints | Simple client configuration | Routing/control-plane behavior still needs operation |

Use the platform's existing discovery mechanism before adding another registry. A single Django application and PostgreSQL do not automatically need a service mesh. Extra registries can disagree about health, identity, or membership and create another recovery dependency.

## DNS is cached naming, not instant failover

DNS TTL controls a record's intended cache lifetime, but clients have resolver caches, process caches, and open connection pools. Updating a DNS answer does not terminate connections already established to the old endpoint. Negative answers can also be cached, delaying visibility of a newly created name.

A/AAAA records identify addresses; SRV records can describe service targets and ports when the client supports that lookup mechanism. Do not assume an ordinary HTTP library automatically performs SRV-based discovery. Configure resolver timeout, retry, caching, and refresh behavior in the actual client/runtime.

Kubernetes provides DNS-based discovery for Services. A normal Service commonly resolves to a stable service address, while a headless Service exposes endpoint addresses and leaves more selection behavior to clients. Read the deployed platform's rules, including readiness publication and namespace-qualified names. [Kubernetes service discovery](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

Example logical name in a Kubernetes cluster, not a hostname available on your Windows host:

```text
task-api.application.svc.cluster.local
```

The cluster domain can be configured differently. Explicitly distinguish application namespace, service name, port, and protocol. Accidentally resolving a short name in the wrong namespace can route a request to an unintended service.

## Registration, leases, health, and generations

An instance registration should include identity, instance generation, addresses, protocol/port, zone, version/compatibility metadata, and lifecycle state. Require authenticated registration and constrained permissions so an arbitrary workload cannot replace a trusted service endpoint.

A lease must be renewed before expiry. The authority evaluating expiry needs its own time model and failure policy. A paused process may still be alive after its registration expires, so lease expiry is evidence for routing decisions, not a command that physically stops the process.

Use instance generations to reject delayed updates from an earlier incarnation that reused the same logical name. Otherwise a late deregistration from the old process might remove the replacement. A watch should have a version or revision and a resynchronization strategy; a dropped connection can mean missed events, not merely a temporary absence of new updates.

Consul's catalogue associates registered service endpoints with service/health metadata. Its DNS/API interfaces are examples of discovery mechanisms, not reasons to assume every registry read is instantly current. [Consul discovery overview](https://developer.hashicorp.com/consul/docs/discover).

## Lab: expiry and stale registration updates

Save as `discovery_lab.py` and run with Python. Time is injected as a number so the scenario is deterministic. This one-process model has no authentication, consensus, networking, or persistence; its generation numbers illustrate a rule rather than implement a production lease authority. `register()` represents a trusted controller authorizing a new incarnation. An instance may only renew its assigned generation; allowing stale instances to call `register()` freely would let them replace newer registrations and defeat the rule.

```python
from dataclasses import dataclass


@dataclass
class Endpoint:
    address: str
    generation: int
    expires_at: float
    ready: bool = True
    draining: bool = False


class Registry:
    def __init__(self):
        self.revision = 0
        self.entries = {}

    def register(self, name, address, now, ttl):
        if ttl <= 0:
            raise ValueError("positive TTL required")
        self.revision += 1
        entry = Endpoint(address, self.revision, now + ttl)
        self.entries[name] = entry
        return entry.generation

    def renew(self, name, generation, now, ttl):
        entry = self.entries.get(name)
        if (entry is None or entry.generation != generation
                or entry.expires_at <= now or ttl <= 0):
            return False
        entry.expires_at = now + ttl
        self.revision += 1
        return True

    def drain(self, name, generation):
        entry = self.entries.get(name)
        if entry is None or entry.generation != generation:
            return False
        entry.draining = True
        self.revision += 1
        return True

    def resolve(self, now):
        return sorted(entry.address for entry in self.entries.values()
                      if entry.expires_at > now and entry.ready and not entry.draining)


registry = Registry()
old = registry.register("worker-a", "127.0.0.1:3001", now=0, ttl=10)
assert registry.resolve(9) == ["127.0.0.1:3001"]
assert registry.resolve(10) == []
new = registry.register("worker-a", "127.0.0.1:3002", now=11, ttl=10)
assert new != old
assert not registry.renew("worker-a", old, now=12, ttl=100)
assert not registry.drain("worker-a", old)
assert registry.resolve(12) == ["127.0.0.1:3002"]
assert registry.drain("worker-a", new)
assert registry.resolve(12) == []
print("expiry, replacement generation, stale updates, and draining verified")
```

Notice that resolving no longer returns the drained instance, but this does not cancel work already executing there. Add a simulated client cache and show that it can keep selecting a stale endpoint until refresh. Then design an operation that must reject obsolete ownership at the data store using a fencing token; registry exclusion alone cannot enforce it.

## Watching and configuration distribution

For a watch-based client:

1. Obtain an authenticated snapshot with a configuration revision.
2. Install a validated snapshot atomically instead of exposing a half-updated endpoint list.
3. Apply changes only in the protocol's documented revision order.
4. Detect gaps, expired watch history, and parse errors.
5. Re-fetch a snapshot when continuity cannot be established.
6. Back off reconnects with jitter and a cap, while reporting configuration age.

State the maximum age at which an endpoint configuration can still be used. Read-only public content may tolerate older discovery data; routing financial writes to a former owner may require failing closed. Last-known-good routing is an availability mechanism, not blanket permission to ignore revoked service identity or data ownership.

The Kubernetes EndpointSlice API exposes separate readiness/serving/termination information. Consumers and proxies need to interpret that lifecycle according to their implementation; an endpoint's appearance in an object is not enough to conclude it should accept new traffic. [EndpointSlice conditions](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/).

## Failure and capacity worksheet

Assume 2,000 clients poll every five seconds. That creates about 400 registry requests/s before retries and simultaneous deploys. A watch moves cost toward long-lived connections and fanout: 2,000 connected clients still consume resources, and one large change can trigger 2,000 updates. Neither pattern is free.

| Failure | Required observation and policy |
| --- | --- |
| Registry unreachable | Continue bounded cached routing or fail closed by operation; expose configuration age |
| Endpoint removed but TCP connection reused | Drain/reconnect policy eventually prevents new work on the old connection |
| DNS negative answer cached | Retry with a bounded refresh policy; do not flood resolvers |
| Old process sends deregistration | Generation check preserves the replacement |
| Watch reconnect loses changes | New snapshot restores a coherent endpoint set |
| Unauthorized registration | Control-plane identity/ACL rejects it |
| Every endpoint unhealthy | Explicit fallback policy; no accidental unlimited retry loop |

## Local integration exercise and completion

In an owned test cluster or registry, register two harmless local services, observe discovery, remove one, and record detection time at the registry, proxy, and client. Keep a connection open before removal to observe reuse. Interrupt only the discovery path while leaving the data path available, then verify the declared stale-configuration policy. These steps require an explicitly installed registry/platform; the Python lab does not provision one.

- [ ] Discovery, balancing, peer authentication, and authorization have separate owners.
- [ ] Cached names and existing connections are both included in removal timing.
- [ ] Registration updates identify the correct process generation.
- [ ] Watch recovery handles missing history and atomic replacement.
- [ ] Registry failure and stale-route policies are tested per operation.
- [ ] Metrics include discovery latency, update failures, configuration age, and endpoint churn.

Continue with [multi-region routing](11-multi-region-and-failover.md) and [consensus](06-consensus-raft-and-paxos.md) for the control plane's own availability limits.
