← [Part 18](./18-exp-i-combined.md) | [Index](./README.md)

# §34. Anchoring Analysis: Real-World Integration Targets for io_uring

*Brainstorm document. Maps experimental findings (Parts 1-18) onto real-world systems to identify the strongest integration target for io_uring-based Arrow I/O. Focuses on Apache DataFusion as the primary candidate system.*

---

## §34.1 Decomposing the Experimental Insights

The combined finding from Exp I (§33, Finding 52) reports **8.3× vs vanilla, 1.48× vs mmap** for the 4T-pipeline configuration. This headline number conflates four independent techniques. To identify real-world anchors, we must isolate each technique's contribution:

| Technique | Contribution | Evidence | Requirement |
|---|---|---|---|
| I/O-level column projection | 5-10× (dominant) | §29.2: vanilla reads 23.1 MiB vs iouring 3.0 MiB for 3-col query | IPC file format with footer (batch offset table) |
| io_uring batch submission | 1.3-1.8× over mmap | §29.3, Finding 32: load phase 3.1× faster at equal I/O volume | Scattered reads < 128 KiB, cold cache |
| Pipeline overlap (I/O ∥ decode) | 1.86-2.04× | §31, Finding 40: grows with batch count, neutral at 1 batch | ≥4 batches to amortize thread overhead |
| Multi-file threading | 1.8× at 12T (modest) | §32.3, Finding 45: iouring already fast at 1T | Multiple files to parallelize across |

**Key decomposition**: The 8.3× over vanilla = column projection (5×) × batch submission (1.7×). The 1.48× over mmap = batch submission (1.3×) × pipeline overlap contribution at scale. Column projection is the dominant factor, and it is **format-dependent** (requires random-access metadata, e.g. IPC file footer or Parquet row-group index).

### Regime Map: When Each Technique Applies

From Parts 7-10 (Exps C-D), the io_uring batch submission advantage is bounded by:

| Characteristic | Threshold | io_uring vs mmap | Evidence |
|---|---|---|---|
| Read size | < 128 KiB | 2.1-23× | §23.2: crossover at 128-256 KiB |
| Batch depth | ≥ 50 reads | 5-23× | §23.1: EP=3.5× at N=10, 13.6× at N=100 |
| Cache state | Cold | 7.9× | §26: warm=1.03× |
| Concurrency | 1-4 threads | 5.8-7.9× | §26: degrades to 2.6× at 12T |

**Critical constraint**: io_uring's advantage is **entirely in the I/O path**. If data is in the page cache, there is no advantage ($26, warm=1.03×). Any anchor system must have cold-cache access patterns.

---

## §34.2 DataFusion Spill Files — Why They Don't Fit

DataFusion's spill mechanism was the initial candidate. Analysis reveals a fundamental mismatch:

| Spill File Characteristic | io_uring Winning Regime | Match? |
|---|---|---|
| IPC **stream** format (no footer) | Requires footer for random access | ❌ |
| Sequential full reads (batch-by-batch) | Scattered small reads < 128 KiB | ❌ |
| Just written → likely warm cache | Cold cache required | ❌ |
| All columns read back | Column projection = dominant gain | ❌ |
| Under memory pressure | io_uring needs buffer lifetime | ❌ |

**Current implementation**: `std::fs::File` + `BufReader` + `spawn_blocking` per batch read (DataFusion `physical-plan/src/spill/mod.rs`). No async I/O, no column projection, no batch-level filtering.

### Partial Opportunities Within Spill

Despite the overall mismatch, two sub-scenarios may benefit:

1. **K-way merge prefetch**: ExternalSorter merges N sorted runs. Currently each run spawns a blocking task per batch read. An io_uring ring could prefetch batch N+1 from all N files while decoding batch N. Expected gain: ~1.5-2× from pipeline overlap (§31 analog), but only when merge reads are cold (large working set exceeding RAM).

2. **Format migration (stream → file)**: If DataFusion adopted IPC file format for spill, column projection and batch-level filtering become possible. This would unlock the full 5-10× column projection advantage. However, this is a major architectural change — IPC file format requires writing the footer at file close, conflicting with incremental `append_batch` semantics. DataFusion issue #14078 explored optimized spill formats but was closed.

**Verdict**: Spill files are a weak anchor. The dominant technique (column projection) doesn't apply, and the incremental io_uring gain (batch submission) is neutralized by warm cache and sequential access.

---

## §34.3 Candidate Anchor 1: Parquet Column Chunk Reads

**Fit assessment: STRONG**

### How DataFusion Reads Parquet

```
ParquetExec → ParquetOpener → AsyncFileReader (parquet crate)
  → object_store::local::LocalFileSystem
    → tokio::fs::File
      → spawn_blocking + std::fs::File::read_at
```

Parquet files have a columnar layout with row groups. DataFusion already performs:
- **Column pruning**: Only requested columns are read (projection pushdown)
- **Row group pruning**: Row groups are skipped based on min/max statistics (= zone-map analog)

The resulting I/O pattern after pruning:

| Property | Parquet Read Pattern | io_uring Regime? |
|---|---|---|
| Access pattern | Scattered column chunks at different file offsets | ✅ Non-sequential |
| Read size | Column chunks: typically 8 KiB - 2 MiB | ✅ Often < 128 KiB for narrow columns |
| Batch depth | Multiple columns × selected row groups | ✅ 10-100+ reads per query |
| Cache state | Analytical queries on large datasets → often cold | ✅ Cold |
| Concurrency | Multiple partitions reading in parallel | ✅ 1-8 threads typical |

### The Gap

DataFusion's Parquet reader issues byte-range requests **individually** through `object_store`. For local files, each request becomes a `spawn_blocking` + `read()` syscall. There is no batching of column-chunk reads.

The `object_store` trait has `get_ranges(&self, location, ranges) → Vec<Bytes>` which is semantically a batch read — but the local implementation executes ranges sequentially.

### Predicted Gain

Using the regime map from §34.1:

- Narrow columns (Int32, Float64): chunk size 8-64 KiB → **2-13× faster reads** (§23.2)
- Typical query projecting 3-5 columns from 10 row groups: batch depth 30-50 → **5-10× EP** (§23.1)
- Cold cache (dataset > RAM): **full advantage applies** (§26)
- Realistic end-to-end with decode overhead: **1.5-3×** (informed by §29.3 — decode cost compresses advantage)

For wide columns (String, List): chunk size > 128 KiB → **mmap wins** (§23.2). An adaptive strategy is needed.

### Integration Path

Two levels of integration, non-exclusive:

**Level A: `object_store` `get_ranges` with io_uring** (infrastructure)
- Implement `ObjectStore` trait backed by io_uring for local files
- `get_ranges()` maps directly to batched io_uring submissions
- Benefits ALL local file reads (Parquet, Arrow IPC, CSV, JSON) across the entire Arrow ecosystem (DataFusion, delta-rs, Lance, Ballista)
- Does not require changes to the Parquet reader itself

**Level B: Parquet reader-level batching** (application)
- Modify `ParquetExec` to collect all needed column-chunk offsets for a row group, then issue them as a single `get_ranges` call
- Currently, column chunks may be requested individually or in small groups
- This maximizes the batch depth seen by the io_uring backend

### Risks and Complications

1. **Column chunk size variance**: Narrow columns fit the < 128 KiB regime, but wide/nested columns can be multi-MiB. Need adaptive crossover (fall back to sequential I/O or mmap above 128 KiB).

2. **object_store abstraction layer**: io_uring is Linux-only. `object_store` is cross-platform. The io_uring backend would be a Linux-specific `ObjectStore` implementation, gated behind `#[cfg(target_os = "linux")]`. This is architecturally clean but adds a platform-specific code path.

3. **Async runtime interaction**: io_uring has its own completion model. Bridging io_uring completions into tokio's async model requires either `tokio-uring` or a custom `Future` implementation. `tokio-uring` requires its own runtime (not compatible with standard tokio), which is a significant constraint. The `io-uring` crate (low-level) can be used with a dedicated I/O thread + channel bridge to tokio.

4. **Buffer management**: Parquet column chunks are decoded in-place by the `parquet` crate. io_uring reads into user-provided buffers. The buffer hand-off must be zero-copy or the copy cost erodes the I/O gain (§29.3, Finding 33: O_DIRECT buffers incur decode copy cost).

### Community Signals

- DataFusion issue #15321: "Improve Spill Performance: mmap the spill files" — shows community interest in I/O optimization.
- `object_store` is actively maintained by the Apache Arrow team. PRs improving local file performance are welcome.
- PostgreSQL 18 shipped io_uring for reads (March 2025), establishing industry precedent.

---

## §34.4 Candidate Anchor 2: Arrow IPC File Reader (ArrowExec)

**Fit assessment: DIRECT MATCH (most aligned with experiments)**

### How DataFusion Reads Arrow IPC

DataFusion's `ArrowExec` reads Arrow IPC files using the `arrow-ipc` crate's `FileReader`. The current implementation reads **full batch messages** — all columns in each batch — regardless of query projection.

This means DataFusion's Arrow IPC reader is equivalent to the **vanilla** strategy in our experiments.

### The Full-Stack Opportunity

Our experiments demonstrate a complete alternative read path for Arrow IPC files:

| Component | Current DataFusion | Our Experiments | Expected Gain |
|---|---|---|---|
| Column projection | ❌ Reads all columns | ✅ I/O-level via footer metadata | 5-10× I/O reduction (§29.3, Finding 35) |
| Batch filtering | ❌ Reads all batches | ✅ Zone-map on batch statistics | 10-90× batch skip (§27) |
| I/O mechanism | `BufReader` + `read()` | io_uring batch submission | 1.3-1.8× (§29.3, Finding 32) |
| Decode pipeline | Sequential | Producer-consumer overlap | 1.86× at ≥21 batches (§31, Finding 40) |

**Combined predicted gain**: 8.3× vs current ArrowExec (directly from §33, Finding 52).

### Integration Path

Replace or augment `ArrowExec` with an io_uring-backed reader that:

1. Reads the IPC file footer to obtain batch offsets and column metadata
2. Evaluates zone-map predicates (if batch-level statistics exist) to select batches
3. Computes column buffer offsets for projected columns in selected batches
4. Submits all reads via io_uring in one batch
5. Decodes column buffers into RecordBatches with pipeline overlap

This is essentially what our `ipc-bench` crate already does. The integration work is bridging it into DataFusion's execution framework.

### Risks and Complications

1. **Arrow IPC is a secondary format in DataFusion**. Most users use Parquet. Community investment in IPC reader optimization may be limited. However, this is changing as Arrow IPC gains traction for inter-process data exchange and caching layers.

2. **Zone-map statistics are not standard in Arrow IPC**. The IPC footer contains batch offsets and schema, but not min/max statistics per batch. Zone-map filtering requires either: (a) a sidecar metadata file, (b) a custom metadata extension in the IPC footer, or (c) a first-pass scan to build statistics. Our experiments used approach (c) — reading the footer to identify batch offsets, then using domain knowledge (sorted `pickup_datetime` column) for filtering.

3. **Dictionary replacement**: DataFusion chose IPC stream format for spill files partly because it supports dictionary replacement (evolving dictionaries across batches). IPC file format also supports dictionaries but with different semantics. This is relevant only if the IPC reader is also used for spill read-back.

### Publishability

This is the **most publishable** anchor because:
- We have end-to-end validated data across 18 experiments
- The comparison is apples-to-apples (same dataset, same query, same output)
- The 8.3× speedup is large and well-decomposed into independent factors
- The adaptive crossover (< 128 KiB → io_uring, ≥ 128 KiB → mmap) is a novel contribution

---

## §34.5 Candidate Anchor 3: `object_store` Local Backend

**Fit assessment: BROADEST IMPACT**

### What `object_store` Is

`object_store` (apache/arrow-rs) is the I/O abstraction layer used by DataFusion, delta-rs, Lance, Ballista, and other Rust data systems. It provides a unified `ObjectStore` trait for S3, GCS, Azure Blob, HDFS, and local files.

For local files, the implementation (`LocalFileSystem`) uses `tokio::fs::File` which wraps `std::fs::File` in `spawn_blocking`.

### The `get_ranges` API

```rust
/// Read multiple byte ranges from a file.
async fn get_ranges(
    &self,
    location: &Path,
    ranges: &[Range<usize>],
) -> Result<Vec<Bytes>>;
```

This API is **semantically identical** to our `read_many()` function in `aligned_io.rs`. It accepts multiple byte ranges and returns the data for each. The current local implementation executes ranges sequentially. An io_uring backend would submit all ranges in a single batch.

### Integration Design

```
object_store::local::LocalFileSystem (current)
  → tokio::fs::File::read_at() per range (sequential)

object_store::local::IoUringFileSystem (proposed)
  → io_uring batch submission for get_ranges()
  → single read_at() fallback for get_range()
  → adaptive: io_uring for ranges < 128 KiB, sequential for larger
```

### Predicted Gain

The gain depends entirely on the caller's I/O pattern:

| Caller Pattern | Example | Predicted Gain |
|---|---|---|
| Many small scattered reads | Parquet column chunks (narrow cols) | 2-13× (§23) |
| Few large sequential reads | CSV full scan | Negligible |
| Mixed sizes | Parquet with wide+narrow columns | Requires adaptive crossover |
| Warm cache | Repeated queries | None (§26: warm=1.03×) |

### Risks and Complications

1. **Platform specificity**: io_uring is Linux-only (kernel 5.6+). `object_store` is cross-platform. The io_uring backend must be a compile-time feature (`#[cfg(target_os = "linux")]`) with graceful fallback.

2. **Runtime bridging**: `object_store` is async (tokio). io_uring needs its own event loop or a bridge. Options:
   - **tokio-uring**: Requires `tokio_uring::Runtime`, incompatible with standard `tokio::Runtime`. Ruled out for library use.
   - **Dedicated I/O thread**: Spawn a thread running io_uring event loop, communicate via channels. Our `aligned_io.rs` already uses this pattern (synchronous io_uring on the calling thread).
   - **`io-uring` crate direct**: Use the low-level crate, submit from `spawn_blocking`, poll completions in-band. Most compatible with existing tokio usage.

3. **Buffer ownership**: `object_store` returns `Bytes` (reference-counted, immutable). io_uring reads into `Vec<u8>` or aligned buffers. The conversion `Vec<u8> → Bytes` is zero-copy (`Bytes::from(vec)`). For O_DIRECT, aligned buffers need a custom `Bytes` implementation or a copy step.

4. **File descriptor management**: io_uring can register file descriptors for reduced per-op overhead (`IORING_REGISTER_FILES`). For scan-heavy workloads with many files, this optimization is significant but adds lifecycle complexity.

### Ecosystem Impact

An io_uring `object_store` backend would automatically benefit:

| System | How It Benefits |
|---|---|
| DataFusion | Parquet, Arrow IPC, CSV reads |
| delta-rs | Delta Lake Parquet reads |
| Lance / LanceDB | Lance columnar format reads |
| Ballista | Distributed query local I/O |
| Iceberg-rust | Iceberg table Parquet reads |
| Any `object_store` user | Automatic via trait dispatch |

---

## §34.6 Candidate Anchor 4: Lance / LanceDB

**Fit assessment: NICHE BUT PRECISE**

### Why Lance

Lance is a Rust columnar format designed for ML/AI workloads. Key characteristics:

- **Wide tables**: Embedding vectors (hundreds of Float32 columns) + metadata columns
- **Selective column reads**: ML inference reads 3-5 feature columns from 200+ column tables
- **Large datasets**: Multi-GB to TB scale, often exceeding RAM
- **Uses `object_store`**: Local file I/O goes through the same abstraction layer

This maps directly to **Workload 3: Feature Store Column Extraction** (§11):
- 3-5 columns × many batches = 300-500 scattered reads
- Each read: 4 KiB to 64 KiB (Float32/Float64 vectors)
- 95% of file skipped
- **Predicted EP: 8-15×**

### Integration Path

Same as §34.5 — an io_uring `object_store` backend would benefit Lance automatically. Alternatively, Lance has its own `Reader` trait that could be directly backed by io_uring.

### Risks

- Niche audience (ML practitioners, not general analytics)
- Lance format is evolving rapidly; API stability is uncertain
- Community is smaller than DataFusion's

---

## §34.7 Comparative Assessment

| Criterion | Parquet Reader | Arrow IPC Reader | `object_store` Backend | Lance |
|---|---|---|---|---|
| **Alignment with experiments** | Strong (scattered reads pattern matches) | Direct (experiments are on IPC) | Indirect (infrastructure layer) | Strong (Workload 3 match) |
| **Predicted E2E gain** | 1.5-3× over current | 8.3× over vanilla | Depends on caller | 8-15× for feature extraction |
| **Community impact** | High (primary DF format) | Medium (secondary format) | Very High (ecosystem-wide) | Low (niche) |
| **Integration effort** | Medium | Low (code exists) | Medium | Medium |
| **Publishability** | High | Highest (validated data) | Medium (infra, not flashy) | Medium |
| **Novel contribution** | Incremental (Parquet already has projection) | Full stack (projection + batching + pipeline) | Infrastructure (enabling layer) | Application of existing insights |

---

## §34.8 Recommended Strategy

### Two-Layer Approach

**Layer 1 (Infrastructure): `object_store` io_uring backend**

Build a Linux-specific `ObjectStore` implementation backed by io_uring. This is the infrastructure layer that enables all downstream applications. Key design decisions:

- Use `io-uring` crate (low-level) with synchronous submission from `spawn_blocking` context
- Implement `get_ranges()` as batched io_uring submission
- Adaptive crossover: io_uring for ranges < 128 KiB, fall back to `pread64` for larger ranges
- Return `Bytes` via zero-copy `Vec<u8>` conversion
- Feature-gate behind `#[cfg(target_os = "linux")]` + cargo feature `io_uring`

**Layer 2 (Application): DataFusion showcase**

Demonstrate the `object_store` backend on a concrete DataFusion workload. Two options, non-exclusive:

- **Option A: Parquet on ClickBench** — Run ClickBench queries with the io_uring backend vs default. Measure cold-cache performance on column-selective queries. This is the most community-relevant benchmark.
- **Option B: Arrow IPC full-stack reader** — Port our `ipc-bench` pipeline into a DataFusion `ExecutionPlan` implementation. This demonstrates the full 8.3× advantage and is the most publishable result.

### Phasing

1. **Phase 1**: `object_store` io_uring backend with `get_ranges` batching. Validate with micro-benchmarks (equivalent to Exp C — isolated batch reads at varying sizes and depths).

2. **Phase 2**: Integrate with DataFusion Parquet reader. Run ClickBench cold-cache subset. Measure and report end-to-end gains.

3. **Phase 3**: Arrow IPC full-stack reader as a DataFusion `ExecutionPlan`. This is the "research paper" result — full pipeline with column projection, zone-map filtering, io_uring batching, and decode pipelining.

4. **Phase 4 (stretch)**: Adaptive I/O strategy that automatically selects io_uring vs mmap based on read size, batch depth, and cache state at runtime. This is the "adaptive-io" vision that gives the project its name.

---

## §34.9 Open Questions

1. **What does DataFusion's Parquet reader's actual I/O pattern look like?** We predict scattered small reads, but need to instrument `object_store::get_ranges` to measure real column-chunk sizes and batch depths on ClickBench queries. If column chunks are consistently > 128 KiB, the io_uring advantage disappears.

2. **Can `tokio-uring` be avoided?** `tokio-uring` requires its own runtime, making it unsuitable for library use. The alternative (dedicated I/O thread + channels) adds latency. Need to benchmark the overhead of the bridge vs the io_uring gain.

3. **Does O_DIRECT help or hurt for Parquet?** Our experiments showed O_DIRECT avoids page cache pollution but incurs buffer copy cost during decode (§29.3, Finding 33). For Parquet, where decoded data is further transformed (decompression, dictionary decoding), the extra copy may be negligible — or it may compound.

4. **What are the memory overhead implications?** io_uring requires ring buffers (SQ/CQ entries) and I/O buffers that must remain alive until completion. For a system under memory pressure, this overhead may be unacceptable. Need to quantify: how much memory does a 128-entry io_uring ring + registered buffers consume?

5. **How does the io_uring advantage interact with NVMe vs SATA vs cloud storage?** Our experiments are on NVMe. The batch submission advantage depends on device command latency (~85µs for NVMe page fault). On SATA (~5ms) or network storage, the relative advantage may be larger (more latency to hide) or negligible (bottlenecked on bandwidth, not latency).

---

## §34.10 Evidence Index

All claims in this document reference experimental findings from Parts 1-18:

| Claim | Finding | Source |
|---|---|---|
| Column projection = 5-10× I/O reduction | Finding 35 | [§29.3](./14-exp-g-e2e-comparison.md) |
| io_uring batch = 1.3-1.8× over mmap | Finding 32 | [§29.3](./14-exp-g-e2e-comparison.md) |
| Pipeline overlap = 1.86-2.04× | Finding 40, 47 | [§31](./16-exp-h2-pipeline.md), [§33](./18-exp-i-combined.md) |
| Combined 4T-pipeline = 8.3× vs vanilla | Finding 52 | [§33](./18-exp-i-combined.md) |
| Combined 4T-pipeline = 1.48× vs mmap | Finding 50 | [§33](./18-exp-i-combined.md) |
| Crossover at 128 KiB | — | [§23.2](./07-exp-c-fault-isolation.md) |
| Warm cache = no advantage | — | [§26](./10-exp-d-concurrent-scattered.md) |
| Cold concurrent = 2.6-7.9× | — | [§26](./10-exp-d-concurrent-scattered.md) |
| Zone-map filtering = 30-36× | — | [§27](./12-exp-e-zone-map-query.md) |
| Decode cost compresses advantage | Finding 33 | [§29.3](./14-exp-g-e2e-comparison.md) |
| Thread scaling modest for io_uring | Finding 45 | [§32.3](./17-exp-h3-cross-file.md) |
| Wide-table projection at 64 KiB | Finding 37 | [§30](./15-exp-h1-wide-table.md) |

← [Part 18](./18-exp-i-combined.md) | [Index](./README.md)
