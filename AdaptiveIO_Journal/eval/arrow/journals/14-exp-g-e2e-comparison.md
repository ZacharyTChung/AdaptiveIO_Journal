← [Part 13](./13-exp-f-multi-file-scan.md) | [Index](./README.md) | [Next →](./15-exp-h1-wide-table.md)

**§29. Experiment G: End-to-End Zone-Map Query Comparison**

Methodology: First apples-to-apples benchmark comparing 3 complete read-decode-filter pipelines on the same zone-map filtered query:
- **vanilla**: Arrow `FileReader` with column projection + `set_index()` batch selection → `RecordBatch` → `count_matching()`. Reads full batch messages (all columns in selected batches).
- **mmap**: memmap2 + manual column buffer extraction via `ipc_metadata` APIs + `construct_record_batch()` → `count_matching()`. Column projection at I/O level.
- **iouring**: O_DIRECT `DirectFile` + `read_many()` batch submission of column buffers + `construct_record_batch()` → `count_matching()`. Column projection at I/O level with batched I/O.

Query: `SELECT pickup_datetime, trip_miles, base_passenger_fare WHERE trip_miles > 5.0` with time-range zone-map filter on `pickup_datetime`. Dataset: fhvhv_tripdata_2021-01.arrow (2064 MiB, 91 batches). Cold cache (posix_fadvise + drop_caches) before each round.

**§29.1 End-to-End Performance**

Source: `results/exp_g_20260403_233911/` (20 files, 5-round averages)

| Window | Batches | vanilla total (ms) | mmap total (ms) | iouring total (ms) | vanilla/iouring | mmap/iouring |
|---|---|---|---|---|---|---|
| 1h | 1/91 | 42.9 | 8.0 | 4.5 | 9.5× | 1.79× |
| 4h | 2/91 | 80.9 | 16.2 | 9.0 | 9.0× | 1.80× |
| 1d | 4/91 | 159.5 | 33.4 | 18.5 | 8.6× | 1.81× |
| 1w | 21/91 | 797.5 | 170.3 | 102.0 | 7.8× | 1.67× |

*Units: milliseconds (total wall time including load + decode + filter). 5-round cold-cache averages.*

**§29.2 Phase Breakdown**

| Window | Strategy | load (ms) | decode (ms) | filter (ms) | I/O (MiB) | Page Faults |
|---|---|---|---|---|---|---|
| 1h | vanilla | 42.0 | 0.0 | 1.0 | 23.1 | 5,831 |
| 1h | mmap | 6.8 | 0.3 | 0.9 | 3.9 | 849 |
| 1h | iouring | 2.2 | 1.4 | 1.0 | 3.0 | 774 |
| 4h | vanilla | 78.9 | 0.0 | 2.0 | 45.8 | 11,652 |
| 4h | mmap | 13.9 | 0.3 | 2.0 | 7.7 | 1,695 |
| 4h | iouring | 4.1 | 2.9 | 2.0 | 6.0 | 1,548 |
| 1d | vanilla | 155.1 | 0.0 | 4.4 | 91.3 | 23,285 |
| 1d | mmap | 28.8 | 0.3 | 4.3 | 15.2 | 3,364 |
| 1d | iouring | 8.1 | 6.2 | 4.2 | 12.0 | 3,096 |
| 1w | vanilla | 768.4 | 0.0 | 29.2 | 477.6 | 122,183 |
| 1w | mmap | 147.1 | 0.3 | 22.9 | 78.9 | 17,742 |
| 1w | iouring | 43.5 | 36.0 | 22.5 | 63.2 | 16,682 |

**§29.3 Analysis**

Findings:

- **Finding 31: io_uring is 7.8-9.5× faster than vanilla Arrow FileReader end-to-end.** Vanilla reads full batch messages (all columns), while iouring reads only the 3 projected columns. At 1h (1 batch), vanilla reads 23.1 MiB vs iouring's 3.0 MiB — a 7.7× I/O reduction.

- **Finding 32: io_uring is 1.67-1.81× faster than mmap end-to-end.** Both use column projection. The advantage comes entirely from the load phase: iouring load=2.2ms vs mmap load=6.8ms at 1h (3.1× I/O speedup). This is smaller than Exp E's 30× because column buffers are ~1 MiB each (near the ~128 KiB crossover).

- **Finding 33: Decode cost is the asymmetry.** Vanilla's decode=0ms (Arrow FileReader decodes during load). mmap's decode=0.3ms (near-zero with page cache). iouring's decode=1.4-36ms (O_DIRECT buffers must be copied into Arrow arrays). At 1w, iouring's decode (36ms) nearly equals its load (43.5ms), compressing the end-to-end advantage.

- **Finding 34: Filter time is strategy-independent** (~1ms at 1h, ~29ms at 1w). This confirms the benchmark is apples-to-apples: all strategies produce identical RecordBatch data for filtering.

- **Finding 35: The practical speedup over vanilla (7.8-9.5×) comes from two sources: column projection (7.7× I/O reduction) and batched I/O (3.1× faster per-byte load).** Column projection alone (mmap) gives 5.4-4.7× over vanilla. Adding io_uring batch submission gives an additional 1.7-1.8×.

← [Part 13](./13-exp-f-multi-file-scan.md) | [Index](./README.md) | [Next →](./15-exp-h1-wide-table.md)
