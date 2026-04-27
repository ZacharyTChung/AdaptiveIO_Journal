# Two Types of Coordination: Metadata vs I/O — 2026-04-10

**Milestone:** M1 — Ablation + Landscape Analysis
**Tasks completed:** M1-T1, M1-T2, M1-T3, M1-T4
**System:** Linux 6.x, NVMe SSD, Intel CPU (see eval/clickbench/results/run_20260410_071914/k_sweep/ for system_info)

---

## 1. K-Sweep Decomposition Reveals Two Distinct Coordination Layers (C5)

**Claim tested:** C5 — Metadata cache and lookahead batching are independently beneficial

**Setup:**
```bash
HITS=/home/ybyan/dataset/clickbench/hits.parquet \
  bash eval/clickbench/scripts/run_k_sweep.sh
# K ∈ {1,2,4,8,16}, queries {q0,q1,q3,q7}, 5 iterations, 2 stores × 2 factories
# 400 total cold-cache runs, drop_caches between each
```

**Results (H1 = median wall_ms ratio L,D / L,C):**

| Query | K=1 | K=2 | K=4 | K=8 | K=16 |
|-------|-----|-----|-----|-----|------|
| q0 | 2.170× | 2.139× | 2.193× | 2.296× | 2.181× |
| q1 | 1.571× | 1.595× | 1.615× | 1.489× | 1.547× |
| q3 | 1.516× | 1.495× | 1.464× | 1.004× | 0.883× |
| q7 | 1.677× | 1.685× | 1.702× | 1.818× | 1.697× |

**Decomposition (metadata_fraction = (H1_K1 - 1) / (H1_K4 - 1)):**

| Query | H1_K1 | H1_K4 | metadata_fraction | lookahead_fraction |
|-------|-------|-------|-------------------|-------------------|
| q0 | 2.170× | 2.193× | 0.981 | 0.019 |
| q1 | 1.571× | 1.615× | 0.929 | 0.071 |
| q3 | 1.516× | 1.464× | 1.112 | -0.112 |
| q7 | 1.677× | 1.702× | 0.964 | 0.036 |

**Key findings:**
- SharedMetadataCache alone (K=1) accounts for 93–98% of measured H1 speedup on q0/q1/q7
- Lookahead prefetch (K>1) contributes 2–7% marginal gain on these queries
- q3 regresses severely at K≥8 (1.004× at K=8, 0.883× at K=16) — filter-heavy query fetches row groups that zone-map pruning would skip
- C5 PASS: K=1 H1 > 1.0 for all 4 queries AND K=4 > K=1 for 3/4 queries

**Decision:** Metadata coordination is the dominant lever in the current architecture. Lookahead prefetch should be pruning-aware to avoid the q3 regression.

---

## 2. Two Types of Coordination — The Core Insight

The K-sweep conflated two fundamentally different interventions. Distinguishing them is critical for evaluating the I/O scheduler research path.

### Metadata Coordination

**Was:** Each DataFusion partition reader independently reads the Parquet footer (124 footer reads per 10-query matrix).
**Now:** SharedMetadataCache eliminates redundant footer reads (124 → 2), reducing ObjectStore calls from 350 → 77 per query (Journal 22 §50).
**Why:** Footer reads are pure overhead — every partition reader needs the same structural metadata. Caching is a user-space optimization that reduces call count, not per-call cost.
**Impact:** This is what K=1 measures. It is orthogonal to io_uring — no kernel interface change can eliminate redundant user-space calls.

### I/O Coordination

**Was:** Each partition reader issues independent `ObjectStore::get_ranges` calls, each dispatched as an individual `pread` via `spawn_blocking`.
**Now (hypothetical):** A process-wide I/O scheduler intercepts get_ranges from all concurrent streams, coalesces adjacent ranges, and submits batched io_uring SQE groups with O_DIRECT.
**Why:** With buffered I/O, the kernel page cache already coalesces nearby reads — making explicit batching redundant (H2 = 0.998×). With O_DIRECT, each I/O is a real NVMe command; batching reduces per-call overhead.
**Impact:** The K-sweep does NOT measure this. It varies intra-stream lookahead depth while keeping the I/O dispatch path unchanged. Cross-stream aggregation + O_DIRECT is untested.

### Distinction Table

| Dimension | Metadata Coordination | I/O Coordination |
|---|---|---|
| Reduces | Number of I/O calls | Cost per I/O call |
| Mechanism | Cache structural metadata | Batch SQE submission + O_DIRECT |
| Scope | Per-reader (intra-stream) | Cross-stream (process-wide) |
| K-sweep measured? | Yes (K=1 isolates it) | No (not implemented) |
| Current contribution | 93–98% of H1 | Unmeasured |
| io_uring relevant? | No (pure user-space) | Yes (SQE batching) |
| Requires O_DIRECT? | No | Likely (page cache masks benefit) |

---

## 3. Research Direction Decision

**Was:** Unclear whether the K-sweep's 96% metadata-fraction invalidates the I/O scheduler path (Path A).
**Now:** It does not. The K-sweep and the I/O scheduler address orthogonal coordination layers.
**Why:**
1. K=1 measures metadata coordination (eliminating redundant calls). This is solved — 93–98% captured.
2. An I/O scheduler would test I/O coordination (reducing per-call cost via batching + O_DIRECT). This is unmeasured.
3. The evidence pyramid has a gap: "below" (H2 = 0.998×), "at" (reactor H2 = 0.887×), "above" (H1 = 1.438×) are all measured with buffered I/O. O_DIRECT changes the economics.
**Impact:** Path A remains viable but requires O_DIRECT as a prerequisite. Buffered I/O + batching is a proven null result (H2 = 0.998×, reactor H2 = 0.887×).

### Risks Illuminated by M1

1. **Buffered I/O makes batching redundant.** Page cache coalesces nearby reads → H2 ≈ 1.0×. O_DIRECT is prerequisite for any scheduler benefit.
2. **O_DIRECT may hurt warm workloads.** Bypassing page cache means every read is a real NVMe command. Warm-cache speedup (2.03×) would be lost.
3. **Engineering effort vs marginal gain.** LanceDB's scheduler (PR #1918 + #2928) was substantial. If NVMe gain is <10%, contribution may not justify complexity.

### LanceDB Precedent

LanceDB: "Adding io_uring by itself didn't help performance at all. Adding the scheduler rework along with io_uring gave us the result" — 1.5M IOPS. However, Lance uses 64 KiB default chunks (vs Parquet's 50–500 KiB), and their workload profile differs from ClickBench analytical queries.

---

## Design Ramifications for M2+

1. **M2 should include K=1** as a comparison point alongside K=4 across all 43 queries, to confirm metadata-dominance generalizes.
2. **Path A requires O_DIRECT.** Any I/O scheduler evaluation must bypass page cache. Buffered I/O batching is a proven null result.
3. **Adaptive prefetch needed.** q3's K≥8 regression (0.883× at K=16) shows prefetch must check zone-map statistics before fetching row groups — avoid the fetch-then-prune antipattern.
