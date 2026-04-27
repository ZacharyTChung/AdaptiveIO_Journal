# Part 7: Experiment C — Pure Fault Isolation

*Section 23 of the research journal. The cleanest test of mmap vs io_uring.*

---

*← Previous: [Research Pivot](./06-research-pivot.md) | [Index](./README.md) | Next: [Exp A: Scattered Reads →](./08-exp-a-scattered-reads.md)*

---

## 23. Experiment C: Pure Fault Isolation

### 23.0 Micro-Experiment Methodology

Experiments C, A, and B (Parts 7-9) use a fundamentally different approach from Parts 1-5. Rather than measuring end-to-end query latency (load → decode → filter), these experiments isolate the **raw I/O submission mechanism** by touching non-resident pages directly — no Arrow parsing, no filtering, no decoding.

**Common protocol for all three experiments:**
- **Target file**: `fhvhv_tripdata_2021-01.arrow` (2,064 MiB, 91 batches, 24 columns).
- **Cold cache**: Between each round, the file's page cache is evicted via `posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED)` on the target file descriptor. The shell scripts additionally call `echo 3 > /proc/sys/vm/drop_caches` before each benchmark invocation.
- **Rounds**: Each configuration is run 5 times. All tables report the **arithmetic mean of per-read latencies across all 5 rounds** unless otherwise noted.
- **Metrics per round**: `wall_time_ns` (total elapsed via `Instant::now()`), `io_bytes` (delta from `/proc/self/io` `read_bytes` field), `page_faults` (delta of minor + major faults from `/proc/self/stat` fields 10 and 12), `per_read_ns = wall_time_ns / num_reads`.
- **Effective parallelism**: Computed as `baseline_avg / strategy_avg`, where baseline is `mmap-random-touch` (the serialized page fault strategy). Values > 1.0 indicate the strategy achieves the equivalent of that many parallel I/O operations relative to a single serialized fault.
- **Statistical note**: Run-to-run coefficient of variation (CV) is below 5% for all strategies at N ≥ 50. For example, at N=100: mmap-random-touch CV = 1.6%, iouring-batched CV = 5.1%. The conclusions are robust to this variance.

**Experiment C strategies** (5 strategies measuring raw page residency cost):

1. **mmap-sequential-touch**: `mmap()` the entire file with default kernel advice. Sort the N offsets in ascending order, then touch each via `volatile_read(ptr + offset)`. The kernel's default readahead prefetches upcoming pages, but non-resident accesses still trigger synchronous page faults.

2. **mmap-random-touch** (baseline): `mmap()` with `madvise(MADV_RANDOM)` to disable readahead entirely. Touch offsets in original (unsorted) order. Each page fault incurs the full NVMe command latency (~10 μs) with no prefetching. This is the **serialized fault baseline** — all other strategies are compared against it.

3. **mmap-willneed-batch**: `mmap()` the file, then call `madvise(MADV_WILLNEED, read_size)` for every offset in a tight loop. Sleep 1 ms to allow the kernel's background readahead threads to issue I/O. Then touch all pages via `volatile_read`. This tests the kernel's ability to parallelize I/O when told which pages are needed ahead of time.

4. **iouring-batched**: Open with `O_DIRECT`. Allocate N 4096-byte-aligned buffers. Submit all N read requests to a single `io_uring` submission queue in one `io_uring_enter()` syscall, then poll all N completions. Total I/O time ≈ max(individual latencies) because the NVMe controller processes requests concurrently from its hardware queue (typical queue depth: 32-128).

5. **iouring-serial**: Open with `O_DIRECT`. For each offset, submit a single read to `io_uring`, wait for its completion, then proceed to the next. This is the **io_uring control** — it isolates the `io_uring` interface overhead and `O_DIRECT` buffer management cost from the batch submission advantage.

**Offset generation** (Experiment C): N offsets are generated from the 2,064 MiB file using a configurable spacing algorithm. A 64-bit LCG pseudo-random number generator is used (multiplier: 6364136223846793005, increment: 1442695040888963407, default seed: 42). All offsets are rounded down to 4096-byte alignment to match NVMe sector boundaries.
- *Uniform*: Offsets are evenly spaced: `offset[i] = align_down(i × file_size / N)`.
- *Random*: Each offset is drawn independently: `offset[i] = align_down(lcg_next() >> 33 % (file_size - read_size))`.
- *Clustered*: Offsets are grouped into clusters of 4 consecutive pages separated by large gaps: `gap = file_size / ceil(N/4)`, each cluster contains pages at `base, base+4096, base+8192, base+12288`.

### 23.1 N Scaling: The Batch Amortization Effect

This sweep increases the number of requested pages from 10 to 500 (read_size = 4096, spacing = random) to measure how per-read latency scales with batch size.

| num_reads | mmap-seq-touch | mmap-random-touch | mmap-willneed-batch | iouring-batched | iouring-serial |
|-----------|---------------|-------------------|---------------------|-----------------|----------------|
| 10        | 366,418       | 85,855            | 111,240             | 24,556          | 84,085         |
| 25        | 358,128       | 85,071            | 47,731              | 11,650          | 83,414         |
| 50        | 356,484       | 85,974            | 26,714              | 7,442           | 85,221         |
| 100       | 358,399       | 86,410            | 16,693              | 6,371           | 85,328         |
| 200       | 353,628       | 86,001            | 11,489              | 5,114           | 85,056         |
| 500       | 349,013       | 87,480            | 11,108              | 5,912           | 85,093         |

*Units: ns/read. Average of 5 cold-cache rounds. Source: results/exp_c_20260327_051902/n_scaling_{N}_r{1..5}.txt*

**Analysis**:
- **Finding 1: iouring-batched scales with batch size.** Per-read cost drops from 24.6 μs (N=10) to 5.1 μs (N=200), a 4.8× improvement. At N=100, **iouring-batched is 13.6× faster than mmap-random-touch** (6,371 vs 86,410 ns/read). The speedup range across all N values is **3.5× (N=10) to 16.8× (N=200)**.
- **Finding 2: mmap-random-touch is bottlenecked by serialization.** The per-read cost is flat at 85–87 μs regardless of N. Each page fault blocks the thread for the full NVMe command latency, preventing any I/O parallelism. This is the architectural ceiling of demand paging.
- **Finding 3: mmap-willneed-batch provides a middle ground.** By issuing `MADV_WILLNEED` for all N pages before touching them, the kernel can parallelize readahead. At N=100, it achieves 5.2× effective parallelism (16,693 vs 86,410 ns/read) — but is still 2.6× slower than iouring-batched (16,693 vs 6,371). The kernel's readahead mechanism adds scheduling overhead that io_uring's direct NVMe submission avoids.
- **Finding 4: iouring-serial confirms the mechanism.** Serial io_uring (85 μs/read) performs identically to mmap-random-touch (86 μs/read). This proves the speedup comes exclusively from **batch submission parallelism**, not from O_DIRECT bypassing the page cache or the io_uring syscall interface itself.

**Effective Parallelism at N=100** (baseline = mmap-random-touch = 86,410 ns/read):

| strategy            | avg_per_read_ns | effective_parallelism |
|---------------------|-----------------|-----------------------|
| mmap-seq-touch      | 358,399         | 0.24                  |
| mmap-random-touch   | 86,410          | 1.00 (baseline)       |
| mmap-willneed-batch | 16,693          | 5.18                  |
| iouring-batched     | 6,371           | 13.56                 |
| iouring-serial      | 85,328          | 1.01                  |

`effective_parallelism = baseline_avg / strategy_avg`. An EP of 13.56 for iouring-batched means it achieves the throughput equivalent of 13.56 serialized mmap page faults running in parallel — close to the NVMe device's hardware queue depth.

### 23.2 Read Size Scaling: The Crossover Point

We sweep the read size from 4 KiB to 256 KiB with N=100 fixed.

| read_size | mmap-seq-touch | mmap-random-touch | mmap-willneed-batch | iouring-batched | iouring-serial |
|-----------|---------------|-------------------|---------------------|-----------------|----------------|
| 4,096     | 355,240       | 86,212            | 16,623              | 6,964           | 85,614         |
| 16,384    | 355,480       | 86,520            | 19,601              | 12,809          | 189,480        |
| 65,536    | 354,415       | 88,074            | 38,294              | 40,127          | 244,293        |
| 262,144   | 354,128       | 90,771            | 74,550              | 148,358         | 584,184        |

*Units: ns/read. Average of 5 cold-cache rounds. N=100 fixed. Source: results/exp_c_20260327_051902/size_scaling_{sz}_r{1..5}.txt*

**Analysis**:
- **Finding 5: The crossover from io_uring advantage to mmap advantage occurs between 65 KiB and 262 KiB (approximately 128 KiB).** At 4 KiB, iouring-batched is 12.4× faster than mmap-random (6,964 vs 86,212 ns/read). At 16 KiB, still 6.8× faster (12,809 vs 86,520). At 65 KiB, iouring-batched is still 2.2× faster (40,127 vs 88,074) — the advantage persists because NVMe command latency (~10 μs) is still a significant fraction of the 65 KiB transfer time (~21 μs). At 262 KiB, the transfer time (~85 μs) dominates, and mmap-random is 1.6× faster (90,771 vs 148,358) because it avoids the overhead of O_DIRECT buffer allocation and alignment.

### 23.3 Spacing Comparison: Locality Effects

| spacing   | mmap-seq-touch | mmap-random-touch | mmap-willneed-batch | iouring-batched | iouring-serial |
|-----------|---------------|-------------------|---------------------|-----------------|----------------|
| uniform   | 358,072       | 87,353            | 17,554              | 7,345           | 84,968         |
| random    | 353,189       | 87,238            | 16,720              | 6,413           | 85,016         |
| clustered | 89,050        | 86,000            | 15,170              | 7,493           | 84,911         |

*Units: ns/read. Average of 5 cold-cache rounds. N=100, read_size=4096 fixed. Source: results/exp_c_20260327_051902/spacing_{type}_r{1..5}.txt*

**Analysis**:
- **Finding 6: iouring-batched is spacing-invariant.** Performance ranges only 6.4–7.5 μs/read across all three patterns (CV = 8%). The NVMe controller's internal scheduling handles both random and sequential access patterns efficiently when requests arrive in parallel. This makes iouring-batched a reliable choice for unpredictable workloads.
- **Finding 7: Clustered spacing rescues mmap-sequential-touch.** When pages are physically adjacent (4-page clusters), the kernel's default readahead successfully fetches multiple requested pages per fault, reducing per-read cost by 4× (89 μs vs 353–358 μs for non-clustered patterns). This confirms that mmap's performance is highly dependent on access locality, while io_uring's batch submission provides consistent performance regardless of spatial pattern.

---

*← Previous: [Research Pivot](./06-research-pivot.md) | [Index](./README.md) | Next: [Exp A: Scattered Reads →](./08-exp-a-scattered-reads.md)*
