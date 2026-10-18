# Load balancing and traffic management

[Curriculum](../README.md) · [System-design map](README.md) · Prerequisites: HTTP, networking, [capacity planning](05-system-design-and-capacity-planning.md)

## Outcome and design boundaries

Choose a routing layer, distribute work according to useful signals, and maintain capacity while instances fail or leave. A load balancer chooses an eligible destination; it does not make application state consistent, authorize a tenant, or guarantee a request's side effects occur once.

Start by identifying the unit being assigned: packets, transport connections, HTTP requests, HTTP/2 streams, or long-lived sessions. An even number of connections can hide very uneven request volume. One persistent connection might carry thousands of requests while another carries one.

```text
Client -> regional/global traffic entry -> regional load balancer
       -> application instances -> bounded database connections -> PostgreSQL
                              \-> bounded worker queue
```

Each arrow has its own deadlines, retry behavior, and resource limits. Adding application instances increases potential database concurrency; it does not create database capacity.

## Layer 4 versus layer 7

| Choice | Useful routing information | Benefits | Costs and limits |
| --- | --- | --- | --- |
| L4 connection routing | Transport addresses/ports and connection state | Protocol independence; TLS can remain end-to-end | Little application-level routing visibility; long connections can skew load |
| L7 HTTP routing | Host, path, method, headers, request/stream metadata | API routing, policy, request observability | Must understand the protocol; termination/proxying adds trust and resource boundaries |
| Client-side balancing | Discovered endpoints and local measurements | Avoids an extra balancing hop | Every client needs correct discovery, retry, and draining behavior |
| Proxy/sidecar balancing | Central configuration plus local endpoint observations | Consistent policy across clients | More processes, configuration distribution, and operational complexity |

TLS passthrough, termination, and re-encryption are separate decisions. A TLS-terminating proxy sees sensitive plaintext and needs restricted access and redacted logs. If traffic crosses another trust boundary after termination, authenticate and encrypt that hop too. Do not enable Django's forwarded-host or HTTPS settings without defining which proxy can set those headers.

Global routing selects a region or cluster. Local balancing selects an endpoint inside that cluster. Envoy distinguishes control-plane placement decisions from each proxy's local balancing and health observations. [Envoy load-balancing architecture](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview).

## Algorithms and what they actually observe

Round robin rotates through eligible endpoints. Weighted round robin gives larger instances more assignments, assuming weight correlates with useful capacity. It is simple and works well when request costs and servers are comparable.

Least connections chooses the endpoint with fewer active connections; least requests uses in-flight requests or streams when available. Neither sees the true cost of a query. A single expensive export can be worse than many cheap reads. Normalize measurements for server capacity and consider separate pools for workloads with different latency objectives.

Random selection is simple; power-of-two choices compares two sampled candidates instead of scanning every endpoint. Latency-sensitive algorithms can react to recent performance, but noisy measurements can shift all load at once and oscillate. Slow start gradually introduces a new or recovered endpoint so its cold caches do not receive a full burst immediately.

Hash-based routing preserves affinity for a chosen key. It can improve cache locality but creates hot-key and membership-change concerns. Use [consistent hashing](07-sharding-and-consistent-hashing.md) when limited remapping matters. Client IP is often a poor identity: NAT can group many people, and mobile clients change addresses. NGINX documents the supported balancing algorithms and their configuration-specific behavior; features differ between its open-source and commercial products. [NGINX HTTP load balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/).

## Lab: observe variable-cost requests

Save this standalone Python simulation as `balance_lab.py` and run `py balance_lab.py`. Each backend has one execution slot. Its reported outstanding count includes queued work. This deliberately simplified signal is more informative than a real proxy's connection count; it is not an implementation of NGINX or Envoy.

```python
from math import isclose


def simulate(policy):
    arrivals = [(0.0, 10.0), (0.1, 1.0), (0.2, 1.0), (1.3, 1.0)]
    finishes = [[], [], []]
    assignments, latencies = [], []
    for index, (arrival, duration) in enumerate(arrivals):
        for pending in finishes:
            pending[:] = [end for end in pending if end > arrival]
        if policy == "round_robin":
            selected = index % len(finishes)
        else:
            selected = min(range(len(finishes)), key=lambda i: (len(finishes[i]), i))
        pending = finishes[selected]
        start = max(arrival, pending[-1] if pending else arrival)
        end = start + duration
        pending.append(end)
        assignments.append(selected)
        latencies.append(end - arrival)
    return assignments, latencies


rr_nodes, rr_times = simulate("round_robin")
least_nodes, least_times = simulate("least_outstanding")
assert rr_nodes == [0, 1, 2, 0]
assert least_nodes == [0, 1, 2, 1]
assert round(rr_times[-1], 1) == 9.7
assert isclose(least_times[-1], 1.0)
print("round robin:", rr_nodes, "last latency:", round(rr_times[-1], 1))
print("least outstanding:", least_nodes, "last latency:", round(least_times[-1], 1))
```

The fourth request waits behind a long request under round robin. This workload demonstrates why the balancing unit matters, not that one algorithm wins for every workload. Repeat with stale load measurements, different server speeds, and simultaneous arrivals. Observe that equal outstanding counts can still represent unequal remaining work.

## Health, ejection, and overload are different

An active probe periodically checks an endpoint. Passive detection observes failures from real traffic. A process can be alive but not ready; a dependency can be slow without the process being broken. A probe that performs a costly database scan can become an outage amplifier.

Use thresholds and recovery hysteresis to avoid removing and re-adding instances on every transient failure. Distinguish a client-caused 400 from a connection failure when deciding whether an upstream is unhealthy. Correlated dependency failures should not necessarily cause every application instance to restart simultaneously.

Specify what happens when all endpoints are unhealthy. Some systems fail closed, while others have configurable fallback or panic behavior. Check the deployed product's exact policy. Routing to a known unhealthy endpoint may be preferable for one read workload and unacceptable for a mutation whose invariants cannot be checked.

At overload, bound queued requests, concurrent requests, connection count, and body size. Reject excess work early according to the documented contract instead of accepting it into an unbounded queue. A queue with 5,000 items and a 100/s service rate already implies roughly 50 seconds to drain without new arrivals. Most interactive requests should not wait that long.

## Draining and safe deployment

1. Mark an instance unavailable for new assignments.
2. Allow the discovery and proxy configuration change to propagate.
3. Stop accepting new work at the server boundary.
4. Complete bounded in-flight requests within a grace period.
5. Reconnect or close long-lived streams according to protocol policy.
6. Terminate after the deadline and reconcile interrupted durable work.

A long-lived WebSocket is not migrated to another process merely because routing changes. HTTP/2 connections can continue creating streams unless the protocol/server communicates shutdown correctly. Account for connection reuse and request deadlines when choosing the drain period.

## Local reverse-proxy configuration exercise

The following is an **HTTP-context fragment** for an existing local NGINX installation. Start two local copies of your practice application on ports 3001 and 3002 first; add an instance identifier to a harmless test response. This is not a complete production configuration.

```nginx
upstream tutorial_apps {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    keepalive 16;
}
server {
    listen 127.0.0.1:8080;
    location / {
        proxy_pass http://tutorial_apps;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_connect_timeout 1s;
        proxy_read_timeout 5s;
        proxy_send_timeout 5s;
        proxy_next_upstream off;
    }
}
```

Validate the full configuration with the installed server's configuration-check command. Repeat read requests, stop one local backend, and record errors and recovery. This fragment disables automatic upstream retries; enabling retries later requires an operation-aware policy. Read/send timeouts here are not a universal end-to-end deadline.

## Capacity, verification, and completion

If 12 instances each safely handle 80 requests/s for the tested mix, 960/s is a nominal aggregate. With one of three equally sized zones unavailable, only eight remain: 640/s before extra headroom and dependency constraints. A design expected to survive a zone loss must budget around the surviving capacity.

Track per-endpoint request rate, in-flight work, queue time, upstream connection errors, latency distributions, ejections, and retries. Compare client-observed errors with proxy and application outcomes: a proxy timeout can occur after an application commit.

- [ ] I can explain the routing unit and L4/L7 boundary.
- [ ] Sticky state has a restart/reassignment policy.
- [ ] Health checks do not confuse overload with process death.
- [ ] Retry ownership prevents multiplied attempts across layers.
- [ ] Surviving instances can meet the intended failure capacity.
- [ ] Draining is verified with ordinary requests and long-lived connections.

Exercise: design separate pools for interactive API calls and 60-second exports. Specify queue bounds, deadlines, worker capacity, and what clients observe when either pool saturates. Continue with [discovery](10-service-discovery-and-routing.md) and [regional failover](11-multi-region-and-failover.md).
