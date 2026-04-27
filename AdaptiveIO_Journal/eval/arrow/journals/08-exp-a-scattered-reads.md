# Part 8: Experiment A — Small Scattered Reads

*Section 24 of the research journal. Validating Exp C with real Arrow IPC metadata.*

---

*← Previous: [Exp C: Fault Isolation](./07-exp-c-fault-isolation.md) | [Index](./README.md) | Next: [Exp B & Final Verdict →](./09-exp-b-concurrent-verdict.md)*

---

## 24. Experiment A: Small Scattered Reads

Experiment A tests whether the pure fault isolation results from Exp C ([Part 7](./07-exp-c-fault-isolation.md)) hold in a realistic Arrow IPC workload. Instead of synthetic offsets, Exp A generates offsets from **actual IPC file metadata**: for a given column and selectivity, it reads the Arrow IPC footer, iterates all 91 batch blocks, finds the column's values buffer in each batch, divides each buffer into `read_size`-byte chunks, and selects `ceil(num_chunks × selectivity)` chunks per batch using the LCG PRNG. The selected offsets are rounded to 4096-byte alignment, deduplicated, and sorted. This simulates the access pattern a zone-map-filtered scan would produce: many small, non-contiguous reads scattered across the file.

**Experiment A strategies** (4 strategies):

1. **mmap-serial**: `mmap()` the file with default advice. Touch offsets in sorted (ascending) order via `volatile_read`. The kernel's readahead can prefetch ahead of the access pattern, but faults are still synchronous.

2. **mmap-random**: `mmap()` with `madvise(MADV_RANDOM)`. Touch offsets in sorted order. Readahead is disabled, so every non-resident page incurs a full NVMe command latency. This is the mmap baseline for scattered access.

3. **iouring-batched**: Open with `O_DIRECT`. Submit all reads in a single `io_uring` batch via `read_many()`, then poll all completions. Same mechanism as Exp C but with metadata-derived offsets.

4. **iouring-one-at-a-time**: Open with `O_DIRECT`. Submit and complete each read serially. The io_uring control for Exp A.

**Parameter sweeps**:
- *Sweep 1 (selectivity)*: 0.01, 0.05, 0.10, 0.50 — column `trip_miles`, read_size 4096. Tests how the number of reads affects per-read latency.
- *Sweep 2 (read size)*: 4096, 16384, 65536, 262144 — column `trip_miles`, selectivity 0.05. Tests the IOPS-bound to bandwidth-bound crossover with real metadata.
- *Sweep 3 (column variation)*: `trip_miles`, `trip_time`, `base_passenger_fare` — selectivity 0.05, read_size 4096. Tests whether the results are column-independent.

### 24.1 Selectivity Scaling: Amortizing Command Latency

Column: `trip_miles`, read_size: 4096. Averages over 5 cold-cache rounds.

| selectivity | num_reads | mmap-serial | mmap-random | iouring-batched | iouring-one-at-a-time |
|-------------|-----------|-------------|-------------|-----------------|----------------------|
| 0.01        | 272       | 306,467     | 85,449      | 4,899           | 84,338               |
| 0.05        | 1,158     | 191,916     | 84,799      | 4,243           | 83,944               |
| 0.10        | 2,257     | 127,750     | 85,124      | 4,284           | 84,338               |
| 0.50        | 9,168     | 33,283      | 85,756      | 3,710           | 84,237               |

*Units: ns/read (avg_per_read_ns). Source: results/exp_a_20260327_062157/*

**Analysis**:
- **Finding 8: iouring-batched is 17.4× faster than mmap-random at 1% selectivity** (4,899 vs 85,449 ns/read). The speedup grows to **23×** at 50% selectivity (3,710 vs 85,756 ns/read) as more reads are batched together, further amortizing NVMe command latency.
- **Finding 9: mmap-serial degrades sharply at low selectivity.** At 1% selectivity (272 reads), mmap-serial reads 30 MiB sequentially to touch 272 scattered pages — 306 μs/read. At 50% selectivity (9,168 reads), the sequential scan amortizes better: 33 μs/read. This confirms that mmap-serial is only competitive when access is nearly sequential.
- **Finding 10: mmap-random is selectivity-invariant** (~85 μs/read across all selectivity values). Each page fault costs a fixed ~10 μs NVMe command regardless of how many are issued.
- **Finding 11: iouring-one-at-a-time matches mmap-random** (~84 μs/read). Serial io_uring provides no parallelism benefit — the advantage is exclusively from batch submission.

### 24.2 Read Size Scaling: Re-confirming the Crossover

Column: `trip_miles`, selectivity: 0.05. Averages over 5 cold-cache rounds.

| read_size | num_reads | mmap-serial | mmap-random | iouring-batched | iouring-one-at-a-time |
|-----------|-----------|-------------|-------------|-----------------|----------------------|
| 4,096     | 1,158     | 192,058     | 84,906      | 4,121           | 83,731               |
| 16,384    | 357       | 304,085     | 85,667      | 10,973          | 189,193              |
| 65,536    | 91        | 350,526     | 85,812      | 40,392          | 243,292              |
| 262,144   | 91        | 357,193     | 87,210      | 148,461         | 588,734              |

*Units: ns/read (avg_per_read_ns). Source: results/exp_a_20260327_062157/*

**Analysis**:
- **Finding 12: The iouring-batched crossover is between 16 KiB and 65 KiB.** At 16 KiB, iouring-batched is 7.8× faster than mmap-random (10,973 vs 85,667 ns/read). At 65 KiB, the gap narrows to 2.1× (40,392 vs 85,812). At 256 KiB, iouring-batched is slower (148,461 vs 87,210 ns/read) — transfer time now dominates over command latency.
- **Finding 13: iouring-one-at-a-time degrades rapidly with read size.** At 16 KiB it is 2.2× slower than mmap-random (189,193 vs 85,667), confirming that serial io_uring adds syscall overhead without any parallelism benefit for large reads.

### 24.3 Column Variation: Strategy Independence

Selectivity: 0.05, read_size: 4096. Averages over 5 cold-cache rounds.

| column              | num_reads | mmap-serial | mmap-random | iouring-batched | iouring-one-at-a-time |
|---------------------|-----------|-------------|-------------|-----------------|----------------------|
| trip_miles          | ~1,146    | 192,031     | 85,906      | 4,461           | 84,611               |
| trip_time           | ~1,158    | 196,958     | 85,440      | 5,540           | 85,402               |
| base_passenger_fare | ~1,153    | 195,026     | 85,093      | 4,411           | 85,246               |

*Units: ns/read (avg_per_read_ns). Source: results/exp_a_20260327_062157/*

**Analysis**:
- **Finding 14: Results are largely column-invariant, with a notable exception.** The mmap strategies (serial: 192–197 μs, random: 85.1–85.9 μs) and iouring-one-at-a-time (84.6–85.4 μs) show ≤3% variation across columns — the I/O pattern, not the data type, determines performance. However, **iouring-batched shows a 24% anomaly for `trip_time`** (5,540 ns/read) versus `trip_miles` (4,461) and `base_passenger_fare` (4,411). This likely reflects differences in physical buffer layout or alignment within the IPC file for `trip_time`, which affects the NVMe controller's ability to coalesce adjacent reads in the hardware queue. The effect is specific to the batched strategy because only batch submission exposes inter-request locality to the NVMe scheduler.

---

*← Previous: [Exp C: Fault Isolation](./07-exp-c-fault-isolation.md) | [Index](./README.md) | Next: [Exp B & Final Verdict →](./09-exp-b-concurrent-verdict.md)*
