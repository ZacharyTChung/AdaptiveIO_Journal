← [Part 14](./14-exp-g-e2e-comparison.md) | [Index](./README.md) | [Next →](./16-exp-h2-pipeline.md)

# §30. Experiment H1: Wide-Table Small-Buffer Projection

Methodology: Dataset `wide_200col.arrow` (200 Float64 columns, 100 batches, 8192 rows/batch, 64 KiB per column buffer). Zone-map: 1/100 batches hit (1h time window on `ts` column). 3 strategies: vanilla Arrow FileReader, mmap, iouring — all end-to-end with decode+filter.

| Projected Cols | vanilla (ms) | mmap (ms) | iouring (ms) | vanilla/iouring | mmap/iouring |
|---|---|---|---|---|---|
| 5 | 25.9 | 1.1 | 0.9 | 29.7× | 1.30× |
| 10 | 23.8 | 1.7 | 1.3 | 18.5× | 1.32× |
| 20 | 26.8 | 3.6 | 2.4 | 11.0× | 1.49× |
| 50 | 26.2 | 7.8 | 5.2 | 5.0× | 1.49× |

## Analysis

- **Finding 36: vanilla constant ~26ms (reads ALL 200 cols regardless of projection).** Lacks I/O-level column projection.
- **Finding 37: iouring 1.3-1.5× faster than mmap at 64 KiB buffers — modest advantage at this buffer size (just below crossover).**
- **Finding 38: At 5 cols, iouring reads 0.3 MiB vs vanilla 13.4 MiB — 44× less I/O. Column projection is the dominant factor.**

← [Part 14](./14-exp-g-e2e-comparison.md) | [Index](./README.md) | [Next →](./16-exp-h2-pipeline.md)
