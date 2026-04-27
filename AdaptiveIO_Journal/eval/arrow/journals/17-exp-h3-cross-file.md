← [Part 16](./16-exp-h2-pipeline.md) | [Index](./README.md) | [Part 18 →](./18-exp-i-combined.md)

# §32. Experiment H3: Cross-File Multi-Threaded Query

Methodology: Dataset 12 Uber NYC IPC files (~30 GiB), 3 columns, threshold=5.0. 3 strategies: vanilla, mmap, iouring — round-robin file distribution, barrier+channel timing.

## §32.1 Narrow window (2h, 1/12 files hit)

| threads | vanilla (ms) | mmap (ms) | iouring (ms) | vanilla/iouring | mmap/iouring |
|---|---|---|---|---|---|
| 1 | 43.4 | 7.8 | 4.3 | 10.1× | 1.81× |
| 12 | 41.3 | 7.8 | 4.3 | 9.6× | 1.82× |

## §32.2 Wide window (full year, 12/12 files hit, 1338 batches, 175M rows)

| threads | vanilla (ms) | mmap (ms) | iouring (ms) | vanilla/iouring | mmap/iouring |
|---|---|---|---|---|---|
| 1 | 48,794 | 10,186 | 5,584 | 8.7× | 1.82× |
| 2 | 28,488 | 5,380 | 3,433 | 8.3× | 1.57× |
| 4 | 19,558 | 3,494 | 3,056 | 6.4× | 1.14× |
| 8 | 18,745 | 3,357 | 2,881 | 6.5× | 1.17× |
| 12 | 18,727 | 3,351 | 3,092 | 6.1× | 1.08× |

## §32.3 Analysis

- **Finding 42: Narrow window — thread scaling irrelevant (only 1 file has data). Results match Exp G (iouring 1.8× vs mmap).**
- **Finding 43: Wide window 1 thread — iouring 5.6s vs mmap 10.2s vs vanilla 48.8s. Column projection dominates (5.2 GiB vs 31.7 GiB).**
- **Finding 44: At 12 threads, mmap nearly catches iouring (3.4s vs 3.1s, only 1.08×) because decode+filter time (1.6s) dominates and parallelizes well.**
- **Finding 45: iouring scales modestly (5.6s→3.1s, 1.8× with 12T) because it's already fast at 1 thread. mmap scales better (10.2s→3.4s, 3.0× with 12T).**
- **Finding 46: vanilla reads 31.7 GiB constant regardless of threads. mmap 5.2 GiB. iouring 4.2 GiB.**

← [Part 16](./16-exp-h2-pipeline.md) | [Index](./README.md) | [Part 18 →](./18-exp-i-combined.md)
