# Murally CI Build Times Research

**Date:** 2026-02-06
**Workflow:** `CI (if needed)` (ID: 33654084)
**Repository:** `tactivos/murally`
**Period:** 2026-01-12 to 2026-02-06 (~25 days)

---

## Data Collection

- **Source:** GitHub Actions API via `gh` CLI
- **Total unique runs collected:** 2,794
- **Successful runs with valid duration:** 1,553
- **Failed runs:** 773
- **Cancelled runs:** 468
- **Success rate:** 55.6%

Duration is measured as `updated_at - run_started_at` for completed, successful runs only.

---

## Summary Statistics

### All Successful Runs (n=1,553)

| Metric       | Value     |
|-------------|-----------|
| Min          | 0.43 min  |
| Max          | 56.90 min |
| Mean         | 7.90 min  |
| Std Dev      | 5.92 min  |
| Median (P50) | 8.97 min  |
| P75          | 11.15 min |
| P90          | 13.78 min |
| **P95**      | **18.47 min** |
| P99          | 23.62 min |

### Full CI Builds Only (>=2 min, n=1,107)

446 runs (28.7%) completed in under 2 minutes (mean 0.67 min). These are short-circuit/skip runs where CI determined no build was needed. Excluding those:

| Metric       | Value     |
|-------------|-----------|
| Min          | 2.12 min  |
| Max          | 56.90 min |
| **Mean**     | **10.81 min** |
| **Std Dev**  | **4.44 min**  |
| Median (P50) | 10.10 min |
| P75          | 12.13 min |
| P90          | 15.12 min |
| **P95**      | **21.60 min** |
| P99          | 24.67 min |

---

## Histogram — Full CI Builds (>=2 min)

```
  Duration    | Distribution                                               Count
  ------------|------------------------------------------------------------
   2-  4 min  | ########                                                    48
   4-  6 min  | ########                                                    48
   6-  8 min  | #################                                           97
   8- 10 min  | ############################################################338
  10- 12 min  | ###################################################         288
  12- 14 min  | ##########################                                  147
  14- 16 min  | #######                                                     43
  16- 18 min  | ##                                                          16
  18- 20 min  | ###                                                         22
  20- 22 min  | ##                                                          16
  22- 24 min  | #####                                                       30
  24- 26 min  | #                                                           7
  26- 28 min  |                                                             3
  28- 30 min  |                                                             2
  30- 32 min  |                                                             1
  56- 58 min  |                                                             1 (outlier)
```

**Distribution shape:** The bulk of builds (83%) fall in the 6–14 minute range, with a clear mode at 8–12 minutes. There is a long right tail with occasional builds extending to 20+ minutes.

---

## Daily Average Build Time

| Date       | Avg Build Time | Run Count |
|-----------|---------------|-----------|
| 2026-01-12 | 7.78 min       | 47        |
| 2026-01-13 | 8.65 min       | 86        |
| 2026-01-14 | 8.13 min       | 76        |
| 2026-01-15 | 9.66 min       | 103       |
| 2026-01-16 | 9.01 min       | 90        |
| 2026-01-17 | 10.29 min      | 7         |
| 2026-01-19 | 10.45 min      | 49        |
| 2026-01-20 | 9.66 min       | 104       |
| 2026-01-21 | 8.60 min       | 71        |
| 2026-01-22 | 7.78 min       | 56        |
| 2026-01-23 | 7.84 min       | 69        |
| 2026-01-26 | 8.04 min       | 48        |
| 2026-01-27 | 8.70 min       | 89        |
| 2026-01-28 | 6.48 min       | 58        |
| 2026-01-29 | 5.42 min       | 77        |
| 2026-01-30 | 6.63 min       | 65        |
| 2026-01-31 | 5.51 min       | 5         |
| 2026-02-01 | 7.19 min       | 2         |
| 2026-02-02 | 6.23 min       | 86        |
| 2026-02-03 | 8.25 min       | 98        |
| 2026-02-04 | 6.24 min       | 97        |
| 2026-02-05 | 7.62 min       | 80        |
| 2026-02-06 | 6.32 min       | 90        |

**Trend:** Average build times appear to have decreased slightly from mid-January (~9-10 min) to early February (~6-8 min). Weekends (Jan 17-18, Jan 31-Feb 1) show very low volume.

---

## Run Outcome Breakdown

| Conclusion  | Count | Percentage |
|------------|-------|------------|
| Success     | 1,553 | 55.6%      |
| Failure     | 773   | 27.7%      |
| Cancelled   | 468   | 16.7%      |

---

## Key Findings

1. **Typical build time is ~10 minutes.** The median full build takes 10.10 minutes, with most builds completing in the 8–12 minute range.

2. **P95 is ~21.6 minutes for full builds.** 1 in 20 builds takes over 21 minutes. This is roughly 2x the median, suggesting queue contention or resource constraints on slower runs.

3. **28.7% of CI runs are skipped** (< 2 min), meaning the "CI (if needed)" conditional logic is working — almost a third of pushes don't trigger a full build.

4. **~45% of runs fail or get cancelled.** The 27.7% failure rate and 16.7% cancellation rate are notable and may warrant separate investigation.

5. **The 22-24 min bucket has a small secondary bump (30 runs)** which could indicate a subset of builds hitting a specific bottleneck (e.g., runner queuing, specific test suite timing out and retrying).

6. **One extreme outlier at ~57 minutes** — likely a runner issue or resource starvation event.

7. **Build times trended down ~20-30% from mid-Jan to early Feb**, from averages of 9-10 min to 6-8 min. This could reflect infrastructure improvements, caching improvements, or changes in PR volume/complexity.

---

## Methodology Notes

- Data was collected via GitHub Actions API (`gh api repos/tactivos/murally/actions/workflows/...`) paginated in 100-run batches.
- GitHub API caps results at 1,000 per query window; three overlapping queries were used to cover the full period, then deduplicated by run ID.
- Duration = `updated_at - run_started_at` (excludes queue wait time before the run starts).
- Only `conclusion: success` runs are included in timing analysis. Failed/cancelled runs were excluded since their duration doesn't reflect full build time.
- Runs with duration > 180 minutes were filtered as data anomalies.
