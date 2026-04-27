# mmap vs io_uring for Arrow IPC: Research Journal

A systematic investigation of when and why `io_uring` batch submission outperforms `mmap` demand paging for reading Apache Arrow IPC files on NVMe storage.

## Reading Guide

This journal is organized chronologically as the research evolved. **Start at Part 1 for full context, or jump to Part 7 for the key pivot, or Part 9 for the final verdict.**

| Part | Title | Sections | Summary |
|------|-------|----------|---------|
| [01](./01-methodology.md) | Methodology & Setup | §1-3 | Dataset, 11 strategies, 5 hypotheses |
| [02](./02-baseline-experiments.md) | Baseline Experiments | §4-7 | Cold/warm cache results — O_DIRECT wins cold, mmap wins warm |
| [03](./03-hypothesis-evaluation.md) | Hypothesis Evaluation | §8-10 | H1 confirmed, H2 partial, H3 confirmed warm / refuted cold |
| [04](./04-throughput-and-scaling.md) | Throughput & Scaling | §11-18 | Multi-file, concurrent, pipeline, memory pressure |
| [05](./05-validation-experiments.md) | Validation Experiments | §19-21 | User-space cache (6,000× speedup), wide-table crossover |
| [06](./06-research-pivot.md) | **Research Pivot** | §22 | From "less work" to "faster work" — the key insight |
| [07](./07-exp-c-fault-isolation.md) | Exp C: Fault Isolation | §23 | 5 strategies, N scaling, size scaling — **13.6× speedup** |
| [08](./08-exp-a-scattered-reads.md) | Exp A: Scattered Reads | §24 | Real IPC metadata offsets — **17-23× speedup** |
| [09](./09-exp-b-concurrent-verdict.md) | Exp B & Final Verdict | §25 | Concurrent scaling + **definitive answer** |
| [10](./10-exp-d-concurrent-scattered.md) | **Exp D: Cold Concurrent** | §26 | Cold-cache scattered + threads — **2.6-7.9× speedup** |
| [11](./11-workload-recommendations.md) | **Workload Recommendations** | — | Where io_uring wins: 5 realistic workloads with predicted EP |
| [12](./12-exp-e-zone-map-query.md) | Exp E: Zone-Map Query | §27 | Real metadata-driven offsets — **30-36× speedup** |
| [13](./13-exp-f-multi-file-scan.md) | Exp F: Multi-File Scan | §28-29 | Scaling across 12 files — **8-9× speedup** |
| [14](./14-exp-g-e2e-comparison.md) | End-to-End Comparison | §29 | vanilla vs mmap vs iouring on same pipeline |
| [15](./15-exp-h1-wide-table.md) | H1: Wide-Table Projection | §30 | 200-column projection at 64 KiB |
| [16](./16-exp-h2-pipeline.md) | H2: Pipelined I/O + Decode | §31 | Overlapping producer/consumer phases |
| [17](./17-exp-h3-cross-file.md) | H3: Cross-File Query | §32 | Multi-threaded scaling across 12 files |
| [18](./18-exp-i-combined.md) | Exp I: Combined Pipeline | §33 | Pipelining + Multi-threading E2E results |
| [19](./19-anchoring-analysis.md) | **Anchoring Analysis** | §34 | Real-world integration targets — DataFusion, object_store, Lance |
| [20](./20-anchoring-experiment-plan.md) | **Anchoring Experiment Plan** | §35-43 | Strategy for DataFusion/Parquet integration |
| [21](./21-anchoring-results.md) | **Anchoring Experiment Results** | §44-47 | (In Progress) Results for io_uring ObjectStore |
| [22](./22-coalescing-results.md) | **Coalescing Reader Results** | §48-52 | **H1=1.438× PASS (batching), H2=0.998× FAIL (io_uring), H3=1.428× PASS (combined)** — user-space fix wins, io_uring neutral |
| [23](./23-reactor-experiment.md) | **Reactor Experiment (Negative)** | §53-59 | Persistent ring + reactor **made io_uring 11% worse** (H2=0.887×) — serialization destroyed parallelism, ObjectStore API incompatible with io_uring's batch model |
| [24](./24-k-sweep-landscape.md) | **K-Sweep & Landscape Survey** | §60-63 | **C5 PASS (K-sweep), C6 PASS (landscape)** — metadata cache is the dominant lever (96%+); request-response pattern shared by 6/9 engines. |

## Narrative Arc

```
Parts 1-5: "When should you use io_uring instead of mmap?"
           Answer: When you can skip I/O (column projection, caching).

Part 6:    "Wait — we never tested equal-work scenarios."
           The pivot: from I/O avoidance to I/O efficiency.

Parts 7-9: "Is mmap's page fault model fundamentally suboptimal?"
           Answer: YES, for reads < ~128 KiB. 3.5-23× slower.

Part 10:   "Does the advantage hold under concurrent load?"
           Answer: YES for cold cache (2.6-7.9×). NO for warm cache (3%).

Part 11:   "Now what? Five concrete workloads to prove it."
           Practical prescriptions for the next-generation Arrow engine.

Phase 6:   "Realistic Workload Validation (Exp E, F)"
           Validating prescriptions with zone-maps and multi-file scans.

Phase 7:   "End-to-end validation (Part 14)"

Phase 8:   "Stress Testing & Parallelism (Parts 15-17)"
            Pushing the limits with wide tables, pipelining, and multi-file query scaling.

Phase 9:   "Combined Optimizations (Part 18)"
            Final end-to-end evaluation of pipelined multi-threading.

Phase 10:  "Anchoring Analysis (Part 19)"
            Real-world integration targets: DataFusion Parquet, Arrow IPC, object_store, Lance.

Phase 11:  "Anchoring Experiment (Parts 20-21)"
            Implementation and results for io_uring ObjectStore integration.
```


## Key Findings (Quick Reference)

| Finding | Source | One-liner |
|---------|--------|-----------|
| io_uring batch = 13.6× faster at 4 KiB | [Part 7, §23.1](./07-exp-c-fault-isolation.md) | N=100 random offsets, cold cache |
| Crossover at ~128 KiB | [Part 7, §23.2](./07-exp-c-fault-isolation.md) | mmap wins above this read size |
| 30-36× faster with real zone-maps | [Part 12, §27.1](./12-exp-e-zone-map-query.md) | Real IPC metadata-driven offsets (Exp E) |
| mmap-serial matches io_uring for clustered access | [Part 13, §28.1](./13-exp-f-multi-file-scan.md) | Kernel readahead is effective for 1-2 batch hits (Exp F) |
| mmap_lock contention is NOT the bottleneck | [Part 9, §25.1](./09-exp-b-concurrent-verdict.md) | Per-fault serialization dominates |
| Cold-cache concurrent: 2.6-7.9× speedup | [Part 10, §26.1](./10-exp-d-concurrent-scattered.md) | io_uring advantage persists under concurrent load |
| NVMe queue saturation degrades EP at 12 threads | [Part 10, §26.2](./10-exp-d-concurrent-scattered.md) | EP drops from 7.9× (1 thread) to 2.6× (12 threads) |
| Pipeline overlap gives 1.86× speedup | [Part 16, §31.2](./16-exp-h2-pipeline.md) | Overlapping I/O with decode+filter (Exp H2) |

## Data & Code

- **Source code**: [`../ipc-bench/src/`](../ipc-bench/src/) — Rust implementation of all strategies
- **Benchmark results**: [`../ipc-bench/results/`](../ipc-bench/results/) — Raw output files cited in tables
- **Dataset**: `fhvhv_tripdata_2021-{01..12}.arrow` (~30 GiB total, 91 batches per file)
- **Scripts**: [`../ipc-bench/scripts/`](../ipc-bench/scripts/) — `run_exp_{a..f}.sh`, `check_correctness.sh`

## Lineage

This research was conducted in phases:

1. **Phase 1** (Parts 1-5): Initial hypothesis-driven comparison of mmap vs io_uring strategies with varying I/O volumes.
2. **Phase 2** (Part 6): Critical reassessment — realized prior experiments measured I/O *avoidance*, not I/O *efficiency*.
3. **Phase 3** (Parts 7-9): Designed equal-work micro-experiments to isolate the architectural difference. Produced the definitive finding: **batch submission parallelism vs serialized page faults**.
4. **Phase 4** (Part 10): Validated that the advantage persists under realistic concurrent cold-cache workloads. Discovered NVMe queue saturation as a scaling limit.
5. **Phase 5** (Part 11): Translated experimental results into 5 concrete workload recommendations for systems engineers.
6. **Phase 6** (Parts 12-13): Validated recommendations with realistic zone-map filtered queries and multi-file partition scans (Exps E and F).
7. **Phase 7** (Part 14): End-to-end validation comparing vanilla Arrow with I/O-level projection.
8. **Phase 8** (Parts 15-17): Specialized stress tests for wide-table projection, pipelined overlap, and multi-file query parallelism.
9. **Phase 9** (Part 18): Combined pipeline + multi-threading — final end-to-end evaluation.
10. **Phase 10** (Part 19): Anchoring analysis — mapping experimental insights onto real-world systems (DataFusion Parquet reader, Arrow IPC reader, `object_store` backend, Lance).

Each part builds on the previous. Cross-references use `§N` notation (e.g., "see §22.3") and link to the containing part file.
