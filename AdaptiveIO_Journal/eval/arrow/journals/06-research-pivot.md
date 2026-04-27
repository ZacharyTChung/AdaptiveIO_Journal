# Part 6: Research Pivot — From "Less Work" to "Faster Work"

*Section 22 of the research journal. This is the key turning point.*

---

*← Previous: [Validation Experiments](./05-validation-experiments.md) | [Index](./README.md) | Next: [Exp C: Fault Isolation →](./07-exp-c-fault-isolation.md)*

---

## 22. Reassessing the Research Direction: From "Less Work" to "Faster Work"

### 22.1 The Problem With Our Wins So Far

Sections 1-21 ([Parts 1-5](./01-methodology.md)) document a series of experiments comparing io_uring-backed O_DIRECT against mmap for Arrow IPC reads. The clearest wins for io_uring came from **column projection** (H1: 8× cold speedup by reading 5% of the file) and **user-space caching** (H1 cache: 6,000× warm load speedup by eliminating I/O entirely). Both are engineering optimizations — they make io_uring win by having it *do less work* than mmap.

But this sidesteps the fundamental systems research question: **when both strategies read the same bytes, can io_uring do the I/O faster?**

Our prior experiments cannot answer this because they were designed to compare strategies doing *different amounts* of work. The column projection strategies read 95 MiB while mmap strategies read 2 GiB. The cache strategies read 0 bytes on warm iterations. Comparing wall times across strategies that read different byte volumes tells us about I/O avoidance, not I/O efficiency.

### 22.2 The Architectural Difference We Haven't Measured

mmap and io_uring have a fundamental architectural difference in how they handle non-resident data:

**mmap page fault model**: When a thread touches a non-resident page, the CPU generates a synchronous trap. The kernel handles it: find/allocate a physical page, initiate a disk read, block the thread until the read completes, update page tables, return to userspace. The thread then touches the next non-resident page, triggering another trap. Each fault is **serialized** — the thread is blocked for the duration of each individual disk read.

**io_uring batch submission model**: The thread fills a submission queue with N read requests and calls `io_uring_enter()` once. The kernel drains the submission queue, posting all N requests to the NVMe submission queue. The SSD's internal controller processes multiple requests concurrently (modern NVMe devices support 32-128 queue depth). The thread then polls the completion queue for all N results. Total I/O time ≈ max(individual read latencies), not sum.

The theoretical speedup from batching is **up to N×** for N concurrent reads, bounded by the SSD's queue depth.

### 22.3 Why We Didn't See It: Bandwidth-Bound vs IOPS-Bound

Our experiments used Arrow IPC column buffers of ~1 MiB. At this size, each read is bandwidth-bound:

- 1 MiB transfer at 3 GB/s = **333μs** 
- NVMe command latency = **~10μs**
- Serialization penalty per fault = 10μs / 333μs = **3% overhead**

For 54 column reads, the total serialization penalty is 54 × 10μs = 540μs — less than 3% of the 18ms total I/O. This is invisible in noisy benchmarks. The kernel's readahead mechanism further masks the issue by prefetching upcoming pages while the current fault is being serviced.

But the math changes dramatically for small reads:

| Read Size | Transfer Time | NVMe Latency | Serial Penalty | N=100 Serial | N=100 Batched | Speedup |
|-----------|--------------|--------------|----------------|-------------|---------------|---------|
| 4 KiB     | 1.3μs       | 10μs         | 88%            | 1.13ms      | ~15μs         | **75×** |
| 16 KiB    | 5.3μs       | 10μs         | 65%            | 1.53ms      | ~16μs         | **96×** |
| 64 KiB    | 21μs        | 10μs         | 32%            | 3.1ms       | ~31μs         | **100×** |
| 256 KiB   | 85μs        | 10μs         | 11%            | 9.5ms       | ~95μs         | **100×** |
| 1 MiB     | 333μs       | 10μs         | 3%             | 34.3ms      | ~343μs        | **100×** |

*Note: At N=100, the batched time is bounded by max(transfer_time) because the SSD processes all 100 reads concurrently (within its queue depth). The speedup saturates at min(N, queue_depth).*

The insight: **io_uring's batching advantage is always present, but it only produces a meaningful wall-time speedup when reads are small enough that command latency is a significant fraction of total per-read time.**

### 22.4 Evidence Hiding in Our Existing Data

The serialization cost IS visible in our data, buried inside phase timing:

**Cold cache, stride=5, 3 columns** (from `results/run_20260326_045556/`):
```
odirect-column:   load=66.16ms   filter=25.75ms   (data pre-loaded, filter at memory speed)
mmap-projection:  load=0.01ms    filter=105.73ms   (page faults occur DURING filter scan)
```

The filter phase iterates over column values. For `odirect-column`, all data was loaded into RAM during the explicit load phase (66ms), so the filter runs at pure memory bandwidth (26ms). For `mmap-projection`, each non-resident page accessed during filtering triggers a synchronous page fault. The filter time inflates from 26ms to 106ms — an **80ms penalty** from page fault serialization during the hot path.

This is the smoking gun: the same compute work (scanning the same column values for `trip_miles > 5.0`) takes 4× longer when data must be faulted in on-demand versus pre-loaded. The 80ms gap is the cost of ~1,400 serialized page faults (each blocking the thread for ~57μs on average).

However, this comparison is confounded — the two strategies also use different decode paths and read different byte volumes. A clean experiment must isolate the fault-serialization cost by having both strategies access the **exact same bytes**.

### 22.5 The Next Experiments

To cleanly measure io_uring's batch submission advantage over mmap's serialized faults, we need experiments where:

1. **Both strategies read the same bytes** — no "doing less work" confound
2. **Reads are small enough to be IOPS-bound** — where serialization dominates
3. **Access is non-sequential** — defeating kernel readahead

Three experiments are designed:

**Experiment C (Pure Fault Isolation)**: The cleanest test. Select K non-contiguous 4 KiB offsets from the IPC file. Measure time to make all K pages resident via (a) mmap sequential touch, (b) mmap with MADV_RANDOM, (c) io_uring batched read. No Arrow parsing, no filtering — just page residency. Expected speedup: 20-70× at K=100 depending on readahead effectiveness. → [Part 7](./07-exp-c-fault-isolation.md)

**Experiment A (Small Scattered Reads)**: Simulates zone-map-filtered access. Instead of reading entire 1 MiB column buffers, read only the 4-16 KiB pages that contain rows matching a predicate. Hundreds of small, non-contiguous reads per query. Sweep read count (10-1000), read size (4 KiB-256 KiB), and selectivity (1%-50%) to map the crossover from IOPS-bound to bandwidth-bound. → [Part 8](./08-exp-a-scattered-reads.md)

**Experiment B (Concurrent Multi-Query)**: Multiple threads query different column pairs simultaneously. mmap threads share page tables and contend on the kernel's per-process `mmap_lock` during fault handling. io_uring threads have independent rings with no shared state. Tests whether io_uring's isolation advantage scales with concurrency. → [Part 9](./09-exp-b-concurrent-verdict.md)

### 22.6 What This Changes About the Research Narrative

Sections 1-21 answered: *"When should a data engine use io_uring instead of mmap?"* Answer: when it can skip I/O via column projection or caching.

The next experiments answer a deeper question: *"Is mmap's synchronous page fault model fundamentally suboptimal for concurrent, non-sequential I/O?"* This is a systems architecture question independent of any specific file format or workload optimization. The answer has implications for every system that uses mmap for file I/O — databases, search engines, ML model serving, and beyond.

If confirmed, the finding would be: **io_uring batch submission achieves N× I/O parallelism that mmap's serial fault handling cannot, for IOPS-bound access patterns below ~64 KiB read size.** This is a statement about the Linux I/O architecture, not about Arrow file engineering.

---

*← Previous: [Validation Experiments](./05-validation-experiments.md) | [Index](./README.md) | Next: [Exp C: Fault Isolation →](./07-exp-c-fault-isolation.md)*
