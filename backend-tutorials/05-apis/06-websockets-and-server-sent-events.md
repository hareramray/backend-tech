# WebSockets, server-sent events, and realtime delivery

[Curriculum](../README.md) · Prerequisites: async Python, authorization, queues, and HTTP

## Choose the simplest delivery model that fits

Polling repeatedly requests current state. Long polling holds a request until something changes or its timeout expires. Server-sent events (SSE) use a persistent HTTP response for server-to-client text events. WebSockets support bidirectional messages over a long-lived connection.

| Requirement | Starting point |
| --- | --- |
| Infrequent status changes | Polling with validators and a sensible interval |
| Live notifications to a browser | SSE or a managed push service |
| Bidirectional collaboration | WebSocket protocol with explicit message schema |
| Durable history and replay | Durable event log plus a live delivery channel |

A socket is a transport, not a durable queue. Disconnecting a client can lose ephemeral notifications. Decide whether an event means “refresh your view” or is a durable business record.

## Understand SSE framing

An SSE response uses `text/event-stream`; blank lines delimit events. The `id` field can support resumption through `Last-Event-ID`, but the server must actually retain and replay the relevant history. Browser reconnection does not create a durable event log for you. [WHATWG server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html).

```text
id: 1042
event: task.updated
data: {"task_id":"22222222-2222-4222-8222-222222222222","version":2}

```

Do not concatenate unescaped attacker-controlled lines into the event format. Encode the data as JSON and control event names/IDs. Heartbeats can keep an idle stream visible to intermediaries but do not replace useful timeout and disconnect handling.

## Django deployment implications

For long-lived asynchronous streaming, use an ASGI deployment and async-compatible components. A synchronous streaming response under WSGI can occupy a worker for its entire lifetime. Streaming iterator type must match the serving interface; accidentally adapting a synchronous iterator can defeat the intended resource behavior. [Django StreamingHttpResponse](https://docs.djangoproject.com/en/5.2/ref/request-response/#streaminghttpresponse-objects).

Django Channels is a common integration for WebSocket consumers, routing, and channel layers. It is an optional dependency, not part of the capstone. An in-memory channel layer is per process; multiple server instances need a suitable shared layer. Channel-layer delivery is not a substitute for persistent event storage. [Channels documentation](https://channels.readthedocs.io/en/stable/).

## Authentication and authorization over time

Authenticate the connection and authorize each subscription. A client must not subscribe to `tenant:anything` simply by naming a room. For writes over WebSocket, validate every message and authorize its object and operation, just as for HTTP.

Cookie-authenticated WebSockets require origin validation to prevent unauthorized cross-site use. Tokens in query strings can leak through logs; choose a documented handshake/authentication mechanism and redact it. Decide what happens when a session expires or membership is revoked during an existing connection. A once-valid connection should not retain access indefinitely.

## Backpressure and reconnection

Bound per-connection output queues, message size, subscription count, and event rate. Slow consumers can otherwise accumulate unbounded memory. Options include dropping replaceable notifications, sending a resync marker, or disconnecting with a documented retry policy.

Attach monotonically meaningful sequence positions or version numbers where ordering matters. Clients should deduplicate and detect gaps. A snapshot-plus-stream protocol needs a boundary that avoids missing events between fetching the snapshot and subscribing. One design returns a snapshot watermark and replays events after that watermark.

## Practical design and implementation lab

1. Add an outbox event when a task version changes.
2. Create a read model of events with tenant, sequence, type, and allowed payload.
3. Implement an SSE endpoint under an ASGI server, or a Channels consumer, following the chosen integration's official setup.
4. Authorize the subscription from verified membership; do not trust a tenant supplied in the URL alone.
5. Send only a task ID/version notification first, then refetch through the existing authorized API.
6. Disconnect and reconnect using the last processed sequence; replay retained events.
7. Simulate a slow client and verify queue bounds.
8. Revoke membership while connected and verify the selected revocation policy.

This is an extension lab: the baseline capstone does not already install Channels, Redis, an ASGI server, or event retention. Record those prerequisites before running the experiment.

## Checklist and exercises

- [ ] Delivery guarantees, replay window, and gap behavior are explicit.
- [ ] Subscription and message authorization include tenant scope.
- [ ] Slow clients cannot grow memory without bound.
- [ ] Multi-instance fanout is tested.
- [ ] Deployment drains or reconnects long-lived clients deliberately.

Exercise: define the JSON protocol for a collaborative task board, including subscribe, event, acknowledgement, heartbeat, permission loss, and resync messages. Explain which events can be safely coalesced.
