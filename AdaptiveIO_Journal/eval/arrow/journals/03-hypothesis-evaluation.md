# Part 3: Hypothesis Evaluation

*Sections 8-10 of the research journal.*

---

*← Previous: [Baseline Experiments](./02-baseline-experiments.md) | [Index](./README.md) | Next: [Throughput & Scaling →](./04-throughput-and-scaling.md)*

---

## 8. H1 Evaluation: Column Projection Reduces I/O

**Hypothesis**: O_DIRECT column projection reduces total time by avoiding unnecessary I/O.
**Status**: **CONFIRMED** for cold starts.

- **Metric**: `odirect-column` reads ~91 MiB for `trip_miles` vs ~2064 MiB for strategies that read the full file.
- **Speedup**: 21-25% faster cold total time.
- **Surprise**: The speedup is primarily achieved during the *filter* phase, not the *load* phase, by avoiding serialized page fault stalls.

---

## 9. H2 Evaluation: Pipeline Overlap Hides Latency

**Hypothesis**: Pipelining I/O and CPU work hides latency.
**Status**: **PARTIALLY CONFIRMED**.

- **Metric**: `odirect-column-pipeline` (~189ms) is ~8% faster than `odirect-column` (~206ms) cold.
- **Analysis**: The benefit of pipelining is modest for single-column queries because the I/O volume is low enough that the NVMe bandwidth isn't fully utilized. The advantage grows as more columns are projected (increasing I/O volume).

---

## 10. H3 Evaluation: Full Sequential Scan - mmap Wins

**Hypothesis**: mmap is superior for full sequential file scans.
**Status**: **CONFIRMED** for warm, **REFUTED** for cold.

- **Warm**: `mmap-naive` (~446ms) is significantly faster than `odirect-full` (~2.5s) as it hits the page cache.
- **Cold**: `mmap-naive` (~1.6s) is actually faster than `odirect-full` (~2.6s), but both are dominated by the sheer volume of data. Interestingly, `mmap-populate` (509ms) is the sleeper hit for full scans, as it uses the kernel's highly optimized bulk faulting.

---

*← Previous: [Baseline Experiments](./02-baseline-experiments.md) | [Index](./README.md) | Next: [Throughput & Scaling →](./04-throughput-and-scaling.md)*
