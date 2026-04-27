# Journal 21: Anchoring Experiment Results

**Status**: Complete
**Predecessor**: §35–§43 (Journal 20 — Anchoring Experiment Plan)
**Run ID**: run_20260409_072628
**Scope**: End-to-end validation of io_uring ObjectStore on ClickBench/DataFusion

---

## §44 — Experimental Setup

*   **Hardware**: Intel Xeon Gold 6530 (62 cores), 251 GiB RAM, Micron 7450 NVMe SSD, Linux 6.8
*   **Software**: DataFusion 47, arrow/parquet 55, object_store 0.12, io-uring 0.7, kernel 6.8
*   **Dataset**: ClickBench `hits.parquet` (14.78 GiB, 226 row groups, 105 columns)
*   **Queries**: Subset of 10 queries (0, 1, 2, 3, 5, 7, 8, 13, 29, 42) covering narrow/wide column scans and aggregations
*   **Method**: 5 iterations per (query × store × mode), warm and cold cache variants, cold via `sync && echo 3 > /proc/sys/vm/drop_caches`

### §44.1 Parquet Metadata Characterization

Key numbers from metadata analysis:
*   226 row groups × 105 columns = 23,730 total column chunks
*   **Size histogram**:
    | Size Range | Chunk Count |
    |---|---|
    | <4 KiB | 8,410 |
    | 4–16 KiB | 2,023 |
    | 16–64 KiB | 3,744 |
    | 64–128 KiB | 2,727 |
    | 128–512 KiB | 3,391 |
    | 512 KiB – 2 MiB | 1,965 |
    | >2 MiB | 1,470 |
*   **59.7% of chunks are <64 KiB** (narrow numeric/flag columns)
*   Wide columns (URL, Title, Referer, SearchPhrase) have p95 chunk sizes 20–33 MiB
*   Narrow columns (IDs, flags, booleans) have p95 chunk sizes <128 KiB
*   Phase 0 regime prediction (p95 < 128 KiB → io_uring_favorable) classified ~40% of columns as favorable

---

## §45 — Phase 0: I/O Tracing

### §45.1 Unexpected Finding: Parquet Footer Storm

Trace analysis revealed every query reads the **same 2.4 MiB Parquet footer 63 times** (once per partition per worker), plus 63 footer-length probes (8 bytes each). Total: **126 redundant footer reads = ~153.7 MiB of wasted I/O per query**. This was constant across all queries regardless of data reads.

**Root cause**: DataFusion's `ParquetExec` creates one `ParquetObjectReader` per partition (target_partitions ≈ num_cpus = 62–63), and each independently reads the footer. Page cache absorbs the cost after the first read, but syscall count is still 63x.

### §45.2 Get_ranges Batch Size Analysis

Critical finding: DataFusion calls `ObjectStore::get_ranges()` with **tiny batches** (1–5 ranges per call):

| Query | get_ranges calls | Max batch | Median batch |
|---|---|---|---|
| q1 (COUNT WHERE AdvEngineID<>0) | 212 | 1 | 1 |
| q2 (SUM/AVG/COUNT) | 226 | 2 | 2 |
| q3 (AVG UserID) | 226 | 1 | 1 |
| q42 (date aggregate) | 3 | 5 | 5 |

This is the **fundamental limitation**: io_uring's main advantage is batched SQE submission. With batch size = 1, each `get_ranges()` call creates a fresh ring with one SQE—degenerating to async pread.

### §45.3 Per-Query Data I/O Profile

| Query | data_MB | footer_MB | small_pct | data_ranges | Predicted Regime |
|---|---|---|---|---|---|
| q0 | 0.0 | 2.4 | 0.0% | 0 | neutral_or_unfavorable |
| q1 | 0.9 | 153.7 | 100.0% | 212 | strongly_favorable |
| q2 | 42.0 | 153.7 | 65.9% | 452 | strongly_favorable |
| q3 | 270.1 | 153.7 | 1.8% | 226 | neutral_or_unfavorable |
| q5 | 373.1 | 153.7 | 8.8% | 226 | neutral_or_unfavorable |
| q7 | 0.9 | 153.7 | 100.0% | 212 | strongly_favorable |
| q8 | 328.6 | 153.7 | 12.6% | 452 | neutral_or_unfavorable |
| q13 | 643.2 | 153.7 | 5.3% | 452 | neutral_or_unfavorable |
| q29 | 41.1 | 153.7 | 31.9% | 226 | neutral_or_unfavorable |
| q42 | 4.8 | 153.7 | 80.0% | 15 | weakly_favorable |

---

## §46 — Phase 2: Benchmark Results

### §46.1 Protocol
*   10 queries × 2 stores (local LocalFileSystem, iouring IoUringStore) × 5 iterations = 100 runs per mode
*   Row counts verified identical between stores (correctness ✓)
*   Reported: inner wall_ms (DataFusion query execution), Welch t-statistic for significance

### §46.2 Warm Cache Results

| Q | local_med | iour_med | speedup | delta_ms | t_stat | verdict |
|---|---|---|---|---|---|---|
| 0 | 48 | 46 | 1.043x | +2 | 1.49 | NEUTRAL |
| 1 | 455 | 439 | 1.036x | +16 | 1.04 | NEUTRAL |
| 2 | 411 | 376 | 1.093x | +35 | 3.85 | FASTER |
| 3 | 449 | 432 | 1.039x | +17 | 1.61 | NEUTRAL |
| 5 | 672 | 682 | 0.985x | -10 | -0.73 | NEUTRAL |
| 7 | 414 | 430 | 0.963x | -16 | -0.09 | NEUTRAL |
| 8 | 676 | 683 | 0.990x | -7 | -0.57 | NEUTRAL |
| 13 | 895 | 944 | 0.948x | -49 | -3.55 | SLOWER |
| 29 | 696 | 626 | 1.112x | +70 | 3.66 | FASTER |
| 42 | 450 | 430 | 1.047x | +20 | 1.84 | NEUTRAL |

**Summary stats**: Median speedup: 1.038x, Range: 0.948x–1.112x.

### §46.3 Cold Cache Results

| Q | local_med | iour_med | speedup | delta_ms | t_stat | verdict |
|---|---|---|---|---|---|---|
| 0 | 100 | 101 | 0.990x | -1 | -1.07 | NEUTRAL |
| 1 | 527 | 511 | 1.031x | +16 | 1.74 | NEUTRAL |
| 2 | 502 | 450 | 1.116x | +52 | 4.11 | FASTER |
| 3 | 593 | 548 | 1.082x | +45 | 4.28 | FASTER |
| 5 | 769 | 831 | 0.925x | -62 | -2.92 | SLOWER |
| 7 | 486 | 493 | 0.986x | -7 | 0.06 | NEUTRAL |
| 8 | 732 | 760 | 0.963x | -28 | -1.56 | NEUTRAL |
| 13 | 1056 | 1138 | 0.928x | -82 | -3.12 | SLOWER |
| 29 | 768 | 692 | 1.110x | +76 | 4.64 | FASTER |
| 42 | 485 | 483 | 1.004x | +2 | 1.29 | NEUTRAL |

**Summary stats**: Median speedup: 0.997x, Range: 0.925x–1.116x.

### §46.4 Statistical Significance

Welch t-statistics for key results:
*   **q2 cold**: t=+4.11 (strongly significant faster)
*   **q3 cold**: t=+4.28 (strongly significant faster)
*   **q29 cold**: t=+4.64 (strongly significant faster)
*   **q5 cold**: t=-2.92 (significant slower)
*   **q13 cold**: t=-3.12 (significant slower)

### §46.5 I/O Time vs CPU Time Decomposition

For most queries, (cold\_ms − warm\_ms) ≈ disk I/O time, with the remainder being CPU-bound (decode + filter + aggregate).

| Query | Cold (ms) | Warm (ms) | I/O Delta (ms) | I/O % |
|---|---|---|---|---|
| q1 | 527 | 455 | 72 | 13.7% |
| q2 | 502 | 411 | 91 | 18.1% |
| q3 | 593 | 449 | 144 | 24.3% |
| q5 | 769 | 672 | 97 | 12.6% |
| q13 | 1056 | 895 | 161 | 15.2% |
| q29 | 768 | 696 | 72 | 9.4% |

Even if io_uring eliminated all I/O latency, maximum possible speedup would be 1.1x–1.3x bounded by Amdahl's Law.

---

## §47 — Analysis & Conclusions

### §47.1 Phase 0 Prediction Failure

The Phase 0 regime prediction based on chunk size distribution **did not correlate** with measured speedup:
*   Pearson r(small_pct, warm_speedup) = +0.125 (weak)
*   Pearson r(small_pct, cold_speedup) = +0.242 (weak)

The prediction assumed: many small reads → io_uring batching advantage. But DataFusion's Parquet reader calls `get_ranges()` with batch size 1–5, so **io_uring batching never activates**. Each call is effectively "io_uring with a single SQE" ≈ async pread.

### §47.2 The Real Signal

A different metric showed moderate correlation:
*   Pearson r(total_data_mb, cold_speedup) = **-0.556**

Interpretation: **smaller data volume → better iouring relative performance; large sequential scans → pread+kernel readahead wins**. The Micron 7450 NVMe's hardware readahead engine and the kernel's `read_ahead_kb` optimize sequential pread to ~5+ GB/s, which io_uring cannot beat for streaming large chunks.

### §47.3 Concrete Winners and Losers

*   **Winners** (iouring ≥ +8%): q2, q3, q29 — all have moderate data volumes (40–270 MB) and mixed column widths.
*   **Losers** (iouring ≤ -5%): q5 (373 MB, wide SearchPhrase), q13 (643 MB, wide SearchPhrase).
*   **Clear pattern**: io_uring hurts wide-column full-scan queries where readahead is dominant.

### §47.4 Hypothesis Revision

Original hypothesis (from §34.1 regime map): io_uring batching gives 1.3–1.8× speedup for scattered small reads. **This does not hold** in DataFusion's Parquet stack because:

1.  **ParquetObjectReader issues size-1 batches**: The Parquet async reader fetches column chunks one at a time per row group partition, not as a coalesced batch. The `get_byte_ranges()` API is called with len=1 in practice.
2.  **Footer re-reads dominate I/O volume**: 153 MiB of redundant footer reads swamp actual data reads for narrow-column queries.
3.  **Kernel readahead beats io_uring for streaming**: Large sequential column chunks (>1 MiB) read faster via pread+readahead than via io_uring submission.
4.  **I/O is not the bottleneck for most queries**: Warm baseline shows 400–900 ms wall time, meaning CPU decode/filter dominates over disk I/O (typically 50–200 ms).

### §47.5 Actionable Findings

For io_uring's promised gains to materialize in this stack, upstream changes are needed:
1.  **Batch get_ranges in ParquetObjectReader**: Collect all column chunks for a row group into a single `store.get_ranges()` call with Vec<Range> of length N columns.
2.  **Cache the Parquet footer**: Share the footer across partitions instead of re-reading it 63x.
3.  **Use adaptive backend**: io_uring for scattered narrow-column reads (<128 KiB), pread for large sequential reads (>1 MiB).

### §47.6 Final Verdict

**The naive `LocalFileSystem → IoUringStore` swap in DataFusion yields no systematic benefit on ClickBench.** The median cold-cache speedup is 0.997x (essentially neutral), with 3 queries faster, 2 slower, and 5 neutral. The anchoring experiment **does not replicate** the earlier journal findings (1.3–1.8× in microbenchmarks) because the DataFusion layer neutralizes io_uring's batching advantage.

This is a **valuable negative result**: the Parquet reader's access pattern matters more than the underlying I/O backend. Future work should target batch consolidation in the Parquet reader itself, then revisit the io_uring comparison.

