# Stream processing, windows, watermarks, and checkpoint recovery

## Purpose and prerequisites

A queue worker answers “perform this job.” A stateful stream processor can answer “what is each tenant's task-creation rate during the last five minutes?” continuously, while events arrive out of order, partitions pause, and workers restart. Correct results require explicit time, state, and replay semantics.

Read [distributed consistency](03-distributed-systems-and-consistency.md), [storage engines](13-storage-engines-lsm-trees-and-bloom-filters.md), and [capacity planning](05-system-design-and-capacity-planning.md). Examples use a Django service that publishes committed task events through an outbox. The Python lab is a deterministic model, not a Flink implementation or a production event pipeline.

## Define the event and the clock

Use a stable event identifier, schema version, tenant identifier, event type, occurrence timestamp, and payload. A logical event identifier and a broker offset solve different problems: the identifier recognizes a business event republished at a different offset, while the offset locates a record within a partition for replay.

**Event time** is the timestamp assigned to the event's occurrence. **Processing time** is the worker's current wall-clock time when it handles the record. **Ingestion time** describes arrival at an ingestion boundary. A device can upload an hour-old event now; replay can process last month's events today. Decide which clock the question actually requires.

For task-created analytics, assign occurrence time on the trusted server handling the transaction, or define how client clocks are validated. Preserve the original timestamp during replay. Also retain arrival time for lag analysis. Timestamp validation must handle future dates, missing zones, and impossible values; a single far-future event must not blindly advance progress and discard valid data.

## Choose window membership before implementation

Windows divide an unbounded stream into finite aggregation scopes. A key, such as tenant ID, defines which records share aggregation state. Tumbling and sliding time windows have deterministic boundaries; session windows group nearby activity and may merge when later records connect sessions. [Flink window concepts](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/operators/windows/).

| Window | Rule | Worked event-time example |
|---|---|---|
| Tumbling | Non-overlapping fixed length | With ten-second windows, time 17 belongs to `[10, 20)` |
| Sliding | Fixed length, started every slide interval | With length ten and slide five, time 17 belongs to `[10, 20)` and `[15, 25)` |
| Session | Activity separated by a configured inactivity gap | With gap five, times 1, 4, and 12 initially form two sessions |

Use half-open intervals consistently: an event at 20 is excluded from `[10, 20)` and included in `[20, 30)`. A late event at time 8 can bridge sessions formed by 1, 4, and 12 when every adjacent activity gap is at most five. Merging changes both aggregate value and session identity, so a sink may need to retract earlier outputs.

Sliding windows can multiply state updates. A 60-second window sliding every five seconds assigns an ordinary interior event to 12 windows. Incremental pane aggregation can reuse work for associative aggregates, but median, distinct counts, and custom business logic need their own algorithms and error guarantees.

## Watermarks express progress assumptions

A watermark estimates event-time progress; it is not proof that every older event has arrived. One policy computes a partition's watermark from its maximum observed event time minus an out-of-order allowance. When an operator consumes several active inputs, progress is constrained by the slowest input watermark. Idle-input handling can remove silent inputs from this minimum. [Flink watermark generation and idleness](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/event-time/generating_watermarks/).

For this tutorial's simplified convention, watermark `W` means events with timestamps below `W` are no longer expected. A window ending at `E` first emits when `W >= E`. Engine APIs can use different boundary details, such as the largest timestamp in a window; check the actual framework rather than copying equality tests blindly.

Suppose partition A reports 12 and B reports 3. The operator watermark is three, so `[0, 10)` remains open. When B reaches ten, the operator can emit that window even if A has already passed it. A fast partition cannot establish completeness for the slow one.

If B is declared idle and excluded, A may advance progress. When B resumes with old records, the global watermark does not move backward; those records may now be late. Thus an idle timeout changes the correctness/latency tradeoff. Choose it using observed silence patterns, consumer lag, and the consequence of missing late data. A partition with a backlog is not automatically idle just because a worker is slow.

## Separate first results from finality

An allowed-lateness interval retains window state after its first emission so late events can revise the result. Define whether downstream receives updated absolute values, additive deltas, or retractions. Late records beyond retention can go to a side output for reconciliation, be rejected with metrics, or trigger a separate correction job.

Worked example: `[0, 10)` contains values three and seven, so it first emits ten at watermark ten. A value-five event at time six arrives while the watermark is ten. With five seconds of allowed lateness, accept it and revise the result to 15. When the watermark reaches 15, this tutorial considers the window expired; a later time-eight event goes to the late-record output. “Final” means final under that retention policy, not complete for arbitrarily delayed real-world input.

Keep output identity stable, for example `(tenant_id, window_start, window_end, calculation_version)`, and include a revision or other ordering contract. An absolute upsert to 15 can be replayed safely when its identity and revision are stable. Repeating `total = total + 5` cannot be treated as idempotent without an atomic event-deduplication rule.

## Checkpoints capture a recoverable computation

A checkpoint must associate source positions with the operator state produced from those positions. For multiple inputs, a distributed snapshot protocol establishes a consistent cut; checkpoint barriers and alignment are one way to coordinate this. Restore state and replay source records after the recorded positions. [Flink fault-tolerance concepts](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/).

Include keyed aggregates, window timers, necessary deduplication state, and compatible serialization. A source that cannot replay retained data cannot meet the same recovery promise as a durable partitioned log. Retention must cover the longest plausible outage plus restore and catch-up time. Distinguish checkpoints used for automatic recovery from intentionally managed snapshots used for upgrades or migration; verify the framework's lifecycle rules.

If, at failure, the latest completed checkpoint's captured source positions are 30 seconds behind current ingestion at 2,000 events/s, roughly 60,000 events need replay under steady assumptions. Measure that distance from source offsets: checkpoint alignment, snapshot duration, and existing source lag can make it much larger than the time since checkpoint completion. A 90-second restore adds 180,000 new arrivals. At a processing capacity of 5,000 events/s with 2,000/s continuing ingress, the 240,000-event backlog drains in approximately `240,000 / (5,000 - 2,000) = 80` seconds. A processor only just matching ingress never catches up.

## Exactly-once is scoped to an effect

Exactly-once state recovery does not mean each record's code executes once. Execution can repeat after failure while the restored state behaves as though each input contributed once. End-to-end effects additionally depend on the source and sink contracts; supported transactional sinks or correctly idempotent operations can cooperate with checkpointing. [Flink connector fault-tolerance guarantees](https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/datastream/guarantees/).

Consider a worker that updates PostgreSQL and crashes before recording its broker position. Replay repeats the update. Committing the position first has the opposite gap: a crash can omit the update. A robust PostgreSQL consumer can insert a unique processed-event marker and apply its business change in one database transaction, then acknowledge the source. The replayed marker prevents another change. Retain markers long enough for the supported replay horizon.

For aggregate sinks, an idempotent upsert needs monotonic versions or another rule preventing an older replay from overwriting a newer value. A distributed transaction protocol is another option for compatible sinks, with its own coordination and timeout costs. An email or arbitrary HTTP call does not join a database transaction automatically; use a downstream idempotency contract or an outbox plus explicit duplicate handling.

## Local lab: late events and a crash after sink output

1. Save this block as `stream_lab.py` and run `python stream_lab.py` with Python 3.10 or later.
2. Observe that the first result is ten, a late event revises it to 15, and a duplicate contributes nothing.
3. The simulated crash retains the sink but restores an earlier processor snapshot. Replaying the late event attempts the same revision again without another visible sink change.
4. A second late event arrives after expiration and is routed separately.

```python
from copy import deepcopy


class Sink:
    def __init__(self):
        self.rows = {}
        self.visible_changes = 0

    def upsert(self, start, revision, total):
        previous = self.rows.get(start)
        if previous is None or revision > previous[0]:
            self.rows[start] = (revision, total)
            self.visible_changes += 1
        elif revision == previous[0]:
            assert total == previous[1], "same revision must mean same result"


class Processor:
    def __init__(self):
        self.partition_watermarks = {"A": float("-inf"), "B": float("-inf")}
        self.watermark = float("-inf")
        self.windows = {}
        self.seen = set()
        self.too_late = []
        self.next_offset = 0  # Offset in this synthetic ordered command log.

    def apply(self, command, sink):
        self.next_offset += 1
        if command[0] == "watermark":
            _, partition, value = command
            assert value >= self.partition_watermarks[partition]
            self.partition_watermarks[partition] = value
            self.watermark = max(
                self.watermark, min(self.partition_watermarks.values())
            )
            for start, (total, revision) in list(self.windows.items()):
                if self.watermark >= start + 10:
                    sink.upsert(start, revision, total)
                if self.watermark >= start + 15:
                    del self.windows[start]
            return

        _, partition, event_id, timestamp, value = command
        assert partition in self.partition_watermarks
        if event_id in self.seen:
            return
        self.seen.add(event_id)
        start = timestamp // 10 * 10
        if self.watermark >= start + 15:
            self.too_late.append(event_id)
            return
        total, revision = self.windows.get(start, (0, 0))
        self.windows[start] = (total + value, revision + 1)
        if self.watermark >= start + 10:
            sink.upsert(start, revision + 1, total + value)


commands = [
    ("event", "A", "a", 2, 3),
    ("event", "B", "b", 4, 7),
    ("watermark", "A", 12),
    ("watermark", "B", 3),
    ("watermark", "B", 10),
    ("event", "A", "c", 6, 5),
    ("event", "A", "c", 6, 5),  # Same logical event again.
    ("watermark", "A", 20),
    ("watermark", "B", 15),
    ("event", "B", "d", 8, 11),
]
processor, sink = Processor(), Sink()
for command in commands[:4]:
    processor.apply(command, sink)
assert processor.watermark == 3 and sink.rows == {}
processor.apply(commands[4], sink)
assert sink.rows == {0: (2, 10)}
checkpoint = deepcopy(processor)

processor.apply(commands[5], sink)
assert sink.rows == {0: (3, 15)}
processor = deepcopy(checkpoint)  # Crash: sink persists; processor rolls back.
for command in commands[processor.next_offset:]:
    processor.apply(command, sink)

assert sink.rows == {0: (3, 15)}
assert sink.visible_changes == 2
assert processor.too_late == ["d"]
assert processor.windows == {}
print("window [0, 10):", sink.rows[0])
print("visible sink changes:", sink.visible_changes)
print("too late:", processor.too_late)
```

Expected output is revision/total `(3, 15)`, two visible sink changes, and `['d']`. The fixture uses a single deterministic command order, ideal atomic sink operations, and in-memory snapshots. The revision counter is valid only under those assumptions; arbitrary reordering during replay requires a stronger output-version protocol. The unbounded `seen` set deliberately omits production retention. No disk durability, source retention, transactional checkpoint coordinator, or Flink connector is implemented here.

## State growth and operating limits

At 100,000 active tenant-window keys with 200 bytes of aggregate state each, the bare state is approximately 20 MB before serialization, indexes, timers, and backend overhead. Keeping 12 overlapping windows can raise the naive aggregate portion to 240 MB. Deduplicating 2,000 events/s for one hour requires 7.2 million identifiers; even 32 bytes each is 230.4 MB before data-structure overhead.

Observe source lag, event-time lag, watermark skew, late-event counts, state size, checkpoint age and duration, restore time, sink commit latency, and backpressure. A slow sink can delay upstream processing and checkpoints. Increasing parallelism cannot split a single heavily used key unless the aggregate supports a safe partial-aggregation strategy.

## Exercises and completion checklist

Add an idle partition to the model and describe the consequences when it resumes with old events. Then replace sums with session counts and design sink retractions when a bridging event merges two already-emitted sessions.

- [ ] Event timestamps, window boundaries, and late-data policies are explicit.
- [ ] Every active input's progress constrains watermark advancement.
- [ ] Idleness and timestamp validation do not silently change correctness promises.
- [ ] Checkpoints couple state with replayable source positions.
- [ ] Sink deduplication or transactions cover the actual external effect.
- [ ] State retention, source retention, and recovery capacity are budgeted together.
- [ ] A crash after output and before checkpoint completion is tested.
