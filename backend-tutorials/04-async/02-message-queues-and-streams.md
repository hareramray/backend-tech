# Message queues and streams

Prerequisites: [jobs and scheduling](01-jobs-and-scheduling.md), [transactions](../02-data/02-transactions-and-concurrency.md), and the [course map](../README.md). The lab is a dependency-free Python simulation; broker-specific operations are labeled pseudocode.

## Objectives

Choose between work queues and retained streams, distinguish publisher confirmation from consumer acknowledgement, and reason about duplicates, ordering, offsets and poison messages.

## Work distribution versus retained history

A work queue distributes tasks among consumers; successful acknowledgement normally retires a delivery from that queue. A stream retains an ordered log according to a retention policy, with consumers tracking their positions. Multiple consumer groups can independently replay a stream. Neither model automatically makes every downstream action exactly once.

Queue systems may route messages through exchanges, topics or subscriptions. Streams often partition records by a key. Use a stable business key when related records must share an ordering domain. Partitioning increases parallelism but usually limits ordering to the partition; one global order can constrain throughput.

Distinguish a command, which asks an owner to perform work, from an event, which reports a fact that already happened. A command should have a clear intended handler and outcome. Events can have multiple independent subscribers, each responsible for its own projection and failure handling.

## Acknowledgements cover different boundaries

Publisher confirmation means the broker accepted responsibility according to its configuration; consumer acknowledgement means a consumer reports processing completion. They are independent. A publisher can lose a confirmation and resend a message the broker already stored. A consumer can commit a database change and crash before acknowledging. Both create duplicates.

RabbitMQ documents these confirmation and acknowledgement boundaries, including automatic requeue of unacknowledged deliveries after a consumer connection closes. Acknowledgement must follow the durable effect the consumer promises to perform. [RabbitMQ acknowledgements and confirms](https://www.rabbitmq.com/docs/confirms)

With a stream, a committed offset records progress for a group. Commit before processing and a crash can skip work; commit after processing and a crash can repeat work. Batch processing must not advance beyond a failed record unless the chosen policy explicitly records and handles that gap.

Kafka's exactly-once facilities cover specific transaction and stream-processing scopes; external database or HTTP effects still require coordination or idempotency appropriate to their boundaries. Read the configured producer, transaction and consumer behavior rather than treating the phrase as an application-wide guarantee. [Kafka delivery and processing design](https://kafka.apache.org/42/design/design/)

## Lab: make the duplicate window visible

Save this dependency-free Python as `delivery_demo.py` in a temporary learning directory and run `python delivery_demo.py`. It simulates a durable effect and a lost acknowledgement:

~~~python
message = {"id": "event-1", "tenant": "tenant-a", "delta": 1}
durable_count = 0
acknowledged = False

def unsafe_consumer(crash_after_effect):
    global durable_count, acknowledged
    durable_count += message["delta"]
    if crash_after_effect:
        raise RuntimeError("process died before ACK")
    acknowledged = True

try:
    unsafe_consumer(True)
except RuntimeError as error:
    print(error)
if not acknowledged:
    unsafe_consumer(False)
print("Unsafe effect count:", durable_count)

durable_seen = set()
safe_count = 0

def simulated_atomic_transaction(crash_after_commit):
    global safe_count
    # This models one database transaction.
    # A Python set is not durable storage in a real worker.
    key = (message["tenant"], message["id"])
    if key not in durable_seen:
        durable_seen.add(key)
        safe_count += message["delta"]
    if crash_after_commit:
        raise RuntimeError("process died after commit")

try:
    simulated_atomic_transaction(True)
except RuntimeError:
    pass
simulated_atomic_transaction(False)
print("Deduplicated effect count:", safe_count)
~~~

Expected output ends with unsafe count 2 and deduplicated count 1. The important requirement is not the set: it is atomic persistence of the deduplication identity and the effect in a real datastore. See [idempotency](04-idempotency-and-deduplication.md) for the SQL pattern.

## Broker worker protocol

This is broker-neutral pseudocode requiring a configured client, durable broker topology and application database:

~~~text
receive message with bounded concurrency
validate envelope and supported schema version
begin local database transaction
  insert processed identity if absent
  if inserted:
    apply tenant-scoped business effect
commit
acknowledge delivery or safely advance processed offset
~~~

On transient failure, roll back and retry after a bounded delay. On malformed payload, unsupported schema or permanent policy failure, record the error and route to a dead-letter process. Avoid immediate infinite requeue: it consumes capacity without making progress.

Set prefetch or maximum in-flight work so the worker does not reserve more messages than it can process before acknowledgement deadlines. Separate CPU-heavy and I/O-heavy consumers where their capacities differ. Graceful shutdown should stop intake, finish bounded in-flight work and leave unfinished deliveries eligible for recovery.

## Ordering, parallelism and replay

Receiving records in order does not guarantee completing them in order when processing concurrently. If aggregate version 8 must follow 7, serialize work per aggregate or use conditional application of versions. A missing earlier event needs a defined wait, repair or refresh policy.

Rebalances can move partition ownership while old work is still running. Consumers need cancellation and ownership-aware progress commits. Poison messages can block ordered partitions; routing one aside may preserve throughput but changes ordering semantics, so record the decision and provide repair tooling.

Retention determines how far back a stream can replay. A consumer that falls behind retention may need a snapshot rebuild. Replay must avoid re-sending customer emails or reissuing payments; distinguish rebuildable projections from irreversible external effects. Encrypt or minimize sensitive event content and plan how retention interacts with deletion requirements.

Monitor oldest unprocessed age, consumer lag, acknowledgements, redeliveries, dead-letter growth, partition skew and processing duration. Lag in records and lag in time answer different questions.

## Failure tests, mistakes and exercises

Crash after database commit but before acknowledgement. Disconnect the publisher after sending but before confirmation. Send a poison message before valid messages. Slow one partition and verify other independent partitions progress. Replay the same event IDs and verify stable effects.

Common mistakes include acknowledging before durable work, committing offsets past failures, assuming message order equals effect order, storing giant binary payloads in the broker, and treating dead-letter storage as a complete recovery process.

Design a command queue for image resizing and an event stream for task-status analytics. Specify routing keys, retention, retry behavior, concurrency and replay rules.

- [ ] Delivery and processing guarantees are defined separately.
- [ ] Confirmation, commit and acknowledgement boundaries are clear.
- [ ] Duplicate delivery cannot duplicate a committed effect.
- [ ] Ordering scope and concurrency agree.
- [ ] Poison messages have an owner and recovery procedure.
- [ ] Retention and rebuild requirements are compatible.
