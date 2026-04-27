# Part 10: Experiment D — Cold-Cache Concurrent Scattered Reads

*Section 26 of the research journal. Tests whether io_uring's batch advantage persists under realistic cold-cache concurrent workloads.*

---

*← Previous: [Exp B & Verdict](./09-exp-b-concurrent-verdict.md) | [Index](./README.md) | Next → [Workload Recommendations](./11-workload-recommendations.md)*

---

## 26. Experiment D: Cold-Cache Concurrent Scattered Reads

Experiment B (§25) found only a 3% difference between mmap and io_uring under concurrent load. However, that experiment measured *warm-cache page-touch latency* on sequential first-page offsets — no actual I/O was performed after the first round. Experiment D tests the hypothesis that **cold-cache scattered reads under concurrency restore io_uring's batch advantage**.

**Experiment D methodology:**

- **Design**: Combines [Exp A](./08-exp-a-scattered-reads.md)'s scattered-read pattern with [Exp B](./09-exp-b-concurrent-verdict.md)'s concurrent thread model. Each thread reads scattered 4 KiB pages from its assigned column pair across all 91 batches, with page cache dropped before every round.
- **Offset generation**: For each of 12 column pairs, `generate_column_offsets` (from `scattered.rs`) is called for both columns in the pair with selectivity=0.05, producing ~1,100-1,800 offsets per pair depending on column size. Offsets are merged, sorted, and deduplicated per pair. All precomputation happens once before timing begins.
- **Thread model**: Same as Exp B: `pair_idx = thread_id % 12`, columns = `(2 × pair_idx, 2 × pair_idx + 1)`. Barrier synchronization with channel-based timing.
- **Cold cache**: `posix_fadvise(POSIX_FADV_DONTNEED)` + `/proc/sys/vm/drop_caches` before every round.
- **Three modes**:
  - `mmap-shared`: Single `Arc<Mmap>` shared across threads, `MADV_RANDOM` hint, `volatile_read` per offset
  - `mmap-private`: Each thread opens and mmaps the file independently, `MADV_RANDOM` hint
  - `iouring-batched`: Each thread opens its own `DirectFile`, submits ALL reads via `read_many` in one batch (chunked at 1024 for large N)
- **Key difference from Exp B**: Reads are *scattered* (Exp A style, ~1,100-1,800 per thread) not sequential first-page touches (~182 per thread). io_uring uses *batched* submission, not serial one-at-a-time.
- **Metrics**: total_wall_time_ns (barrier release to last join), contention_factor (max_thread/min_thread), effective_parallelism (mmap-shared baseline / mode wall time).
- **Rounds**: 5 rounds per configuration, averages reported. Source: `results/exp_d_20260327_222740/`

### 26.1 Thread Scaling: Cold-Cache Scattered Reads

Selectivity: 0.05, read_size: 4096. Averages over 5 cold-cache rounds.

| threads | mmap-shared wall (ms) | mmap-shared CF | mmap-private wall (ms) | mmap-private CF | iouring-batched wall (ms) | iouring-batched CF | EP |
|---------|----------------------|----------------|------------------------|-----------------|---------------------------|--------------------|----|
| 1       | 151.6                | 1.000          | 153.2                  | 1.000           | 19.1                      | 1.000              | 7.94 |
| 2       | 158.1                | 1.017          | 159.4                  | 1.019           | 22.2                      | 1.041              | 7.13 |
| 4       | 204.9                | 1.307          | 204.1                  | 1.296           | 35.5                      | 1.191              | 5.77 |
| 8       | 210.1                | 1.298          | 211.1                  | 1.303           | 70.0                      | 1.689              | 3.00 |
| 12      | 212.8                | 6.277          | 214.9                  | 6.371           | 81.1                      | 8.250              | 2.63 |

*EP = effective_parallelism = mmap-shared wall / iouring-batched wall. CF = max_thread_time / min_thread_time. Source: results/exp_d_20260327_222740/*

### 26.2 Analysis

**Finding 18: Cold-cache restores io_uring's batch advantage under concurrency.** At 1 thread, iouring-batched is **7.94× faster** (19.1ms vs 151.6ms). Compare with [Exp B](./09-exp-b-concurrent-verdict.md) (warm cache): iouring was only 1.03× faster (17.0ms vs 17.5ms). The cold-cache scattered pattern forces actual NVMe I/O, where batch submission's command-latency parallelism dominates.

**Finding 19: Effective parallelism degrades with thread count (NVMe queue saturation).** EP drops from 7.94× (1 thread) to 2.63× (12 threads). At high concurrency, all threads submit batches simultaneously, saturating the NVMe controller's finite IOPS. The shared NVMe queue depth (~32-128 entries) is consumed by 12 threads' concurrent batches, reducing per-thread parallelism. Even so, iouring-batched remains **2.63× faster** at 12 threads — a substantial advantage over the 3% margin seen in warm-cache Exp B.

**Finding 20: mmap-shared ≈ mmap-private for cold-cache scattered reads.** Wall times differ by <2% across all thread counts (e.g., 212.8ms vs 214.9ms at 12 threads). This confirms [Exp B](./09-exp-b-concurrent-verdict.md)'s finding: the mmap_lock / shared-page-table overhead is negligible. The dominant cost is per-fault NVMe command serialization, which is identical for both mmap modes.

**Finding 21: Contention factor explodes at 12 threads for ALL modes.** CF reaches 6.3 (mmap) and 8.3 (iouring) at 12 threads. This is NOT caused by lock contention — it is caused by **asymmetric column sizes**. The 24 Arrow columns have vastly different buffer sizes: large columns (Float64 data buffers, ~96 MiB across 91 batches) produce ~1,800 offsets per pair, while small columns (null bitmaps, validity arrays) produce ~100 offsets. Threads assigned to small pairs finish in ~9ms while large-pair threads take ~76ms. The CF measures this workload imbalance, not I/O contention.

**Finding 22: io_uring wall time scales worse than mmap (+324% vs +40%).** From 1→12 threads, mmap-shared wall time grows 40% (152→213ms) while iouring-batched grows 324% (19→81ms). This apparent paradox is explained by NVMe physics: mmap was already serialized (~85µs per fault), so adding threads only adds moderate queueing overhead. io_uring's single-thread advantage came from filling the NVMe queue depth with one thread's batch — 12 threads compete for the same finite queue, diluting the per-thread parallelism. The absolute wall time (81ms) is still much lower than mmap's (213ms), but the *scaling gradient* favors mmap.

### 26.3 Exp B vs Exp D: The Complete Concurrency Picture

| Metric | Exp B (warm, §25) | Exp D (cold, §26) | Explanation |
|--------|-------------------|-------------------|-------------|
| mmap-shared 12 threads | 17.5 ms | 212.8 ms | Warm: no I/O needed. Cold: ~1,800 page faults per thread |
| iouring 12 threads | 17.0 ms | 81.1 ms | Warm: no I/O needed. Cold: batch submission parallelizes NVMe |
| Speedup | 1.03× | **2.63×** | Cold cache restores io_uring advantage |
| CF at 12 threads | 1.035 / 1.036 | 6.277 / 8.250 | Column size asymmetry dominates cold-cache CF |
| Bottleneck | None (page cache) | NVMe IOPS / column asymmetry | Different regimes entirely |

The key insight: **Exp B and Exp D measure different things.** Exp B measures thread synchronization overhead on already-cached data. Exp D measures actual I/O contention on cold-cache scattered reads. For real-world query workloads where data is not always cached, Exp D's results are more representative.

### 26.4 Updated Verdict

The addition of Experiment D strengthens the original verdict from [§25.3](./09-exp-b-concurrent-verdict.md):

**CONFIRMED: mmap's synchronous page fault model is suboptimal for concurrent, cold-cache, non-sequential reads below ~128 KiB.**

- **Single-thread**: 7.9× speedup (Exp D), consistent with 3.5-23× range from [Exps C](./07-exp-c-fault-isolation.md) and [A](./08-exp-a-scattered-reads.md)
- **12-thread concurrent**: 2.6× speedup despite NVMe queue saturation — demonstrating that io_uring's advantage persists under realistic concurrent load
- **Warm cache**: Only 3% difference ([Exp B](./09-exp-b-concurrent-verdict.md)) — confirming that the advantage is specifically in the I/O path, not in data access patterns

The practical implication is refined: **io_uring batch submission is most valuable for cold-cache workloads with scattered small reads. As working sets warm and cache hit rates increase, the advantage diminishes toward zero.**

---

*← Previous: [Exp B & Verdict](./09-exp-b-concurrent-verdict.md) | [Index](./README.md) | Next → [Workload Recommendations](./11-workload-recommendations.md)*
