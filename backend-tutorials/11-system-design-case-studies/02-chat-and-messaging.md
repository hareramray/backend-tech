# Case study: durable chat with reconnect and replay

## Product contract and boundaries

Design private direct messages and group conversations for a fictional collaboration product. Members send text, attach files, page through history, and see optional delivery/read indicators. A user may connect several devices. Groups have at most 500 members initially. Voice, live video, end-to-end encryption, and public broadcast channels are separate projects. This independent design uses Django, an ASGI/WebSocket gateway, and PostgreSQL; it is not a feature already present in the task-service capstone.

Prerequisites are [distributed consistency](../08-architecture/03-distributed-systems-and-consistency.md) and [traffic management](../08-architecture/08-load-balancing-and-traffic-management.md). The central promise is that an acknowledged message is durably committed according to the database deployment's stated failure model. The product distinguishes **accepted** by the service, **delivered** to at least one authorized recipient device, and **read** as explicitly reported by a client. Network transmission alone proves neither delivery nor reading.

Fictional targets: p99 message acceptance below 300 ms within the home region; 99% of accepted messages offered to online recipients within one second; reconnect history available within two seconds for up to 1,000 missed messages. No global ordering between unrelated conversations is promised. Read and dispatch authorization decisions made after membership revocation commits must deny that member. Previously authorized bytes in flight may still arrive, and downloaded content cannot be recalled.

## Capacity estimates

Assume 250,000 daily active users sending 40 messages each, an eightfold message peak, 100,000 simultaneous connections, and 1 KiB of logical message metadata/text.

```text
Messages/day     = 250,000 * 40 = 10,000,000
Average ingress  = 10,000,000 / 86,400 = 115.7 messages/s
Peak ingress     = 115.7 * 8 = 925.9 messages/s
Logical storage  = 10,000,000 * 1 KiB = 9.54 GiB/day
90-day retention = 9.54 * 90 = 858.3 GiB logical
```

At an assumed 2.5 receiving devices per message, peak delivery work is approximately 2,315 device deliveries/s. That average hides a 500-member conversation with several devices per member. Bound each conversation's send rate and account for fanout independently of ingress.

If one connection consumes 32 KiB of measured gateway memory, 100,000 connections consume about 3.05 GiB before process overhead, TLS buffers, and queued payloads. A 30-second heartbeat interval creates approximately 3,333 heartbeat exchanges/s. Connection density is therefore a memory, socket, network, and failure-blast-radius question, not only CPU requests per second.

## Architecture and durability boundary

```text
Phones/browsers --> load balancer --> ASGI gateways
      |                                  |
      +--> Django history/send API ------+--> message transaction
                                              |
                                         PostgreSQL + outbox
                                              |
                                         delivery workers
                                              |
                               ephemeral connection routing layer
                                              |
                                         ASGI gateways

Attachment client --> private object storage --> scan/validation workers
```

The message table is authoritative. The routing layer helps find live connections and can lose its contents without losing message history. Django Channels offers channel layers for communication between application instances; treat such a layer as live dispatch infrastructure, not the sole durable message archive. [Channels channel layers](https://channels.readthedocs.io/en/stable/topics/channel_layers.html).

An ordinary WebSocket connection provides framing and bidirectional communication, not application-level durable delivery across disconnects. Define message IDs, replay, receipts, and deduplication above the transport. [WebSocket protocol](https://www.rfc-editor.org/rfc/rfc6455.html).

## Interfaces and schema

```json
{
  "type": "message.send",
  "conversation_id": "c42",
  "client_message_id": "device-uuid:operation-uuid",
  "text": "The deployment review starts at 14:00.",
  "attachment_ids": []
}
```

The reply is `message.accepted` with a server message ID, conversation sequence, and server timestamp. HTTP `POST /v1/conversations/{id}/messages` uses the same operation identifier and transaction path. `GET .../messages?after_seq=120&limit=100` provides ascending replay; `before_seq` provides bounded history. A forbidden or nonexistent conversation returns the same externally documented response shape to reduce membership probing.

These are logical table definitions, not a complete migration:

```text
conversation(tenant_id, id, next_seq, membership_version)
  PRIMARY KEY (tenant_id, id)
membership(tenant_id, conversation_id, user_id, role, active, joined_seq)
  UNIQUE (tenant_id, conversation_id, user_id)
message(tenant_id, conversation_id, seq, id, sender_id,
        client_message_id, body, created_at, edited_version, deleted)
  PRIMARY KEY (tenant_id, conversation_id, seq)
  UNIQUE (tenant_id, conversation_id, sender_id, client_message_id)
receipt(tenant_id, conversation_id, user_id, delivered_seq, read_seq)
  PRIMARY KEY (tenant_id, conversation_id, user_id)
outbox(event_id, tenant_id, conversation_id, seq, event_type, published_at)
```

Use composite foreign keys to prevent rows from connecting objects across tenants. Membership checks use tenant and conversation together. A message ID is convenient for references, while `seq` defines order. The sequence is assigned within the conversation transaction; UUID timestamp ordering or client clocks do not define the canonical message order.

## Send, fanout, and replay flows

1. Validate authentication, tenant membership, payload size, rate limits, and attachment ownership. Resolve the sender from authentication rather than trusting a payload field.
2. Begin a short transaction and lock the conversation row. Membership changes acquire the same lock, giving send and revoke operations a documented ordering.
3. Look for an existing sender/client-message-ID pair. If found, compare the canonical request hash and return its result; conflicting reuse is rejected.
4. Verify active membership, allocate the current `next_seq`, increment the counter, and insert the message plus its outbox event atomically. Commit before returning `accepted`.
5. Delivery workers process the outbox, resolve currently eligible recipients, and offer events to their connected devices. Gateways obtain a trustworthy membership decision before sending content; stale membership caches cannot authorize a new dispatch after revocation.
6. A recipient acknowledges its highest contiguous delivered sequence; read receipts advance only after an explicit client signal. Store monotonic maxima, never move a receipt backward because an older device reports later.

The conversation lock serializes a single busy room. That is intentional for a simple ordering contract, but it becomes the room's throughput ceiling. Store large attachments outside the transaction and never call external services while holding the lock. PostgreSQL row locks block conflicting row changes until transaction completion. [PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html).

For reconnect, authorize the session, establish live subscription, capture a durable high-water sequence, and replay from the client's last contiguous position through that high-water mark while buffering live events. Then merge buffered events by sequence and deduplicate. The live channel is a hint: periodic high-water comparison and gap replay from PostgreSQL repair missed dispatches. If the replay is too large, return a paginated synchronization mode instead of buffering unlimited data.

Deleted messages retain sequence tombstones so a client does not interpret deliberate deletion as a missing event. Membership start boundaries can exclude earlier history; explain that filtered ranges are intentional rather than promising every integer corresponds to a visible message. Never let a client acknowledge an arbitrary future sequence: validate the receipt against authorized offered history.

## Failure semantics and operational choices

A connection can drop after commit but before acceptance reaches the sender. The sender retries the same client-message ID, getting the committed result. If it invents a new ID, the server cannot reliably distinguish a retry from a deliberately repeated message. If a worker crashes after sending but before marking its event complete, recipients can see a duplicate event; client and server deduplication make it one visible message.

Per-conversation order does not imply delivery order across gateways. A device receiving sequence 123 before 122 buffers briefly, requests the gap, and renders in canonical order. Buffer limits and timeout behavior are part of the protocol. If storage is unavailable, reject or explicitly mark sends as local drafts; do not acknowledge durable acceptance based only on in-memory enqueueing.

Presence and typing indicators are ephemeral. Store short-lived heartbeats by device, aggregate online state, and tolerate temporary errors. A device going offline without a clean close eventually expires. Presence never serves as proof of authorization or as a reason to skip durable history. Push notifications for offline users contain minimal information and point back to authorized message retrieval.

Attachments use private object IDs, per-conversation access checks, content-size quotas, and a quarantine/scan state before becoming downloadable. Do not accept an arbitrary storage key from a sender. Deleting a message schedules removal according to retention policy, while a signed attachment URL's remaining lifetime limits immediate revocation. Avoid including secrets or message bodies in traces.

## Scaling, security, and recovery

Start with one message store and several stateless gateways. Add connection routing and worker partitions when measured load requires them. Partition workers by conversation ID to reduce reordering, while retaining sequence-based correctness. Later, shard storage by tenant/conversation so a conversation's messages and ordering authority stay together. Very large broadcast rooms need a separate fanout and history design rather than one database row per recipient per message.

Long-lived connections must handle authentication expiry and logout. Validate WebSocket origins for cookie-authenticated browser connections, authorize every message operation, and enforce frame/message size limits. An authenticated socket is not blanket access to every conversation. Channels documents origin restrictions because browser WebSocket requests can carry existing cookies. [Channels security](https://channels.readthedocs.io/en/stable/topics/security.html).

Track acceptance latency, commit failures, outbox age, delivery delay, replay size, sequence gaps, active connections, slow-consumer drops, and per-conversation hotspots. A bounded gateway queue protects other users; disconnect a persistently slow device with a resumable position rather than retaining unlimited messages. Separate successful acceptance from successful online delivery in dashboards.

Restore message data, membership history, sequence counters, receipts, and outbox records consistently. Rebuild ephemeral routing from reconnects. If a failover can lose acknowledged transactions, disclose that RPO; synchronous durability choices require measurement and failure testing. A recovery audit should find duplicate operation keys, sequence inconsistencies, orphan attachments, and messages whose outbox events were never dispatched.

## Tabletop lab: reconnect under failure

Create a worksheet with conversation C, active users A/B, `next_seq=10`, and B's contiguous receipt at 9. Use the schedules below independently. Record server rows, outbox state, client state, and the next authorized action.

| Schedule | Expected result |
|---|---|
| A sends operation X; sequence 10 commits; response is lost; X retries | One stored message, one canonical sequence, original acceptance returned |
| Worker sends 10 then crashes before marking published | Redelivery is possible; B displays sequence 10 once |
| B receives 11 before 10 | B detects a gap, replays 10, then advances its contiguous receipt |
| Subscribe, capture high-water 12, then message 13 commits during replay | Replay covers through 12; buffered/live delivery or later gap repair retrieves 13 |
| Admin revokes B, then A sends another message | B receives no newly authorized content and new history requests fail |
| Gateway loses routing data | Durable messages remain; reconnect rebuilds routing and replays gaps |
| Device reports read sequence 1,000 when only 13 exists | Invalid advance is rejected; other receipts remain intact |

Repeat the worksheet with a slow device and a queue limit of three events. Explain exactly how disconnection and replay avoid unbounded memory. Then model a 500-member group sending 20 messages/s and calculate at least 10,000 recipient deliveries/s before multiple devices.

## Exercises and completion checklist

Implement operation deduplication and sequence allocation in a disposable Django/PostgreSQL project. Test two simultaneous sends, an acceptance timeout, and send-versus-revoke ordering. For a design exercise, compare server-readable search with end-to-end encryption: key distribution, device recovery, moderation, and search requirements all change.

- [ ] Acceptance, delivery, and read receipts have different explicit meanings.
- [ ] Replay works even if the live transport loses an event.
- [ ] Ordering, idempotency, and membership changes share a coherent authority.
- [ ] Slow devices, offline users, and attachments have bounded resource behavior.
- [ ] Recovery restores message and authorization state together.

Continue with [news feeds](03-news-feed.md) or return to the [case-study index](README.md).
