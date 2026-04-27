# Research Plan: Page-Granular I/O for Selective Columnar Analytics

> **Status:** Active — pivoted from negative result to new format design
> **Target venue:** USENIX ATC 2026
> **Submission deadline:** TBD (winter Jan or spring Apr round)
> **Last updated:** 2026-04-13 (M6 task files amended with gap analysis — T1 expanded for all indexes, T3 late-mat deferred, T4 rewritten to factor validate.rs, T5 async bridge added, T6 data conversion prerequisite added)
> **Core claim:** Columnar analytics engines fail to exploit NVMe random I/O because file formats (Parquet, ORC) produce 50–500 KiB column chunk reads — too large and too few for io_uring batching. A page-granular columnar format with fixed 4–8 KiB pages, independently decompressible, and per-page zone maps produces hundreds of small scattered reads per selective query, enabling batched io_uring submission that delivers ≥3× E2E speedup on queries with ≤5% selectivity while maintaining parity on full scans.
> **Current milestone:** M6 — Page-Granular Reader + io_uring Integration
> **Current task:** M6-T1 — MicaReader::open (metadata + all indexes)
>   → See `tasks/M6/PROGRESS.md` for status; `tasks/M5/PROGRESS.md` for M5 completion summary
> **Future work (separate paper):** See `OSDI_PLAN.md` for cross-query I/O batching + multi-tenant cooperative scheduling

---

## 1. Overview

Modern columnar analytics engines (DataFusion, Polars, DuckDB) access storage through abstraction layers like Apache Arrow's `ObjectStore` trait. When io_uring emerged as a high-performance Linux I/O interface promising batched syscall submission, the natural hypothesis was that swapping `pread`-based storage backends for io_uring would accelerate analytical queries. Prior work on custom Arrow IPC readers showed 1.5–9× speedups with io_uring (Journals 13–18 in this repository), reinforcing this expectation.

**Act 1 — Diagnosis.** We implemented a `CoalescingParquetFileReaderFactory` that eliminates redundant footer reads and batches column-chunk fetches with a prefetch window, achieving H1 = 1.438× cold / 2.025× warm on ClickBench. Simultaneously, we replaced `LocalFileSystem` with an io_uring-based `IoUringStore` and measured H2 = 0.998× — no effect. The coalescing reader's win is *larger* in warm cache than cold, proving it is a syscall/await-reduction optimization rather than an I/O bandwidth optimization. A K-sweep ablation (K=1..16) further showed that SharedMetadataCache alone accounts for ~96% of the speedup.

**Act 2 — Decomposition.** Systematic 3-layer experimental decomposition revealed why io_uring fails through ObjectStore: (1) ephemeral ring setup per call (~150–400µs), (2) buffered I/O (page cache masks kernel-side batching), (3) batch depth 1–4 per submission vs. 512 in IPC bench, (4) large read sizes (50–500 KiB makes syscall overhead <2% of I/O). A reactor variant that aggregated concurrent requests made it 11% *worse* (H2 = 0.887×) by serializing previously parallel `spawn_blocking` I/O. A landscape survey confirmed 6/9 analytics engines share the request-response ObjectStore pattern.

**Act 3a — Single-file Resolution Attempt (Negative Result).** The I/O fraction under CoalescingReader is 18–59% of query time (corrected from the misleading 10–25% measured on the unoptimized baseline), making Amdahl's ceiling for an I/O scheduler 1.10–1.42× E2E. Building on 11 transferable insights from 25 prior experiments, we designed a purpose-built I/O scheduler with O_DIRECT + io_uring, persistent ring, registered buffer pool, and batch assembly. On single-file ClickBench, the scheduler delivered C7 = 1.023× — no improvement. Root cause: single-file Parquet produces few large sequential column chunks (50–500 KiB, 28.8% of chunks >128 KiB), most of which exceed the 128 KiB io_uring crossover point and route to pread fallback. Even for queries where ALL I/O goes through io_uring (q1, q7: 300+ SQEs, 0 preads), speedup is ~1.0× because the saved syscall overhead (~260µs) is negligible against 270ms query time. Range splitting and O_DIRECT pread were both attempted and reverted as harmful.

**Act 3b — Multi-file Hypothesis Falsified.** The single-file negative result reveals a structural mismatch, not a scheduler bug. We hypothesized that multi-file partitioned layouts (100+ files) would produce enough small scattered reads to reach io_uring's sweet spot. We validated on two benchmarks: (1) Split ClickBench (same data, N∈{1,10,50,100,226} files) and (2) TPC-H SF=10 (lineitem partitioned into 100 files). Results: C11 = 0.985× (100-file ClickBench), C12 = flat ~1.0× across all N (no monotonic increase), C13 = 0.97× (TPC-H). Root cause: column chunk size (50–500 KiB) is a **data format property**, not a function of file count. Splitting into more files doesn't produce smaller reads. An inline io_uring reader (zero thread hops, no channel overhead) improved to 1.054× (Exp-14) but still fell far short. Even queries with 100% io_uring routing (Q7: 309 SQEs, 0 preads) show 0.70× due to architectural overhead. The binding constraint is read granularity.

**Act 4 — Page-Granular Format + Batched io_uring.** The diagnosis reveals that the obstacle is not the I/O interface or the scheduling architecture — it is the **read granularity** dictated by the columnar format. Parquet and ORC encode data in variable-size column chunks (50–500+ KiB), each compressed as a single unit, making sub-chunk random access impossible. This contrasts with our IPC bench, which fragments columns into 4 KiB blocks and achieves 8–13× speedup via io_uring batching. Meanwhile, production analytics workloads are far more selective than benchmarks suggest: Snowflake's 667M-query analysis (VLDB 2025) shows 38% of filters have ≤5% selectivity, and 99.4% of micro-partitions are pruned across their platform (SIGMOD 2025). The Lance format (arXiv 2025) demonstrates 60× better random access than Parquet through 4–8 KiB mini-blocks. We build a minimal purpose-built page-granular columnar format with: (1) fixed 4–8 KiB pages, independently decompressible; (2) lightweight per-page compression (FOR, DELTA, DICT); (3) per-page zone maps enabling fine-grained predicate pushdown. Combined with batched io_uring via our inline reader (zero thread hops, O_DIRECT), selective queries (≤5% selectivity) produce hundreds of small scattered reads from zone-map-qualified pages — exactly the regime where io_uring batching delivers massive speedups. The paper story becomes: NVMe hardware can deliver 12M random 4K IOPS (VLDB 2023), but columnar formats waste this by requesting few large sequential reads. We measure the read-size crossover (first systematic study), build a format that exploits it, and demonstrate when fine-grained I/O beats coarse-grained scanning.

---

## 2. Claims & Hypotheses

### Act 1–2 Claims (Validated)

| ID | Claim | Metric | Target | Experiment | Status |
|----|-------|--------|--------|------------|--------|
| C1 | Coalescing reader delivers ≥1.4× cold-cache speedup | Median wall_ms ratio (L,D)/(L,C) across 10 queries | ≥1.4× | Exp-1 | ✅ 1.438× |
| C2 | io_uring provides no measurable speedup through ObjectStore | Median wall_ms ratio (L,C)/(I,C) | within ±5% of 1.0 | Exp-2 | ✅ 0.998× |
| C3 | Store-level reactor aggregation is actively harmful | Median H2 with reactor vs without | reactor H2 < 0.95× | Exp-3 | ✅ 0.887× |
| C4 | Coalescing benefit is syscall/await reduction, not I/O bandwidth | Warm H1 > Cold H1 | ratio > 1.2 | Exp-4 | ✅ ratio 1.41 |
| C5 | Metadata cache and lookahead are independently beneficial | H1 at K=1 > 1.0, H1 at K=4 > H1 at K=1 | Both measurable | Exp-5 | ✅ meta_frac ≈96% |
| C6 | Request-response storage API pattern shared by majority | Engine survey count | ≥5 of 9 engines | Exp-6 | ✅ 6/9 |

### Act 3a Claims (Single-file — Validated Negative Results)

| ID | Claim | Metric | Target | Experiment | Status |
|----|-------|--------|--------|------------|--------|
| C7 | Single-file scheduler adds ≤1.05× E2E speedup (structural ceiling) | Median wall_ms ratio across 10 queries, cold cache | ≤1.05× | Exp-7 | ✅ 1.023× |
| C8 | O_DIRECT is prerequisite — scheduler with buffered I/O shows ≤1.03× | Median wall_ms ratio scheduler+buffered vs baseline | ≤1.03× | Exp-8 | ✅ 1.011× |
| C9 | Batch depth insensitive on single-file (too few small reads to batch) | Speedup variation across N=1,10,50,100,256 | ≤1.05× variation | Exp-9 | ✅ 0.967× max |

### Act 3b Claims (Multi-file — New)

| ID | Claim | Metric | Target | Experiment | Status |
|----|-------|--------|--------|------------|--------|
| C11 | Multi-file partitioned scheduler delivers ≥1.3× median E2E speedup (split ClickBench) | Median wall_ms ratio (no-scheduler) / (scheduler) on 100-file partition, 10+ queries, cold cache | ≥1.3× | Exp-11 | ❌ FAIL (0.985×) |
| C12 | Speedup increases monotonically with file count (regime characterization) | Median speedup at N=1,10,50,100,226; monotonic trend | Monotonic, ≥1.3× at N≥100 | Exp-12 | ❌ FAIL (flat ~1.0× at N≥10) |
| C13 | TPC-H partitioned scheduler delivers ≥1.3× median E2E speedup (standard benchmark) | Median wall_ms ratio (no-scheduler) / (scheduler) on TPC-H SF=10 lineitem partitioned into 100+ files, I/O-intensive queries, cold cache | ≥1.3× | Exp-13 | ❌ FAIL (0.97×) |

### Act 4 Claims (Page-Granular Format — New)

| ID | Claim | Metric | Target | Experiment | Status |
|----|-------|--------|--------|------------|--------|
| C14 | Read-size crossover: io_uring batching transitions from beneficial to neutral at ~16–32 KiB | Speedup (io_uring / pread) at read sizes 512B, 1K, 2K, 4K, 8K, 16K, 32K, 64K, 128K, 256K, 512K; identify crossover | Crossover at ≤32 KiB; ≥2× at 4K | Exp-15 | ⬜ Pending |
| C15 | Page-granular format + io_uring delivers ≥3× E2E speedup on selective queries (≤5% selectivity) vs Parquet + DataFusion default | Median wall_ms ratio (Parquet default) / (page-granular + io_uring), 10+ selective queries, cold cache | ≥3× | Exp-16 | ⬜ Pending |
| C16 | Speedup scales with selectivity: monotonically increasing from full-scan (1.0×) to point lookup | Speedup at selectivity = 100%, 50%, 20%, 10%, 5%, 1%, 0.1% | Monotonic, ≥3× at ≤5% | Exp-17 | ⬜ Pending |
| C17 | Page-granular format imposes ≤20% overhead on full scans vs Parquet | Median wall_ms ratio (page-granular full scan) / (Parquet full scan) | ≤1.20× | Exp-19 | ⬜ Pending |
| C18 | Page-granular format + io_uring delivers ≥2× on TPC-H selective queries (Q6, Q12, Q14 with date range predicates) | Median wall_ms ratio, cold cache | ≥2× | Exp-18 | ⬜ Pending |

**Primary claim:** C15 — the paper requires ≥3× on selective queries to demonstrate that page-granular format + batched io_uring is a fundamentally better I/O strategy for the dominant production workload (38% of queries have ≤5% selectivity per Snowflake VLDB 2025).
**Diagnostic foundation:** C1–C6 (validated) explain WHY io_uring fails through ObjectStore. C7–C9 (validated) confirm single-file is the wrong regime. C11–C13 (validated negative) prove file count is irrelevant — read granularity is the binding constraint.
**Novel measurement:** C14 — first systematic read-size crossover study for io_uring on NVMe. No prior work measures where batching transitions from beneficial to neutral.
**Resolution:** C15+C18 show that with the RIGHT format (page-granular, 4–8 KiB), batched io_uring delivers the speedups that Parquet's column chunks prevent.
**Completeness:** C16 characterizes the selectivity-speedup curve. C17 bounds worst-case overhead (full scans should not regress significantly).
**Deferred:** C10 (I/O fraction correlation) may be computed from Exp-16 data if per-query variance is informative.

---

## 3. System Architecture

### 3.1 Component Map

```
                     DataFusion ParquetExec
                            |
                  +---------+---------+
                  |                   |
          DefaultFactory      CoalescingFactory (Act 1)
                  |                   |
          ParquetObjectReader   CoalescingParquetReader
                  |                   |
                  |            +------+------+
                  |            | SharedMeta  |  RangeCache
                  |            | Cache       |  + Async Prefetch
                  |            +------+------+
                  |                   |
                  |          +--------+--------+
                  |          | I/O Scheduler   |  (Act 3 — NEW)
                  |          |  Request Queue  |
                  |          |  Batch Assembly  |
                  |          |  Pipeline Mgr   |
                  |          +--------+--------+
                  |                   |
            +-----+-----+     +------+------+
            | ObjectStore|     | Direct I/O  |
            | (pread)    |     | (O_DIRECT + |
            |            |     |  io_uring)  |
            +------------+     +-------------+
```

### 3.2 Component Registry

#### CoalescingParquetFileReaderFactory — `eval/clickbench/src/coalescing_reader.rs`
- **Role:** Custom `ParquetFileReaderFactory` with shared metadata cache and per-reader range cache.
- **Interface exposed:** `ParquetFileReaderFactory::create_reader` → `Box<dyn AsyncFileReader>`
- **Interface consumed:** `Arc<dyn ObjectStore>` (Act 1) or `Arc<IoScheduler>` (Act 3)
- **Supports claims:** C1, C4, C5, C7, C11
- **Key design decisions:** Split-lock metadata cache (std::sync::Mutex outer + tokio::sync::OnceCell inner). Async prefetch via tokio::spawn fire-and-forget. K=4 default prefetch window.

#### IoUringStore — `eval/object-store-iouring/src/store.rs`
- **Role:** `ObjectStore` implementation using io_uring for `get_ranges`. Used in Act 1–2 experiments.
- **Supports claims:** C2, C3
- **Key design decisions:** Ephemeral ring per call (reactor variant documented as negative result in Journal 23).

#### I/O Scheduler — `eval/clickbench/src/io_scheduler.rs` (NEW — Act 3)
- **Role:** Process-wide I/O coordination service. Collects requests from all concurrent readers, batches into SQE groups, submits via persistent io_uring ring(s), dispatches completions.
- **Interface exposed:** `IoScheduler::submit_ranges(path, ranges) → Vec<Bytes>` (async)
- **Interface consumed:** Linux io_uring syscalls + O_DIRECT file descriptors
- **Supports claims:** C7, C8, C9, C11, C12
- **Key design decisions (derived from 11 journal insights):**

| # | Insight | Source | Design Decision |
|---|---------|--------|-----------------|
| 1 | 128 KiB crossover | J7 §23.2 | Size-adaptive routing: reads <128 KiB → io_uring; ≥128 KiB → pread fallback |
| 2 | Batch ≥50 SQEs | J7 §23.1 | Accumulate requests until batch ≥50 or deadline timer (1ms) fires |
| 3 | Pipeline overlap 1.86–2.04× | J16 §31 | Pipeline: submit batch N+1 while processing CQEs from batch N |
| 4 | Thread scaling modest 1.8× | J17 §32.3 | Single dedicated scheduler thread managing ring(s) |
| 5 | O_DIRECT mandatory | J22-23-25 | All reads via O_DIRECT; aligned buffer pool |
| 6 | Metadata 96% solved | J24-25 | Scheduler targets remaining per-call I/O dispatch cost |
| 7 | Spacing-invariant CV=8% | J7 §23.3 | No spatial reordering of SQEs — submit in arrival order |
| 8 | Decode cost compression | J14 §29.3 | Registered buffer pool; zero-copy via `io_uring_register_buffers` |
| 9 | NVMe queue saturation 12T | J10 §26.2 | Bound in-flight SQEs at 128–256 max |
| 10 | Winning IPC config 1.48× | J18 §33 | Expected 1.3–1.8× I/O path improvement |
| 11 | Corrected Amdahl I/O=18–59% | J21+J25 | Meaningful E2E ceiling of 1.10–1.42× |

#### Phase 2b Benchmark Suite — `eval/clickbench/scripts/run_phase2b*.sh`
- **Role:** Reproducible benchmark harness for the 2×2 matrix.
- **Supports claims:** C1–C5

#### K-Sweep Benchmark — `eval/clickbench/scripts/run_k_sweep.sh`
- **Role:** Ablation harness sweeping K∈{1,2,4,8,16} across queries.
- **Supports claims:** C5

### 3.3 Data & Control Flow

**Act 1 (current):**
1. DataFusion `ParquetExec` creates N partition streams
2. Each stream gets a `CoalescingParquetReader` (shared metadata cache, own range cache)
3. On `get_byte_ranges`: check cache → metadata lookup (shared) → compute prefetch → fetch (await) → spawn async prefetch → return
4. ObjectStore call goes to `LocalFileSystem` (pread) or `IoUringStore` (io_uring)

**Act 3a (scheduler, single-file — negative result):**
1. CoalescingReader submits ranges to `IoScheduler::submit_ranges()` instead of ObjectStore
2. Scheduler accumulates requests in a lock-free MPSC queue
3. Scheduler thread: drain queue → assemble batch (≥50 SQEs, capped at 256) → submit to io_uring ring → poll CQEs → dispatch completions to waiting futures
4. Pipeline: while CQEs from batch N arrive, accept requests for batch N+1
5. O_DIRECT: all I/O bypasses page cache; aligned buffer pool provides memory
6. Size-adaptive: reads ≥128 KiB routed to pread fallback (io_uring overhead > benefit)
7. **Result:** On single-file, most column chunks are >128 KiB → pread dominates → scheduler adds no benefit (1.023×)

**Act 3b (scheduler, multi-file — falsified):**
1. DataFusion `ListingTable` scans directory of Parquet files → creates ParquetExec with many partitions
2. Each file gets a `CoalescingParquetReader`, all sharing the same `Arc<IoScheduler>`
3. **Result:** Column chunk size is format-invariant — splitting into N files doesn't produce smaller reads.
4. Scheduler metrics (SQEs, preads) nearly identical across N=1..226 (Exp-12). File count is orthogonal to read granularity.
5. Inline io_uring reader (Exp-14, zero thread hops) improved to 1.054× but still far short. Q7 with 100% io_uring (309 SQEs) shows 0.70× — overhead exceeds savings at 50–500 KiB read sizes.
6. **Conclusion:** Read granularity is a data format property. Must change the format, not the scheduler.

**Act 4 (page-granular format + inline io_uring — new):**
1. New page-granular columnar format: fixed 4–8 KiB pages, each independently compressed (FOR/DELTA/DICT)
2. Multi-index pruning stack: per-page zone maps (range on sort-key), Bloom filters (equality on high-cardinality), per-page bitmaps (equality on low-cardinality) — dynamically selected per column at write time, dynamically routed per predicate at read time (see §3.5)
3. Selective query (≤5% selectivity): evaluate multi-index predicates → identify qualifying pages (e.g., 50 of 1000) → read only those pages
4. Each qualifying page is 4–8 KiB → hundreds of small scattered reads → exactly io_uring's sweet spot
5. Inline io_uring reader (thread-local rings, zero channel hops, O_DIRECT) submits all page reads as batched SQEs
6. Full scan path: read all pages sequentially (same as Parquet column scan, ≤20% overhead from page headers)
7. **Expected:** ≥3× on selective queries (selectivity ≤5%), ~1.0× on full scans

### 3.4 Act 4 Component Registry

#### Mica Format Writer — `mica/src/writer/` (NEW — Act 4)
- **Role:** Converts Arrow RecordBatches to Mica page-granular columnar files with multi-index metadata.
- **Interface exposed:** `MicaWriter::new(path, schema, config) → MicaWriter`, `MicaWriter::write_batch(batch)`, `MicaWriter::finish() → FileMetadata`
- **Key design decisions:** Fixed page size (default 8 KiB). Rows packed greedily into pages. Lightweight compression per page: PLAIN (no compression), FOR (frame-of-reference for integers), DELTA (sorted/monotonic), DICT (low-cardinality strings). Multi-index metadata computed per page during write: zone maps (always), Bloom filters (high-cardinality columns, >100 distinct), per-page bitmaps (low-cardinality columns, ≤100 distinct). Index selection is automatic based on per-column cardinality analysis (configurable threshold). Footer written last with offsets to all metadata/index sections. See §3.5 for full index design rationale.
- **Supports claims:** C15, C16, C17, C18

#### Mica Format Reader — `mica/src/reader/` (NEW — Act 4)
- **Role:** Reads Mica page-granular files with multi-index predicate pushdown + batched io_uring I/O.
- **Interface exposed:** `MicaReader::open(path) → MicaReader`, `MicaReader::scan(projection, predicate) → RecordBatchStream`
- **Interface consumed:** `ReadEngine` trait (`mica/src/io/`) — inline io_uring (`uring.rs`) or pread fallback (`fallback.rs`)
- **Key design decisions:** All index metadata loaded eagerly on open (zone maps always; Bloom/bitmaps if present — contiguous per column, single read each). Multi-index predicate routing: range predicates → zone maps, equality/IN → bitmap (exact) or Bloom (probabilistic) based on `index_flags`. Compound predicates AND-composed across columns. Only qualifying pages read via batched io_uring. Each page decompressed independently. Late materialization: position lists from one column applied to other columns. See §3.5 for full predicate routing logic.
- **Supports claims:** C14, C15, C16, C18

#### Inline io_uring Reader — `eval/clickbench/src/inline_uring_reader.rs` (existing, Act 3 → reusable)
- **Role:** Zero-hop io_uring I/O engine. Thread-local rings, no channels, no batching delay.
- **Interface exposed:** `inline_read_ranges(path, ranges, ring_size, use_direct_io, metrics) → Vec<Bytes>`
- **Key design decisions:** `thread_local!` IoUring ring per blocking-pool thread. O_DIRECT with aligned buffers. All reads through io_uring (no pread fallback). Validated in Exp-14 (1.054× on Parquet — modest due to large reads; expected ≥3× on 4–8 KiB pages).
- **Supports claims:** C14, C15, C18

#### Page-Granular File Layout

```
┌──────────────────────────────────────┐
│ File Header (32 bytes)               │  magic, version, row_count, col_count
├──────────────────────────────────────┤
│ Column Metadata × N                  │  name, type, encoding, page_count,
│  (fixed-size per column)             │  index_flags, zone_map_offset,
│                                      │  bloom_offset, bitmap_offset,
│                                      │  data_offset
├──────────────────────────────────────┤
│ Zone Maps                            │  per-page min/max for each column
│  (contiguous per column,             │  ALWAYS present (16 bytes/page/col)
│   sorted by column then page)        │  one read loads all zone maps
├──────────────────────────────────────┤
│ Bloom Filters                        │  per-page Bloom filter per column
│  (contiguous per column)             │  ONLY for high-cardinality columns
│                                      │  (index_flags & BLOOM)
├──────────────────────────────────────┤
│ Per-Page Bitmaps                     │  per-page distinct-value bitmap
│  (contiguous per column)             │  ONLY for low-cardinality columns
│                                      │  (index_flags & BITMAP)
├──────────────────────────────────────┤
│ Column Pages                         │  4-8 KiB aligned pages
│  Column 0: [Page 0][Page 1]...       │  each independently compressed
│  Column 1: [Page 0][Page 1]...       │  page header: row_count, encoding,
│  ...                                 │  compressed_size, uncompressed_size
├──────────────────────────────────────┤
│ Footer (fixed size)                  │  offsets to metadata sections
└──────────────────────────────────────┘
```

### 3.5 Multi-Index Pruning Stack

Page-level zone maps alone are insufficient for most workloads. The birthday problem kills zone-map effectiveness on unsorted data: for a page with R=200 rows and selectivity s=5%, P(page skipped) = (1−s)^R = 0.003%. Even at 1% selectivity, only 13.4% of pages are skipped. The crossover for >50% skip rate is s < ln(2)/R ≈ 0.35% — far below the 5% threshold that covers 38% of Snowflake queries.

We address this with a **multi-index pruning stack** that covers complementary query patterns:

#### Index Structure Comparison

| Index | Query Pattern | Space/Page (R=200) | False Positives | When Effective |
|-------|--------------|-------------------|-----------------|----------------|
| **Zone maps** (min/max) | Range on sort-key | 16 bytes/col | None | Sorted/clustered columns only |
| **Bloom filters** | Point/equality on high-cardinality | ~275 bytes/col | Configurable FPR (~1%) | Any column (unsorted OK), cardinality >100 |
| **Per-page bitmaps** | Point/equality on low-cardinality | 1–12 bytes/col | None (exact) | Any column (unsorted OK), cardinality ≤100 |
| **Z-order/Hilbert clustering** | Multi-dimensional range | 0 (write-time layout cost) | N/A | 2–3 sort dimensions |
| *(gap — no solution)* | Range on unsorted non-sort-key | — | — | Full scan required |

#### Advantages and Disadvantages

**Zone Maps (min/max per page)**
- ✅ Negligible space (16 bytes/page/col). Zero false positives. Fast O(1) evaluation.
- ✅ Excellent for range predicates on sorted/clustered columns.
- ❌ Completely useless on unsorted data — birthday problem guarantees every page spans the full value range.
- ❌ Cannot help equality/IN predicates on high-cardinality columns.
- *Always present* — cost is negligible and they enable range pruning on the sort key.

**Bloom Filters (per-page, ~10 bits/element)**
- ✅ Works on ANY column regardless of sort order. Point/equality/IN pruning.
- ✅ Configurable space-accuracy tradeoff via FPR parameter.
- ✅ On unsorted high-cardinality data (e.g., user_id with 1M distinct values), P(value in page) = R/cardinality = 200/1M = 0.02% — skips 99.98% of pages.
- ❌ Cannot answer range queries (>, <, BETWEEN).
- ❌ Space overhead: ~275 bytes/page/col at 10 bits/element × 200 elements. For 15 GB dataset indexing 5 columns: ~625 MB ≈ 4%.
- ❌ False positives at configured rate (~1% default). FPR worsens with higher page load factor.
- *Present when* `index_flags & BLOOM` — writer enables per column based on cardinality analysis.

**Per-Page Bitmaps (exact distinct-value set)**
- ✅ Zero false positives — exact set membership test.
- ✅ Tiny space for low-cardinality: 10 values → 10 bits/page (1.25 bytes), 50 → 50 bits (6.25 bytes).
- ✅ Equivalent to ClickHouse `set(N)` skip index — proven concept.
- ❌ Space grows linearly with cardinality — impractical above ~100 distinct values.
- ❌ Cannot answer range queries (only equality/IN).
- ❌ Every page likely contains all values at low cardinality — limits I/O skipping (but compound predicates with zone maps or Bloom filters on other columns still help).
- *Present when* `index_flags & BITMAP` — writer enables per column based on cardinality analysis.

**Z-Order/Hilbert Clustering (write-time layout)**
- ✅ No read-time overhead — embedded in data order.
- ✅ Enables zone-map pruning on 2–3 dimensions simultaneously.
- ✅ Z-order formula: effective selectivity = s^(1/D) for D dimensions.
- ✅ 200-row pages amplify clustering benefit (finer zone-map granularity than 450K-row Parquet RGs).
- ❌ Write-time decision — cannot be changed without rewriting data.
- ❌ Diminishing returns beyond 3 dimensions. Hilbert 24–40% better than Z-order but more complex.
- ❌ Optimizes for anticipated query patterns — hurts unanticipated patterns.
- *Applied at write time* — orthogonal to index structures.

#### Dynamic Index Selection (Write-Time)

The writer automatically selects which indexes to build per column based on column statistics computed during the first pass over the data:

```
per column:
  zone_maps = ALWAYS                           # 16 bytes/page — negligible
  if cardinality(column) ≤ 100:
    bitmap  = YES                              # 1–12 bytes/page — exact membership
    bloom   = NO                               # redundant: bitmap is exact
  elif cardinality(column) > 100:
    bloom   = YES                              # ~275 bytes/page — probabilistic membership
    bitmap  = NO                               # impractical: too many bits

  index_flags |= ZONE_MAP                      # always set
  index_flags |= BLOOM   if bloom enabled
  index_flags |= BITMAP  if bitmap enabled
```

The cardinality threshold of 100 is based on: at 100 distinct values × 200 rows/page, the birthday problem gives P(all values present) ≈ 86%, limiting bitmap-based page skipping. Beyond 100, Bloom filters become more space-efficient. The threshold is configurable via writer config.

Optional: user can force specific indexes via writer config (e.g., force Bloom on a column the writer would skip, disable bitmap to save space).

#### Dynamic Predicate Routing (Read-Time)

The reader evaluates each predicate against the most efficient available index for that column:

```
fn evaluate_predicate(pred: &Predicate, col_meta: &ColumnMeta) -> PageBitmap {
    match pred.op {
        // Range predicates: zone maps only
        Gt | Lt | Ge | Le | Between =>
            evaluate_zone_maps(pred, col_meta.zone_maps),

        // Equality/IN predicates: best available index
        Eq | In => {
            if col_meta.index_flags.has(BITMAP) {
                evaluate_bitmap(pred, col_meta.bitmaps)     // exact, zero FPR
            } else if col_meta.index_flags.has(BLOOM) {
                evaluate_bloom(pred, col_meta.blooms)        // probabilistic
            } else {
                evaluate_zone_maps(pred, col_meta.zone_maps) // fallback
            }
        }

        // NotEq: complement of Eq
        Ne => !evaluate_predicate(&pred.as_eq(), col_meta),
    }
}

fn evaluate_compound(preds: &[Predicate], metadata: &FileMetadata) -> PageBitmap {
    let mut result = PageBitmap::all_set(metadata.page_count);
    for pred in preds {
        // AND composition: intersect page bitmaps
        result &= evaluate_predicate(pred, &metadata.columns[pred.column_idx]);
    }
    result
}
```

**Compound pruning example** (demonstrating why multiple indexes are essential):
- Query: `WHERE timestamp BETWEEN '2023-01-01' AND '2023-01-07' AND product_id = 42`
- Zone maps prune on timestamp (sorted column) → 1% of pages survive (5 of 500)
- Bloom filter prunes on product_id (unsorted, high-cardinality) → 0.02% of remaining
- Net: read ~0.0002% of total pages — massive I/O reduction from compound index evaluation

#### Known Limitation: Range on Non-Sort-Key Columns

Range predicates (>, <, BETWEEN) on columns NOT in the sort key remain unprunable by any of the three index structures. This is a fundamental limitation: zone maps require sorted data for effective min/max ranges, and Bloom filters/bitmaps cannot answer range queries. SuRF (SIGMOD 2018) requires sorted keys for trie construction; SNARF (VLDB 2022) has false negative issues; Rosetta (SIGMOD 2020) targets LSM-trees. No production columnar format has solved per-page range pruning on unsorted data.

For the ATC paper, we acknowledge this gap and scope our claims to: (a) range predicates on sort-key columns (zone maps), (b) equality/IN predicates on any column (Bloom/bitmap), and (c) compound predicates combining both. The Z-order/Hilbert clustering extends zone-map effectiveness to 2–3 dimensions. Full-scan fallback handles the unprunable case.

---

## 4. Milestones

### Milestone Map

```
                           DIAGNOSTIC ARC (complete)                                 RESOLUTION ARC (active)
M0 (Results) ──> M1 (Ablation) ──> M2 (Scheduler) ──> M3 (Single-file) ──> M4 (Multi-file) ──> M5 (Format) ──> M6 (Reader) ──> M7 (Experiments) ──> M8 (Paper)
     ✅               ✅              ✅              ✅ (negative)        ✅ (negative)          ✅              ⬜               ⬜                  ⬜
                                                   C7=1.023×            C11=0.985×           page-granular    inline io_uring   crossover +
                                                   regime boundary      read granularity      4-8K pages       batched reader    selective queries
                                                                       is binding constraint
```

### MVP Path
> **Diagnostic arc (M0–M4):** Complete. Comprehensive characterization of why io_uring fails for columnar analytics.
> C1–C6 diagnose the ObjectStore bottleneck. C7–C9 show single-file is wrong regime. C11–C13 prove file count is irrelevant — read granularity is the binding constraint.
> **Resolution arc (M5–M7):** Build a format with the RIGHT read granularity (4–8 KiB pages) and demonstrate that batched io_uring then delivers the expected speedups.
> M5 builds the page-granular format + writer. M6 integrates the inline io_uring reader. M7 runs experiments.
> M8 writes the paper. Story: diagnosis → root cause (format, not interface) → crossover measurement → page-granular resolution.
> **If M7 C15 < 1.5×:** Format overhead dominates savings — investigate compression, page alignment, decompression cost. Still publishable as comprehensive negative result with crossover measurement (C14).
> **OSDI follow-up (separate paper):** Cross-query I/O batching, multi-tenant cooperative scheduling. See `OSDI_PLAN.md`.

---

### M0: Consolidate Existing Results ✅
**Goal:** All Phase 2b v1 and v2 results organized and validated.
**Status:** Complete. Journals 22 (coalescing results) and 23 (reactor negative result).
→ Full task breakdown: `tasks/M0/`

### M1: Ablation + Landscape Analysis ✅
**Goal:** K-sweep ablation (C5), landscape survey (C6), warm/cold decomposition (C4).
**Status:** Complete. Journal 24 (K-sweep + landscape). All 6 Act 1–2 claims validated.
→ Full task breakdown: `tasks/M1/`

### M2: I/O Scheduler Design + Implementation ✅
**Goal:** Purpose-built I/O scheduler with O_DIRECT, io_uring, pipelined submission, and registered buffer pool. Integrated with CoalescingReader.
**Duration:** 3–4 weeks
**Status:** Complete. 69/69 tests pass. Full correctness validation.
→ Full task breakdown: `tasks/M2/`

### M3: Single-file Scheduler Benchmarks ✅ (Negative Result)
**Goal:** Run Exp-7 (E2E speedup), Exp-8 (O_DIRECT prerequisite), Exp-9 (batch depth sensitivity). Validate C7, C8, C9.
**Duration:** 1–2 weeks
**Status:** Complete. C7 = 1.023× (negative), C8 = 1.011× (pass), C9 = 0.967× (insensitive). Scheduler adds no measurable E2E speedup on single-file. Root cause: structural mismatch between single-file I/O pattern and io_uring sweet spot.
→ Full task breakdown: `tasks/M3/`
→ Journal: `docs/journal/2026-04-10-scheduler-negative-result.md`

### M4: Multi-file Partitioned Benchmark ✅ (Negative Result)
**Goal:** Validate workload regime hypothesis on two complementary benchmarks.
**Status:** Complete. C11 = 0.985× (FAIL), C12 = flat (FAIL), C13 = 0.97× (FAIL). Multi-file hypothesis falsified. Column chunk size is a format property, not a function of file count. Inline io_uring (Exp-14) improved to 1.054× but still modest.
→ Full task breakdown: `tasks/M4/`
→ Journal: `docs/journal/2026-04-11-multifile-hypothesis-falsified.md`

### M5: Page-Granular Format Design + Writer ✅
**Goal:** Design and implement a minimal page-granular columnar format with fixed 4–8 KiB pages, lightweight per-page compression, and per-page zone maps. Build a writer that converts Arrow RecordBatches to this format. Convert ClickBench and TPC-H lineitem datasets.
**Duration estimate:** 2 weeks
**Blocks:** M6
**Done when:** (1) Format spec document written. (2) Writer library produces valid page-granular files. (3) ClickBench HITS and TPC-H lineitem converted. (4) Read-back validation passes (row count + checksum match).
→ Full task breakdown: `tasks/M5/`

### M6: Batched io_uring Reader + DataFusion Integration ⬜
**Goal:** Build a page-granular file reader that uses zone-map predicate pushdown to identify qualifying pages and reads them via batched inline io_uring. Integrate as a DataFusion `TableProvider` for E2E query execution.
**Duration estimate:** 2 weeks
**Blocks:** M7
**Done when:** (1) Reader loads zone maps, evaluates predicates, produces page bitmaps. (2) Qualifying pages read via `inline_read_ranges()`. (3) DataFusion queries on page-granular files produce correct results. (4) Selective queries show significantly fewer I/O bytes than full scan.
→ Full task breakdown: `tasks/M6/`

### M7: Crossover + Selective Query Experiments ⬜
**Goal:** Run the evaluation experiments. Measure read-size crossover (Exp-15), selective query speedup (Exp-16), selectivity sweep (Exp-17), TPC-H selective queries (Exp-18), and full-scan overhead (Exp-19). Validate C14–C18.
**Duration estimate:** 1–2 weeks
**Blocks:** M8
**Done when:** All 5 experiments complete, verdicts produced, Journal 28 written with cross-experiment analysis.
→ Full task breakdown: `tasks/M7/`

### M8: Paper Writing & Submission ⬜
**Goal:** Complete ATC paper draft. Story: diagnosis (why io_uring fails on columnar) → root cause (format granularity) → crossover measurement (first systematic study) → page-granular resolution (format + batched io_uring) → evaluation.
**Duration estimate:** 2–3 weeks
**Blocks:** Submission
**Done when:** Paper submitted.
→ Full task breakdown: `tasks/M8/`

---

## 5. Evaluation Plan

### Testbed / Experimental Setup

- **Hardware:** Intel Xeon Gold 6530, 32 cores / 64 threads, 251 GiB DDR5, 894 GB NVMe (ext4 on LVM), AVX-512
- **OS / kernel:** Linux 6.8.0-106-generic SMP PREEMPT_DYNAMIC (Ubuntu)
- **Software:** DataFusion 47, Arrow 55, Parquet 55, object_store 0.12, io-uring 0.7, Rust stable (release builds)
- **Dataset:** ClickBench `hits.parquet` (14.78 GiB, 226 row groups × 105 columns); multi-file partitioned versions at N=10,50,100,226 files; TPC-H SF=10 (~10 GiB total, lineitem ~6 GiB, 100-file flat split); Mica format versions of both datasets (generated by `MicaWriter`)
- **Reproducibility:** All scripts in `eval/clickbench/scripts/` and `mica/scripts/`; `DRY_RUN=1` for validation; `HITS` env var for dataset path.

### Experiments

#### Exp-1: Coalescing Reader Speedup (H1) — validates C1
- **Metric:** Median wall_ms ratio (L,D)/(L,C) across 10 queries
- **Status:** ✅ DONE — H1 = 1.438× (Journal 22)

#### Exp-2: io_uring Marginal Benefit (H2) — validates C2
- **Metric:** Median wall_ms ratio (L,C)/(I,C)
- **Status:** ✅ DONE — H2 = 0.998× (Journal 22)

#### Exp-3: Reactor Negative Result — validates C3
- **Metric:** H2 with reactor vs without
- **Status:** ✅ DONE — reactor H2 = 0.887×, 11% regression (Journal 23)

#### Exp-4: Warm/Cold Decomposition — validates C4
- **Metric:** Warm H1 / Cold H1
- **Status:** ✅ DONE — ratio = 1.41 (Journal 22)

#### Exp-5: K-Sweep Ablation — validates C5
- **Metric:** H1 at K=1 vs K=4
- **Status:** ✅ DONE — meta_frac ≈96%, C5 PASS (Journal 24)

#### Exp-6: Engine Landscape Survey — validates C6
- **Metric:** Count of request-response engines
- **Status:** ✅ DONE — 6/9 (Journal 24, landscape-survey.md)

#### Exp-7: Single-file Scheduler E2E Speedup — validates C7
- **Metric:** Median wall_ms ratio (Coalescing+pread) / (Scheduler+O_DIRECT+io_uring) across 10 queries, cold cache
- **Setup:** Coalescing-reader-only as baseline. Scheduler+CoalescingReader as treatment. 5 iterations per cell, cold cache (drop_caches).
- **Status:** ✅ DONE — C7 = 1.023× (negative result, Journal 26)
- **Key finding:** Even queries with 300+ SQEs through io_uring show ~1.0× speedup. Scheduler overhead cancels syscall savings.
- **Tasks:** `tasks/M3/M3-T2-exp7-run.md`

#### Exp-8: O_DIRECT Prerequisite — validates C8
- **Metric:** Median wall_ms ratio with scheduler+buffered vs coalescing-reader-only
- **Status:** ✅ DONE — C8 = 1.011× (PASS — buffered scheduler is within ≤1.03×, Journal 26)
- **Tasks:** `tasks/M3/M3-T3-exp8-run.md`

#### Exp-9: Batch Depth Sensitivity (Single-file) — validates C9
- **Metric:** wall_ms at batch sizes N=1,10,50,100,256
- **Status:** ✅ DONE — C9 = 0.967× max variation (insensitive, Journal 26)
- **Key finding:** On single-file, too few small reads exist for batching to matter.
- **Tasks:** `tasks/M3/M3-T4-exp9-run.md`

#### Exp-11: Multi-file Scheduler A/B Comparison — validates C11
- **Metric:** Median wall_ms ratio (no-scheduler on 100-file) / (scheduler on 100-file) across 10 queries, cold cache
- **Setup:** 100-file partition of HITS. Same queries as Exp-7. CoalescingReader-only vs CoalescingReader+IoScheduler. 5 iterations per cell, cold cache.
- **Baselines:** CoalescingReader on multi-file partition without scheduler (pread, buffered I/O)
- **Expected result:** ≥1.3× median. Multi-file layout produces 300–1000+ small scattered reads across 100 fds. Narrow column chunks <128 KiB → io_uring path instead of pread fallback.
- **Failure mode:** If <1.05×, multi-file hypothesis is falsified — pivot to pure negative-result paper. If 1.05–1.3×, partial success — check if higher N (Exp-12) reaches target.
- **Status:** ✅ DONE — C11 = 0.985× median speedup (FAIL, Journal 27)
- **Key finding:** 100-file split produces same read sizes as single-file. Pread-dominated I/O pattern unchanged.
- **Tasks:** `tasks/M4/M4-T4-exp11-multifile-ab.md`

#### Exp-12: File Count Sweep — validates C12
- **Metric:** Median speedup at N=1,10,50,100,226 files. Speedup = median(no-scheduler@N) / median(scheduler@N).
- **Setup:** 3 representative queries (q2, q7, q42). 5 iterations per cell. Both scheduler-on and scheduler-off at each N. Cold cache.
- **Expected result:** Monotonic increase. N=1 ≈1.02× (matches M3). N≥100 ≥1.3×. Transition from "no benefit" to "benefit" at N≈50.
- **Failure mode:** If not monotonic, check for confounding factors (row group distribution, ListingTable overhead). If plateau before N=50, scheduler saturates early — still useful (low transition point).
- **Status:** ✅ DONE — C12 FAIL: flat ~1.0× at N≥10, N=1 anomalous outlier (Journal 27)
- **Key finding:** Scheduler metrics (SQEs, preads) nearly identical across N=1..226. File count is orthogonal to read size.
- **Tasks:** `tasks/M4/M4-T5-exp12-filecount-sweep.md`

#### Exp-13: TPC-H Multi-file Scheduler A/B Comparison — validates C13
- **Metric:** Median wall_ms ratio (no-scheduler) / (scheduler) on TPC-H SF=10 lineitem, I/O-intensive queries, cold cache
- **Setup:** TPC-H SF=10 lineitem table partitioned into ~100+ Parquet files by `l_shipdate`. Select 6–10 I/O-intensive TPC-H queries (Q1, Q3, Q5, Q6, Q10, Q12, Q14, Q19). Register all tables via `ListingTable` with `CoalescingParquetFileReaderFactory`. IoScheduler shared across all tables. Baseline = CoalescingReader only (no scheduler). Treatment = CoalescingReader + IoScheduler. 5 iterations per cell, cold cache.
- **Baselines:** (1) CoalescingReader on TPC-H multi-file without scheduler (pread, buffered I/O) — same code path, scheduler disabled. (2) Implicit comparison to M3 single-file ClickBench result (1.02×).
- **Expected result:** ≥1.3× median on I/O-intensive queries. TPC-H lineitem partitioned by date produces ~100 files × narrow column projections → many scattered reads <128 KiB across distinct fds → matches io_uring regime.
- **Failure mode:** If <1.05×, the multi-file regime hypothesis fails on a standard benchmark — more concerning than ClickBench split failure since TPC-H is a recognized workload. If 1.05–1.3×, check whether query selection skewed toward compute-heavy queries and whether lineitem partition layout produces sufficient small reads.
- **Status:** ✅ DONE — C13 = 0.97× median speedup (FAIL, Journal 27)
- **Key finding:** Even with 100 lineitem files and heavy I/O (Q1: 5439 total ops), scheduler adds overhead. Higher io_uring utilization correlates with worse regression.
- **Tasks:** `tasks/M4/M4-T10-exp13-tpch-ab.md`

#### Exp-14: Inline io_uring vs Default (100-file ClickBench) — validates architectural overhead hypothesis
- **Metric:** Median wall_ms ratio (default factory) / (inline-uring factory) across 10 queries, cold cache
- **Setup:** Same 100-file partition as Exp-11. Baseline = `--factory default --store local`. Treatment = `--factory inline-uring --store local`. 5 iterations per cell, cold cache.
- **Status:** ✅ DONE — 1.054× median speedup. Eliminated catastrophic regressions from old scheduler (Q5: 0.63→1.15×, Q13: 0.59→1.07×). Zero preads, zero stalls. But still modest — column chunks are 50–500 KiB.
- **Key finding:** Architecture matters (3 thread hops hurt) but read granularity is the dominant factor. Even with zero-hop io_uring, 50–500 KiB reads don't benefit from batching.
- **Results:** `results/run_20260411_100148/exp14/`

#### Exp-15: Read-Size Crossover Micro-benchmark — validates C14
- **Metric:** Throughput (GB/s) and latency (µs) of io_uring batched reads vs pread at read sizes from 512B to 512 KiB. Identify crossover point where io_uring speedup transitions from >1.5× to <1.1×.
- **Setup:** Synthetic file (1 GiB random data). Read N ranges at fixed size, random offsets. Measure: io_uring batched (batch_size=64) vs sequential pread. Both O_DIRECT. Sweep read_size ∈ {512, 1K, 2K, 4K, 8K, 16K, 32K, 64K, 128K, 256K, 512K}. QD=64. 100 iterations per configuration.
- **Expected result:** ≥2× at 4K, crossover (1.0×) at 16–32 KiB. Below crossover: io_uring batching wins (many syscalls saved). Above crossover: per-call overhead dwarfed by I/O time.
- **Failure mode:** If crossover is at 2 KiB or below, page-granular format needs very small pages (impractical due to header overhead). If crossover is at 128+ KiB, our Parquet results don't make sense (should have seen benefit).
- **Status:** ⬜ Pending
- **Tasks:** `tasks/M7/M7-T1-exp15-crossover.md`

#### Exp-16: Selective Query A/B (Page-Granular vs Parquet) — validates C15
- **Metric:** Median wall_ms ratio (Parquet + DataFusion default) / (page-granular + io_uring) on 10+ selective ClickBench queries, cold cache
- **Setup:** Same ClickBench data in both Parquet and page-granular format. Select queries with natural selectivity ≤5% (e.g., Q0 CounterID=62, Q8 URL LIKE, Q29 EventDate range, Q42 CounterID+EventDate). Add synthetic selective variants if needed. 5 iterations per cell, cold cache.
- **Baselines:** DataFusion + Parquet (default factory, page cache) — the standard stack.
- **Expected result:** ≥3× median on selective queries. Zone maps prune 95%+ of pages → 50 pages × 8 KiB = 400 KiB total I/O vs Parquet reading entire column chunks.
- **Failure mode:** If <1.5×, decompression overhead or zone-map evaluation cost dominates. If 1.5–3×, partial success — check which component limits (I/O, decode, zone-map eval).
- **Status:** ⬜ Pending
- **Tasks:** `tasks/M7/M7-T2-exp16-selective-ab.md`

#### Exp-17: Selectivity Sweep — validates C16
- **Metric:** Speedup (Parquet / page-granular+io_uring) at selectivity = 100%, 50%, 20%, 10%, 5%, 1%, 0.1%.
- **Setup:** Synthetic filter on ClickBench (e.g., `EventDate >= X AND EventDate <= Y` with varying X,Y to control selectivity). Measure wall_ms at each selectivity level.
- **Expected result:** Monotonic: ~1.0× at 100% (full scan, format overhead cancels), increasing speedup as selectivity decreases, ≥3× at ≤5%.
- **Failure mode:** Non-monotonic curve (unexpected). Or speedup plateau before reaching 3× even at 0.1%.
- **Status:** ⬜ Pending
- **Tasks:** `tasks/M7/M7-T3-exp17-selectivity-sweep.md`

#### Exp-18: TPC-H Selective Queries — validates C18
- **Metric:** Median wall_ms ratio (Parquet + DataFusion) / (page-granular + io_uring) on TPC-H queries with natural selectivity (Q6 date+discount, Q12 shipmode, Q14 date range).
- **Setup:** TPC-H SF=10 lineitem in both Parquet and page-granular format. Select 3–5 queries with date range predicates that naturally achieve ≤10% selectivity on lineitem. 5 iterations, cold cache.
- **Expected result:** ≥2× median. TPC-H's l_shipdate ranges typically select 1–15% of rows.
- **Failure mode:** If <1.5×, TPC-H queries may be too compute-heavy (hash joins dominate). Focus on lineitem-only aggregation subqueries.
- **Status:** ⬜ Pending
- **Tasks:** `tasks/M7/M7-T4-exp18-tpch-selective.md`

#### Exp-19: Full Scan Overhead — validates C17
- **Metric:** Median wall_ms ratio (page-granular full scan) / (Parquet full scan). No predicate pushdown — read all data.
- **Setup:** Full-table scans on ClickBench (all rows, all queried columns). Compare page-granular format vs Parquet. 5 iterations, cold cache. Measure both wall time and I/O bytes.
- **Expected result:** ≤1.20× (≤20% overhead). Page headers add overhead but pages are read sequentially.
- **Failure mode:** If >1.5×, format overhead too high — investigate: are page headers too large? Is per-page decompression less efficient than per-chunk? Is sequential read pattern hurt by O_DIRECT?
- **Status:** ⬜ Pending
- **Tasks:** `tasks/M7/M7-T5-exp19-fullscan-overhead.md`

### What "winning" looks like

**Strong outcome (target):** C15 ≥3× on selective ClickBench queries AND C18 ≥2× on TPC-H selective queries. C14 shows clear crossover at ~16–32 KiB (validates format design choice). C16 shows monotonic selectivity-speedup curve. C17 confirms ≤20% overhead on full scans. Combined with M0–M4 diagnostic arc, the paper tells a complete story: existing formats prevent io_uring from helping → we prove it with negative results on every angle → we measure the crossover → we build a format that exploits it → ≥3× on the dominant production workload (38% of queries per Snowflake).

**Acceptable outcome:** C15 = 1.5–3×, C18 = 1.3–2×. Still publishable with tempered claim. Crossover measurement (C14) and diagnostic arc (C1–C13) carry the paper. Partial speedup with clear selectivity-dependent curve is a meaningful contribution.

**Fallback (negative on format too):** C15 < 1.5×. The paper becomes a comprehensive negative result: neither the I/O interface, nor the scheduling architecture, nor the file layout, nor the format granularity can make io_uring consistently help analytical queries. Still publishable at ATC as experience/measurement paper with the crossover finding (C14). The crossover measurement alone is novel — no prior work has done this systematically.

---

## 6. Risk & Fallback Map

| ID | Risk | Likelihood | Impact | Mitigation | Fallback |
|----|------|-----------|--------|------------|---------|
| R1 | Amdahl ceiling: I/O is only 18–59% of query time | High | Med | Target I/O-heavy queries (q0, q2, q3, q23). Multi-file layout increases I/O fraction. | Report per-query results; compute-bound queries won't benefit — that's the story |
| R2 | O_DIRECT warm regression: bypassing page cache hurts warm queries | Med | Med | Measure warm performance explicitly. Consider hybrid mode (O_DIRECT cold, buffered warm). | Scope claim to cold-cache workloads; note warm as limitation |
| R3 | Reviewer says "just engineering applying known io_uring patterns" | Med | High | Evidence pyramid (3-layer decomposition) is methodological contribution. Landscape survey (9 engines) shows ecosystem-wide impact. Single-file negative result proves this is NOT simple. | Emphasize negative results + diagnostic methodology |
| R4 | Jasny et al. VLDB 2026 overlap | Med | Med | They study buffer manager + network; we study analytical query I/O through ObjectStore. Different workload, different findings. | Cite and contrast explicitly in related work |
| R5 | LanceDB overlap | Med | Med | LanceDB uses Lance format, not Parquet; their scheduler is format-specific. No peer-reviewed publication. | Show our approach works within existing DataFusion/Parquet stack without format change |
| R6 | Single-file negative result undermines credibility | Low | Med | Frame as workload regime characterization, not failure. C7–C9 are positive results — they successfully identify WHERE io_uring doesn't help. | Strong negative results are publishable at systems venues |
| R7 | O_DIRECT alignment requirements break Parquet reads | Low | High | Parquet column chunks are often aligned; measure actual alignment distribution. | Read at aligned boundaries and trim; overhead should be minimal |
| R8 | Insufficient temporal overlap to batch requests | Med | Med | Multi-file layout creates temporal overlap naturally — many files produce concurrent readers. CoalescingReader async prefetch adds more overlap. | If batch depth still low on multi-file, issue is structural — report as finding |
| R9 | ListingTable overhead masks scheduler benefit | Low | Med | Profile ListingTable vs single-file registration overhead independently. Use same ListingTable for both baseline and treatment. | Any overhead is shared between baseline and treatment — cancels in ratio |
| R10 | Partitioned file sizes differ from production layouts | Med | Low | Use realistic N values (10–226). Production Hive/Iceberg tables typically have 100–10000 files. | Report file count sweep so practitioners can extrapolate to their layout |
| R11 | TPC-H queries are join-heavy; I/O may be masked by hash join compute | Med | Med | Select I/O-intensive queries (Q1, Q6, Q12 — simple aggregations on lineitem). Report both I/O-intensive and compute-intensive subsets. | If joins dominate, report per-table I/O metrics separately; lineitem-only queries should show clearest benefit |
| R12 | TPC-H partition layout differs from split ClickBench (date-partitioned vs row-group split) | Low | Low | This is a feature, not a bug — TPC-H partitioning is production-realistic. Document the layout difference. | Difference in results between ClickBench split and TPC-H explains generalizability bounds |
| R13 | tpchgen-rs data generation quality or DataFusion TPC-H query compatibility issues | Low | Med | Validate with single-file first (correctness check against known TPC-H answers). Use DataFusion's standard SQL for TPC-H queries. | Fall back to pyarrow data generation + manual query validation |
| R14 | Page-granular format decompression overhead exceeds scan savings | Med | High | Lightweight compression only (FOR/DELTA/DICT, no Snappy/Zstd). Page header overhead minimized (8–16 bytes). Benchmark decompression in isolation first. | Offer PLAIN (uncompressed) encoding as default; show raw I/O benefit without compression cost |
| R15 | Zone-map selectivity insufficient on ClickBench columns (min/max too wide) | Med | Med | Choose columns with natural clustering (EventDate, CounterID). Add synthetic pre-sorted datasets if needed. Report zone-map effectiveness per column. | Use columns known to cluster well. If zone maps too coarse, this is itself a finding about format limitations |
| R16 | Lance format comparison: reviewers ask "why not just use Lance?" | High | High | Lance is optimized for vector/AI workloads (full-zip encoding for embeddings). Our format targets pure columnar analytics. Lance has no io_uring integration. We provide direct A/B if possible. | Position as complementary: Lance targets random access on embeddings; we target selective scans on analytics columns. Cite Lance paper and differentiate clearly |
| R17 | Crossover measurement is hardware-specific (different NVMe may differ) | Med | Low | Report NVMe model, firmware, queue depth. Discuss expected variation. | The measurement methodology transfers; absolute crossover point is parameterized by hardware. This is a feature, not a bug — practitioners measure their own crossover |
| R18 | Page-granular format loses Parquet ecosystem compatibility (tools, metadata, statistics) | Med | Med | Position as research prototype, not production replacement. Parquet remains the storage format; page-granular is the I/O-optimized analytical format. | Frame as "what if" experiment: given optimal format, how much can io_uring help? The gap between Parquet and page-granular quantifies format cost |
| R19 | Bloom filter space overhead exceeds I/O savings | Med | Med | ~275 bytes/page/col × 5 columns ≈ 4% of dataset. Bloom filters loaded once per column (contiguous read). Compare: Parquet column-chunk Bloom filters are similarly sized. Benchmark index load time vs page skip savings. | Make Bloom optional — run experiments with and without. If overhead too high, reduce to 2–3 columns with highest cardinality |
| R20 | Per-page bitmap ineffective on low-cardinality columns (every page has all values) | High | Low | Birthday problem: 10 values × 200 rows → 100% presence probability. Bitmap confirms presence, cannot skip. But compound predicates with zone maps or Bloom on OTHER columns still reduce page set. | Position bitmap as AND-composition accelerator, not standalone. Show compound predicate examples where bitmap + zone map together achieve high skip rates |
| R21 | Dynamic index selection threshold (cardinality=100) is wrong for some data distributions | Low | Low | Threshold is configurable in writer config. Sensitivity analysis: try 50, 100, 200. | Report sensitivity in paper. 100 is a reasonable default per ClickHouse set(N) precedent |

---

## 7. Iteration Checkpoints

### CP-1: After M2 (Scheduler implementation) ✅
**Question:** Does the scheduler pass correctness tests and run end-to-end?
- YES → Proceed to M3 benchmarks
- **Result:** YES — 69/69 tests pass. Proceeded to M3.

### CP-2: After M3 (Single-file benchmarks) ✅ (Negative)
**Question:** Does the scheduler show ≥1.10× median E2E improvement on single-file?
- YES (≥1.15×) → C7 validated, proceed to full evaluation
- MARGINAL (1.05–1.15×) → Profile bottleneck
- NO (<1.05×) → Investigate workload regime mismatch; consider multi-file
- **Result:** NO — 1.023×. Root cause identified: single-file column chunks too large for io_uring sweet spot. Multi-file hypothesis formulated (Act 3b).

### CP-3: After M4-T1+T2 (Partitioned dataset + runner) ✅
**Question:** Does multi-file runner produce correct results with scheduler?
- **Result:** YES — correct results with both scheduler on and off. Proceeded to benchmarks.

### CP-4: After M4-T4 (Exp-11 — split ClickBench) ✅ (Negative)
**Question:** Does multi-file scheduler show ≥1.10× median E2E improvement?
- **Result:** NO — 0.985×. Proceeded to TPC-H and file count sweep anyway.

### CP-4b: After M4-T10 (Exp-13 — TPC-H) ✅ (Negative)
**Question:** Does TPC-H partitioned scheduler show ≥1.10× median E2E improvement?
- **Result:** NO — 0.97×. Both C11 and C13 failed. Multi-file hypothesis falsified. Pivoted to page-granular format approach (Act 4).

### CP-5: After M5 (Format writer complete) ✅
**Question:** Does the page-granular format writer produce valid files that can be read back correctly?
- YES → Proceed to M6 (reader integration)
- NO → Debug writer before building reader
- **Result:** YES — 72 tests pass (19 zone map + 22 encoding + 7 writer + 13 bloom + 11 bitmap). Format spec written. Writer produces valid files. Convert + validate CLIs compile. Proceeded to M6. **Note:** Actual data conversion (ClickBench + TPC-H) deferred to M6-T6-ST0.

### CP-6: After M6 (Reader + DataFusion integration)
**Question:** Do selective queries on page-granular format skip ≥90% of pages via zone maps?
- YES → Proceed to M7 (experiments). The I/O reduction should translate to speedup.
- NO → Zone maps too coarse, or predicates not pushable. Investigate column selection and predicate types.

### CP-7: After M7-T1 (Exp-15 crossover)
**Question:** Is the read-size crossover at ≤32 KiB?
- YES → Validates 4–8 KiB page design. Proceed to full experiments.
- NO (crossover at >64 KiB) → Our page size is in the wrong regime. Consider larger pages or re-evaluate.

### CP-8: After M7-T2 (Exp-16 selective A/B)
**Question:** Does page-granular + io_uring show ≥1.5× on selective queries?
- YES (≥3×) → C15 validated. Full paper proceeds.
- MARGINAL (1.5–3×) → Publishable with tempered claim. Analyze bottleneck.
- NO (<1.5×) → Format overhead dominates. Profile decompression, zone-map eval, I/O. Consider simpler encoding.

### CP-9: Before submission
**Question:** Is the story coherent across all experiments (M0–M7)?
- YES → Submit
- CONTRADICTIONS → Revisit; may need additional experiment or revised claim

---

## 8. Related Work & Positioning

| Paper | Venue/Year | What it does | How we differ |
|-------|-----------|--------------|---------------|
| Jasny et al. "io_uring for High-Performance DBMSs" | VLDB 2026 | Ring-per-thread + fibers for buffer mgr + network shuffle. 2.05× buffer mgr, 2.31× network. | We study analytical query I/O through ObjectStore; they study transactional buffer manager. Complementary domains. |
| Haas & Leis "What Modern NVMe Storage Can Do" | VLDB 2023 | io_uring slightly slower than libaio in default mode | We explain WHY (request-response API, large reads, buffered I/O) and show HOW to fix it |
| Spilly/Umami | SIGMOD 2025 | io_uring + hash partitioning for NVMe query processing | Full custom engine; we work within DataFusion's existing architecture |
| DataFusion | SIGMOD 2024 Industry | Architecture paper | No I/O optimization discussion; we fill this gap |
| LanceDB "Quest for 1M IOPS" | Blog 2024 | I/O scheduler + io_uring for Lance format | Confirms "io_uring by itself didn't help"; no peer-reviewed publication. Lance format, not Parquet. |
| DuckDB async I/O | PR #7331 | Async sink/source infrastructure | Internal refactoring; no published io_uring results |
| Umbra/CedarDB | VLDB 2020+ | Coroutines + io_uring for OLAP | Custom engine with async-submit API; confirms landscape analysis |
| Apache Arrow ObjectStore | Crate 0.12 | Storage abstraction for Arrow ecosystem (54.5M downloads, 314 reverse deps) | We characterize its request-response pattern as structural bottleneck and show how to work around it |
| Lance format | arXiv 2025 | Mini-block encoding (4–8K), 60× better random access than Parquet. Designed for AI/vector workloads. | We target pure columnar analytics with simpler encoding. Lance has no io_uring integration. Our crossover measurement applies to Lance's mini-blocks too. |
| FastLanes | VLDB 2025 | Fine-grained batch-level access with lightweight compression (FOR, DELTA, RLE). | Compression approach aligns with ours. They focus on decode throughput; we focus on I/O scheduling. Complementary. |
| F3: Future-proof File Format | SIGMOD 2025 | Decouples I/O unit from row group. Format evolution framework. | We take an opinionated approach (fixed pages) rather than a general framework. F3 doesn't study io_uring. |
| Snowflake Pruning | SIGMOD 2025 | 99.4% micro-partitions pruned. Production selectivity analysis. | Motivates our claim that selective queries dominate. We cite their data to justify page-granular design. |
| Snowflake Workload Analysis | VLDB 2025 | 38% of filters have ≤5% selectivity. 667M production queries analyzed. | Key motivation for C15. Our format targets this dominant workload segment. |
| TreeLine | VLDB 2022 | Record-oriented NVMe-native storage. Shows random ≈ sequential on NVMe with parallelism. | Supports our premise that NVMe random I/O is cheap. We apply this insight to columnar analytics. |
| "When DB Meets New Storage" | VLDB 2023 | 17 performance mismatches between DBs and NVMe. "Random-to-sequential may HURT." | Supports our claim that optimizing for sequential I/O is misguided on NVMe. |
| SuRF (Succinct Range Filters) | SIGMOD 2018 Best Paper | Truncated trie for point + range queries. ~14 bits/key. Requires sorted keys. | Demonstrates range filtering is possible but requires sorted data — our zone maps fill this role on sort-key columns. SuRF targets LSM-trees, not columnar formats. |
| Column Sketches | SIGMOD 2018 | Lossy per-value codes for scan acceleration. 3–6× improvement. | In-memory scan accelerator, not I/O skipping. Explicitly states: "Neither traditional indexing nor lightweight data skipping techniques provide performance benefits for queries with moderate selectivity over unclustered data." Motivates our multi-index approach. |
| Hippo | VLDB 2016 | Page-range histograms, 100× smaller than B+tree. | Supports per-page lightweight indexing concept. We use fixed-size pages (not variable ranges), enabling exact page-level I/O skipping rather than range approximation. |
| Cuckoo Index | VLDB 2020 | Cuckoo filters per block for data skipping. | Similar goal to our Bloom filters. They use cuckoo filters at coarse block granularity; we use Bloom filters at fine page granularity (200 rows vs thousands). |
| Column Imprints | SIGMOD 2013 | Per-cacheline bit-vectors for range scan acceleration. | In-memory technique — 64 bits per cacheline. At page granularity (200 rows), bucket occupancy too high for effective pruning. Designed for CPU cache, not I/O skipping. |

---

## 9. Open Questions

- [x] Does K-sweep show K=1 is sufficient? — ANSWERED: metadata_frac ≈96%; lookahead adds 2–7%
- [x] Is the I/O scheduler in scope? — ANSWERED: Yes, it is the Act 3 resolution
- [x] Does the scheduler deliver speedup on single-file? — ANSWERED: No (C7 = 1.023×). Structural mismatch with io_uring sweet spot.
- [x] Should the scheduler support hybrid O_DIRECT/buffered mode? — ANSWERED: O_DIRECT pread experiment showed 1.69× regression. Buffered pread is correct for large reads.
- [x] Does the 128 KiB crossover hold for Parquet reads? — ANSWERED: Yes. Single-file chunks >128 KiB go to pread.
- [x] Does multi-file partitioned layout enable sufficient io_uring batching? — ANSWERED: No (Exp-11, C11=0.985×). Column chunks remain large regardless of file count.
- [x] What is the file count transition point? — ANSWERED: None. Speedup flat ~1.0× at all N (Exp-12). File count orthogonal to read granularity.
- [x] Does TPC-H partitioned lineitem produce reads <128 KiB? — ANSWERED: Some do, but most are still large. Even with high SQE counts (Q1: 3885), scheduler adds overhead.
- [x] Do TPC-H join-heavy queries mask I/O improvement? — ANSWERED: Even lineitem-only queries (Q6) show 0.93×. Not a masking issue.
- [x] Does tpchgen-rs produce valid TPC-H data? — ANSWERED: Yes. All 8 queries return correct results.
- [ ] What is the exact read-size crossover point for io_uring vs pread on this NVMe? — needs: Exp-15 — blocks: M7 format design validation
- [ ] Does per-page FOR/DELTA compression add significant decompression overhead vs Parquet Snappy? — needs: micro-benchmark — blocks: M5 encoding choice
- [ ] Can zone maps on ClickBench EventDate/CounterID achieve ≥90% page pruning at ≤5% selectivity? — needs: zone-map effectiveness analysis — blocks: M6 predicate pushdown
- [ ] Does O_DIRECT + small pages (4–8 KiB) cause excessive TLB misses or alignment overhead? — needs: perf counters during Exp-16 — blocks: performance interpretation
- [ ] How does page-granular format perform on warm cache vs cold cache? — needs: warm/cold comparison in Exp-16 — blocks: claim scoping
- [ ] What is the Bloom filter space overhead as % of dataset, and does index load time negate page skip savings? — needs: space/time measurement in M5/M6 — blocks: R19 risk assessment
- [ ] At what cardinality threshold should the writer switch from bitmap to Bloom filter? — needs: sensitivity sweep (50, 100, 200) on ClickBench columns — blocks: writer config
- [ ] Can compound predicates (zone map on sort-key + Bloom on non-sort-key) achieve ≥95% page pruning on ClickBench? — needs: Exp-16 with compound queries — blocks: C15 claim strength
- [ ] Is per-page bitmap actually useful for low-cardinality columns, or does the birthday problem make it always read all pages? — needs: empirical test on ClickBench status/region columns — blocks: bitmap inclusion decision

---

## 10. Agent Continuity Block

> READ THIS FIRST if you are an AI agent picking up this project.

**Project state:** M0–M5 complete (diagnostic arc + format writer done). M6 task files amended with gap analysis findings.
**Current milestone:** M6 — Page-Granular Reader + io_uring Integration
**Current task:** M6-T1 — MicaReader::open (metadata + ALL indexes: zone maps, Bloom, bitmaps)
**Last completed task:** M5-T7 — Validation CLI (all M5 tasks done, 72 tests, 2026-04-13)
**Blocking issues:** none

**Session 1 progress (2026-04-13):**
- M5-T1 DONE: `docs/mica-spec.md` — full byte-level format spec (header, col meta, zone maps, bloom, bitmaps, pages, encodings, footer, alignment, overhead analysis)
- M5-T2 DONE: `mica/` crate — Cargo.toml, lib.rs, 6 modules (format, index, writer, reader, io, datafusion), 20 source files, cargo check clean

**Session 2 progress (2026-04-13):**
- M5-T5 DONE: `mica/src/index/zone_map.rs` (~300 lines) — ZoneMapEntry compute for all 15 DataType variants, serialize/deserialize 16B entries, 19 tests
- M5-T3 DONE: `mica/src/format/encoding.rs` (1088 lines) — PLAIN/FOR/DELTA/DICT encode+decode, auto-selection heuristic (DICT>DELTA>FOR>PLAIN), bit-packing helpers, 22 tests
- M5-T4 DONE: `mica/src/writer/mod.rs` (810 lines) — MicaWriter::new/write_batch/finish, multi-batch accumulation, page alignment, zone map integration, CRC32C checksums, 5 tests
- Total: 46 tests passing, cargo check clean

**Session 3 progress (2026-04-13):**
- M5-T5b DONE: `mica/src/index/bloom_filter.rs` (524 lines) — BloomFilter struct, xxhash64 double-hashing, construct from Arrow arrays, serialize/deserialize, FPR tests, 13 tests
- M5-T5c DONE: `mica/src/index/page_bitmap.rs` (936 lines) — PageBitmapIndex struct, dictionary construction, per-page bitvecs, contains/contains_in, serialize/deserialize, 11 tests
- Writer integration DONE: `mica/src/writer/mod.rs` updated to 1058 lines — bloom/bitmap computation per column based on cardinality threshold, multi-section file layout (Header→ColMeta→ZoneMaps→Bloom→Bitmap→Data→Footer), 7 writer tests
- M5-T6 DONE: `mica/src/bin/convert.rs` — Parquet→Mica CLI with schema filtering, LargeUtf8/LargeBinary casting, stats output
- M5-T7 DONE: `mica/src/bin/validate.rs` — sequential Mica reader, row count + per-column checksum + 10-row spot-check comparison against Parquet
- Total: 72 tests passing, cargo check clean, both binaries compile

**M6 gap analysis amendments (2026-04-13):**
- T1 expanded: now loads ALL index metadata eagerly (zone maps + Bloom filters + bitmaps), not just zone maps. Estimate 1d→1.5d.
- T2 amended: added ALL_NULL_SENTINEL handling note + decode() API contract note (non_null_count, not row_count).
- T3 amended: ST-3 (late materialization) marked as stretch goal / M6.5. Initial impl uses single-pass reads for qualifying pages.
- T4 rewritten: approach changed from "implement decode_page from scratch" to "factor validate.rs reader logic into library". Saves ~500 lines of duplication.
- T5 amended: added ST-0 (async/sync bridge via spawn_blocking for io_uring reads). Added filter pushdown pattern notes.
- T6 amended: added ST-0 (run mica-convert + mica-validate on actual data before SQL queries). M5 built CLIs but never ran them.

**How to resume:**
1. Read §1 (Overview) — Acts 1–3b are the diagnostic arc (WHY io_uring fails on Parquet). Act 4 is the resolution (page-granular format).
2. Read §2 (Claims) — C1–C13 are complete (diagnostic). C14–C18 are the new claims (Act 4, pending).
3. Read §3.4 for Act 4 architecture — page-granular format layout, writer, reader, inline io_uring.
4. **Read §3.5 for multi-index pruning stack** — zone maps + Bloom filters + per-page bitmaps. Dynamic write-time index selection and read-time predicate routing. This is a crucial design choice.
5. **Read `docs/mica-spec.md`** — the authoritative byte-level format spec. All implementation must match this.
6. Read §4 for milestone map — M5 ✅ (format), M6 ⬜ (reader), M7 ⬜ (experiments), M8 ⬜ (paper).
7. Key insight: column chunk granularity (50–500 KiB) prevents io_uring from helping. Fixed 4–8 KiB pages with multi-index pruning (zone maps for range, Bloom for high-cardinality equality, bitmaps for low-cardinality equality) enable fine-grained selective reads on diverse query patterns.
8. Inline io_uring reader already exists (`eval/clickbench/src/inline_uring_reader.rs`) — reusable for page-granular format.
9. IPC bench reference (`eval/arrow/ipc-bench/src/aligned_io.rs`) shows 8–13× with 4K reads — the target regime.
10. **CRITICAL: `decode()` takes non_null_count, NOT row_count.** See `mica/src/bin/validate.rs` for the correct pattern.
11. **CRITICAL: Bloom deserialization needs BloomHeader(16B) read first** — `BloomFilter::from_bytes()` takes raw bits only, not the header. Reader must parse header to get `bytes_per_filter`, then slice per-page.
12. **Reference reader in validate.rs:** ~1069 lines of working page reader logic. T4 should factor this into library code, not rewrite.

**Key paths:**
- Existing source: `eval/clickbench/src/` (runner, inline_uring_reader, coalescing_reader, io_scheduler, buffer_pool)
- Mica format source: `mica/` (standalone crate at workspace root, to be created in M5)
- IoUringStore: `eval/object-store-iouring/src/`
- Eval scripts: `eval/clickbench/scripts/`, `mica/scripts/` (to be created)
- Results: `eval/clickbench/results/run_YYYYMMDD_HHMMSS/`
- Partitioned data: `eval/clickbench/data/hits_N{10,50,100,226}/`
- TPC-H data: `eval/clickbench/data/tpch_sf10/{single,split}/`
- IPC bench: `eval/arrow/ipc-bench/src/aligned_io.rs`
- Arrow Journals: `eval/arrow/journals/`
- Research Journals: `docs/journal/`
- OSDI future plan: `OSDI_PLAN.md`

**Key prior results (diagnostic arc):**
- Phase 2b (Journal 22–23): H1=1.438×, H2=0.998×, reactor=0.887×
- K-sweep (Journal 24): meta_frac ≈96%
- Single-file scheduler (Journal 26): C7=1.023×, C8=1.011×, C9=0.967×
- Multi-file scheduler (Journal 27): C11=0.985×, C12=flat, C13=0.97×
- Inline io_uring (Exp-14): 1.054× — architecture helps but read granularity dominates
- IPC bench: 8–13× with 4K scattered reads (the target regime for page-granular format)

**Key research references:**
- Snowflake VLDB 2025: 38% of queries have selectivity ≤5%
- Snowflake SIGMOD 2025: 99.4% micro-partitions pruned
- Lance arXiv 2025: 60× better random access than Parquet with mini-blocks
- VLDB 2023 (Haas & Leis): NVMe random within 20% of sequential
- VLDB 2026 (Jasny): io_uring batched reads, 18% improvement
- No prior study measures read-size crossover point (C14 is novel)

**Key benchmark result directories:**
- v1: `results/run_20260409_224511/` (cold), `run_20260409_225058/` (warm)
- v2: `results/run_20260410_004046/` (cold), `run_20260410_004628/` (warm)
- K-sweep: `results/run_20260410_071914/k_sweep/`
- M3 Exp-7/8/9: `results/run_20260410_132015/exp7/`, `.../exp8/`, `.../exp9/`
- M4 Exp-11: `results/run_20260411_061504/exp11/`
- M4 Exp-12: `results/run_20260411_061823/exp12/`
- M4 Exp-13: `results/run_20260411_061613/tpch/`
- Exp-14 (inline io_uring): `results/run_20260411_100148/exp14/`
