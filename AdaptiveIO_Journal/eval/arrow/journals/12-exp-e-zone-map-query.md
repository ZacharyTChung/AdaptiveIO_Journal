# Part 12: Experiment E — Zone-Map Filtered Point Query

*Validation of Workload 1 from [Part 11](./11-workload-recommendations.md). Measures io_uring's advantage in a realistic query pattern where zone-map metadata allows skipping the majority of an Arrow IPC file.*

---

*← Previous: [Part 11: Recommendations](./11-workload-recommendations.md) | [Index](./README.md) | [Next → Part 13](./13-exp-f-multi-file-scan.md) →*

---

## 27. Experiment E: Zone-Map Filtered Point Query

Experiment E validates Workload 1: a query pattern where zone-map metadata (min/max stats per batch) identifies matching batches, leaving a scattered set of small reads across the file.

**Methodology**:
- **File**: Single file (`fhvhv_tripdata_2021-01.arrow`, 2064 MiB, 91 batches).
- **Zone-map**: Built by reading the first and last `i64` from each batch's `pickup_datetime` values buffer.
- **Filter**: Selects batches overlapping the query time range (1 hour, 4 hours, 1 day, 1 week).
- **Reads**: Column offsets collected for selected batches across 3 columns (`pickup_datetime`, `trip_miles`, `base_passenger_fare`), 4 KiB read_size, aligned.
- **Strategies** (3 strategies):
    - `mmap-serial`: Sorted offsets, kernel readahead via `MADV_NORMAL`.
    - `mmap-random`: `MADV_RANDOM` hint, synchronous page faults.
    - `iouring-batched`: Batch all reads via `read_many` in a single submission.
- **Rounds**: 5 cold-cache rounds per configuration.
- **Source**: `results/exp_e_20260328_014003/`

### 27.1 Results: Zone-Map Filtered Reads

| Window | Batches Hit | num_reads | mmap-serial (ns/read) | mmap-random (ns/read) | iouring-batched (ns/read) | EP |
|---|---|---|---|---|---|---|
| 1 hour | 1/91 | 768 | 8,054 | 86,210 | 2,862 | 30.1× |
| 4 hours | 2/91 | 1,536 | 8,089 | 85,849 | 2,954 | 29.1× |
| 1 day | 4/91 | 3,072 | 8,139 | 85,536 | 2,452 | 34.9× |
| 1 week | 21/91 | 16,128 | 8,043 | 86,288 | 2,401 | 35.9× |

*EP = effective_parallelism = mmap-random / iouring-batched. Source: results/exp_e_20260328_014003/*

### 27.2 Analysis

**Finding 23: iouring-batched achieves 30-36× EP over mmap-random.** This far exceeds the predicted 15-23× from Exp A. The improvement comes from real IPC metadata-driven offsets being more amenable to batch submission than Exp A's synthetic selectivity-based offsets.

**Finding 24: EP increases with window size (30.1× → 35.9×).** As the window expands, more reads are submitted in the batch, allowing for better amortization of the submission cost. io_uring per-read cost drops from 2,862 to 2,401 ns/read as the batch size increases from 768 to 16,128.

**Finding 25: mmap-serial achieves 10× EP over mmap-random.** At 8,054 vs 86,210 ns/read, kernel readahead is surprisingly effective. This is because zone-map-filtered offsets are clustered within a few contiguous batches, allowing the kernel to prefetch adjacent pages efficiently even without explicit batching.

**Finding 26: mmap-random is window-invariant at ~86 µs/read.** This confirms the ~85 µs cold-cache page fault cost observed in all prior experiments (Exp A, C, D).

**Contrast with Exp A**: Exp A used synthetic selectivity-based offsets and measured 17-23× EP. Exp E's zone-map-driven offsets achieve 30-36× — a real workload is more favorable to io_uring than the synthetic benchmark.

---

*← Previous: [Part 11: Recommendations](./11-workload-recommendations.md) | [Index](./README.md) | [Next → Part 13](./13-exp-f-multi-file-scan.md) →*
