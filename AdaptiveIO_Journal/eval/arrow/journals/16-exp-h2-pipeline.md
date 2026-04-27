← [Part 15](./15-exp-h1-wide-table.md) | [Index](./README.md) | [Next →](./17-exp-h3-cross-file.md)

# §31. Experiment H2: Pipelined I/O + Decode

Methodology: Dataset Uber NYC IPC `fhvhv_tripdata_2021-01.arrow`, 3 columns, threshold=5.0. 4 strategies: vanilla, mmap, iouring (sequential), iouring-pipeline (producer-consumer overlap).

| Window | Batches | vanilla (ms) | mmap (ms) | iouring (ms) | pipeline (ms) | pipeline/iouring |
|---|---|---|---|---|---|---|
| 1h | 1 | 49.1 | 8.5 | 4.1 | 4.8 | 0.85× (thread overhead) |
| 4h | 2 | 100.0 | 16.5 | 8.3 | 7.8 | 1.06× |
| 1d | 4 | 193.8 | 32.3 | 16.4 | 13.4 | 1.22× |
| 1w | 21 | 963.1 | 168.9 | 109.7 | 59.0 | 1.86× |

## Analysis

- **Finding 39: Pipeline is SLOWER for 1-batch queries (4.8ms vs 4.1ms) due to thread spawn + channel overhead.**
- **Finding 40: Pipeline advantage grows with batch count. At 21 batches: 59ms vs 110ms = 1.86× speedup. Producer I/O (57.6ms) overlaps with consumer decode+filter (59ms).**
- **Finding 41: Pipeline achieves near-perfect I/O + CPU overlap when both phases are roughly equal duration.**

← [Part 15](./15-exp-h1-wide-table.md) | [Index](./README.md) | [Next →](./17-exp-h3-cross-file.md)
