# Research Journal

Research journal for the Adaptive I/O project targeting USENIX ATC 2026.

## Entries

| Date | Title | Milestone | Summary |
|------|-------|-----------|---------|
| 2026-04-10 | [Metadata vs I/O Coordination](2026-04-10-metadata-vs-io-coordination.md) | M1 | K-sweep shows 96% gain from metadata cache; I/O coordination untested |
| 2026-04-10 | [The Scheduler Hypothesis Fails](2026-04-10-scheduler-negative-result.md) | M3 | C7 FAIL (1.02×), C8 PASS (1.01×), C9 FAIL (0.97×) — scheduler adds no E2E speedup |
| 2026-04-11 | [Multi-file Hypothesis Falsified](2026-04-11-multifile-hypothesis-falsified.md) | M4 | C11 FAIL (0.985×), C12 FAIL (no trend), C13 FAIL (0.97×) — file count orthogonal to read size |
| 2026-04-15 | [Reviewer Objections: Why Mica, Not Parquet+Extensions](2026-04-15-reviewer-objections-mica-vs-parquet.md) | M6 | 4 structural incompatibilities prevent retrofitting Parquet; equality-filter benchmarks needed for ≥3× |
| 2026-04-18 | [Static Data-Read Policy A/B: Adjacent vs 4 KiB Gap Bridge](2026-04-18-m6-static-gap4k-policy-ab.md) | M6 | Cold A/B: gap+4K helps direct sparse reads, hurts buffered path |
| 2026-04-20 | [Full-Scan Refresh + Paper-Claim Calibration](2026-04-20-m6-fullscan-refresh-and-claim-calibration.md) | M6 | Full-scan Mica/Parquet = 1.24×/1.47× slower (not 2×); three paper claims calibrated |
