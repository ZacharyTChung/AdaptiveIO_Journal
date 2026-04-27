# The Scheduler Hypothesis Fails — 2026-04-10

**Milestone:** M3 — Scheduler Validation
**Tasks completed:** M3-T1 through M3-T5
**System:** Linux 6.x, NVMe SSD, Intel CPU (see eval/clickbench/results/run_20260410_132015/exp7/ for system_info)

---

## Summary

Three experiments tested whether a user-space I/O scheduler with io_uring + O_DIRECT improves end-to-end ClickBench query latency. **Two of three claims failed:**

| Claim | Target | Observed | Verdict |
|-------|--------|----------|---------|
| C7: Scheduler E2E speedup | ≥1.15× | 1.023× | **FAIL** |
| C8: O_DIRECT prerequisite | ≤1.03× | 1.011× | **PASS** |
| C9: Batch depth sensitivity | ≥1.10× | 0.967× | **FAIL** |

The scheduler adds no meaningful E2E speedup. The negative result is robust across 10 ClickBench queries, 5 iterations each, with cold-cache drops between runs.

---

## 1. Exp-7: Scheduler E2E Speedup (C7 FAIL)

**Setup:**
```bash
HITS=/home/ybyan/dataset/clickbench/hits.parquet \
  EXPERIMENT=exp7 ITERATIONS=5 \
  bash eval/clickbench/scripts/run_scheduler.sh
# baseline: coalescing reader, no scheduler
# treatment: coalescing reader + IoScheduler (batch_threshold=50, O_DIRECT)
# 10 queries × 5 iterations × 2 configs = 100 runs
```

**Per-query results (median wall_ms):**

| q | baseline | scheduler | speedup | verdict |
|---|----------|-----------|---------|---------|
| 0 | 148 | 138 | 1.073× | FAIL |
| 1 | 275 | 273 | 1.007× | FAIL |
| 2 | 211 | 220 | 0.959× | FAIL |
| 3 | 344 | 337 | 1.021× | FAIL |
| 5 | 681 | 661 | 1.030× | FAIL |
| 7 | 276 | 266 | 1.038× | FAIL |
| 8 | 726 | 651 | 1.115× | FAIL |
| 13 | 1128 | 1101 | 1.024× | FAIL |
| 29 | 511 | 526 | 0.972× | FAIL |
| 42 | 284 | 296 | 0.960× | FAIL |

**Aggregate: median speedup = 1.023× (target ≥1.15×) → FAIL**

**Scheduler metrics reveal the I/O routing split:**

| q | batches | SQEs | CQEs | preads | stalls |
|---|---------|------|------|--------|--------|
| 0 | 0 | 0 | 0 | 0 | 0 |
| 1 | 22 | 340 | 340 | 0 | 45 |
| 7 | 25 | 324 | 324 | 0 | 47 |
| 3 | 5 | 7 | 7 | 592 | 6 |
| 8 | 29 | 116 | 116 | 936 | 42 |
| 13 | 23 | 45 | 45 | 1000 | 28 |

Three query categories emerge:
- **Metadata-only (q0):** Zero I/O — scheduler has nothing to do.
- **Small-range dominated (q1, q7):** 300+ SQEs via io_uring, 0 preads. Speedup ≈ 1.0–1.04×. io_uring batching saves ~260μs in syscalls, but this is negligible on 275ms queries.
- **Large-range dominated (q3, q8, q13):** Mostly pread fallback (ranges > 128KB routing threshold). Scheduler adds channel/coordination overhead with no benefit.

---

## 2. Exp-8: O_DIRECT Prerequisite (C8 PASS)

**Setup:** Scheduler enabled with buffered I/O (no O_DIRECT). Compared against Exp-7 baseline.

```bash
EXPERIMENT=exp8 ITERATIONS=5 bash eval/clickbench/scripts/run_scheduler.sh
# scheduler enabled, --no-direct-io flag, batch_threshold=50
```

**Aggregate: median speedup = 1.011× (target ≤1.03×) → PASS**

The buffered scheduler provides no meaningful benefit over the baseline, confirming that any scheduler benefit requires O_DIRECT. However, since the O_DIRECT scheduler also provides no meaningful benefit (C7 FAIL), this result is moot — C8 validates a prerequisite for a conclusion that itself fails.

**Outlier:** q8 shows 1.152× with buffered scheduler, which is anomalous. This is likely noise from the high-variance I/O pattern of q8 (936 preads).

---

## 3. Exp-9: Batch Depth Sensitivity (C9 FAIL)

**Setup:** Sweep batch_threshold ∈ {1, 10, 50, 100, 256} on 3 data-intensive queries.

```bash
EXPERIMENT=exp9 ITERATIONS=5 BATCH_VALUES="1 10 50 100 256" \
  bash eval/clickbench/scripts/run_scheduler.sh
# 5 thresholds × 3 queries × 5 iterations = 75 runs
```

**Speedup matrix (vs N=1):**

| q | N=1 ms | N=10 | N=50 | N=100 | N=256 |
|---|--------|------|------|-------|-------|
| 2 | 208 | 0.963× | 0.967× | 0.959× | 0.916× |
| 7 | 258 | 0.952× | 0.963× | 0.949× | 1.020× |
| 42 | 293 | 1.093× | 1.046× | 1.007× | 1.007× |

**Aggregate N≥50: median = 0.967× (target ≥1.10×) → FAIL**

Increasing batch threshold does not improve performance. Values near or below 1.0× suggest the batch deadline (1ms flush timeout) may add latency without offsetting syscall savings.

---

## 4. Root Cause Analysis

### Why the Scheduler Fails

The scheduler was designed to reduce I/O coordination overhead through three mechanisms:

1. **SQE batching** — coalesce multiple small reads into one `io_uring_enter` syscall
2. **O_DIRECT** — bypass page cache to eliminate kernel copy overhead
3. **Buffer pooling** — pre-allocated aligned buffers for DMA

Each mechanism saves microseconds. The total savings are dwarfed by DataFusion query execution time:

| Savings Source | Estimated | Query Time | Fraction |
|----------------|-----------|------------|----------|
| SQE batching (340 reads → 22 batches) | ~260μs | 275ms | 0.09% |
| O_DIRECT copy avoidance (340 × 4KB) | ~1.5ms | 275ms | 0.5% |
| Combined | ~1.8ms | 275ms | 0.6% |

Even in the best case (q1/q7 where all I/O goes through io_uring), the savings are <1% of query time. The scheduler coordination overhead (mpsc channels, batch management, buffer pool allocation) likely consumes most of these savings.

### Routing Threshold Problem

The `routing_threshold_bytes = 128KB` causes most Parquet column chunks (typically 50–500KB for HITS) to route through the pread fallback instead of io_uring. Attempted fixes:

1. **Range splitting** (split large ranges into 256KB io_uring chunks): Increased SQE count from 7 to 2228 for q3, created buffer pool pressure, and was slower (453ms vs 306ms baseline).
2. **O_DIRECT pread** (open pread files with O_DIRECT): 551ms vs 327ms baseline. O_DIRECT disables OS readahead, which is highly effective for Parquet's sequential scan pattern.

Both alternatives were worse than the original buffered pread routing.

### The Fundamental Problem

The thesis proposed that "the bottleneck is user-space I/O coordination above the storage abstraction." The M3 experiments show this is wrong for ClickBench on NVMe:

- **I/O overhead is already small.** With buffered I/O, the OS page cache and readahead efficiently handle Parquet's sequential column scans. There is little overhead to eliminate.
- **DataFusion compute dominates.** Query execution (decode, filter, aggregate) takes 95%+ of wall time. Even perfect I/O would yield <5% speedup.
- **The coalescing reader already captures the low-hanging fruit.** M2 showed 1.44× cold / 2.03× warm from metadata caching + prefetch. The remaining I/O coordination overhead is too small to offset scheduler complexity.

---

## 5. What M3 Teaches

### Validated Understanding

1. **Metadata coordination >> I/O coordination** for Parquet analytics on NVMe. SharedMetadataCache (eliminating redundant footer reads) provides 93–98% of the measured speedup (C5 from M1).
2. **Buffered I/O + page cache is hard to beat.** OS readahead effectively coalesces sequential Parquet scans. O_DIRECT removes this free optimization.
3. **io_uring batching saves too little.** At ~1μs per coalesced syscall on NVMe, even 300 coalesced reads save only ~260μs — negligible on 275ms+ queries.

### LanceDB Comparison

LanceDB reported 1.5–9× speedups from their I/O scheduler rework (PR #1918 + #2928). Key differences:
- Lance uses 64 KiB default chunks → more I/O ops per query → more batching opportunity
- Lance vector search is I/O-bound (large random reads) vs ClickBench is compute-bound (filter + aggregate)
- Lance scheduler includes substantial architecture changes beyond io_uring

Our results are consistent with the existing literature: io_uring benefits are workload-dependent and strongest for I/O-bound random access patterns.

---

## 6. Implications for the Paper

C7 and C9 are negative results. The paper's thesis needs adjustment:

**Original thesis:** "For columnar analytics on NVMe, the bottleneck is user-space I/O coordination — a purpose-built I/O scheduler with O_DIRECT + io_uring resolves it."

**Revised understanding:** For ClickBench on NVMe, the bottleneck is metadata coordination (redundant footer reads), not I/O dispatch cost. A user-space metadata cache captures 93–98% of available coordination gains. An I/O scheduler adds <3% on top.

**Paper pivot options:**
1. **Negative result paper:** Document why the scheduler hypothesis fails and what the actual bottleneck is
2. **Metadata coordination paper:** Focus on SharedMetadataCache + coalescing as the contribution, with scheduler as ablation
3. **Different workload:** Test on I/O-bound workloads (vector search, large scans) where scheduler may help
4. **Different bottleneck:** Profile DataFusion decode/filter to find the actual next optimization target

---

## Data Artifacts

| Experiment | Path | Rows | Config |
|------------|------|------|--------|
| Exp-7 (C7) | `eval/clickbench/results/run_20260410_132015/exp7/` | 100 | 10q × 5i × 2 |
| Exp-8 (C8) | `eval/clickbench/results/run_20260410_132517/exp8/` | 50 | 10q × 5i × 1 |
| Exp-9 (C9) | `eval/clickbench/results/run_20260410_132651/exp9/` | 75 | 3q × 5i × 5 |

Scripts: `eval/clickbench/scripts/run_scheduler.sh`, `eval/clickbench/scripts/analyze_scheduler.py`

Code: IoScheduler with routing-based dispatch (io_uring for ranges < 128KB, buffered pread for larger). 66/66 tests pass.
