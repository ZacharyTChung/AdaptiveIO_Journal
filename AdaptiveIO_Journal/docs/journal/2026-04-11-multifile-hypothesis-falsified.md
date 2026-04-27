# Journal 27: Multi-file Hypothesis Falsified

**Date:** 2026-04-11
**Milestone:** M4 — Multi-file Partitioned Benchmark
**Verdict:** C11 ❌ FAIL (0.985×) · C12 ❌ FAIL (no monotonic trend) · C13 ❌ FAIL (0.97×)

## Summary

| Experiment | Claim | Target | Result | Verdict |
|-----------|-------|--------|--------|---------|
| Exp-11: 100-file ClickBench A/B | C11 | ≥1.3× | 0.985× | ❌ FAIL |
| Exp-12: File count sweep (N=1..226) | C12 | Monotonic + ≥1.3× at N≥100 | Flat ~1.0× at N≥10 | ❌ FAIL |
| Exp-13: TPC-H SF=10 A/B | C13 | ≥1.3× | 0.97× | ❌ FAIL |

All three M4 claims fail. Combined with M3 (C7=1.023×), the IoScheduler delivers no measurable E2E benefit under ANY tested regime — single-file, multi-file split, or standard TPC-H benchmark. The multi-file hypothesis is falsified.

## Experiment Setup (common)

```bash
# Build
cargo build --release --manifest-path eval/clickbench/Cargo.toml

# Baseline: CoalescingReader only (no scheduler)
cargo run --release -- bench --directory data/hits_N100 --factory coalescing --store tracing --no-direct-io

# Treatment: CoalescingReader + IoScheduler
cargo run --release -- bench --directory data/hits_N100 --factory coalescing --store tracing --scheduler --no-direct-io

# TPC-H mode
cargo run --release -- bench --tpch-dir data/tpch_sf10 --factory coalescing --store tracing [--scheduler] --no-direct-io
```

- 5 iterations per query-config pair, cold cache (`drop_caches` between iterations)
- Data: HITS 14.78 GiB (99,997,497 rows, 226 row groups), TPC-H SF=10 lineitem 2.4 GiB (59,986,052 rows, 524 row groups)
- Partitioned data created via `partition_hits.py` (round-robin by row group)
- TPC-H lineitem split into 100 flat files via same script

## Exp-11: Split ClickBench 100-file A/B (C11)

100-file partition of HITS. 10 queries × 5 iterations × 2 configs = 100 runs.

### Per-query Results

| Query | Baseline (ms) | Scheduler (ms) | Speedup | Verdict |
|-------|--------------|----------------|---------|---------|
| Q0  | 71  | 38   | 1.868× | ✅ PASS |
| Q1  | 87  | 71   | 1.225× | ❌ FAIL |
| Q2  | 134 | 132  | 1.015× | ❌ FAIL |
| Q3  | 249 | 261  | 0.954× | ❌ FAIL |
| Q5  | 564 | 900  | 0.627× | ❌ FAIL |
| Q7  | 95  | 135  | 0.704× | ❌ FAIL |
| Q8  | 498 | 597  | 0.834× | ❌ FAIL |
| Q13 | 862 | 1452 | 0.594× | ❌ FAIL |
| Q29 | 417 | 403  | 1.035× | ❌ FAIL |
| Q42 | 143 | 97   | 1.474× | ✅ PASS |

**Aggregate median speedup: 0.985×** (threshold ≥1.3×)

Only 2/10 queries pass individually. Q13 and Q5 show severe regressions (0.59×, 0.63×).

### Scheduler Metrics

| Query | SQEs | Preads | Stalls | SQE/(SQE+Pread) |
|-------|------|--------|--------|------------------|
| Q0  | 0    | 0      | 0      | — (never triggered) |
| Q1  | 292  | 0      | 25     | 100% io_uring |
| Q2  | 443  | 233    | 80     | 66% io_uring |
| Q3  | 7    | 356    | 7      | 2% io_uring |
| Q5  | 24   | 316    | 22     | 7% io_uring |
| Q7  | 309  | 0      | 65     | 100% io_uring |
| Q8  | 79   | 623    | 36     | 11% io_uring |
| Q13 | 30   | 652    | 28     | 4% io_uring |
| Q29 | 86   | 191    | 54     | 31% io_uring |
| Q42 | 32   | 8      | 22     | 80% io_uring |

**Key observation:** Most queries are pread-dominated. The scheduler routes >60% of reads to pread fallback because column chunks exceed the 128 KiB threshold. Even Q7 (100% io_uring) shows 0.70× — pure scheduler thread contention overhead.

## Exp-12: File Count Sweep (C12)

N∈{1,10,50,100,226} × 10 queries × 5 iterations × 2 configs = 500 runs.

### Speedup vs File Count

| N | Median Speedup | Best Query | Worst Query |
|---|---------------|------------|-------------|
| 1   | 1.485× | Q2 3.73× | Q42 0.94× |
| 10  | 1.015× | Q0 1.89× | Q13 0.64× |
| 50  | 1.011× | Q0 1.91× | Q13 0.60× |
| 100 | 1.010× | Q0 1.87× | Q13 0.59× |
| 226 | 0.994× | Q0 1.86× | Q13 0.59× |

**NOT monotonic.** N=1 is an anomalous outlier (driven by Q2=3.73× and Q3=3.02×). At N≥10, speedup is flat at ~1.0× with no scaling trend.

### Scheduler Metrics vs File Count

| N | SQEs | Preads | Stalls |
|---|------|--------|--------|
| 1   | 72 | 202 | 28 |
| 10  | 70 | 205 | 30 |
| 50  | 69 | 205 | 29 |
| 100 | 50 | 198 | 22 |
| 226 | 37 | 157 | 18 |

**Critical finding:** Scheduler metrics are nearly IDENTICAL across all N values. Increasing file count does NOT change the I/O pattern seen by the scheduler. DataFusion's CoalescingReader produces the same read sizes regardless of how many files the data is split across.

### The N=1 Anomaly

N=1 shows 1.485× while all other N values show ~1.0×. This is NOT a scheduler benefit — it's a DataFusion planning artifact:
- Q2 jumps from 132ms (N=100) to 36ms (N=1 scheduler) = 3.73×
- Q3 jumps from 261ms (N=100) to 86ms (N=1 scheduler) = 3.02×
- These queries likely benefit from simpler scan planning on a single file (fewer file groups, less coordination overhead)
- The "speedup" is from reduced DataFusion overhead, not scheduler-accelerated I/O

## Exp-13: TPC-H SF=10 A/B (C13)

100 lineitem files, 8 TPC-H queries × 5 iterations × 2 configs = 80 runs.

### Per-query Results

| Query | Baseline (ms) | Scheduler (ms) | Speedup | Verdict |
|-------|--------------|----------------|---------|---------|
| Q1  | 742  | 1052 | 0.705× | ❌ FAIL |
| Q3  | 959  | 1082 | 0.886× | ❌ FAIL |
| Q5  | 1250 | 1263 | 0.990× | ❌ FAIL |
| Q6  | 551  | 591  | 0.932× | ❌ FAIL |
| Q10 | 1149 | 1104 | 1.041× | ❌ FAIL |
| Q12 | 906  | 835  | 1.085× | ❌ FAIL |
| Q14 | 809  | 859  | 0.942× | ❌ FAIL |
| Q19 | 1174 | 1003 | 1.171× | ❌ FAIL |

**Aggregate median speedup: 0.97×** (threshold ≥1.3×)

No query passes individually. Q1 (0.71×) and Q3 (0.89×) show significant regressions.

### Scheduler Metrics

| Query | SQEs | Preads | Stalls | SQE/(SQE+Pread) |
|-------|------|--------|--------|------------------|
| Q1  | 3885 | 1554 | 536  | 71% io_uring |
| Q3  | 923  | 2769 | 327  | 25% io_uring |
| Q5  | 896  | 2688 | 365  | 25% io_uring |
| Q6  | 1684 | 1684 | 262  | 50% io_uring |
| Q10 | 1722 | 1722 | 395  | 50% io_uring |
| Q12 | 872  | 3488 | 215  | 20% io_uring |
| Q14 | 865  | 2595 | 342  | 25% io_uring |
| Q19 | 3256 | 1628 | 519  | 67% io_uring |

TPC-H produces MUCH higher I/O volumes than split ClickBench (Q1: 3885 SQEs + 1554 preads = 5439 total ops vs ClickBench Q1: 292 SQEs). Despite this heavier I/O workload, the scheduler still fails to deliver benefit. Even Q1 with 71% io_uring routing and 3885 SQEs shows 0.71× — the highest io_uring utilization correlates with the worst regression.

## Cross-Experiment Synthesis

### The Complete Picture (M3 + M4)

| Experiment | Regime | Speedup | io_uring % | Verdict |
|-----------|--------|---------|------------|---------|
| Exp-7 (M3) | Single file, 226 RGs | 1.023× | Variable | ❌ FAIL |
| Exp-11 (M4) | 100 files, ~2 RGs/file | 0.985× | 2-100% | ❌ FAIL |
| Exp-12 (M4) | 1-226 files, sweep | 1.0-1.5× | Similar | ❌ FAIL |
| Exp-13 (M4) | 100 files, TPC-H joins | 0.97× | 20-71% | ❌ FAIL |

**No tested regime produces scheduler benefit.** The hypothesis that multi-file workloads create the right I/O pattern for io_uring batching is false.

### Why Multi-file Doesn't Help

The multi-file hypothesis rested on a chain of reasoning:

1. ✅ More files → more file descriptors → more scattered I/O opportunities
2. ❌ More files → smaller reads per file → reads below 128 KiB threshold
3. ❌ Smaller reads → io_uring routing → batching benefit

**Link 2 is broken.** DataFusion's Parquet reader reads entire column chunks (row_group × column = one read). Column chunk size is determined by:
- Number of rows per row group (fixed at write time)
- Column data type and encoding (data property)

Neither factor depends on file count. Splitting 226 row groups across 100 files gives ~2 RGs per file, but each RG's column chunks are exactly the same size as in the single-file case. The Exp-12 scheduler metrics confirm this — SQEs and preads are nearly identical from N=1 to N=226.

**Link 3 is broken even when Link 2 holds.** Q7 in Exp-11 achieves 100% io_uring routing (all reads < 128 KiB), yet shows 0.70× — the scheduler thread (MPSC channel + io_uring submission) adds more latency than it saves through batching. Q1 in Exp-13 has 71% io_uring and 3885 SQEs — the highest volume — yet shows 0.71×.

### The Fundamental Mismatch

The IPC benchmark (Journal 22) showed io_uring benefits at the raw I/O layer: `io_uring_read` vs `pread` on many small aligned buffers shows 1.44×. But this benefit cannot propagate through the Parquet reader because:

1. **Read granularity mismatch:** IPC uses fixed-size buffers (64-256 KiB). Parquet uses variable-size column chunks (50-500+ KiB, often exceeding 128 KiB).
2. **Coalescing eliminates small reads:** The CoalescingReader merges adjacent/nearby column reads before they reach the scheduler, further increasing effective read sizes.
3. **Thread overhead dominates:** The scheduler's single-thread MPSC processing adds latency that exceeds any batching savings on the actual I/O operations.

### Comparison to M3 Analysis

M3 (Journal 26) identified these root causes for single-file failure:
1. Read sizes too large (128-512 KiB) → pread fallback
2. Too few reads per query → insufficient batching
3. Scheduler thread contention

M4 confirms that **all three persist in multi-file regimes:**
1. Read sizes unchanged (column chunks are a data property, not a file property)
2. More total reads (TPC-H Q1: 5439 ops) but still pread-dominated
3. More thread contention from higher I/O volume (Q1 stalls: 536)

## What M4 Teaches

### 1. Column chunk size is the binding constraint
The scheduler's 128 KiB routing threshold was set based on IPC bench results. In practice, Parquet column chunks are frequently larger, making the threshold irrelevant. The scheduler cannot benefit reads it doesn't route to io_uring.

### 2. File count is orthogonal to read size
The core assumption — that splitting data creates smaller reads — is wrong. File count affects file descriptor count and row group distribution, but not column chunk sizes. The I/O pattern is determined by the intersection of (schema, encoding, row group size), not (file count, partition strategy).

### 3. CoalescingReader and IoScheduler work at cross purposes
CoalescingReader INCREASES read sizes by merging adjacent column chunks. IoScheduler NEEDS small reads to route to io_uring. These two components cannot both be optimal simultaneously under the current architecture.

### 4. Even pure io_uring routing doesn't help
Q7 (100% io_uring, 309 SQEs) and Q1-TPC-H (71% io_uring, 3885 SQEs) both show regressions. The scheduler thread overhead exceeds batching benefits regardless of routing fraction or volume.

## Implications for Paper

### All Performance Claims Falsified

| Claim | Result | Status |
|-------|--------|--------|
| C7 (single-file scheduler) | 1.023× | ❌ FAIL |
| C8 (stall metric) | 1.011× | ✅ PASS (moot) |
| C9 (batch threshold) | 0.967× | ❌ FAIL |
| C11 (multi-file split ClickBench) | 0.985× | ❌ FAIL |
| C12 (file count scaling) | No trend | ❌ FAIL |
| C13 (TPC-H) | 0.97× | ❌ FAIL |

The diagnostic claims C1–C6 remain valid. The structural diagnosis (ObjectStore overhead, copy overhead, format mismatch) correctly identifies WHY io_uring cannot benefit through the standard DataFusion path. But the custom scheduler — designed to bypass those bottlenecks — still fails because the Parquet reader's read granularity doesn't match io_uring's strength regime.

### Paper Direction: Full Negative-Result Paper

The paper must pivot to a comprehensive negative-result contribution:

1. **Diagnosis remains valuable:** C1–C6 explain precisely why io_uring doesn't help through ObjectStore — these insights apply to any analytics engine using object_store.
2. **Scheduler attempt is instructive:** "We built the custom reader that SHOULD work, and it still doesn't" — demonstrates that the problem is deeper than the I/O interface layer.
3. **Regime characterization is novel:** Exp-12 sweep provides the first systematic evidence that file count does not create an io_uring-favorable regime in columnar analytics.
4. **Root cause is architectural:** The mismatch is between column-chunk granularity (set at write time) and io_uring's batching advantage (requires many small reads).

Story: Diagnosis → Structural mismatch → Regime investigation → Negative resolution → The real bottleneck is read granularity, not I/O interface.

## Data Artifacts

| Artifact | Path |
|----------|------|
| Exp-11 results | `eval/clickbench/results/run_20260411_061504/exp11/` |
| Exp-11 analysis | `analysis_exp11.json`, `summary_exp11.txt` |
| Exp-12 results | `eval/clickbench/results/run_20260411_061823/exp12/` |
| Exp-12 analysis | `analysis_exp12.json`, `summary_exp12.txt` |
| Exp-13 results | `eval/clickbench/results/run_20260411_061613/tpch/` |
| Exp-13 analysis | `analysis_tpch.json`, `summary_tpch.txt` |
| Partitioned data | `eval/clickbench/data/hits_N{10,50,100,226}/` |
| TPC-H data | `eval/clickbench/data/tpch_sf10/{single,split}/` |
| TPC-H queries | `eval/clickbench/queries/tpch/q{01,03,05,06,10,12,14,19}.sql` |
| Benchmark scripts | `scripts/run_multifile.sh`, `scripts/run_tpch.sh` |
| Analysis scripts | `scripts/analyze_multifile.py`, `scripts/analyze_tpch.py` |
| M4 task files | `tasks/M4/M4-T{1..10}.md`, `tasks/M4/PROGRESS.md` |
