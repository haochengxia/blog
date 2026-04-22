---
layout: post
title:  "CacheBench Final Report"
date:   2025-09-01 22:00:00 +0800
categories: jekyll update
---

## Project scope

[CacheBench](https://ucsc-ospo.github.io/project/osre25/harvard/cachebench/) aims to build a benchmarking suite for cache performance evaluation with three concrete goals: reproducing eviction algorithms at scale in libCacheSim, building reusable benchmarking infrastructure for future cache research, and supporting a community-facing leaderboard. We covered over 7000 real-world traces.


## CacheBench artifacts

- [libCacheSim-python](https://github.com/cacheMon/libCacheSim-python) provides Python bindings for libCacheSim, including ratio-based cache sizing, fast simulation, and plugin-based prototyping.
- [cache_dataset](https://github.com/cacheMon/cache_dataset) provides the open trace corpus and tutorials used to reproduce simulations on OracleGeneral-formatted traces.

## Experimental slice

- Source cache size ratios: `0.001`, `0.01`, and `0.1`.
- Public comparison coverage: 7,875 traces at `0.001`, 7,881 traces at `0.01`, and 7,875 traces at `0.1`.
- Public algorithms covered: ARC, FIFO, LFU, LHD, LIRS, LRB, LRU, S3FIFO, Sieve, ThreeLCache, TwoQ, and WTinyLFU.
- Primary metric: request miss ratio.
- Derived metrics used in this report:
  - Mean and median request miss ratio.
  - Delta vs FIFO in percentage points, computed as `FIFO miss ratio - algorithm miss ratio`.
  - Tie-shared best-on-trace share, where ties for the lowest miss ratio split credit evenly.
  - Near-total-miss rate, defined as runs with miss ratio at or above `99.9%`.

The source CSV files include `S4FIFO`, but this report excludes it because it is an internal method and should not appear in the public comparison. All tables and plots below are regenerated after filtering it out.

The report uses delta vs FIFO in percentage points instead of relative percentage improvement because relative ratios become numerically unstable on traces where FIFO already has an extremely small miss ratio.

## Headline results

| Cache ratio | Lowest mean miss ratio | Largest mean delta vs FIFO | Largest tie-shared best-on-trace share |
| --- | --- | --- | --- |
| `0.001` | `ThreeLCache` at `50.23%` | `ThreeLCache` at `+9.29 pp` | `ThreeLCache` at `38.98%` |
| `0.01` | `ARC` at `36.98%` | `ThreeLCache` at `+5.04 pp` | `ThreeLCache` at `26.79%` |
| `0.1` | `LIRS` at `23.55%` | `ThreeLCache` at `+4.37 pp` | `WTinyLFU` at `22.93%` |

The public-only picture is more nuanced than the original mixed report.

- `ARC` is now the strongest public algorithm by absolute miss ratio, with the best overall mean miss ratio at `37.52%` and a mean delta vs FIFO of `+4.78 pp`.
- `ThreeLCache` is the strongest public algorithm when judged by average gain over FIFO. Its overall mean delta vs FIFO is `+6.23 pp`, it beats FIFO on `96.7%` of traces, and it has the largest overall tie-shared best-on-trace share at `28.06%`.
- `LHD`, `LIRS`, and `TwoQ` form a tight second tier on absolute miss ratio, clustered between `38.34%` and `38.48%` mean miss ratio.
- `LRB` remains strong on improvement over FIFO with an overall mean delta of `+4.81 pp`, but its overall mean miss ratio is weaker than the best classical baselines.
- `WTinyLFU` and `LFU` are not robust on this benchmark slice. `WTinyLFU` produces `3,979` near-total-miss runs and `LFU` has the worst overall mean miss ratio at `59.34%`.

## Overview plots

Figure 1 shows the three most useful leaderboard-style metrics side by side after removing the internal method. `ARC` leads on overall absolute miss ratio, while `ThreeLCache` leads on improvement over FIFO across all three cache ratios.

![CacheBench overview heatmaps]({{ '/images/cachebench/cachebench_overview_heatmaps.png' | relative_url }})

Figure 2 answers a different question: how often is an algorithm actually the best choice for a workload? Ties are common enough that they must be handled explicitly even after removing `S4FIFO`. The tied-best rate is about `9.6%` of traces at `0.001`, `5.7%` at `0.01`, and `5.1%` at `0.1`.

![Tie-shared best-on-trace share]({{ '/images/cachebench/cachebench_best_trace_share.png' | relative_url }})

Figure 3 highlights per-trace variability for the leading public algorithms. The median gains stay positive across all six plotted methods, but the spread is widest at the smallest cache ratio, where workload sensitivity is highest.

![Per-trace improvement distribution]({{ '/images/cachebench/cachebench_top_algorithm_distribution.png' | relative_url }})

Figure 4 is the robustness check. This plot matters because best-share or average improvement alone can hide pathological failures. `ThreeLCache` and `LRB` remain stable with only `27` and `20` near-total-miss runs, while `WTinyLFU` and `S3FIFO` fail on a substantial fraction of runs.

![Near-total-miss robustness check]({{ '/images/cachebench/cachebench_near_total_miss_rates.png' | relative_url }})

## Interpretation

This public benchmark slice suggests that CacheBench should not reduce leaderboard quality to a single number.

- `ARC` is the strongest public headline result if the benchmark prioritizes the lowest average miss ratio.
- `ThreeLCache` deserves equal attention because it delivers the largest average improvement relative to FIFO and the largest overall tie-shared best-on-trace share.
- `LHD`, `LIRS`, and `TwoQ` remain valuable classical baselines because they stay competitive on absolute miss ratio without the instability seen in `WTinyLFU` and `LFU`.
- `WTinyLFU` leading the tie-shared best-on-trace share at ratio `0.1` while still performing poorly on robustness is a concrete example of why best-share cannot be used alone.
- The presence of hundreds of tied-best traces per ratio means CacheBench should make tie policy explicit in any published leaderboard.

## Recommendations for the CacheBench leaderboard

CacheBench should report at least four columns for every submitted public algorithm:

1. Mean request miss ratio.
2. Mean delta vs FIFO in percentage points.
3. Tie-shared best-on-trace share.
4. Near-total-miss rate.

Using only one of these metrics would hide important behavior. Absolute miss ratio favors `ARC`, delta vs FIFO favors `ThreeLCache`, and robustness separates stable algorithms from unstable ones. Best-on-trace share is useful, but it must be paired with the other metrics to avoid misleading conclusions.

## Limitations

This report is intentionally scoped to the current baseline CSV slice. It does not yet cover runtime cost, byte miss ratio, warmup sensitivity, or significance testing across workload families. Those should be added in the next iteration of CacheBench so the benchmark suite can evaluate not only miss ratio quality, but also efficiency and operational stability.
