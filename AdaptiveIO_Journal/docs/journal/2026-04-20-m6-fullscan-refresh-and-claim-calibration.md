# Full-Scan Refresh + Paper-Claim Calibration — 2026-04-20

**Milestone:** M6 — Page-Granular Reader + io_uring Integration
**Tasks completed:** None (results refresh + paper-positioning session)
**System:** Linux 6.8.0-106-generic, 1× Intel Xeon Gold 6530 (32 cores / 64 threads), NVMe `nvme0n1`

---

## 1. Full-Scan Mica vs Parquet vs Arrow — Cold Refresh

**Claim tested:** C17 — Page-granular format imposes ≤1.20× overhead on full scans vs Parquet.

**Setup:**

```bash
cargo build --release --manifest-path eval/native-format-bench/Cargo.toml \
  --bin native-format-bench

./eval/native-format-bench/target/release/native-format-bench run \
  --cache cold \
  --workload sf_projection_scan \
  --workload mf_full_year_filter \
  --reader parquet_native \
  --reader arrow_mmap_pruned \
  --reader mica_v3_native_mirrored_adaptive_sparse_raw_coalesced \
  --iterations 10 \
  --output eval/native-format-bench/results/nyc-fhv-guardrail
```

- Cold protocol: `posix_fadvise(..., POSIX_FADV_DONTNEED)` on each data file before every iteration ([eval/native-format-bench/src/run.rs:920](eval/native-format-bench/src/run.rs#L920)).
- Iterations: 10 per (workload, reader).
- Dataset: NYC FHV 2021, single-file `fhvhv_tripdata_2021-01.parquet/arrow/mica` (11,908,468 rows) and 12-file multi ({2021-01..2021-12}).
- Projected columns: `pickup_datetime`, `trip_miles`, `base_passenger_fare`.

**Results (median wall-clock, ms):**

| Workload | Parquet | Arrow mmap pruned | Mica v3 coalesced | Mica / Parquet |
|---|---:|---:|---:|---:|
| `sf_projection_scan` (11.9 M rows, single file) | **333.277** | 3324.054 | 413.488 | **1.24× slower** |
| `mf_full_year_filter` (175 M rows, 12 files) | **4255.967** | 42763.974 | 6248.085 | **1.47× slower** |

**p95 (ms):**

| Workload | Parquet | Arrow mmap pruned | Mica v3 coalesced |
|---|---:|---:|---:|
| `sf_projection_scan` | 338.694 | 3343.955 | 514.770 |
| `mf_full_year_filter` | 5252.324 | 42892.426 | 6776.708 |

**Delta vs stale 2026-04-15 numbers:**

| Workload | Old Mica/Parquet | New Mica/Parquet | Improvement |
|---|---:|---:|---:|
| `sf_projection_scan` | 0.48× (2.09× slower) | **0.81× (1.24× slower)** | +69% |
| `mf_full_year_filter` | 0.48× (2.07× slower) | **0.68× (1.47× slower)** | +42% |

**Key findings:**
- Single-file `sf_projection_scan` is within **4% of the C17 1.20× ceiling**. Multi-file `mf_full_year_filter` still exceeds C17 (1.47× vs 1.20× target) but by 23 pp, not 87 pp.
- Arrow mmap pruned is 6.8–8.0× slower than Mica on both workloads because Arrow IPC has no column-level byte-range projection at the I/O layer — it pages in all 24 columns.
- Mica v3 vs prior v3 adaptive: already identical within noise on these full-scan workloads (coalesced is the latest default).

**Decision:** C17 remains OPEN but substantially de-risked. Do not block the paper submission on C17; revisit after the compression option (below) is prototyped. Headline for §6 (Evaluation) will use **1.24× / 1.47×**, not the 2.09× / 2.07× the old review session cited.

**Data artifact:** [`native_format_bench_20260420_101732/`](../../eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260420_101732/).

---

## 2. Bytes-Read Verification and Per-Byte Decode Speed

**Claim tested:** Validates the C17 picture by isolating I/O volume from decode cost — informs whether a compression option is the right lever.

**Setup:** Aggregated median `os_read_bytes` and `engine_read_bytes` from the same 10-iteration run ([results.csv](../../eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260420_101732/results.csv)). Derivation assumes NVMe sequential throughput ≈ 3 GB/s for the I/O-time back-of-envelope.

**Measured bytes (median over 10 iterations):**

| Workload | Reader | OS bytes read | Engine bytes read | Units hit |
|---|---|---:|---:|---:|
| `sf_projection_scan` | parquet_native | 97.9 MB | — | 12 row groups |
| `sf_projection_scan` | mica_v3 coalesced | 245.7 MB | 241.6 MB | 59,543 pages |
| `sf_projection_scan` | arrow_mmap_pruned | 2,164.7 MB | — | 91 batches |
| `mf_full_year_filter` | parquet_native | 1,371.9 MB | — | 174 row groups |
| `mf_full_year_filter` | mica_v3 coalesced | 3,613.8 MB | 3,541.9 MB | 872,988 pages |
| `mf_full_year_filter` | arrow_mmap_pruned | 31,711.0 MB | — | 1,338 batches |

**Derived per-byte decode advantage:**

| Workload | Mica bytes / Parquet bytes | Mica latency / Parquet latency | Mica decodes faster per byte by |
|---|---:|---:|---:|
| `sf_projection_scan` | 2.51× | 1.24× | **2.02×** |
| `mf_full_year_filter` | 2.63× | 1.47× | **1.79×** |

**Key findings:**
- Projection pushdown is active on both formats: Parquet reads 97.9 MB of its 337 MB on-disk single-file (3 of 24 columns); Mica reads 245.7 MB of its 981 MB on-disk (matching the (3/24) × 2.91× size ratio).
- Arrow mmap fetches the entire file (2.16 GB) because Arrow IPC's mmap vanilla path has no column-level byte-range projection at I/O level — this is the measured origin of Arrow's 8× regression.
- Mica pays a ~2.5–2.6× extra-bytes tax (uncompressed vs Snappy Parquet) but recovers most of it through decode-pipeline efficiency: ~2.02× faster wall-time per byte on sf_projection_scan.
- On both full-scan workloads, I/O is only ~20–30% of wall time at NVMe speeds — both formats are **decode-bound**, not I/O-bound.

**Decision:** Add per-page LZ4/FSST as an optional encoding in the Mica spec and re-measure before submission. Projected close-the-gap at 2× compression: sf_projection_scan → ~1.12×, mf_full_year_filter → ~1.33× (see §3).

---

## 3. Compression Lever Projection

**Was:** "Mica is 3× the bytes and 2× slower on full scans — structural flaw." (2026-04-15 framing.)
**Now:** "Mica pays a 1.24–1.47× full-scan overhead from being uncompressed. Decode is ~2× faster per byte. Projected overhead with LZ4 at 2× compression is 1.12–1.33×."
**Why:** Measured per-byte decode speed (§2) combined with a conservative LZ4 decompress cost model (~3 GB/s). Scaling: Mica I/O halves; LZ4 decompress adds cost roughly equal to the saved I/O time since I/O is only 20–30% of wall time.
**Impact:** C17 is not a format-level show-stopper; it's a compression-option TODO. §6 of the paper can honestly claim "within ~25–50% of Parquet on full scan, projected ≤1.3× with optional LZ4" without over-promising parity.

---

## 4. Scope Correction: `mf_full_year_filter` is I/O-Wise a Full Scan

**Was:** Old journal labeled `mf_full_year_filter` as "~30% selectivity" (carrying `trip_miles > 5.0` output ratio).
**Now:** Output selectivity ~30%; **I/O selectivity ~100%**.
**Why:** `trip_miles` is not the sort key (pickup_datetime is). Zone maps on a high-cardinality float crossing the 5.0 threshold cannot prune any page under the birthday-problem regime at 200 rows/page — measured `pages_hit = 872,988` (all pages) confirms this. No Bloom/bitmap index exists on `trip_miles` (range predicate is unprunable by Bloom/bitmap by definition).
**Impact:** `mf_full_year_filter` functions as a dedicated **full-scan C17 probe** with mandatory materialization (the predicate prevents short-circuiting). Future paper §6 must describe the two full-scan workloads as `sf_projection_scan` (project-only, no filter) and `mf_full_year_filter` (unprunable range filter, forces full decode) — both I/O-full, differently decode-heavy.

---

## 5. Paper-Claim Calibration: Three Defensive Claims

**Was:** Three candidate framings for the paper motivation, not yet calibrated against the evidence.
**Now:** Revised into defensible statements with specified supporting experiments.

### 5.1 Claim A — "NVMe enables Mica because of random access + batching"

**Defensible form:** *"NVMe delivers throughput-competitive random access at queue depth ≥32 via its 32–128-entry hardware command queue; this regime did not exist on HDD and was unexploitable on NVMe until io_uring exposed batch submission to user space."*

**Supporting evidence available:** [07-exp-c-fault-isolation.md §23.1](../../eval/arrow/journals/07-exp-c-fault-isolation.md) — iouring-batched = 13.56× effective parallelism over serialized mmap at 4 KiB / N=100 cold.
**Still missing:** QD-sweep micro-benchmark (random 4 KiB at QD ∈ {1, 4, 8, 16, 32, 64, 128}) showing the crossover from latency-bound to throughput-competitive on the target hardware. Plan: extend Exp-C's `iouring-batched` strategy to sweep N (effective QD).
**Reviewer risk:** Storage-specialist reviewer will expect hardware-specific QD curves. Mitigation: produce one figure.

### 5.2 Claim B — "Existing formats cannot be hacked into this"

**Defensible form:** *"Any change to Parquet large enough to support page-granular batch-random I/O is itself a new format. Mica is a point design in this inevitable redesign space, alongside Lance, Nimble, and Vortex."*

**Supporting evidence available:** [2026-04-15 §3.1–§3.4](2026-04-15-reviewer-objections-mica-vs-parquet.md) — four structural incompatibilities (page-dictionary coupling, monolithic Thrift footer, default 1 MB page, I/O model). Empirical: H2=0.998× ([journal 22](../../eval/arrow/journals/22-coalescing-results.md)) and reactor H2=0.887× ([journal 23](../../eval/arrow/journals/23-reactor-experiment.md)) prove the I/O-interface layer cannot close the gap alone.
**Still missing:** Direct Mica-vs-Lance head-to-head on the 6 selective + 2 full-scan workloads. Without this, claim B cannot be fully defended — reviewers will assume Lance already solves the problem.
**Reviewer risk:** HIGHEST of the three. Mica vs Lance is the #1 anticipated objection.

### 5.3 Claim C — "io_uring is the software enabler"

**Defensible form:** *"io_uring is the first general-purpose Linux interface that exposes NVMe's batch-parallel queue semantics at the application level without O_DIRECT-only or dedicated-core constraints (unlike libaio and SPDK respectively)."*

**Supporting evidence available:** [07-exp-c-fault-isolation.md §23.1](../../eval/arrow/journals/07-exp-c-fault-isolation.md) — iouring-serial at 85 μs/read matches mmap-random (86 μs/read) exactly, proving speedup comes from **batch submission**, not io_uring brand. iouring-batched at 6.4 μs/read is 13.6× faster.
**Still missing:** One paragraph in related work acknowledging libaio (O_DIRECT-only, semantic limits) and SPDK (user-space, dedicated cores).
**Reviewer risk:** Kernel-systems reviewer asking "why not libaio/SPDK?" Low effort to pre-empt.

---

## 6. Updated Headline Numbers for Paper §6

Consolidated summary of Mica vs Parquet across all measured workloads (single-file NYC FHV unless noted, cold cache, median over 10 iterations):

| Workload | Regime | Index exercised | Mica/Parquet | Source |
|---|---|---|---:|---|
| `sf_point_query` | very selective point | zone + bloom | **126.8× faster** | [matrix 0417](../../eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260417_113215/reader_comparison_matrix_10it_cold.md) |
| `sf_equality_filter` | selective equality | bloom | **8.57× faster** | matrix 0417 |
| `mf_time_2h_filter` | multi-file time range | zone map | **5.74× faster** | matrix 0417 |
| `sf_time_1h_filter` | selective time range | zone map | **5.17× faster** | matrix 0417 |
| `sf_compound_filter` | compound selective | zone + bloom | **4.87× faster** | matrix 0417 |
| `sf_in_list_filter` | wider equality IN(2) | bitmap | **3.38× faster** | matrix 0417 |
| `sf_projection_scan` | full-scan projection | — | **1.24× slower** | this session |
| `mf_full_year_filter` | full-scan unprunable | — | **1.47× slower** | this session |

Selectivity-speedup is monotonic (C16 shape holds) spanning **126.8× → 0.68×** end-to-end, crossing 1.0× between `sf_in_list_filter` and `sf_projection_scan`.

---

## Open Questions

- **Answered:** Is C17 a format-level show-stopper? → No. Overhead is 1.24× / 1.47× uncompressed; per-byte decode is ~2× faster than Parquet; adding LZ4 projectably closes most of the residual.
- **Answered:** Is `mf_full_year_filter` actually a selective workload? → No; output 30% but I/O 100%. Treat as full-scan probe.
- **New:** What is the actual measured Mica performance with optional per-page LZ4? Projected 1.12× / 1.33× vs Parquet; needs implementation + re-run.
- **New:** What is the QD-scaling curve of random 4 KiB reads on the Xeon Gold 6530 + nvme0n1 hardware? Needed for claim A defense.
- **New:** How does Mica compare to Lance on the 8-workload matrix? Needed for claim B defense.
- **New:** Should `engine_read_bytes` be instrumented for Parquet (currently 0 in the CSV)? Would strengthen the per-byte decode derivation.

---

## Design Ramifications for M7

1. **Add LZ4 compression as an optional page encoding before submission-time experiments.** The compression lever is the single highest-leverage change for closing the full-scan gap; the spec already reserves it as an open question in [mica-spec.md §Appendix C](../mica-spec.md). Re-run §1's two workloads and update C17 verdict.
2. **Add a QD-sweep micro-benchmark variant to `profile_mica` or a new binary.** Measure random 4 KiB read throughput at QD ∈ {1, 4, 8, 16, 32, 64, 128} on the Xeon Gold 6530. Produces one figure for §2 (Background).
3. **Add Lance as a reader kind in `native-format-bench`** (parallel to `parquet_native`), including the write-side conversion. Run the 8 workloads. This is the single most important missing experiment for submission credibility.
4. **Instrument `engine_read_bytes` for Parquet_native** so the per-byte decode comparison stops relying on OS-level bytes alone. Small addition to the bench harness.
5. **Stop referencing the 2026-04-15 numbers in any paper draft.** They are obsolete; all current framing should reference this entry or the 2026-04-17 matrix.
