# Part 5: Validation Experiments

*Sections 19-21 of the research journal.*

---

*← Previous: [Throughput & Scaling](./04-throughput-and-scaling.md) | [Index](./README.md) | Next: [Research Pivot →](./06-research-pivot.md)*

---

## 19. H1 Validation: User-Space Column Cache

To address the warm-cache disadvantage of O_DIRECT, we implemented `odirect-column-cached`, which maintains a user-space cache of column buffers.

| Iteration | Load (ms) | I/O Bytes |
|---|---|---|
| 1 (Cold) | 123.60 | 91.28 MiB |
| 2 (Warm) | 0.01 | 0 B |

**Finding 10: 6,000× warm speedup.**
By eliminating I/O entirely on repeated queries, the user-space cache allows O_DIRECT to finally match (and slightly exceed) mmap's warm performance by avoiding the overhead of page table lookups.

---

## 20. H2 Validation: Wide-Table Projection (200 Columns)

We tested a 200-column synthetic table to find the crossover point where the overhead of O_DIRECT's manual management outweighs its I/O selectivity.

- **Narrow Projection (2/200 cols)**: `odirect` reads 20.11 MiB vs `mmap` 22.57 MiB. `odirect` is ~10% faster.
- **Wide Projection (50/200 cols)**: `mmap-projection` is 13% faster than `odirect-column`.

**Finding 11: The Crossover Point.**
As the number of projected columns increases, the kernel's readahead and demand-paging efficiency for `mmap` begins to outperform the overhead of managing hundreds of individual `io_uring` read requests.

---

## 21. H1+H2 Synthesis and Future Directions

### 21.1 Key Findings

**Finding 1: Demand-page precision equals O_DIRECT precision for Arrow IPC projection.**
Both mmap and O_DIRECT read ~1% of file bytes for 1% column selectivity (2/200 columns). Arrow IPC's dense column layout makes the kernel's page-granular demand paging inherently column-precise. This was the most important and surprising result.

**Finding 2: User-space cache is the decisive advantage of O_DIRECT.**
The cache eliminates all repeated I/O (io_bytes=0 from iteration 2), reduces page faults to near-zero, and delivers load times of 0.01ms (vs 120ms uncached). On warm queries, the cached strategy is 7–14% faster than mmap-projection across both the 24-column and 200-column datasets.

**Finding 3: O_DIRECT's cold-start advantage is context-dependent.**
On the 24-column dataset, O_DIRECT is 16–22% faster cold (reads fewer bytes, avoids demand-fault stalls during filtering). On the 200-column dataset with narrow projection, mmap-projection is 10–15% faster cold (overlapped I/O compensates for slightly higher byte reads). The crossover depends on whether the workload is I/O-bound (O_DIRECT wins) or decode/filter-bound (mmap wins via pipelining).

**Finding 4: Total warm latency is filter-dominated, not I/O-dominated.**
On the 24-column dataset, both strategies converge to ~80ms warm (filter alone). On the 200-column dataset, both converge to ~18–21ms warm. The cache eliminates I/O completely; further optimization requires accelerating the filter computation itself (SIMD, columnar pruning, etc.).

### 21.2 Revised Strategy Recommendations

| Condition | Recommended Strategy | Rationale |
|---|---|---|
| Cold first query, narrow projection | O_DIRECT column (24-col) or mmap-projection (200-col) | Depends on file shape; O_DIRECT wins when filter dominates |
| Warm repeated queries | O_DIRECT + user-space cache | Eliminates all I/O and page faults; 7–14% faster than mmap |
| Cold wide projection (50+ cols) | mmap-projection | Overlapped I/O via demand paging; 13% faster at w=50 |
| Stride/batch sampling | O_DIRECT column | Reads only selected batches; 22% faster than mmap at stride=5 |
| Mixed cold+warm workload | O_DIRECT + cache with cold fallback | First query pays O_DIRECT cost; all subsequent queries hit cache |

### 21.3 Implications for Future Hypotheses

**H3 (Concurrent multi-query isolation)**: The user-space cache makes this testable. Different queries can maintain separate cache entries without interfering with each other's page cache state. Expected to show O_DIRECT advantage under concurrent load.

**H4 (Memory-pressure stability)**: The cache is immune to `kswapd` eviction because O_DIRECT data never enters the page cache. Under memory pressure (4G cgroup limit vs 29G dataset), mmap performance should degrade as the kernel reclaims pages, while the user-space cache maintains predictable performance.

**H5 (Deterministic P99 latency)**: With the cache, warm query latency is bounded by filter compute time (~80ms for 3.39M rows, ~18ms for 200-col narrow query). There is no tail latency from I/O variance. mmap's P99 depends on system-wide page cache pressure and is inherently less predictable.

### 21.4 Open Questions

1. **Cache eviction policy**: The current `HashMap` cache is unbounded. For multi-file workloads (H4), an LRU or column-frequency-based eviction policy is needed. What is the right cache size for a 4G memory budget?

2. **Filter acceleration**: With I/O eliminated, the filter phase is the bottleneck. Can SIMD-accelerated comparison (`f64 > threshold` on Arrow buffers) reduce the 80ms filter time? What is the theoretical minimum?

3. **Readahead tuning**: The 2.46 MiB gap between mmap (22.57 MiB) and O_DIRECT (20.11 MiB) on 2-column 200-col queries comes from kernel readahead. Can `madvise(MADV_RANDOM)` close this gap for mmap, and at what cost to sequential access patterns?

4. **Hybrid strategy**: A hybrid that uses mmap for cold first queries (overlapped I/O) and switches to cached O_DIRECT for repeated queries could capture the best of both worlds. The switching cost is the one-time cache population from mmap-served data into user-space buffers.

---

*← Previous: [Throughput & Scaling](./04-throughput-and-scaling.md) | [Index](./README.md) | Next: [Research Pivot →](./06-research-pivot.md)*
