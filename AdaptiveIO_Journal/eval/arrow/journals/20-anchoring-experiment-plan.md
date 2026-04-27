# Journal 20: Anchoring Experiment Plan

**Status**: Plan (pre-implementation)
**Predecessor**: §34 (Journal 19 — Anchoring Analysis)
**Scope**: Validate io_uring advantages on real Parquet/DataFusion workloads

---

## §35 — Motivation & Design Principles

Our 19 journals establish io_uring advantages in a controlled IPC environment.
The regime map (§34.1) identifies four independent techniques:

| Technique | Isolated gain | Regime constraint |
|---|---|---|
| Column projection | 5–10× | Format must support selective reads |
| io_uring batch | 1.3–1.8× over mmap | Read size < 128 KiB, batch ≥ 50 |
| Pipeline overlap | 1.86–2.04× | I/O:CPU ratio ≈ 1:1 |
| Multi-file threading | 1.8× at 12T | Multiple independent files |

**Central question**: Do real Parquet/DataFusion workloads land in the io_uring winning regime, and does the predicted speedup survive the full query stack?

### Design decisions (locked)

1. **No O_DIRECT initially.** Use regular fds with io_uring (page-cache-backed batched syscalls). Systematic O_DIRECT analysis follows separately.
2. **Minimal `object_store` scope.** Implement `get_ranges()` only — the batched-read path.
3. **Workload selection deferred.** ClickBench is the starting point; custom workloads may follow based on Phase 0 findings.
4. **Phase 3 (Arrow IPC full-stack) tabled.** Spec included for future pickup.

---

## §36 — Phase 0: Workload Characterization

**Objective**: Determine whether DataFusion's Parquet I/O patterns naturally fall within the io_uring winning regime, *before* building any integration.

**Deliverables**:
- Column-chunk size distribution for the target dataset(s)
- Per-query I/O trace: read count, sizes, offsets, sequential vs scattered ratio
- Regime-map overlay: which queries hit the sweet spot?
- Go/no-go decision for Phase 1

### §36.1 — Static metadata analysis

Extract column chunk metadata from Parquet files without running queries.

**Approach**:
1. Write a small tool (or extend `parquet-bench`) that reads Parquet footer metadata and emits per-column-chunk statistics: `(row_group, column_index, column_name, offset, compressed_size, uncompressed_size, codec, num_values)`.
2. Run on candidate datasets:
   - **ClickBench `hits.parquet`** — the standard OLAP benchmark. 99 columns, ~14 GiB single file, many narrow integer columns + wide string columns.
   - **Existing taxi data** (if available from prior experiments).
3. Compute distribution:
   - Histogram of compressed chunk sizes (the actual I/O unit)
   - Per-column breakdown: which columns have chunks < 128 KiB?
   - Total read count per row-group for N-column projections (1, 5, 10, 20 columns)

**Prediction from regime map**: Narrow integer/date columns (e.g. `CounterID`, `WatchID`, `EventDate` in ClickBench) should have chunks well under 128 KiB with zstd compression. Wide string columns (`URL`, `Title`, `Referer`) will likely exceed 128 KiB.

**Key output**: A table mapping column name → average chunk size → predicted io_uring advantage. This directly tells us which ClickBench queries will benefit.

### §36.2 — Dynamic I/O tracing

Capture the actual byte-range reads DataFusion's Parquet reader issues.

**Approach**:
1. Create a thin `ObjectStore` wrapper that logs every `get_range()` and `get_ranges()` call: `(timestamp_ns, method, path, offset, length)`.
2. Configure DataFusion with this tracing store.
3. Run a representative subset of ClickBench queries (cold cache each time — `echo 3 > /proc/sys/vm/drop_caches`).
4. Analyze traces:
   - Reads per query
   - Size distribution per query
   - Temporal spacing (can reads be batched?)
   - How does DataFusion's row-group pruning affect read count?

**Why both static + dynamic?** Static analysis tells us the *potential*. Dynamic tracing tells us what DataFusion *actually does* — including metadata reads, page-index reads, row-group pruning decisions, and any internal batching or coalescing already happening.

**Key output**: Per-query trace file + summary table:

```
| Query | Columns | Reads | Median size | P95 size | Batch-eligible | Predicted gain |
|-------|---------|-------|-------------|----------|----------------|----------------|
| Q1    | 3       | 24    | 45 KiB      | 112 KiB  | yes            | 1.5–2×         |
| ...   |         |       |             |          |                |                |
```

### §36.3 — Parquet-bench extension

Before the full `object_store` integration, extend `parquet-bench` to include a **non-O_DIRECT io_uring scattered** strategy. This validates the technique on real Parquet files with minimal infrastructure.

**What to build**:
1. New strategy: `IoUringScattered` — like `ODirectScattered` (using `PrefetchedChunkReader`) but with regular `open()` instead of `O_DIRECT`, and no alignment requirements.
2. This means:
   - New `RegularFile` struct (like `DirectFile` but without `O_DIRECT` flag)
   - Modified `read_many()` variant: no alignment assertions, use regular heap buffers instead of `AlignedBuf`
   - `PrefetchedChunkReader` parameterized to use either file type
3. Run on ClickBench `hits.parquet` with various column selections (1, 5, 10, 20 columns) and compare against `Builtin`, `Mmap`, existing `ODirectScattered`.

**Why this before Phase 1?** It gives us the first real Parquet io_uring numbers with ~50 lines of code change, validating whether the approach works before investing in `object_store` integration.

**Key output**: Timing comparison table (metadata/io/decode/filter breakdown) across strategies and column counts.

### §36.4 — Success criteria

Phase 0 is complete when:
- [ ] Column chunk size distribution analyzed for at least one dataset
- [ ] At least 5 ClickBench queries traced dynamically
- [ ] Non-O_DIRECT io_uring strategy benchmarked in parquet-bench
- [ ] Go/no-go decision for Phase 1 documented with data

**Go condition**: ≥ 30% of traced queries have median read size < 128 KiB and read count ≥ 10.
**No-go pivot**: If most reads are > 128 KiB, pivot to Arrow IPC anchor (Phase 3) where we control the format.

---

## §37 — Phase 1: `object_store` io_uring Backend

**Objective**: Build a minimal `ObjectStore` implementation that batches local-filesystem reads via io_uring, targeting `get_ranges()`.

**Prerequisite**: Phase 0 go decision.

### §37.1 — Architecture

```
┌─────────────────────────────────────┐
│         IoUringObjectStore          │
│  impl ObjectStore                   │
│                                     │
│  ┌──────────────┐  ┌─────────────┐  │
│  │  get_ranges() │  │ get_range() │  │
│  │  (batched)    │  │ (single)    │  │
│  └──────┬───────┘  └──────┬──────┘  │
│         │                  │         │
│         ▼                  │         │
│  ┌──────────────────┐      │         │
│  │ io_uring ring    │◄─────┘         │
│  │ (submit N reads) │                │
│  └──────────────────┘                │
│                                      │
│  Fallback: delegates to              │
│  LocalFileSystem for non-read ops    │
└─────────────────────────────────────┘
```

**Design choices**:

1. **Wrapper, not fork.** Wrap `LocalFileSystem` and override only `get_range()` / `get_ranges()`. All other trait methods (`put`, `list`, `head`, `delete`, `copy`, `rename`) delegate to the inner store.

2. **Ring lifecycle.** One `io_uring::IoUring` instance per `get_ranges()` call. Creating a ring is ~1 μs; the overhead is negligible relative to I/O. This avoids thread-safety complexity.

3. **No O_DIRECT.** Open files with regular `open(O_RDONLY)`. The io_uring advantage comes from batched syscall submission, not from bypassing page cache.

4. **Buffer management.** Allocate one `Vec<u8>` per range. No alignment requirements without O_DIRECT.

5. **Async bridging.** `object_store` trait methods are `async`. Use `tokio::task::spawn_blocking` to run synchronous io_uring submission from within the async context. This is the same pattern DataFusion's `LocalFileSystem` already uses for `pread`.

6. **Adaptive crossover.** Optional: if all ranges are > 128 KiB, fall back to the inner `LocalFileSystem` implementation. This can be added after initial benchmarking validates the concept. Start without it.

### §37.2 — Implementation plan

**New crate**: `eval/object-store-iouring/` (standalone crate, depends on `object_store`, `io-uring`, `tokio`)

**Core types**:

```rust
pub struct IoUringStore {
    inner: Arc<LocalFileSystem>,
    ring_entries: u32,      // default: 128
}

impl IoUringStore {
    pub fn new(inner: LocalFileSystem) -> Self { ... }
}
```

**`get_ranges()` implementation** (the critical path):

```rust
async fn get_ranges(&self, location: &Path, ranges: &[Range<usize>])
    -> object_store::Result<Vec<Bytes>>
{
    let path = self.inner.path_to_filesystem(location)?;
    let ranges = ranges.to_vec();
    tokio::task::spawn_blocking(move || {
        get_ranges_sync(&path, &ranges, ring_entries)
    }).await?
}

fn get_ranges_sync(path: &std::path::Path, ranges: &[Range<usize>], ring_entries: u32)
    -> object_store::Result<Vec<Bytes>>
{
    let fd = open(path, O_RDONLY)?;
    let ring = IoUring::new(ring_entries)?;

    // Allocate buffers
    let mut buffers: Vec<Vec<u8>> = ranges.iter()
        .map(|r| vec![0u8; r.end - r.start])
        .collect();

    // Submit all reads
    // (reuse pattern from aligned_io::read_many, minus alignment checks)

    // Convert to Bytes
    Ok(buffers.into_iter().map(Bytes::from).collect())
}
```

**`get_range()` implementation**: Delegate to `get_ranges()` with a single-element slice. No benefit from io_uring for a single read, but keeps the code path uniform.

### §37.3 — Testing strategy

1. **Correctness**: Read known files, compare byte-for-byte with `LocalFileSystem::get_ranges()`.
2. **Benchmark harness**:
   - Parameterize: range count (1, 10, 50, 100, 500), range size (4 KiB, 32 KiB, 128 KiB, 1 MiB), sequential vs random offsets.
   - Compare: `IoUringStore` vs `LocalFileSystem` (cold cache, warm cache).
   - Output: latency, throughput, syscall count (via `/proc/self/status`).
3. **Integration**: Replace DataFusion's default store with `IoUringStore`, run `parquet-bench` workloads, confirm identical results.

### §37.4 — Success criteria

- [ ] `get_ranges()` produces byte-identical output to `LocalFileSystem`
- [ ] ≥ 1.3× throughput improvement on 50+ ranges of ≤ 128 KiB (cold cache)
- [ ] No regression on large sequential reads (> 128 KiB)
- [ ] Clean integration with tokio runtime (no panics, no deadlocks)

---

## §38 — Phase 2: DataFusion ClickBench Integration

**Objective**: Demonstrate end-to-end query speedup on ClickBench using the io_uring `ObjectStore`.

**Prerequisite**: Phase 1 passing all success criteria.

### §38.1 — Setup

1. **ClickBench dataset**: Download `hits.parquet` (~14 GiB). The standard ClickBench Parquet file.
2. **Query set**: The 43 ClickBench SQL queries. Select a subset (10–15) that Phase 0 identified as io_uring-favorable (narrow column projections, many column chunks).
3. **DataFusion configuration**:
   - Build a minimal DataFusion program that registers `hits.parquet` as a table and executes queries.
   - Swap `ObjectStore`: baseline = `LocalFileSystem`, experiment = `IoUringStore`.
   - Cold cache each run: `sync && echo 3 > /proc/sys/vm/drop_caches`.

### §38.2 — Benchmark protocol

For each query × store combination:

```
1. Drop caches
2. Start wall-clock timer
3. Execute query (collect all results)
4. Stop timer
5. Record: wall time, DataFusion metrics (if available), I/O trace from §36.2 wrapper
```

**Repeat**: 5 cold-cache iterations per configuration. Report median and stddev.

**Control variables**:
- Same NVMe device
- Same CPU pinning (if applicable)
- Same DataFusion version and configuration
- Single-threaded execution first (isolate I/O effect), then multi-threaded

### §38.3 — Expected results (predictions from regime map)

For queries projecting 3–5 narrow columns:
- Column chunks ~20–80 KiB → well within regime
- 10–30 reads per row group × N row groups
- **Predicted**: 1.5–3× E2E cold-cache improvement

For queries projecting 10+ columns or wide string columns:
- Column chunks 200 KiB–2 MiB → outside regime
- **Predicted**: ≤ 1.1× improvement (io_uring overhead ≈ benefit)

For aggregation-heavy queries (SUM/COUNT over single column):
- Very few, very small reads → moderate batch benefit
- Dominated by decode/compute cost
- **Predicted**: ≤ 1.2× improvement

### §38.4 — Analysis framework

For each query, decompose wall time into:

```
Wall = I/O + Decode + Compute + Overhead
```

Compare I/O time between baseline and io_uring. If I/O improvement doesn't translate proportionally to wall-time improvement, the bottleneck is elsewhere (decode, compute). This matches Finding 33 (decode cost asymmetry from Journal 14).

Plot:
1. **Read size vs speedup** (validate regime boundary at ~128 KiB)
2. **Read count vs speedup** (validate batch depth threshold)
3. **I/O fraction vs E2E speedup** (Amdahl's Law visualization)

### §38.5 — Success criteria

- [ ] ≥ 5 ClickBench queries show ≥ 1.3× cold-cache E2E improvement
- [ ] Results consistent with Phase 0 predictions (within 30% of predicted range)
- [ ] No correctness regressions (all queries produce identical results)
- [ ] Regime boundary validated: queries with large reads show ≤ 1.1× change

---

## §39 — Phase 3: Arrow IPC Full-Stack (Tabled — Spec Only)

**Status**: Tabled. Spec preserved for future pickup after Phase 2 validation.

**Objective**: Demonstrate the full 8.3× advantage by building a DataFusion `ExecutionPlan` that replaces `ArrowExec` with our optimized IPC pipeline.

### §39.1 — What this includes

1. **Column-projection IPC reader**: Read only selected columns from IPC file (5–10× I/O reduction). Requires footer parsing + selective message reads (already implemented in `ipc-bench`).

2. **Zone-map filtering**: Skip record batches whose min/max stats exclude the predicate range (10–90× batch filtering). Requires building zone-map metadata on write or as a sidecar (partially implemented in `ipc-bench`).

3. **io_uring batched I/O**: Submit all selected message reads as a single io_uring batch (1.3–1.8× over mmap). Already implemented in `ipc-bench`.

4. **Decode pipeline**: Overlap I/O with Arrow decode using a producer-consumer pipeline (1.86–2.04×). Already implemented in `ipc-bench`.

5. **DataFusion integration**: Wrap as a custom `ExecutionPlan` that DataFusion's optimizer can use in place of `ArrowExec`.

### §39.2 — Integration surface

```rust
struct IoUringIpcExec {
    files: Vec<PathBuf>,
    schema: SchemaRef,
    projection: Option<Vec<usize>>,
    predicate: Option<Expr>,  // For zone-map pruning
    // ... pipeline config
}

impl ExecutionPlan for IoUringIpcExec {
    fn execute(&self, partition: usize, ...) -> SendableRecordBatchStream {
        // 1. Parse IPC footer → identify relevant messages
        // 2. Apply zone-map filter → prune messages
        // 3. Submit io_uring batch for surviving messages
        // 4. Pipeline decode as batches complete
        // 5. Yield RecordBatches
    }
}
```

### §39.3 — Existing assets to leverage

From `ipc-bench/src/`:
- `footer.rs` — IPC footer parsing, column offset extraction
- `zone_map.rs` — Min/max statistics, batch pruning
- `io_uring_reader.rs` — Batched message reads
- `pipeline.rs` — Producer-consumer decode pipeline
- `bench.rs` — Full pipeline orchestration

**Estimated effort**: Medium-high. The components exist but need to be adapted from benchmark harness to DataFusion `ExecutionPlan` contract (partitioning, schema propagation, metric reporting).

### §39.4 — Why this is the strongest anchor

- Combined 8.3× vs vanilla ArrowExec (§34.4)
- Demonstrates all four techniques working together
- Most publishable: complete alternative I/O stack for a common format
- Risk: IPC is secondary in DataFusion; adoption impact lower than Parquet

### §39.5 — Prerequisites for pickup

- Phase 2 complete (validates infrastructure)
- Decision on whether IPC adoption potential justifies the effort
- Decision on write-path: do we need to generate optimized IPC files, or use existing ones?

---

## §40 — Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Most Parquet chunks > 128 KiB | Phase 1-2 show minimal gain | Phase 0 detects early; pivot to Phase 3 (IPC) |
| DataFusion's internal batching already coalesces reads | Reduces io_uring batch depth | Phase 0 dynamic tracing reveals actual call pattern |
| tokio `spawn_blocking` adds overhead | Eats into io_uring gains | Measure overhead in Phase 1 isolation; if > 100 μs, explore alternatives |
| Page cache warmth in multi-iteration benchmarks | Inflates baseline, hides cold advantage | Strict `drop_caches` protocol; report cold-only numbers |
| NVMe queue depth saturation at high thread counts | io_uring degrades (Finding from J10) | Benchmark at 1T and 4T first; thread scaling is Phase 2+ concern |
| `object_store` API changes across versions | Integration breaks | Pin version; current ecosystem uses object_store 0.11.x / 0.12.x |

---

## §41 — Dependencies & Environment

### Software

| Dependency | Version | Purpose |
|---|---|---|
| `arrow` / `parquet` | 55 | Parquet reader, Arrow arrays |
| `io-uring` | 0.7 | io_uring syscall interface |
| `object_store` | TBD (latest compatible) | Storage abstraction |
| `datafusion` | TBD (latest compatible w/ arrow 55) | Query engine |
| `tokio` | 1.x | Async runtime |

### Hardware requirements

- Linux kernel ≥ 5.6 (io_uring support)
- NVMe SSD (to observe I/O advantages; SATA will work but with lower absolute gains)
- ≥ 32 GiB RAM (ClickBench dataset + DataFusion working memory)
- Ability to run `echo 3 > /proc/sys/vm/drop_caches` (root or sudo)

### Data

- **ClickBench `hits.parquet`**: ~14 GiB, downloadable from clickbench.com
- **Existing taxi dataset**: If available from prior experiments (parquet-bench default column is `trip_miles`)

---

## §42 — Execution Sequence

```
Phase 0 ─── §36.1 Static metadata analysis
         ├── §36.2 Dynamic I/O tracing (can parallel with §36.3)
         └── §36.3 Parquet-bench non-O_DIRECT strategy
                │
                ▼
         Go/No-go decision
                │
    ┌───────────┴───────────┐
    │ Go                    │ No-go
    ▼                       ▼
Phase 1 ─── §37 object_store    Pivot to Phase 3
    │       io_uring backend     (Arrow IPC anchor)
    │
    ▼
Phase 2 ─── §38 DataFusion
            ClickBench integration
    │
    ▼
Phase 3 ─── §39 Arrow IPC
            full-stack (if justified)
```

### Estimated effort

| Phase | Effort | Blocking dependency |
|---|---|---|
| Phase 0 | 2–3 days | None (start immediately) |
| Phase 1 | 3–5 days | Phase 0 go decision |
| Phase 2 | 3–5 days | Phase 1 passing |
| Phase 3 | 5–8 days | Phase 2 complete |

---

## §43 — Open Questions (to resolve during execution)

1. **Which `object_store` version?** Need to verify compatibility with `arrow`/`parquet` 55. The `object_store` crate versions track independently from Arrow.

2. **DataFusion's `ParquetExec` coalescing.** Does DataFusion's Parquet reader already batch `get_ranges()` calls internally, or does it issue `get_range()` one-at-a-time? This fundamentally affects whether an io_uring `get_ranges()` backend even gets exercised. Phase 0 dynamic tracing answers this.

3. **Row-group pruning impact.** If DataFusion prunes most row groups via statistics, the surviving read count may be too low for io_uring batching to help. Need to measure with and without pruning.

4. **Memory pressure.** io_uring pre-allocates all read buffers before submission. For 500+ ranges of 64 KiB each, that's 32 MiB pre-allocated. Acceptable, but need to verify it doesn't interact badly with DataFusion's memory management.

5. **Benchmark noise.** NVMe device queue depth, kernel I/O scheduler state, and NUMA effects can add variance. May need CPU pinning and repeated runs.
