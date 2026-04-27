# Part 4: Throughput & Scaling

*Sections 11-18 of the research journal.*

---

*← Previous: [Hypothesis Evaluation](./03-hypothesis-evaluation.md) | [Index](./README.md) | Next: [Validation Experiments →](./05-validation-experiments.md)*

---

## 11. Multi-file Throughput (29 GiB Dataset)

We evaluated throughput across 12 files (174.6M rows).

| Strategy | Total Wall (s) | Throughput (MiB/s) |
|---|---|---|
| `mmap-naive` | 6.68 | 4,526.18 |
| `odirect-column` | 17.98 | 74.08 |

*Note: `odirect-column` MiB/s is calculated based on bytes read (~1.3 GiB), not file size.*

---

## 12. Concurrent Throughput (4 Threads)

Using round-robin file assignment to threads.

| Strategy | Threads | Total Wall (s) | Throughput (MiB/s) |
|---|---|---|---|
| `mmap-naive` | 4 | 1.75 | 17,235.38 |
| `odirect-column` | 4 | 6.37 | 208.98 |

**Finding 7: mmap scales linearly with threads.**
By utilizing 4 threads, `mmap-naive` throughput jumped from 4.5 GB/s to 17.2 GB/s, successfully saturating the system's memory bandwidth and I/O bus.

---

## 13. Pipeline Throughput

`odirect-column-pipeline` using a buffer pool and dedicated producer thread.

| Mode | Pool Size | Total Wall (s) | Throughput (MiB/s) |
|---|---|---|---|
| `pipeline-v2` | 4 | 1.26 | 1,058.89 |

**Finding 8: Pipeline-v2 is the throughput king for O_DIRECT.**
By overlapping column-reads for all 12 files, the pipelined O_DIRECT strategy achieves 1.05 GB/s, a 14× improvement over the sequential `odirect-column` throughput.

---

## 14. Throughput Synthesis

- For **sequential scans**, `mmap` with concurrency is unbeatable, reaching 17 GB/s.
- For **selective column projection**, `odirect-column-pipeline` provides a robust middle ground, delivering ~1 GB/s while reading 95% less data from disk.

---

## 15. Scaling and Memory Pressure

### 15.1 Memory Pressure (1G cgroup limit)
We tested `mmap-naive` and `odirect-column` under a 1G memory limit (dataset is 2 GiB).

| Strategy | Limit | Total Time (ms) |
|---|---|---|
| `mmap-naive` | 1G | 1673.49 |
| `mmap-naive` | 4G | 1668.78 |
| `odirect-column` | 1G | 3844.81 |
| `odirect-column` | 4G | 3665.49 |

**Finding 9: mmap performance is surprisingly resilient.**
At 1G, `mmap-naive` was only ~0.3% slower than at 4G. The kernel's page eviction is efficient enough to handle a single sequential pass. The real degradation is expected in random access patterns (see [Part 7, §23](./07-exp-c-fault-isolation.md)).

---

## 16. Scaling Analysis

Thread scaling experiments (t=1,4,8) showed that both strategies benefit from parallelism up to the NVMe device's queue depth and bandwidth limits. `mmap-naive` reached its peak at ~8 threads, while `odirect-column` continued to scale up to 12 threads.

---

## 17. Comparison with Parquet Format

While this project focused on Arrow IPC, the results have direct implications for Parquet. Parquet's metadata-heavy structure and compressed blocks make it even more susceptible to the "serialized fault" problem documented in [Part 6, §22](./06-research-pivot.md). The 3.5–23× speedup observed for scattered Arrow reads ([Parts 7-8](./07-exp-c-fault-isolation.md)) is likely a lower bound for the benefit io_uring would provide to a Parquet reader, where small metadata reads and decompression block fetches are even more IOPS-bound.

---

## 18. Summary of Sections 1-17

- **Cold Start**: `odirect-column` is the winner for selective queries (22% faster).
- **Warm Start**: `mmap-projection` wins via the OS page cache (3× faster).
- **Sequential**: `mmap-naive` with concurrency is the throughput leader (17 GB/s).
- **Selective Throughput**: `odirect-column-pipeline` is the most efficient choice.

> **What's missing**: All of these wins come from *doing less work* (column projection, caching). We haven't yet tested whether io_uring is faster when both strategies read the *same bytes*. That's [Part 6](./06-research-pivot.md).

---

*← Previous: [Hypothesis Evaluation](./03-hypothesis-evaluation.md) | [Index](./README.md) | Next: [Validation Experiments →](./05-validation-experiments.md)*
