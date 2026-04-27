# Journal 23 — io_uring Reactor Experiment (Negative Result)

**Status**: ✅ Complete — reactor approach refuted, causal analysis documented.
**Precedent**: [Journal 22 §52](./22-coalescing-results.md) established H2=0.998× and identified 4 structural factors explaining the io_uring neutral result.
**Data**:
- v2 Cold: `eval/clickbench/results/run_20260410_004046/phase2b/cold.csv` (200 rows)
- v2 Warm: `eval/clickbench/results/run_20260410_004628/phase2b/warm.csv` (200 rows)
- v2 Tracing: `eval/clickbench/results/run_20260410_010209/phase2b/traces/`
- v2 Analysis: `eval/clickbench/results/run_20260410_004046/phase2b/analysis_phase2b.json`
- v1 Analysis (baseline): `eval/clickbench/results/run_20260409_224511/phase2b/analysis_phase2b.json`

## §53 — Motivation

Journal 22 found H2=0.998× — io_uring provided zero marginal benefit once the coalescing reader was in place. Root-cause investigation identified 4 structural factors explaining why io_uring won 1.5–9× in the IPC experiments (Journals 13–18) but not through the ObjectStore API:

| Factor | IPC bench (wins) | ObjectStore (no win) |
|---|---|---|
| 1. Ring lifetime | Persistent (DirectFile struct) | Ephemeral (new ring per `get_ranges` call) |
| 2. I/O mode | O_DIRECT (async block I/O) | Buffered (VFS/page cache) |
| 3. Batch depth | 100s-512 SQEs per submit | 1-4 SQEs per submit |
| 4. Read size | 4 KiB (syscall-dominated) | 50-500 KiB (I/O-dominated) |

This experiment attempted to fix Factors 1 and 3 by implementing a reactor pattern (persistent ring + request aggregation) and async prefetch (concurrent `get_ranges` calls from the coalescing reader).

## §54 — Implementation

### §54.1 — IoUring Reactor (`reactor.rs`)

Replaced the ephemeral `IoUring::new()` + `File::open()` per `get_ranges` call with a dedicated reactor thread:

- Single `io_uring` ring created once at `IoUringStore` construction, owned by the reactor thread for its entire lifetime.
- `HashMap<PathBuf, File>` fd cache — files opened once, held until shutdown.
- `tokio::sync::mpsc::unbounded_channel` for request submission (Send+Sync, no `spawn_blocking`).
- Reactor loop: `blocking_recv()` for first request → `try_recv()` drain of all pending → `process_batch()` flattens ranges from all concurrent requests into one io_uring submission → distributes results via `oneshot` channels.

Intent: mirror the IPC bench's `DirectFile` + `read_many()` architecture where ring setup is amortized and all reads go through one persistent ring.

### §54.2 — Async Prefetch (coalescing reader)

Split `get_byte_ranges` from one synchronous fetch (current + prefetch ranges) into:

1. **Critical path**: Fetch only the current row group's ranges via `store.get_ranges()` (awaited immediately).
2. **Background**: `tokio::spawn()` a fire-and-forget task to fetch the next K-1 row groups' ranges.

`RangeCache` changed from `RangeCache` (owned) to `Arc<Mutex<RangeCache>>` for shared access between the foreground path and background prefetch tasks.

Intent: create concurrent `get_ranges` calls that the reactor could aggregate into larger SQE batches. With 8 DataFusion streams × 2 concurrent calls per stream (current + prefetch) = up to 16 concurrent requests landing in the reactor's queue.

## §55 — Results (Phase 2b v2)

### §55.1 — Hypothesis Verdicts: v1 vs v2

| Hypothesis | v1 (Journal 22) | v2 (reactor) | Delta |
|---|---:|---:|---:|
| H1 pure batching | 1.438× PASS | 1.474× PASS | +2.5% (noise) |
| **H2 io_uring marginal** | **0.998× FAIL** | **0.887× FAIL** | **−11.1% (regression)** |
| H3 combined | 1.428× PASS | 1.365× PASS | −4.4% |

The reactor made io_uring **actively slower**, degrading H2 from neutral (0.998×) to a measurable regression (0.887×). H1 was unaffected (confirming the coalescing reader changes are orthogonal).

### §55.2 — Per-Query H2 Comparison

| Query | v1 L,C (ms) | v1 I,C (ms) | v1 H2 | v2 L,C (ms) | v2 I,C (ms) | v2 H2 | Δ I,C |
|---|---:|---:|---:|---:|---:|---:|---:|
| q0  | 145 | 147 | 0.986 | 144 | 145 | 0.993 | −1% |
| q1  | 276 | 273 | 1.011 | 281 | 272 | 1.033 | 0% |
| q2  | 224 | 216 | 1.037 | 207 | 255 | 0.812 | +18% |
| q3  | 348 | 353 | 0.986 | 326 | **527** | **0.619** | **+49%** |
| q5  | 650 | 643 | 1.011 | 697 | **882** | **0.790** | **+37%** |
| q7  | 260 | 274 | 0.949 | 253 | 262 | 0.966 | −4% |
| q8  | 655 | 669 | 0.979 | 697 | 784 | **0.889** | +17% |
| q13 | 1064 | 1069 | 0.995 | 1116 | **1552** | **0.719** | **+45%** |
| q29 | 574 | 563 | 1.020 | 503 | 537 | 0.937 | −5% |
| q42 | 295 | 295 | 1.000 | 255 | 288 | 0.885 | −2% |

Queries q3, q5, q13 sustained **40–49% wall-time regressions** on the io_uring path. These are the queries with the most ObjectStore calls (350–450 per query), where the reactor's serialization overhead accumulates.

### §55.3 — Tracing Evidence: Reactor Did NOT Batch

| Metric | v1 coalescing | v2 coalescing |
|---|---:|---:|
| total_calls (median) | 77 | **321** |
| batch_p50 | 4 | **1** |
| batch_p95 | 4 | **3** |
| batch_max | 4 | 3 |
| footer_reads | 2 | 2 |

v2 `total_calls` increased 4× (77 → 321) because the async prefetch splits each cache miss into two separate `get_ranges` calls (current + background prefetch). But the reactor's `batch_p50 = 1` proves it processed **every request individually** — `try_recv()` always found nothing because requests arrive one at a time.

## §56 — Root Cause Analysis

### §56.1 — No Temporal Overlap

The reactor's batching strategy is `blocking_recv()` → `try_recv()` drain → process batch. This only aggregates requests that arrive **while the reactor is blocked on `recv()`**. But each `get_ranges` call completes in microseconds (NVMe I/O + ring submission + CQE drain), so the reactor finishes and loops back to `recv()` before the next request arrives. Result: batch size always 1.

The async prefetch was designed to create concurrent requests, but the prefetch task is spawned AFTER the current fetch completes. By the time the prefetch's `store.get_ranges()` reaches the reactor, the current fetch's `get_ranges()` has already been processed and returned. No temporal overlap.

### §56.2 — Single-Thread Serialization

The original `IoUringStore` used `tokio::task::spawn_blocking(get_ranges_sync)`, which dispatched each `get_ranges` call to a thread from tokio's blocking pool (up to 512 threads). Multiple DataFusion partition streams could execute I/O **in parallel** on different threads.

The reactor funnels ALL I/O through a single thread. With 8 concurrent DataFusion streams:

- **Before (spawn_blocking)**: 8 threads × 1 ring each = 8 concurrent I/O operations (each with its own ephemeral ring, but parallel).
- **After (reactor)**: 1 thread × 1 ring = strictly serial I/O.

For queries with many ObjectStore calls (q3: 447, q5: 388, q13: 400), serializing hundreds of previously-parallel I/O operations into one thread caused the 40–49% regressions.

### §56.3 — Channel Overhead Without Aggregation

Each `get_ranges` call in the reactor path pays ~2–5µs for channel communication (send request, receive oneshot). In the `spawn_blocking` path, the overhead was just `spawn_blocking` scheduling (~1µs). With 300+ calls per query and zero aggregation benefit, this adds ~600–1500µs of pure overhead.

## §57 — Implications

### §57.1 — Why the IPC Bench Pattern Doesn't Transfer

The IPC bench (Journals 13–18) showed io_uring wins because the **application** collected all reads upfront (N=512) and called `read_many()` once. The reactor tried to replicate this by aggregating concurrent requests from the store level. But DataFusion's execution model prevents this:

- Each partition stream processes row groups **sequentially** within a single async task.
- `get_ranges()` is called → awaited → result processed → next call. No overlap within a task.
- Cross-task overlap exists (multiple streams run concurrently), but at microsecond granularity the I/O calls don't actually overlap in time.

The fundamental mismatch: io_uring needs **batch submission** (many SQEs per `io_uring_enter`), but DataFusion produces **streaming requests** (one `get_ranges` at a time, process result, repeat). These are architecturally incompatible at the ObjectStore abstraction layer.

### §57.2 — What This Rules Out

For DataFusion Parquet queries on NVMe + Linux 6.8:

1. **Drop-in ObjectStore replacement**: Neither ephemeral rings (H2=0.998×) nor persistent reactor (H2=0.887×) improve over `LocalFileSystem`. The pread + page cache path is already optimal for the request pattern DataFusion produces.

2. **Store-level request aggregation**: The reactor proved that concurrent callers don't actually overlap at the I/O level. Temporal batching at the store is not viable without artificial delays (which would add latency).

3. **Async prefetch for I/O depth**: Splitting current + prefetch into concurrent calls increased total calls 4× but produced no batching because the calls still serialize at the reactor.

### §57.3 — What Remains Possible

Approaches not tested by this experiment (documented for future investigation):

1. **Custom AsyncFileReader with owned ring**: Bypass ObjectStore entirely. The reader would hold a persistent ring (like IPC bench's `DirectFile`), collect all column chunk ranges for a row group batch, and submit in one `read_many()` call. This is a deeper DataFusion integration than the ObjectStore layer.

2. **DataFusion-native I/O scheduler**: A cross-stream coordinator that collects `get_ranges` requests from all active partition tasks, batches them into large submissions, and dispatches results. This would require changes to DataFusion's ParquetExec internals.

3. **Different storage backend**: On HDD, network storage, or disaggregated storage where individual I/O latency is 100µs–10ms (vs NVMe's ~10µs), io_uring's async submission would matter more because the CPU can do useful work while waiting for I/O completion.

4. **Ring pool** (Factor 1 only): `Mutex<Vec<(IoUring, File)>>` with checkout/checkin in `spawn_blocking`. Eliminates per-call ring setup without serialization. Expected to recover H2 to ~1.00 (neutral) but not produce a positive win.

## §58 — Code Disposition

The reactor implementation (`reactor.rs`) is reverted in favor of a ring-pool approach that addresses Factor 1 (ephemeral ring setup overhead) without introducing serialization. The async prefetch changes to the coalescing reader are retained — they are orthogonal to the io_uring question and show neutral-to-positive H1 impact.

Changes retained:
- `coalescing_reader.rs`: `Arc<Mutex<RangeCache>>`, async prefetch (`tokio::spawn`), `RangeCache::contains()`, `PrefetchMetrics::prefetch_spawned`

Changes reverted:
- `store.rs`: Back to `spawn_blocking` + ring pool (not the original ephemeral pattern, but a clean persistent-ring design)
- `reactor.rs`: Removed (documented as failed experiment in this journal)

## §59 — Conclusion

The reactor experiment is a **definitive negative result** for io_uring on DataFusion's Parquet reader via the ObjectStore API. The attempt to transfer the IPC bench's persistent-ring + batch-aggregation pattern failed because:

1. DataFusion's streaming execution produces temporally non-overlapping I/O requests — the reactor's batch window is always empty.
2. Funneling parallel I/O through one thread serialized previously concurrent work, causing 11% median regression and up to 49% per-query damage (q3).
3. The ObjectStore API's request-response contract (`get_ranges` → `Vec<Bytes>`) is structurally incompatible with io_uring's submit-many-complete-later model.

Combined with Journal 22's H2=0.998× (ephemeral rings) and this journal's H2=0.887× (reactor), the evidence is clear: **for DataFusion Parquet queries on NVMe, the I/O backend is not the bottleneck.** The 1.44× speedup from Journal 22 comes entirely from user-space I/O coordination (shared footer cache + prefetch batching), not from kernel-level I/O dispatch. io_uring's advantages — measured at 1.5–9× in the IPC microbenchmarks — require application-level batch collection that DataFusion's streaming executor cannot provide through the ObjectStore abstraction.
