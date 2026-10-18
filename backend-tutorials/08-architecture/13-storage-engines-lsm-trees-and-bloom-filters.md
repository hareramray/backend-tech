# Storage engines, LSM trees, compaction, and Bloom filters

## Purpose and prerequisites

An API can have excellent queries and still stall because the storage engine is rewriting old data, waiting for durable log flushes, or serving too many random reads. Understanding the physical write and read paths helps explain these symptoms and choose meaningful measurements.

Start with [capacity planning](05-system-design-and-capacity-planning.md) and [distributed consistency](03-distributed-systems-and-consistency.md). This chapter compares storage mechanisms; it does not replace PostgreSQL with a custom Python database. The lab implements only a probabilistic membership filter and needs the Python standard library.

## PostgreSQL is not an LSM engine

The usual PostgreSQL table stores tuples in heap pages; indexes are separate structures, commonly B-trees. A heap page contains a header, item identifiers, free space, and tuple data. A B-tree maps ordered indexed keys toward matching tuples. PostgreSQL's default page size is usually 8 KiB, with build-time alternatives. [PostgreSQL database page layout](https://www.postgresql.org/docs/current/storage-page-layout.html).

An UPDATE normally creates a new tuple version under PostgreSQL's MVCC rules; eligible HOT updates can avoid some index work. VACUUM makes eligible dead-tuple space reusable, and page pruning can reclaim some eligible versions during ordinary access. These mechanisms are distinct from merging immutable sorted tables. Avoid the misleading shorthand that every B-tree database simply overwrites a complete record in place. [PostgreSQL HOT updates](https://www.postgresql.org/docs/current/storage-hot.html), [routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html).

Write-ahead logging means the relevant log records must reach durable storage before the corresponding changed data pages are persisted. Crash recovery can replay the log. Commit durability depends on configuration and the storage stack; “WAL enabled” is not a claim that every acknowledgement waits for disk flush. [PostgreSQL WAL introduction](https://www.postgresql.org/docs/current/wal-intro.html).

For a Django application, adding indexes can reduce search work while increasing update work and occupied space. A slow commit caused by log flush latency needs different treatment from a slow SELECT caused by missing selectivity. Measure query plans, buffer hits, WAL generation, storage latency, and maintenance before selecting a new database category.

## Trace an LSM write through memory and files

An LSM, log-structured merge, design buffers writes in a mutable memtable, commonly records them in a WAL for recovery, flushes full buffers into immutable sorted string table files, or SSTables, and merges files through compaction. RocksDB exposes these components as an embedded key-value storage engine. [RocksDB architecture overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview).

Consider this illustrative sequence with monotonically increasing internal sequence numbers:

```text
sequence 41: PUT task:9 = "open"
sequence 42: PUT task:2 = "open"
flush F1:  task:2@42, task:9@41
sequence 43: PUT task:9 = "done"
sequence 44: DELETE task:2
flush F2:  task:2@44 = tombstone, task:9@43 = "done"
```

The files are individually sorted by key and internal version order; newer files can overlap older key ranges. A current read of `task:9` must choose version 43. A current read of `task:2` must return absent because its newest visible entry is a deletion marker, called a tombstone. Encountering an old value in F1 does not resurrect it.

A typical point lookup checks relevant memory buffers and then candidate files, using key-range metadata, indexes, filters, and caches to avoid unnecessary reads. It selects a version visible to the requested snapshot. A range scan must merge ordered streams and suppress overwritten or deleted entries; a point-key Bloom filter alone cannot enumerate the range.

The WAL and SSTables solve different problems. A WAL supports reconstruction after interruption; an SSTable supports ordered reads over durable data. An engine must also persist which files constitute the database. Completing a file write without safely publishing its metadata is not sufficient for crash consistency. Treat fsync, checksums, manifests, atomic publication, and recovery as engineered protocols rather than optional details of a teaching dictionary.

## Compaction pays for deferred organization

Leveled compaction organizes increasingly large levels and rewrites selected overlapping ranges. It generally aims to reduce redundant live storage and bound lookup work, at the cost of repeated rewriting. Tiered or universal approaches merge groups of sorted runs, often reducing write work while retaining more runs and temporary space. Exact behavior depends on the engine's policy and workload. [RocksDB compaction styles](https://github.com/facebook/rocksdb/wiki/Compaction).

For the example above, a complete merge of F1 and F2 could retain `task:9@43` and remove `task:9@41` when no supported snapshot still needs version 41. Dropping `task:2@44` is safe only when no older retained copy of `task:2` can become visible and no supported historical view requires the previous value. Compacting only F2 and discarding its tombstone while leaving F1 searchable would resurrect deleted data.

A snapshot at sequence 42 needs the earlier `task:9 = open` and `task:2 = open` values. Long-lived snapshots therefore constrain version reclamation. In RocksDB, snapshots influence which versions compaction may remove; they are not simply independent database backups. [RocksDB snapshots](https://github.com/facebook/rocksdb/wiki/Snapshot).

In a distributed store, deletion also interacts with replication and repair: an offline replica may still contain the old value. A local compaction rule is not by itself proof that a distributed tombstone can be discarded. Consult the distributed database's repair and retention protocol separately.

## Quantify amplification with a clear denominator

There is no universal best engine independent of access pattern, memory, device behavior, and durability settings. Name the measurement boundary before comparing ratios:

| Quantity | Example definition | Why it matters |
|---|---|---|
| Write amplification | Storage bytes written / logical application bytes written | Bandwidth, endurance, background work |
| Read amplification | Bytes read from storage / useful response bytes | I/O cost, latency, cache effectiveness |
| Lookup amplification | Candidate runs or blocks examined per lookup | Useful alongside byte measurements |
| Space amplification | Occupied engine bytes / logical live-data bytes | Capacity and compaction headroom |

Worked example: an application writes 100 MiB of logical data. Its WAL writes 100 MiB, initial SSTable flushes write 100 MiB, and compactions write another 800 MiB. Counting all three, write amplification is `(100 + 100 + 800) / 100 = 10`. A metric excluding WAL would report nine for the same experiment. Filesystem metadata, compression, device-level flash rewriting, and backups change other boundaries; document what is included.

If measured total write amplification is ten and the device sustains 250 MB/s for this workload, 25 MB/s of logical writes already consumes that simplified write budget. Foreground reads, checkpoints, metadata, and safety headroom reduce the usable limit. Increasing ingestion indefinitely merely grows compaction debt.

Suppose a missing-key lookup has 12 candidate files, each with a 1% false-positive filter rate. If every filter is available in memory, expected unnecessary data-file probes are approximately `12 * 0.01 = 0.12`, instead of 12. This does not say the lookup costs 0.12 I/O operations in every execution: metadata access, cache misses, and per-file overhead still exist, and any one lookup may probe several files.

## Derive a Bloom filter's size

A standard Bloom filter stores `m` bits and hashes each inserted key to `k` bit positions. Insertion sets those bits. A query with any zero bit is definitely absent from the inserted set; all-one positions mean possibly present. Assuming stable encoding, correct operation, and unchanged bits, inserted keys have no false negatives. False positives require verification against authoritative data. [Bloom's original paper](https://www.cs.princeton.edu/courses/archive/spr05/cos598E/bib/p422-bloom.pdf).

For `n` inserted keys and reasonably uniform, approximately independent hash positions, the usual approximate false-positive probability is:

```text
p ~= (1 - exp(-k*n/m))**k
optimal k ~= (m/n) * ln(2)
m ~= -n * ln(p) / (ln(2)**2)
```

For 1,000 keys and target probability 0.01, round up to 9,586 bits, allocate 1,199 bytes, and use seven hashes. The formulas are a sizing model, not a guarantee for every query set. Overfilling a filter increases its false-positive rate. Clearing a shared bit to delete one key can introduce false negatives for another key; this basic structure does not support deletion.

A storage engine typically creates filters for particular tables or blocks and replaces them with new filters when compacted files change. Filters cost memory and CPU; their value depends on the read workload. RocksDB also discusses implementation choices such as full and partitioned filters. [RocksDB Bloom filters](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter).

## Local lab: test membership and false positives

1. Save this complete block as `bloom_lab.py` in a disposable directory.
2. Run `python bloom_lab.py` with Python 3.10 or later. No packages or services are required.
3. Confirm the no-false-negative assertion and inspect the empirical rate for 10,000 disjoint query keys.
4. Increase `n` without resizing the same filter and compare the rate. Change the target rate and observe the memory and hash-count tradeoff.

```python
import hashlib
import math


class BloomFilter:
    def __init__(self, expected_keys, false_positive_rate):
        if expected_keys <= 0 or not 0 < false_positive_rate < 1:
            raise ValueError("positive key count and probability in (0, 1)")
        self.m = math.ceil(
            -expected_keys * math.log(false_positive_rate)
            / math.log(2) ** 2
        )
        self.k = max(1, round(self.m / expected_keys * math.log(2)))
        self.bits = bytearray((self.m + 7) // 8)

    def positions(self, value):
        encoded = value.encode("utf-8")
        for seed in range(self.k):
            digest = hashlib.sha256(seed.to_bytes(4, "big") + encoded).digest()
            yield int.from_bytes(digest, "big") % self.m

    def add(self, value):
        for position in self.positions(value):
            self.bits[position // 8] |= 1 << (position % 8)

    def might_contain(self, value):
        return all(
            self.bits[position // 8] & (1 << (position % 8))
            for position in self.positions(value)
        )


inserted = {f"task:{i}" for i in range(1000)}
bloom = BloomFilter(len(inserted), 0.01)
for key in inserted:
    bloom.add(key)
assert (bloom.m, bloom.k, len(bloom.bits)) == (9586, 7, 1199)
assert all(bloom.might_contain(key) for key in inserted)

queries = [f"absent:{i}" for i in range(10000)]
assert inserted.isdisjoint(queries)
false_positives = sum(bloom.might_contain(key) for key in queries)
rate = false_positives / len(queries)
assert 0 < rate < 0.025  # A check for this deterministic fixture only.
print("bits, hashes, bytes:", bloom.m, bloom.k, len(bloom.bits))
print("false positives:", false_positives)
print("observed probability:", round(rate, 4))
```

Expected output reports `bits, hashes, bytes: 9586 7 1199`, 93 false positives, and an observed probability of `0.0093`. SHA-256 with a distinct fixed seed prefix makes the fixture reproducible across runs; Python's randomized built-in string hash would not. This implementation emphasizes clarity, uses multiple digest computations, and makes no performance or adversarial-resistance claim.

Never use “possibly present” as proof that an account exists, a payment completed, or authorization is granted. A useful application optimization is to skip an expensive negative lookup when the filter says absent, while always checking real storage when it says possibly present. Keep filter freshness consistent with the set it summarizes; a stale filter can incorrectly exclude newly added keys.

## Diagnose stalls and plan capacity

Track immutable memtables awaiting flush, level-zero file count, pending compaction bytes, compaction throughput, cache hit rate, file-read latency, and write-stall duration. Engines may deliberately slow or stop writes when background work cannot keep pace, protecting bounded resources rather than accepting an unlimited backlog. [RocksDB write stalls](https://github.com/facebook/rocksdb/wiki/Write-Stalls).

Model temporary compaction space separately from steady-state live data. A merge may need both old input files and new outputs until publication and reference release are safe. Long snapshots, large values, skewed hot keys, and an overloaded background I/O budget can substantially alter the expected footprint. A benchmark ending before these costs appear is not a sustained-ingestion benchmark.

## Exercises and completion checklist

Calculate filter storage for ten million keys at 1% and 0.1% target rates. Then trace a delete across three overlapping files and a snapshot that precedes the deletion. Identify exactly which versions may be removed after that snapshot closes.

- [ ] PostgreSQL heap/index/WAL behavior is distinguished from LSM behavior.
- [ ] Read, write, and space amplification specify measurement boundaries.
- [ ] Compaction headroom and sustained throughput appear in capacity estimates.
- [ ] Tombstone removal cannot reveal an older retained value.
- [ ] Snapshot lifetime and replication repair have explicit retention rules.
- [ ] Bloom positives are verified and the basic filter never clears shared bits.
- [ ] Benchmarks include updates, deletions, negative reads, scans, and recovery.
