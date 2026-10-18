# Backend glossary

[Tutorial index](README.md) · Use the linked subject groups for practical explanations.

## Requests, runtimes, and APIs

| Term | Meaning |
| --- | --- |
| API | A defined interface through which one component uses another. |
| Backend | Server-side behavior that enforces rules and manages data/integrations. |
| Client | A caller such as a browser, mobile app, worker, or another service. |
| DNS | The naming system used to resolve service names into records. |
| TCP | Ordered reliable byte-stream transport; application framing remains necessary. |
| TLS | Transport encryption and endpoint authentication based on configured trust. |
| HTTP | Request/response protocol semantics for resources and representations. |
| Origin | Scheme, host, and port used by browser security rules. |
| Reverse proxy | A server that receives requests and forwards them to upstream applications. |
| Load balancer | A component that distributes traffic among eligible instances. |
| Middleware | Behavior surrounding request handling in a configured chain. |
| Serialization | Converting structured values to a transport/storage representation. |
| Validation | Checking that data conforms to accepted shape and meaning. |
| REST | Architectural constraints for distributed resource-oriented interaction. |
| RPC | Invoking an operation remotely through an explicit transport contract. |
| GraphQL | A schema and execution model for client-selected fields. |
| gRPC | A typed RPC framework supporting unary and streaming calls. |
| Protocol Buffers | A schema-based structured data encoding with numbered fields. |
| SSE | Server-to-client events carried over a persistent HTTP response. |
| WebSocket | A long-lived bidirectional message transport. |
| Webhook | An event-triggered HTTP request from one system to another. |
| OpenAPI | A machine-readable description of an HTTP API. |
| Safe method | A method whose requested semantics do not intend state change. |
| Idempotent operation | Repetition has the same intended effect as one execution. |
| ETag | A validator identifying a representation version. |
| Cursor | A token describing a pagination continuation position and scope. |
| Backpressure | A mechanism that limits upstream production when downstream is saturated. |
| Concurrency | Multiple operations making progress during overlapping periods. |
| Parallelism | Multiple operations executing at the same time on execution resources. |
| Deadline | The latest acceptable completion time for an operation. |

Read [foundations](01-foundations/01-client-server-and-request-lifecycle.md) and [API design](05-apis/01-rest-and-resource-design.md).

## Python and Django

| Term | Meaning |
| --- | --- |
| Virtual environment | An isolated Python package environment for a project. |
| Project | Django configuration and entry points assembling installed apps. |
| App | A Django package grouping a feature/domain or reusable behavior. |
| URLconf | The routing declarations mapping URL patterns to views. |
| View | Code translating a request into a response. |
| Model | A Python class describing persistent entities and ORM behavior. |
| ORM | Mapping between application objects and relational queries/data. |
| QuerySet | A composable, generally lazy representation of a database query. |
| Manager | The model-level entry point for query operations. |
| Migration | A versioned change to database schema or historical data state. |
| ModelForm | A form deriving selected validation and fields from a model. |
| Serializer | A DRF component validating input and producing representation data. |
| ViewSet | A DRF grouping of related API actions, often routed automatically. |
| Signal | An in-process notification mechanism; not a durable queue. |
| Management command | An explicit Django CLI operation with settings/model access. |
| WSGI | The conventional synchronous Python web-server interface. |
| ASGI | An asynchronous Python application/server interface. |
| `atomic()` | A Django transaction block with commit/rollback semantics. |
| `on_commit()` | A callback scheduled after successful transaction commit, not durable delivery. |
| `select_related` | ORM eager loading through joins for suitable single-valued relations. |
| `prefetch_related` | ORM eager loading through additional queries and Python association. |
| `F` expression | A database-side reference/expression over a model field. |
| `Q` object | A composable logical query predicate. |
| `AUTH_USER_MODEL` | The setting identifying the project's user model. |
| Static files | Application assets collected and served separately from user uploads. |
| Media files | User-supplied file content requiring independent access/lifecycle policy. |

Read [Django ORM](10-django/01-models-orm-and-querysets.md) and [DRF](10-django/03-django-rest-framework.md).

## Data and storage

| Term | Meaning |
| --- | --- |
| Primary key | A row's unique identifier within a table. |
| Foreign key | A constraint requiring a reference to an existing key. |
| Unique constraint | A database-enforced uniqueness rule. |
| Normalization | Structuring relations to reduce problematic duplication/dependencies. |
| Denormalization | Deliberately duplicating/deriving data for a measured access pattern. |
| Transaction | A group of operations with defined atomicity and isolation boundaries. |
| ACID | Atomicity, consistency, isolation, and durability properties. |
| MVCC | Managing concurrent visibility using row versions/snapshots. |
| Isolation level | Rules for which concurrent changes a transaction can observe. |
| Lost update | One change overwrites another without detecting the conflict. |
| Optimistic concurrency | Checking a version/precondition before accepting a change. |
| Pessimistic locking | Locking data while coordinating an operation. |
| Deadlock | A cycle of transactions waiting on each other's resources. |
| Index | An auxiliary structure supporting selected access patterns. |
| Query plan | The database's chosen operations for executing a query. |
| Selectivity | How much a condition narrows the rows considered. |
| N+1 queries | Repeated per-item queries following an initial collection query. |
| Connection pool | A bounded set of reusable database/client connections. |
| Replica | A maintained copy of another node's data. |
| Replication lag | Delay between a source change and a replica reflecting it. |
| Partitioning | Dividing a logical dataset into portions, often within one database. |
| Sharding | Distributing dataset ownership across independent storage nodes. |
| Cache-aside | Read cache first, fetch from source on miss, then populate cache. |
| TTL | Time after which cached or stored metadata expires under a policy. |
| Cache stampede | Many simultaneous misses triggering duplicate expensive work. |
| Materialized view | Stored query-derived results refreshed according to a policy. |
| CDC | Capturing database changes for downstream processing. |
| WAL | Write-ahead log used by databases for durability/recovery/replication. |
| PITR | Restoring a database to a selected point using backups and logs. |
| Tombstone | A deletion marker retained for synchronization or lifecycle purposes. |

Read [data modeling](02-data/01-relational-modeling-and-sql.md), [transactions](02-data/02-transactions-and-concurrency.md), and [caching](02-data/06-caching-and-redis.md).

## Security and identity

| Term | Meaning |
| --- | --- |
| Authentication | Establishing a principal's identity or credential validity. |
| Authorization | Deciding whether a principal may perform a specific operation. |
| Principal | The identity on whose authority an operation runs. |
| Tenant | An organizational/data-isolation scope within a service. |
| Least privilege | Granting only the authority required for an intended task. |
| RBAC | Access decisions based on assigned roles. |
| ABAC | Access decisions based on attributes of principal, resource, and context. |
| RLS | Database row-level security policies controlling row visibility/actions. |
| Session | Server-recognized authenticated state associated with a client credential. |
| OAuth | A framework for delegated authorization. |
| OIDC | An identity layer on OAuth for authentication workflows. |
| JWT | A structured token format; signatures do not imply encryption. |
| PKCE | A mechanism binding authorization-code redemption to its initiating client. |
| CSRF | Causing a browser to send unintended credential-bearing requests. |
| XSS | Untrusted content executing as script in a trusted web context. |
| CORS | Browser-enforced rules governing access to cross-origin responses. |
| SSRF | Inducing a server to make unintended outbound requests. |
| SQL injection | Altering query structure through incorrectly handled input. |
| HMAC | A keyed message-authentication construction for integrity/authenticity. |
| Salt | A per-value input used with password hashing to defeat shared precomputation. |
| Key rotation | Replacing cryptographic secrets with a controlled transition policy. |
| Replay attack | Reusing a previously valid message or credential in an unauthorized context. |
| Threat model | A structured analysis of assets, actors, boundaries, and abuse paths. |
| Audit trail | Controlled records of security/business-relevant actions and actors. |

Read [security](03-security/01-security-foundations-and-threat-modeling.md).

## Distributed systems and operations

| Term | Meaning |
| --- | --- |
| Queue | A mechanism for assigning work to consumers under delivery semantics. |
| Stream/log | An ordered retained sequence of records with consumer positions. |
| Acknowledgement | A signal that advances delivery/processing state under a protocol. |
| At-least-once | Delivery may repeat; handlers must tolerate duplicates. |
| At-most-once | Delivery avoids repeats but can lose work. |
| Outbox | Durable outgoing event intent committed with business state. |
| Inbox | Durable received-event tracking supporting deduplication and processing. |
| Dead-letter queue | A destination for messages requiring investigation after failure policy. |
| Retry budget | A cap on additional attempts/time/load caused by retries. |
| Circuit breaker | A mechanism temporarily limiting calls to a failing dependency. |
| Bulkhead | Isolation limiting one failure's consumption of shared resources. |
| Saga | A sequence of local transactions with explicit compensating behavior. |
| CQRS | Separating command/write concerns from query/read concerns. |
| Event sourcing | Recording authoritative state changes as an event sequence. |
| Eventual consistency | Replicas/projections converge under specified conditions and delay. |
| Linearizability | Operations behave as if atomically ordered consistently with real-time precedence. |
| Quorum | A required subset of participants for a read/write/decision protocol. |
| Consensus | Agreement on decisions among participants under a failure model. |
| Fencing token | A monotonically ordered authority marker rejecting stale owners. |
| SLI | A measured indicator of user-visible service behavior. |
| SLO | A target for an SLI over an explicit window/population. |
| SLA | A contractual service agreement with defined consequences. |
| Error budget | The tolerated unreliability implied by an SLO. |
| Cardinality | Number of distinct label/value combinations in a metric or dataset. |
| Trace/span | A correlated operation and its timed suboperations across components. |
| Liveness | Whether a process is functioning enough to remain running. |
| Readiness | Whether an instance should receive the intended workload. |
| RPO | Maximum acceptable data-loss window under a recovery objective. |
| RTO | Target time to restore service after an incident. |
| Canary | Releasing a change to a limited population before wider rollout. |
| Blue/green | Maintaining separate release environments and switching traffic. |
| Roll-forward | Recovering by deploying a corrective newer change. |
| IaC | Managing infrastructure through versioned declarative/programmatic definitions. |
| SBOM | An inventory of software components in an artifact. |

Read [architecture](08-architecture/03-distributed-systems-and-consistency.md), [observability](07-operations/05-observability-and-slos.md), and [recovery](07-operations/06-reliability-backups-and-disaster-recovery.md).

## Advanced system design

| Term | Meaning |
| --- | --- |
| Raft term | A logical election epoch used to recognize stale leaders and votes. |
| Log index | An entry's position in a replicated log; distinct from its term. |
| Commit index | The highest log position known committed under the replication protocol. |
| Paxos ballot | An ordered proposal identifier used to coordinate acceptors safely. |
| Joint consensus | A membership transition requiring the relevant majorities of both old and new configurations. |
| Consistent hashing | A placement family designed to limit key movement when membership changes. |
| Rendezvous hashing | Choosing an owner by comparing deterministic key-and-node scores. |
| Virtual node | One of several logical positions assigned to a physical node in a placement scheme. |
| Resharding | Changing data partitions or their ownership while preserving the required correctness and availability. |
| Hot key | A key receiving disproportionate traffic or contention. |
| L4 balancing | Distributing transport-level traffic, commonly by connection or flow. |
| L7 balancing | Routing using application-protocol information, commonly HTTP requests. |
| Draining | Stopping new assignments while existing work completes within a deadline. |
| Control plane | Components that decide and distribute configuration or ownership. |
| Data plane | Components that serve requests using that configuration. |
| Service discovery | Resolving a logical service into eligible endpoints and routing metadata. |
| Lease generation | A version distinguishing an authorized incarnation or claim from stale ones. |
| CDN | A distributed delivery layer that serves content nearer clients and can cache origin responses. |
| Cache key | The identity that determines which requests may reuse a stored representation. |
| Origin shield | An intermediate cache tier that reduces requests reaching the content origin. |
| Active/passive | A regional arrangement with an active serving role and a recovery region under a defined promotion process. |
| Active/active | Multiple regions serving concurrently; write ownership and conflict semantics still require explicit design. |
| Split brain | Competing nodes or regions acting as authoritative owners without the coordination the design requires. |
| Failback | Moving service ownership to a preferred or repaired region after recovery and reconciliation. |
| Happens-before | A causal relation induced by local order, message exchange, and transitivity. |
| Lamport clock | A logical counter whose order respects happens-before without proving its converse. |
| Vector clock | Per-actor counters representing causal context and distinguishing ordered from concurrent changes. |
| Hybrid logical clock | A timestamp combining physical-time information with logical ordering; not a universal proof of causality. |
| CRDT | A replicated data type whose operations or merge rules converge under stated assumptions. |
| G-counter | A grow-only replicated counter commonly merged by componentwise maxima. |
| Observed-remove set | A set tracking add identities so removal affects the adds it observed. |
| Last-write-wins | Resolving competing values by an agreed ordering, which may discard meaningful concurrent changes. |
| Anti-entropy | Comparing and repairing divergent replica state. |
| Merkle tree | A hierarchical hash structure used to narrow differences between datasets. |
| LSM tree | A storage organization that buffers changes and merges sorted persistent runs over time. |
| Memtable | An in-memory ordered write buffer in an LSM-style engine. |
| SSTable | An immutable sorted table file with engine-specific metadata and indexes. |
| Compaction | Rewriting storage runs to merge data, remove obsolete versions when safe, and manage read/write cost. |
| Bloom filter | A probabilistic membership structure with possible false positives and, under its insertion/query assumptions, no false negatives. |
| Write amplification | Physical write work divided by logical write work under a defined measurement scope. |
| Space amplification | Physical storage used relative to the live logical dataset under a defined scope. |
| Event time | The time an event represents in the source domain. |
| Processing time | The time an operator processes a record. |
| Watermark | A stream processor's progress estimate used to decide when event-time results may be emitted or finalized under a policy. |
| Tumbling window | A fixed-duration window that does not overlap neighboring windows. |
| Sliding window | A repeated window that may overlap others when its size exceeds its slide. |
| Session window | A grouping separated by a specified inactivity gap. |
| Checkpoint | A recoverable snapshot of processing state and source positions under the engine's consistency protocol. |
| Barrier alignment | Coordinating input progress around a checkpoint boundary in applicable streaming protocols. |
| UUIDv7 | A UUID format with a Unix-millisecond timestamp prefix and additional bits governed by the format and generator. |
| Worker ID | An allocator identity whose uniqueness and reuse policy help prevent collisions in time/worker/counter IDs. |
| Fanout on write | Materializing recipient-specific work or views when an event is written. |
| Fanout on read | Gathering relevant producers' data when a recipient requests a view. |
| Adaptive bitrate | Playback selecting among media renditions as measured conditions change. |
| Media manifest | A playback description identifying renditions, segments, and related media metadata. |
| Unknown outcome | A state where the caller cannot yet determine whether an attempted external effect occurred. |

Read the [advanced architecture map](08-architecture/README.md) and [worked designs](11-system-design-case-studies/README.md) for mechanisms, limits, and examples.
