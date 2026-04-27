← [Part 17](./17-exp-h3-cross-file.md) | [Index](./README.md)

# §33. Experiment I: Combined Pipeline + Multi-Threading

Methodology: 6 strategies on same workload (12 monthly Arrow IPC files, 3 columns, threshold=5.0). 3 query sizes: narrow 2h (1 file hit), 1 day (1 file, 4 batches), full year (12 files, 1338 batches, 175M rows).

## §33.1 Performance Results

| Strategy | Narrow 2h (ms) | 1 Day (ms) | Full Year (ms) |
|---|---|---|---|
| iouring-1T-sequential | 5.98 | 21.56 | 8,157.8 |
| iouring-1T-pipeline | 7.75 | 17.36 | 3,990.3 |
| iouring-4T-sequential | 4.55 | 18.77 | 3,125.2 |
| iouring-4T-pipeline | 7.03 | 16.20 | 2,365.4 |
| mmap-4T | 7.81 | 36.52 | 3,503.3 |
| vanilla-4T | 43.11 | 173.62 | 19,683.6 |

## §33.2 Analysis

- **Finding 47: Pipeline doubles single-thread throughput.** iouring-1T-pipeline (3,990ms) is 2.04× faster than iouring-1T-sequential (8,158ms) on the full-year scan. The producer thread prefetches batch N+1 via io_uring while the consumer decodes+filters batch N.
- **Finding 48: Pipeline alone does NOT beat mmap-4T.** iouring-1T-pipeline (3,990ms) is 14% slower than mmap-4T (3,503ms). Four mmap threads with independent page fault streams achieve comparable throughput to a single pipelined io_uring thread.
- **Finding 49: Threading alone provides a narrow edge.** iouring-4T-sequential (3,125ms) is only 1.12× faster than mmap-4T (3,503ms).
- **Finding 50: Pipeline + threading compound to a 48% advantage.** iouring-4T-pipeline (2,365ms) is 1.48× faster than mmap-4T (3,503ms). This is the largest realistic end-to-end margin measured in the study.
- **Finding 51: Pipeline overhead penalizes small queries.** For the narrow 2h query (1 batch), pipeline variants are SLOWER due to thread spawn and channel synchronization costs (~2ms).
- **Finding 52 (verdict update): io_uring's strongest realistic configuration is 4T-pipeline.** The combination of three-column projection, zone-map filtering, io_uring batch reads, and pipelined decoding achieves 8.3× vs vanilla Arrow and 1.48× vs mmap on a 30 GiB scan.

← [Part 17](./17-exp-h3-cross-file.md) | [Index](./README.md)
