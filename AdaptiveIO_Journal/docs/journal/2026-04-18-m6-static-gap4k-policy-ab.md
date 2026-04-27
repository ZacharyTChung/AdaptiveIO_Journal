# Static Data-Read Policy A/B: Adjacent vs 4 KiB Gap Bridge — 2026-04-18

**Milestone:** M6 — Page-Granular Reader + io_uring Integration
**Tasks completed:** None (design/debug session)
**System:** Linux 6.8.0-106-generic, 1× Intel Xeon Gold 6530 (32 cores / 64 threads)

---

## 1. Experiment Results: Cold Static Policy A/B for Mica v3

**Claim tested:** C15 — Page-granular format + io_uring delivers ≥3× E2E speedup on selective queries (≤5% selectivity) vs Parquet + DataFusion default. This experiment was a diagnostic sub-experiment for M6/M7: choose the static data-read policy for the Mica reader before Exp-16/Exp-18.

**Setup:** `eval/native-format-bench/target/release/profile_mica`, `--release` build, 8 iterations per arm, `--cache cold`, `--secondary-index-strategy sparse`, `--metadata-open-strategy adaptive`, `--profile-scope full-query`, `--data-read-policy static`, `--data-read-max-coalesced-span-bytes 262144`. No `--workload` flags were passed, so `profile_mica` ran its default workload set: `sf_time_1h_filter`, `sf_equality_filter`, `sf_compound_filter`, `sf_in_list_filter`, `sf_time_1h_bitmap_eq_filter`, `sf_point_query`, `mf_time_2h_filter`. The runner emits both `v2` and `v3` profiles; the decision below uses only `version == "v3"` because M6/M7 targets the page-granular Mica v3 reader. Cold-cache mode is the existing `profile_mica` behavior: `drop_manifest_cache()` calls `posix_fadvise(..., POSIX_FADV_DONTNEED)` on each file before every measured iteration.

```bash
cargo build --manifest-path eval/native-format-bench/Cargo.toml --release --bin profile_mica

eval/native-format-bench/target/release/profile_mica \
  --output eval/native-format-bench/results/nyc-fhv-guardrail \
  --iterations 8 \
  --cache cold \
  --secondary-index-strategy sparse \
  --metadata-open-strategy adaptive \
  --profile-scope full-query \
  --data-read-policy static \
  --data-read-max-coalesced-span-bytes 262144 \
  > eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260418_adjacent_vs_gap4k_cold_profile_mica/buffered_adjacent.json

eval/native-format-bench/target/release/profile_mica \
  --output eval/native-format-bench/results/nyc-fhv-guardrail \
  --iterations 8 \
  --cache cold \
  --secondary-index-strategy sparse \
  --metadata-open-strategy adaptive \
  --profile-scope full-query \
  --data-read-policy static \
  --data-read-max-coalesced-span-bytes 262144 \
  --data-read-max-gap-bytes 4096 \
  --data-read-max-extra-cost-bytes 4096 \
  > eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260418_adjacent_vs_gap4k_cold_profile_mica/buffered_gap4k.json

eval/native-format-bench/target/release/profile_mica \
  --output eval/native-format-bench/results/nyc-fhv-guardrail \
  --iterations 8 \
  --cache cold \
  --secondary-index-strategy sparse \
  --metadata-open-strategy adaptive \
  --profile-scope full-query \
  --use-direct-io \
  --data-read-policy static \
  --data-read-max-coalesced-span-bytes 262144 \
  > eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260418_adjacent_vs_gap4k_cold_profile_mica/direct_adjacent.json

eval/native-format-bench/target/release/profile_mica \
  --output eval/native-format-bench/results/nyc-fhv-guardrail \
  --iterations 8 \
  --cache cold \
  --secondary-index-strategy sparse \
  --metadata-open-strategy adaptive \
  --profile-scope full-query \
  --use-direct-io \
  --data-read-policy static \
  --data-read-max-coalesced-span-bytes 262144 \
  --data-read-max-gap-bytes 4096 \
  --data-read-max-extra-cost-bytes 4096 \
  > eval/native-format-bench/results/nyc-fhv-guardrail/runs/native_format_bench_20260418_adjacent_vs_gap4k_cold_profile_mica/direct_gap4k.json
```

**Buffered I/O (`use_direct_io = false`, v3 only):**

| Workload | Adjacent total (ms) | Gap+4K total (ms) | Delta total | Adjacent read/decode (ms) | Gap+4K read/decode (ms) | Reads (A → G) | Bytes (A → G) |
|--------|--------|--------|--------|--------|--------|--------|--------|
| `sf_time_1h_filter` | 5.522 ms | 5.361 ms | -2.9% | 1.386 ms | 1.278 ms | 3 → 3 | 405,750 B → 405,750 B |
| `sf_equality_filter` | 53.452 ms | 54.511 ms | +2.0% | 41.452 ms | 41.944 ms | 3,796 → 3,412 | 4,845,861 B → 5,437,675 B |
| `sf_compound_filter` | 7.116 ms | 7.606 ms | +6.9% | 2.418 ms | 2.416 ms | 4 → 4 | 444,020 B → 444,020 B |
| `sf_in_list_filter` | 140.344 ms | 155.445 ms | +10.8% | 117.676 ms | 127.793 ms | 12,576 → 8,917 | 17,091,032 B → 23,803,709 B |
| `sf_time_1h_bitmap_eq_filter` | 5.130 ms | 6.028 ms | +17.5% | 1.208 ms | 1.278 ms | 57 → 14 | 99,460 B → 155,447 B |
| `sf_point_query` | 2.004 ms | 2.060 ms | +2.8% | 0.607 ms | 0.497 ms | 18 → 14 | 24,342 B → 34,931 B |
| `mf_time_2h_filter` | 6.893 ms | 7.278 ms | +5.6% | 2.949 ms | 2.468 ms | 5 → 5 | 1,144,149 B → 1,144,149 B |

**Direct I/O (`use_direct_io = true`, v3 only):**

| Workload | Adjacent total (ms) | Gap+4K total (ms) | Delta total | Adjacent read/decode (ms) | Gap+4K read/decode (ms) | Reads (A → G) | Bytes (A → G) |
|--------|--------|--------|--------|--------|--------|--------|--------|
| `sf_time_1h_filter` | 4.653 ms | 4.629 ms | -0.5% | 0.981 ms | 0.971 ms | 3 → 3 | 425,984 B → 425,984 B |
| `sf_equality_filter` | 46.604 ms | 44.361 ms | -4.8% | 36.748 ms | 34.457 ms | 3,796 → 3,412 | 20,303,872 B → 19,353,600 B |
| `sf_compound_filter` | 5.600 ms | 5.564 ms | -0.6% | 2.018 ms | 2.004 ms | 4 → 4 | 466,944 B → 466,944 B |
| `sf_in_list_filter` | 149.310 ms | 122.212 ms | -18.1% | 122.986 ms | 96.965 ms | 12,576 → 8,917 | 68,546,560 B → 60,231,680 B |
| `sf_time_1h_bitmap_eq_filter` | 5.518 ms | 5.242 ms | -5.0% | 1.617 ms | 1.187 ms | 57 → 14 | 348,160 B → 217,088 B |
| `sf_point_query` | 2.100 ms | 2.226 ms | +6.0% | 0.463 ms | 0.491 ms | 18 → 14 | 94,208 B → 90,112 B |
| `mf_time_2h_filter` | 4.958 ms | 6.119 ms | +23.4% | 2.098 ms | 2.865 ms | 5 → 5 | 1,163,264 B → 1,163,264 B |

**Key findings:**
- Buffered cold runs do **not** support replacing adjacent-only with `gap=4 KiB`. The sparse equality/bitmap workloads all regressed despite fewer reads: `sf_equality_filter` `53.452 ms → 54.511 ms` (+2.0%) with `+12.2%` more bytes, `sf_in_list_filter` `140.344 ms → 155.445 ms` (+10.8%) with `+39.3%` more bytes, and `sf_time_1h_bitmap_eq_filter` `5.130 ms → 6.028 ms` (+17.5%) with `+56.3%` more bytes.
- Direct cold runs show the opposite pattern on sparse workloads because bridging small gaps reduces both request count and aligned physical bytes. `sf_equality_filter` improved from `46.604 ms` to `44.361 ms` (-4.8%, `-4.7%` bytes), `sf_in_list_filter` improved from `149.310 ms` to `122.212 ms` (-18.1%, `-12.1%` bytes), and `sf_time_1h_bitmap_eq_filter` improved from `5.518 ms` to `5.242 ms` (-5.0%, `-37.6%` bytes).
- The contiguous/range-clustered workloads (`sf_time_1h_filter`, `sf_compound_filter`, `mf_time_2h_filter`) were already fully merged under adjacent-only. Their read counts and bytes stayed unchanged, so the small timing differences there are measurement noise, not policy signal.
- `sf_point_query` is too small to justify bridging as a default. Even when the read count falls (`18 → 14`), total time changed by only `+2.8%` buffered and `+6.0%` direct.

**Decision:** Do **not** replace adjacent-only with `gap=4 KiB` as the universal static default. Keep adjacent-only for buffered/regular-file runs. Preserve `gap=4 KiB` only as a direct-I/O-specific policy/candidate for sparse workloads.

---

## 2. Static Policy Decision for M6/M7

**Was:** The static default was adjacent-only, but the open question from this session was whether a small positive-gap bridge (`max_gap = 4 KiB`, `max_extra_cost = 4 KiB`) should replace it globally.

**Now:** Adjacent-only remains the buffered static policy. `gap=4 KiB` is retained only for direct-I/O experiments and direct-I/O-aware selector work; it is not a universal replacement.

**Why:** Buffered and direct runs sit in different byte-cost regimes.
- Buffered mode penalizes gap bridging on sparse workloads because fewer requests come with more logical bytes. The worst case in this A/B was `sf_time_1h_bitmap_eq_filter`, where read count collapsed `57 → 14` but physical bytes grew `99,460 B → 155,447 B`, and total time regressed `+17.5%`.
- Direct mode can benefit because 4 KiB bridging often lowers aligned physical bytes as well as request count. The strongest win was `sf_in_list_filter`, where total time improved `149.310 ms → 122.212 ms` (-18.1%) while bytes fell `68,546,560 B → 60,231,680 B` (-12.1%).
- The policy should therefore be mode-specific, not global. A single default would be wrong for at least one of the two execution modes.

**Impact:** M6 reader bring-up and M7 selective-query evaluation should keep separate buffered and direct baselines. Any future adaptive candidate set must treat buffered and direct coalescing as different problems, not one shared ranking.

## Open Questions

- **Answered:** Should `adjacent + 4 KiB gap` replace adjacent-only universally? → No. It hurts buffered cold sparse workloads and helps direct cold sparse workloads.
- **New:** Once M6 reader integration lands, should direct I/O expose a different static default from buffered I/O, or should the split remain an adaptive-selector-only choice?

## Design Ramifications for M7

1. Buffered M7 comparisons should continue to use adjacent-only as the static baseline; swapping in `gap=4 KiB` would make the sparse buffered path look worse before the actual Mica-vs-Parquet comparison.
2. Direct-I/O M7 experiments should include `gap=4 KiB` as an explicit arm because it is already a confirmed win on the current sparse equality/bitmap shapes.
