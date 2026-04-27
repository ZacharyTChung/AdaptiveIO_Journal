# Part 1: Methodology & Setup

*Sections 1-3 of the research journal.*

---

*[Index](./README.md) | Next: [Baseline Experiments →](./02-baseline-experiments.md)*

---

## 1. Introduction

This report documents a comprehensive systems research project investigating the performance characteristics of `io_uring` with `O_DIRECT` compared to traditional `mmap` for reading Apache Arrow IPC files on high-performance NVMe storage. The central research question is: under what specific architectural conditions and access patterns does a manual, batch-submission asynchronous I/O approach outperform the kernel's automated demand-paging and readahead mechanisms?

The experiments leverage a large real-world dataset and simulate various data engine workloads, ranging from full sequential scans to highly selective column projections.

### 1.1 Dataset and Environment
- **Primary Dataset**: `fhvhv_tripdata_2021-01.arrow` (2064 MiB, 91 batches, 11.9M rows, 24 columns).
- **Extended Dataset**: 12 monthly partition files (~29 GiB total).
- **Synthetic Dataset**: 200-column Float64 table (2.0 GiB) for scaling studies.
- **Hardware**: High-speed NVMe SSD (4096-byte sector alignment).

### 1.2 Strategies Under Test
We implemented and evaluated 11 distinct I/O and decoding strategies:

- **mmap variants**:
  - `mmap-naive`: Standard memory mapping with default kernel demand paging.
  - `mmap-sequential`: Memory mapping with `MADV_SEQUENTIAL` advice.
  - `mmap-willneed`: Memory mapping with `MADV_WILLNEED` eager prefetching.
  - `mmap-populate`: Memory mapping with `MAP_POPULATE` to force page table population.
  - `mmap-projection`: Memory mapping utilizing Arrow's `FileDecoder` with specific column projection.

- **O_DIRECT variants (using io_uring)**:
  - `odirect-full`: Eagerly reads the entire file into an aligned user-space buffer.
  - `odirect-pipeline`: Producer-consumer thread model reading the full file in chunks.
  - `odirect-column`: Selective I/O reading only the specific buffers required for the projected columns.
  - `odirect-column-pipeline`: Pipelined version of the selective column reader.

- **Legacy/Baseline**:
  - `pread`: Standard synchronous positional reads.
  - `iouring-prefetch`: A hybrid strategy using `io_uring` to issue `MADVISE` opcodes.

### 1.3 Core Hypotheses
- **H1 (Column Projection)**: Selective O_DIRECT reads will significantly outperform mmap by avoiding I/O for unselected columns.
- **H2 (Pipelining)**: Asynchronous batch submission will hide I/O latency by overlapping it with CPU-bound decoding and filtering.
- **H3 (Sequential Scans)**: For full-file sequential scans, mmap's optimized readahead will remain the performance leader.
- **H4 (Throughput Scaling)**: io_uring will provide better multi-threaded isolation and throughput scaling than mmap.
- **H5 (Memory Pressure)**: O_DIRECT strategies will exhibit more stable performance than mmap under heavy memory pressure by bypassing the OS page cache.

---

## 2. Dataset and Methodology

### 2.1 Arrow IPC File Structure
The test file `fhvhv_tripdata_2021-01.arrow` consists of:
- **91 Record Batches**: 90 batches of 131,072 rows and a final batch of 111,988 rows.
- **24 Columns**: A mix of 8 string columns (3 buffers each) and 16 numeric columns (2 buffers each).
- **Metadata**: Flatbuffer-encoded footer describing batch offsets and buffer locations.

Crucially, buffer offsets in Arrow IPC files are generally not 4096-byte aligned. Our `O_DIRECT` implementation handles this by rounding read ranges to the nearest 4 KiB boundary and performing zero-copy sub-slicing within the aligned buffers.

### 2.2 Measurement Methodology
For each strategy and configuration, we measure:
- **Load Time (ms)**: Time spent making data resident (explicit read or mapping).
- **Decode Time (ms)**: Time spent parsing Arrow IPC metadata and constructing `RecordBatch` objects.
- **Filter Time (ms)**: Time spent executing the predicate (e.g., `trip_miles > 5.0`).
- **Total Time (ms)**: End-to-end wall clock time.
- **Page Faults**: Count of minor and major faults from `/proc/self/stat`.
- **I/O Bytes**: Actual bytes read from block devices from `/proc/self/io`.

### 2.3 Cache Control
- **Cold Cache**: Experiments explicitly drop the OS page cache (`echo 3 > /proc/sys/vm/drop_caches`) between rounds to ensure results reflect disk I/O performance.
- **Warm Cache**: Repeated iterations without dropping caches to measure performance when data is resident in the page cache or user-space cache.

---

## 3. Strategy Overview

The following table summarizes the technical approach of the key strategies evaluated in this research.

| Strategy | I/O Mechanism | Prefetching | Memory Model |
|---|---|---|---|
| `mmap-naive` | `mmap()` | Demand (Kernel) | OS Page Cache |
| `mmap-projection` | `mmap()` | Demand (Kernel) | OS Page Cache (Selective) |
| `odirect-full` | `io_uring` | Eager (Full) | User Aligned Buffer |
| `odirect-column` | `io_uring` | Eager (Selective) | User Aligned Buffer (Sparse) |
| `odirect-pipeline` | `io_uring` + Threads | Overlapped (Full) | SPSC Queue of Buffers |
| `pread` | `pread()` | Synchronous | OS Page Cache |

**Finding 0: Implementation Correctness.** All strategies were verified against a common oracle. For the `trip_miles > 5.0` predicate on the primary dataset, all 11 strategies consistently returned exactly `3,390,038` matched rows.

---

*[Index](./README.md) | Next: [Baseline Experiments →](./02-baseline-experiments.md)*
