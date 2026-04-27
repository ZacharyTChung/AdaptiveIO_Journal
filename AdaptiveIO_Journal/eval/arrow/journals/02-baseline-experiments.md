# Part 2: Baseline Experiments

*Sections 4-7 of the research journal.*

---

*← Previous: [Methodology](./01-methodology.md) | [Index](./README.md) | Next: [Hypothesis Evaluation →](./03-hypothesis-evaluation.md)*

---

## 4. Cold Cache Experiments (Stride=1)

This experiment evaluates performance on a cold start, reading every record batch (Stride=1) and projecting a single column (`trip_miles`).

### 4.1 Results Table (Cold Stride=1)
*Average of 5 rounds. Data Source: results/run_20260326_045556/*

| Strategy | Load (ms) | Decode (ms) | Filter (ms) | Total (ms) | Page Faults |
|---|---|---|---|---|---|
| `mmap-projection` | 0.01 | 33.47 | 228.02 | 261.50 | 2,497 |
| `odirect-column` | 120.22 | 0.37 | 86.07 | 206.66 | 23,465 |
| `odirect-column-pipeline` | 118.92 | 0.37 | 70.44 | 189.73 | 23,455 |
| `mmap-naive` | 0.01 | 1403.66 | 226.11 | 1629.78 | 17,061 |
| `odirect-full` | 2251.72 | 317.22 | 84.10 | 2653.05 | 528,768 |
| `mmap-populate` | 111.47 | 309.40 | 88.38 | 509.26 | 33,126 |
| `pread` | 1098.12 | 304.78 | 96.15 | 1499.05 | 528,584 |

### 4.2 Key Findings
**Finding 1: odirect-column is 21% faster than mmap-projection cold.**
By reading only the ~91 MiB of column data directly into user-space, `odirect-column` avoids the overhead of demand-paging the entire 2 GiB file. While `mmap-projection` only "touches" the relevant pages, the kernel's fault handling remains serialized and incurs a significant penalty during the filtering phase.

**Finding 2: mmap-populate is 2x slower than odirect-column.**
While `mmap-populate` eagerly faults in the data, the cost of the kernel managing the page table entries for the full 2 GiB file far exceeds the cost of a manual O_DIRECT read of the selected 91 MiB.

**Finding 3: Filter phase latency is a smoking gun.**
The filter phase for `mmap-projection` takes ~228ms compared to ~86ms for `odirect-column`. This 142ms difference represents the time the filtering thread spent blocked on synchronous page faults.

---

## 5. Cold Cache Experiments (Stride=5)

In this experiment, we sample every 5th batch, testing the efficiency of sparse I/O.

### 5.1 Results Table (Cold Stride=5)
*Average of 5 rounds. Data Source: results/run_20260326_045556/*

| Strategy | Load (ms) | Decode (ms) | Filter (ms) | Total (ms) |
|---|---|---|---|---|
| `mmap-projection` | 0.01 | 7.37 | 105.70 | 113.09 |
| `odirect-column` | 67.57 | 0.34 | 25.72 | 93.63 |

### 5.2 Key Findings
**Finding 4: O_DIRECT maintains a 17% advantage for sampled access.**
The gap narrows slightly because the total amount of data is smaller, but `odirect-column` still wins by explicitly skipping the I/O for the 80% of batches that are not required. `mmap-projection` still triggers serialized faults for the batches it does touch, inflating the filter time by 4× (105ms vs 25ms).

---

## 6. Warm Cache Experiments (Stride=1)

This experiment measures performance after data has been loaded into the system's memory.

### 6.1 Results Table (Warm Stride=1)
*Average of 5 iterations. Data Source: results/run_20260326_042937/*

| Strategy | Load (ms) | Decode (ms) | Filter (ms) | Total (ms) | Page Faults |
|---|---|---|---|---|---|
| `mmap-projection` | 0.01 | 0.42 | 73.17 | 73.60 | 1,643 |
| `odirect-column` | 124.11 | 0.04 | 90.52 | 214.67 | 23,265 |
| `mmap-naive` | 0.01 | 351.93 | 94.46 | 446.40 | 11,607 |

### 6.2 Key Findings
**Finding 5: mmap-projection wins warm (3× faster than odirect-column).**
This highlights the fundamental disadvantage of standard `O_DIRECT`: it must perform a disk read (or at least a round-trip to the NVMe controller) every time. `mmap-projection` benefits from the OS page cache, turning I/O into simple memory copies.

**Finding 6: The 73ms Filter Floor.**
On a warm cache, the `mmap-projection` total time of ~74ms represents the irreducible CPU cost of scanning the 11.9M rows. Any strategy aiming to beat this must either eliminate I/O or optimize the filter logic itself.

---

## 7. Warm Cache Stride=5

Warm sampled access confirms the same pattern.

| Strategy | Total (ms) |
|---|---|
| `mmap-projection` | ~22.4ms |
| `odirect-column` | ~94.1ms |

---

*← Previous: [Methodology](./01-methodology.md) | [Index](./README.md) | Next: [Hypothesis Evaluation →](./03-hypothesis-evaluation.md)*
