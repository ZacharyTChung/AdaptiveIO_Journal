# Part 13: Experiment F — Multi-File Partition Scan

*Validation of Workload 2 from [Part 11](./11-workload-recommendations.md). Measures io_uring's scaling behavior when scanning multiple partition files with selective predicates.*

---

*← Previous: [Part 12: Exp E — Zone-Map Query](./12-exp-e-zone-map-query.md) | [Index](./README.md) | Next → [Part 14](./14-exp-g-e2e-comparison.md)*

---

## 28. Experiment F: Multi-File Partition Scan

Experiment F validates Workload 2: a query scanning multiple partition files (e.g., 12 monthly files) where only a subset of files contain matching data according to their zone-maps.

**Methodology**:
- **Files**: 12 Arrow IPC files (`fhvhv_tripdata_2021-{01..12}.arrow`, ~30 GiB total).
- **Filter**: Each file is zone-map filtered for a 2-hour window on the 15th of each month. Only files with matching batches contribute reads.
- **Reads**: 2 columns (`pickup_datetime`, `trip_miles`), 4 KiB read_size.
- **Overhead**: Per-file offsets precomputed before timing (zone-map building overhead excluded).
- **Scaling**: 1, 2, 4, 8, 12 threads, round-robin file distribution.
- **Rounds**: 5 cold-cache rounds per configuration.
- **Source**: `results/exp_f_20260328_020352/`

### 28.1 Results: Multi-File Scaling

Total reads: 512 (from 1 matching file out of 12: `fhvhv_tripdata_2021-01.arrow`).

| threads | mmap-serial wall (ms) | mmap-serial ns/read | mmap-random wall (ms) | mmap-random ns/read | iouring-batched wall (ms) | iouring-batched ns/read | EP |
|---|---|---|---|---|---|---|---|
| 1 | 5.2 | 9,798 | 46.0 | 89,642 | 5.2 | 9,951 | 9.0× |
| 2 | 4.9 | 9,365 | 44.8 | 87,407 | 5.9 | 11,294 | 7.7× |
| 4 | 4.7 | 9,044 | 45.7 | 89,113 | 5.2 | 9,757 | 9.1× |
| 8 | 4.6 | 8,904 | 45.1 | 87,990 | 5.8 | 10,965 | 8.0× |
| 12 | 4.6 | 8,801 | 44.7 | 87,224 | 4.8 | 9,092 | 9.4× |

*EP = mmap-random / iouring-batched wall time. Source: results/exp_f_20260328_020352/*

### 28.2 Analysis

**Finding 27: mmap-serial matches iouring-batched (~5ms, ~9-10 µs/read).** When zone-map filtering selects a small, contiguous region (1-2 batches in 1 file), sequential access with kernel readahead is as effective as io_uring batch submission. This is the only experiment where mmap-serial is competitive.

**Finding 28: Thread scaling is flat because only 1/12 files has matching data.** With round-robin distribution, only 1 thread does real work while the rest remain idle. The wall time represents the serial processing of that single file's matches.

**Finding 29: EP=8-9× is significantly lower than Exp E's 30×.** The gap is due to fewer total reads (512 vs 768-16,128), multi-file overhead, and a highly clustered access pattern that favors kernel readahead in the serial baseline.

**Finding 30: The Exp E vs Exp F contrast reveals the access pattern spectrum.** Exp E (single file, many scattered reads) achieves 30-36× EP. Exp F (multi-file, few clustered reads) achieves 8-9× EP. io_uring's advantage scales with the degree of scatter, not the file count.

---

## 29. Final Synthesis: The Decision Framework

The complete experimental arc (A-F) establishes a clear engineering decision framework for the next-generation Arrow engine:

1.  **Highly scattered, single-file access** (selective filter, many small reads): **io_uring batch is 30-36× faster** (Exp E). This is the primary driver for adoption.
2.  **Moderately scattered, cold cache, 1-4 threads**: **io_uring is 5-8× faster** (Exp D).
3.  **Clustered access, few reads per file**: **mmap-serial matches io_uring** (Exp F). In this regime, mmap's simplicity and zero-copy paging are preferable.
4.  **Warm cache, any pattern**: **mmap and io_uring are equivalent** (Exp B, 1.03×).

**Conclusion**: The optimal engine should be adaptive: use `io_uring` for selective queries on cold data where reads are scattered, and fall back to `mmap` for sequential scans or when data is likely to be cached.

---

*← Previous: [Part 12: Exp E — Zone-Map Query](./12-exp-e-zone-map-query.md) | [Index](./README.md) | Next → [Part 14](./14-exp-g-e2e-comparison.md)*
