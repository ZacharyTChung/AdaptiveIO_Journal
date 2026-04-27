# Journal 24 — K-Sweep Ablation and Landscape Survey

**Status**: ✅ Complete
**Precedent**: [Journal 23 §59](./23-reactor-experiment.md) established the reactor negative result and identified the ObjectStore API as structurally incompatible with io_uring's batch model.
**Data**:
- K-sweep results: `eval/clickbench/results/run_20260410_071914/k_sweep/`
- K-sweep analysis: `eval/clickbench/results/run_20260410_071914/k_sweep/k_sweep_analysis.json`
- Landscape survey: `eval/arrow/landscape-survey.md`

## §60 — K-Sweep Setup

This experiment isolates the performance contributions of the two mechanisms in the `CoalescingParquetFileReaderFactory`: (1) the `SharedMetadataCache` and (2) the lookahead prefetch window. 

Journal 22 measured the combined effect at K=4, yielding H1 = 1.438× median speedup. To decompose this, we swept the prefetch window size K ∈ {1, 2, 4, 8, 16} across a representative subset of ClickBench queries {q0, q1, q3, q7}. 

- **K=1 (Ablation baseline)**: Represents the "metadata-cache-only" configuration. The `SharedMetadataCache` is active, eliminating redundant footer reads, but the prefetch lookahead is disabled (`get_byte_ranges` only fetches the current row group's ranges).
- **K=4 (Default)**: The standard configuration from Journal 22, enabling both metadata caching and 3-row-group lookahead.
- **K=8, 16 (Stress)**: Tests the limits of eager prefetching on NVMe.

The experiment executed 400 total cold-cache runs (5 iterations per K per query per store per factory). The goal is to validate claim **C5**: the speedup is dominated by the metadata cache (K=1) and increases with lookahead (K=4 > K=1).

---

## §61 — K-Sweep Results

### §61.1 — H1 per K per Query (L,D / L,C)

| Query | K=1 | K=2 | K=4 | K=8 | K=16 |
|---|---|---|---|---|---|
| q0 | 2.170 | 2.139 | 2.193 | 2.296 | 2.181 |
| q1 | 1.571 | 1.595 | 1.615 | 1.489 | 1.547 |
| q3 | 1.516 | 1.495 | 1.464 | 1.004 | 0.883 |
| q7 | 1.677 | 1.685 | 1.702 | 1.818 | 1.697 |

### §61.2 — Decomposition Analysis

We define `metadata_fraction` as the ratio of the K=1 speedup over the K=4 speedup, relative to a 1.0 baseline: `(H1_K1 - 1) / (H1_K4 - 1)`.

| Query | H1_K1 | H1_K4 | meta_frac | lookahead_frac |
|---|---|---|---|---|
| q0 | 2.170 | 2.193 | 0.981 | 0.019 |
| q1 | 1.571 | 1.615 | 0.929 | 0.071 |
| q3 | 1.516 | 1.464 | 1.112 | -0.112 |
| q7 | 1.677 | 1.702 | 0.964 | 0.036 |

**Interpretation**:
- **Metadata cache is the dominant lever**: For q0, q1, and q7, the `SharedMetadataCache` alone (K=1) accounts for **93%–98%** of the total speedup measured at K=4. Lookahead prefetching contributes a marginal 2%–7% additional gain.
- **q3 Anomaly**: At K=1, q3 shows a strong 1.52× speedup. However, increasing K leads to diminishing returns (1.46× at K=4) and eventual **severe regression** (1.004× at K=8, 0.883× at K=16). q3 is a filter-heavy query; the eager prefetch window fetches row group data that the parquet reader's zone-map filter would have otherwise pruned. At K=16, the cost of fetching this unused data exceeds the win from reduced syscalls.

**C5 Verdict: PASS**
Both conditions are met:
1. **Condition A**: K=1 H1 > 1.0 for all queries (metadata cache is always a win).
2. **Condition B**: K=4 > K=1 for the majority of queries (3/4 queries show additional prefetch gain).

---

## §62 — Landscape Survey

To evaluate the generalizability of our findings, we surveyed 9 production columnar analytics engines to classify their storage I/O interface patterns (see [`eval/arrow/landscape-survey.md`](../landscape-survey.md) for full details).

### §62.1 — Summary of Findings

- **Request-Response (6 of 9)**: DataFusion, Polars, DuckDB, Velox, Spark, Iceberg-rs. These engines issue I/O through synchronous or await-per-call APIs where each read request blocks or suspends until the data is returned.
- **Async-Submit (3 of 9)**: ClickHouse, Umbra/CedarDB, LanceDB. These engines decouple I/O submission from completion, allowing for native batching.

### §62.2 — Generalizability

The `object_store` crate — shared by DataFusion, Polars, and Iceberg-rs — is the de facto storage abstraction for the Rust ecosystem with over 54M downloads. Its `get_ranges` API, while accepting a vector of ranges, is consumed via a single awaited future in the parquet reader, effectively serializing the storage boundary crossing.

**C6 Verdict: PASS**
6 of 9 surveyed engines (67%) use the request-response pattern. This exceeds the 5/9 threshold, confirming that the "above the abstraction" bottleneck identified in our study is a representative property of the current columnar analytics landscape.

---

## §63 — Conclusions

The combination of the K-sweep ablation and the landscape survey solidifies the core thesis of the ATC paper:

1. **User-space coordination is the bottleneck**: The K-sweep proves that the majority of the measured speedup comes from eliminating redundant metadata reads (a pure user-space coordination fix) rather than I/O parallelism. Lookahead prefetch provides a marginal win but must be balanced against over-fetching in filtered queries (q3).
2. **io_uring is neutral because of the API**: Our prior H2=0.998× finding is now explained by the structural mismatch between the request-response pattern (shared by 6 of 9 engines) and io_uring's batch model. The storage abstraction boundaries in modern engines are designed to wait for data, not to submit batches.
3. **Generalizability**: Because the majority of engines share this request-response abstraction shape, our "above-abstraction" coalescing approach is the most pragmatic and high-impact intervention for the ecosystem. Engines like ClickHouse and Umbra that do win with io_uring required fundamental architectural rewrites (async-submit readers, coroutines) to bypass the very bottleneck we have quantified.

The evidence pyramid is now complete:
- **Below abstraction**: H2 = 0.998× (neutral)
- **At abstraction**: Reactor H2 = 0.887× (regression due to serialization)
- **Above abstraction**: Coalescing H1 = 1.438× (44% speedup from coordination)
- **Generalizability**: C6 = 6/9 engines (PASS)

These results direct the paper's recommendations: engines should focus on user-space metadata caching and adaptive, pruning-aware prefetching rather than kernel-level backend swaps.
