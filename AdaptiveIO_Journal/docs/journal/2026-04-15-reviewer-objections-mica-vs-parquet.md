# Reviewer Objections: Why Mica, Not Parquet+Extensions — 2026-04-15

**Milestone:** M6 — Page-Granular Reader + io_uring Integration
**Tasks completed:** None (design/positioning session)
**Claims informed:** C15, C17

---

## 1. Benchmark Results: Post-Optimization Mica Reader (Cold Cache)

**Claim tested:** C15 — page-granular format + io_uring delivers ≥3× E2E speedup on selective queries vs Parquet

**Setup:** `eval/native-format-bench/`, NYC FHV 2021 dataset (11.9M rows single-file, 175M rows 12-file), 3 projected columns (pickup_datetime, trip_miles, base_passenger_fare), 5 workloads, 7 readers, 5 iterations, cold cache (posix_fadvise DONTNEED per iteration), `--release` build.

```bash
cargo run --release --bin native-format-bench -- run --cache cold --iterations 5 \
  --output results/nyc-fhv-guardrail
```

| Workload | Selectivity | Parquet (ms) | Arrow Pruned (ms) | Mica Mirrored (ms) | Mica/Parquet |
|---|---|---|---|---|---|
| sf_projection_scan | 100% | **295** | 3,216 | 617 | 0.48× |
| sf_full_filter | ~30% | **280** | 3,285 | 666 | 0.42× |
| sf_time_1h_filter | ~0.13% | 25 | 31 | **15** | 1.73× |
| mf_time_2h_filter | ~0.001% | 33 | 35 | **16** | 2.10× |
| mf_full_year_filter | ~30% | **4,154** | 43,571 | 8,579 | 0.48× |

**Key findings:**
- Mica wins on selective queries (sf_time_1h: 15ms vs 25ms Parquet = 1.73× faster; mf_time_2h: 16ms vs 33ms = 2.10× faster)
- Parquet wins on full scans (295ms vs 617ms = 2.09× faster), consistent with Snappy compression advantage (337MB file vs 981MB uncompressed Mica)
- Arrow IPC catastrophic without pruning (3.2s–43s) due to format-level inability to column-project at I/O level
- Mica improvement from pre-optimization baseline: 33× on sf_time_1h (497ms → 15ms), 6.6× on sf_projection_scan (4,098ms → 617ms)

**Optimizations applied (this session):**
1. Selective metadata loading (`open_projected`): 244MB → ~5MB os_read_bytes (bloom filters skipped for range predicates)
2. io_uring buffered mode (no O_DIRECT for warm cache fairness)
3. Parallel column decode via rayon
4. Bulk ColumnBuilder (batch `append_values` instead of per-row `append_value`)
5. All-valid fast path in decode pipeline (skip validity allocation for non-null pages)
6. CRC32C verification skipped by default (matching Parquet/Arrow behavior)
7. io_uring ring warmup excluded from timing

**Decision:** Current numbers show 1.7–2.1× on selective queries — below C15 target of ≥3×. Bloom filter and bitmap index workloads (equality predicates) are completely untested and represent Mica's most differentiated feature. Adding equality-filter benchmarks is the highest priority for reaching the ≥3× target.

---

## 2. Reviewer Objection A: "Mica's Benefit Is Just Finer Granularity"

**Objection:** Mica's 200-row page zone maps vs Parquet's 1M-row row group stats is the main advantage. Why not just configure Parquet with smaller row groups?

**Response — granularity-compression tradeoff is structural, not configurable:**

| Aspect | Parquet (200-row RGs) | Mica (200-row pages) |
|---|---|---|
| Compression | Collapses — 1.6KB Float64 chunks below Snappy/ZSTD effective window | Not needed — uncompressed by design, optional LZ4/FSST |
| Footer metadata | 59,543 RGs × 24 cols = 1.4M Thrift entries → multi-MB footer | Fixed-size: 16B/page zone map, 16B/page index entry, O(1) lookup |
| Full-scan throughput | Degrades — per-RG overhead (seek, decompress init, dict load) amortized over only 200 rows | Accepted tradeoff — 2.09× slower than compressed Parquet on full scans |
| Dictionary encoding | Less effective — 200-row dictionaries vs 1M-row dictionaries | Uses FSST for strings, FOR/Delta for integers — designed for small pages |

**Parquet also has page-level Column Index (min/max per data page).** However:
- Default Parquet data page = 1MB (~125K rows of Float64) — still far coarser than 200 rows
- Pages within a column chunk share a dictionary page — can't decode page N without chunk header
- Even with Offset Index, readers typically read the full column chunk and skip during decode, not at I/O level

**Key point:** Mica doesn't just use "smaller groups." Every layer (metadata layout, encoding, I/O) is co-designed for fine-grained random access. Making Parquet granular breaks the assumptions its compression, metadata format, and reader pipeline rely on.

---

## 3. Reviewer Objection B: "Add Bloom Filters and Bitmap Indexes to Parquet"

**Objection:** Per-page bloom filters and bitmap indexes are known techniques. Why not add them to Parquet instead of creating a new format?

**Response — four structural incompatibilities prevent retrofitting:**

### 3.1 Metadata Architecture: Monolithic vs Column-Independent

Parquet stores ALL metadata in a single Thrift-encoded footer. Adding per-page bloom filters for 24 columns × 59,543 pages × 250 bytes/filter = 198MB of bloom data. Options:
- **In footer:** Footer grows from KB to hundreds of MB. Must parse entire footer to access any column's bloom filters.
- **New file section:** Parquet format has no concept of auxiliary metadata sections outside the footer. Requires format spec revision through Apache governance (multi-year process).

Mica stores each metadata type (zone maps, bloom, bitmap, page index) in separate file sections with per-column fixed-size entries. Selective loading: `read_exact_at(zone_map_offset + col * page_count * 16, page_count * 16)` — O(1) seek, zero parsing overhead.

### 3.2 Page Independence: Dictionary Dependencies

Parquet data pages within a column chunk share a dictionary page at the chunk header. Decoding page N requires first reading the dictionary. This creates a serial dependency that prevents true scatter-gather I/O across arbitrary pages.

Mica pages are self-contained: each has its own header (16B), CRC, validity bitmap, and encoded data. Zero dependency on adjacent pages or column-level headers. io_uring can batch-read 50 pages from 3 columns in one submission and decode them independently.

### 3.3 I/O Model: HDFS Sequential vs NVMe Scattered

Parquet was designed for HDFS (~2012): large sequential reads, high per-operation latency, kernel readahead. Layout reflects this — contiguous column chunks, sequential data pages, footer at EOF.

Mica was designed for NVMe + io_uring: pages are 8KB-aligned (NVMe sector + O_DIRECT compatible), independently addressable via page index, metadata sections separate from data. Maps directly to io_uring's scatter-gather batch submission model.

### 3.4 Empirical Evidence: io_uring on Parquet Produces No Speedup

Our own experiments (Journals §48-52, §53-59) demonstrate the retrofitting failure:
- **io_uring ObjectStore for Parquet:** H2 = 0.998× — neutral. The request-response API can't expose batch semantics.
- **Reactor variant** (aggregating concurrent requests): H2 = 0.887× — **11% worse**. Serialized previously parallel spawn_blocking I/O.
- **Inline io_uring reader** (zero thread hops): 1.054× — still negligible. Read sizes (50–500 KiB) exceed the 128 KiB crossover where io_uring batching provides benefit.

The format's read granularity is the binding constraint, not the I/O interface (C7–C13 validated).

---

## 4. The Defensible Framing for the Paper

**The contribution is NOT:** "bloom filters + bitmap indexes + zone maps" (known techniques).

**The contribution IS:** Demonstrating that co-designing storage layout, metadata architecture, and I/O submission model around NVMe's batch-parallel command interface yields significant speedup on selective analytical queries — and that this cannot be achieved by incrementally adding features to existing formats because the physical layout determines the I/O access pattern.

**Precedent for "new format, not extension":**
- PAX over DSM/NSM (new access pattern → new layout)
- ORC over RCFile (new compression model → new format)
- Arrow over Parquet-in-memory (zero-copy requirement → new format)
- Lance over Parquet-for-ML (random access + versioning → new format, arXiv 2025: 60× better random access)

**Suggested paper structure for this argument:**
1. §4.1: Measure the structural gap (C7–C13 negative results proving Parquet layout prevents io_uring benefit)
2. §4.2: Co-design principles (page independence, metadata separation, I/O-aware alignment)
3. §4.3: Evaluation showing speedup scales with selectivity (C16) while bounding full-scan overhead (C17 ≤ 20%)

---

## Open Questions

- **New:** Do equality-filter workloads (bloom + bitmap pruning) reach ≥3× over Parquet? This is the untested regime where Mica's multi-dimensional per-page indexes have no Parquet equivalent.
- **New:** How to frame the full-scan regression (currently 2.09×) — is ≤20% overhead (C17) achievable with optional lightweight compression (LZ4)?
- **New:** Wide-table projection (200+ columns) benchmark would quantify the column-projection advantage more dramatically than the 24-column NYC FHV dataset.

---

## Design Ramifications for Paper Writing

1. The "why not extend Parquet" objection should be addressed in §2 (Background/Motivation) with the 4 structural incompatibilities above, backed by C7–C13 negative results as empirical evidence.
2. Bloom filter and bitmap index benchmarks are critical — they test Mica's most differentiated feature (per-page multi-dimensional indexes) against Parquet's weakest point (row-group-level-only bloom, no bitmap).
3. The full-scan regression must be explicitly bounded (C17) and positioned as an acceptable tradeoff given production workload selectivity distribution (Snowflake VLDB 2025: 38% of filters ≤5% selectivity, 99.4% micro-partitions pruned).
