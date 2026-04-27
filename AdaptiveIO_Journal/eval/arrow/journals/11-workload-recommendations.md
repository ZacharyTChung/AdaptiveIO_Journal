# Part 11: Workload Recommendations — Where io_uring Wins

*Standalone research findings. Translates experimental results (Experiments A-D) into concrete workload prescriptions for io_uring-based Arrow IPC engines.*

---

*← Previous: [Exp D: Cold Concurrent](./10-exp-d-concurrent-scattered.md) | [Index](./README.md) | [Next → Part 12: Exp E](./12-exp-e-zone-map-query.md) →*

---

## The Decision Framework

This table summarizes when `io_uring` batch submission outperforms `mmap` demand paging, based on the architectural findings in [Part 7 (§23)](./07-exp-c-fault-isolation.md) through [Part 10 (§26)](./10-exp-d-concurrent-scattered.md).

| Characteristic | Threshold | io_uring EP | Evidence |
|---|---|---|---|
| Read size | < 128 KiB | 2.1-23× | Exp C §23.2: 2.2× at 64K, mmap wins at 256K |
| Batch depth | ≥ 50 reads per batch | 5-23× | Exp C §23.1: EP=3.5× at N=10, 13.6× at N=100 |
| Access pattern | Non-sequential | 17-23× | Exp A §24.1: scattered IPC metadata offsets |
| Cache state | Cold (working set > RAM) | 7.9× | Exp D §26: cold=7.9×, warm=1.03× |
| Concurrency | 1-4 threads optimal | 5.8-7.9× | Exp D §26: EP degrades to 2.6× at 12 threads |

### Characteristics That Favor io_uring

1.  **Scattered Small Reads (< 64 KiB)**: The dominant advantage. `io_uring` parallelizes NVMe command latency across a batch of reads, whereas `mmap` serializes them through one-at-a-time page faults. (Source: Exp C §23.1)
2.  **High Batch Depth (N > 50)**: The "Effective Parallelism" (EP) scales almost linearly with batch size up to the NVMe queue depth (~32-128). (Source: Exp C §23.1)
3.  **Cold Cache State**: `io_uring`'s advantage is entirely in the I/O path. If data is already in the page cache, the 3-23× speedup vanishes. (Source: Exp D §26)
4.  **Sparse Selectivity (< 20%)**: When a query skips large sections of a file (e.g., via zone-maps or column projection), the resulting reads are scattered, favoring batch submission. (Source: Exp A §24.1)
5.  **Low to Moderate Concurrency (1-4 threads)**: `io_uring` wins most when a single thread can saturate the NVMe queue. At 12+ threads, concurrent batches compete for the same queue depth, reducing the relative advantage over mmap. (Source: Exp D §26.1)

### Characteristics That Favor mmap

| Condition | Why | Evidence |
|---|---|---|
| Read size > 128 KiB | Transfer time dwarfs command latency | Exp C §23.2: mmap 1.7× at 256K |
| Warm cache | No I/O needed; page-touch is faster than syscall | Exp B §25: only 3% difference |
| Full sequential scan | Kernel readahead (e.g., 128K chunks) is highly effective | §4-7 baseline experiments |
| ≥12 concurrent threads + warm cache | Thread overhead and NVMe saturation dominate | Exp B §25, Exp D §26 |

---

## Workload 1: Zone-Map Filtered Point Query

**Description**: A point query or small-range scan on a sorted or clustered column where zone-map metadata (min/max stats per batch) allows skipping 90% or more of the file.

- **Access Pattern**: ~50-200 scattered 4 KiB page reads across a 2 GiB file.
- **Why io_uring Wins**: The query engine can identify all required page offsets from the Arrow IPC footer and submit them in a single `io_uring` batch. This parallelizes the NVMe latency of all 50-200 reads. `mmap` would incur a synchronous ~85µs stall for every single page.
- **Predicted EP**: 15-23× (High confidence — directly confirmed by Exp A §24).
- **Implementation Sketch**: 
    1. Read IPC footer (metadata).
    2. Evaluate zone-map predicate against all 91 batches.
    3. Collect matching batch column offsets.
    4. Submit all offsets in one `io_uring` batch via `readv` or `read_many`.
    5. Decode only the returned pages.
- **Key Parameters**: Selectivity is the critical knob. Below 10% selectivity, `io_uring` is over 20× faster. Above 50%, the advantage drops as reads become more sequential.
- **Limitations**: If the `read_size` exceeds 64 KiB (e.g., very wide columns), `mmap` begins to catch up due to better transfer efficiency.

## Workload 2: Multi-File Partition Scan with Predicate Pushdown

**Description**: A query scanning multiple partition files (e.g., `date=2021-01-01` to `date=2021-01-12`) where each file is filtered by a selective predicate.

- **Access Pattern**: ~12,000-20,000 scattered reads distributed across 12 files.
- **Why io_uring Wins**: `io_uring` supports multi-file batches. A single thread can submit a single ring-entry containing reads from all 12 files simultaneously. This maximizes NVMe queue depth utilization better than any serial approach.
- **Predicted EP**: 5-10× (Medium confidence — extrapolated from Exp A + Exp D).
- **Implementation Sketch**:
    1. For each of 12 files: Read footer and evaluate predicate on zone-maps.
    2. Collect all matching offsets across all 12 files into a single list.
    3. Open all files with `DirectFile` (O_DIRECT).
    4. Batch submit all reads via `io_uring` (chunked at 1024).
- **Key Parameters**: Number of files and selectivity per file.
- **Limitations**: If the predicate has low selectivity (all batches match), the workload becomes a sequential scan, and `mmap`'s kernel readahead wins.

## Workload 3: Feature Store Column Extraction

**Description**: Extracting a small subset of columns (3-5) from an extremely wide table (200+ columns) for machine learning inference.

- **Access Pattern**: 3-5 columns × 91 batches = ~300-500 scattered reads. Each read is the size of the column buffer in that batch (often 4 KiB to 64 KiB).
- **Why io_uring Wins**: In a 200-column table, target columns are physically separated by 195 other columns. This creates a "sparse" read pattern where 95% of the file is skipped. `io_uring` batches these sparse reads, while `mmap` faults them one-by-one.
- **Predicted EP**: 8-15× (Medium confidence — extrapolated from H2 validation §20 + Exp A §24.3).
- **Implementation Sketch**:
    1. Read footer to locate target column buffer offsets across all batches.
    2. Compute 4 KiB page-aligned offsets for these buffers.
    3. Submit as one `io_uring` batch.
- **Key Parameters**: Number of columns extracted.
- **Limitations**: If extracting more than ~50 columns from a 200-column table, the gaps between columns shrink, and the pattern approaches a sequential scan.

## Workload 4: Time-Series Dashboard Cold Refresh

**Description**: Loading a dashboard that queries the last 24 hours of data from disk. The working set (e.g., 48 GiB) exceeds typical server RAM, ensuring a cold-cache state.

- **Access Pattern**: Scattered 4 KiB reads across 24 hourly files for specific "timestamp" and "value" columns.
- **Why io_uring Wins**: The cold-cache state is the "ideal" regime for `io_uring`. Since the data is not in RAM, every access is an I/O. `io_uring`'s 7.9× single-thread advantage (Exp D) provides the fastest possible "Time to First Chart".
- **Predicted EP**: 5-8× (Medium confidence — extrapolated from Exp D §26).
- **Key Insight**: This workload specifically targets the "Cold Concurrent" findings from Part 10.
- **Implementation Sketch**:
    1. Open 24 hourly files.
    2. Scan zone-maps for the specific time range.
    3. Collect column offsets for the two required columns.
    4. Submit all reads in parallel across all files using one or more `io_uring` rings.
- **Limitations**: If the dashboard query is repeated frequently, the data will move into the page cache, and the `io_uring` advantage will drop to ~0% (Exp B).

## Workload 5: Predicate-Pushdown Join Probe

**Description**: A two-phase join where a small "build" table is used to create a Bloom filter, which then probes a large "probe" table.

- **Access Pattern**: Phase 1: Scan join-key column (scattered/sequential). Phase 2: If Bloom filter matches, read "payload" columns for that row (highly scattered).
- **Why io_uring Wins**: Phase 2 is the most scattered access pattern possible in Arrow IPC. Only specific rows from specific batches are read. This produces hundreds of tiny, disconnected reads that perfectly suit `io_uring` batching.
- **Predicted EP**: 3-8× (Low confidence — theoretical, needs validation).
- **Implementation Sketch**:
    1. Build Bloom filter from the small table.
    2. Scan the large table's join-key column using zone-map filtering.
    3. For rows matching the Bloom filter, collect their physical offsets for payload columns.
    4. Batch submit payload reads via `io_uring`.
- **Key Parameters**: Bloom filter false positive rate (FPR). A high FPR leads to more "wasted" reads, which may degrade to a sequential scan.

---

## Implementation Roadmap

1.  **Phase 1: Zone-Map Filtered Point Query (Highest Payoff)**. This is the most "direct" win. Implementation is simple: modify the existing Arrow IPC reader to collect offsets from zone-maps before submitting. Expected speedup: **15-20×**.
2.  **Phase 2: Multi-File Integration**. Enhance the reader to handle multiple `DirectFile` handles in a single batch. This unlocks the partition-scan workload.
3.  **Phase 3: Adaptive Crossover**. Implement a "Read Size Guard". If the requested read size exceeds 128 KiB, the engine should automatically fall back to `mmap` or a large-block `O_DIRECT` read to avoid `io_uring`'s overhead on large transfers.

---

## Appendix: Evidence Summary

| Claim | Source | Evidence |
|---|---|---|
| io_uring batch = 13.6× faster at 4 KiB | [Part 7, §23.1](./07-exp-c-fault-isolation.md) | N=100 random offsets, cold cache |
| Crossover at ~128 KiB | [Part 7, §23.2](./07-exp-c-fault-isolation.md) | mmap wins above this read size |
| 17-23× faster with real IPC offsets | [Part 8, §24.1](./08-exp-a-scattered-reads.md) | Selectivity 1-50%, trip_miles column |
| mmap_lock is NOT the bottleneck | [Part 9, §25.1](./09-exp-b-concurrent-verdict.md) | Per-fault serialization dominates |
| Cold-cache concurrent: 2.6-7.9× speedup | [Part 10, §26.1](./10-exp-d-concurrent-scattered.md) | Advantage persists under load |
| NVMe saturation degrades EP | [Part 10, §26.2](./10-exp-d-concurrent-scattered.md) | EP drops to 2.6× at 12 threads |
