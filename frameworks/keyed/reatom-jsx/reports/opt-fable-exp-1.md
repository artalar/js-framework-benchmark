# Fable experiment 1 — MO removal-walk skip for eagerly torn-down roots

## Algorithm

Fixes the Step-4 regression identified in `opt-fable-validation.md` (double
teardown walk) while keeping the eager-teardown half:

- Module-level `eagerlyDetached = new WeakSet<Node>()`.
- `teardownLinkedListChildren` (linked-list `clear` / version-jump rebuild)
  still runs `cleanupNodes` synchronously on the container children, then marks
  each root in the WeakSet **before** `innerHTML = ''`.
- `processMutations` consumes the mark per removal record
  (`eagerlyDetached.delete(removedNode)`) and skips the removed-subtree walk
  entirely for marked roots. MutationRecords arrive in mutation order, so the
  first (eager) removal record consumes the mark and any later removal of the
  same node is processed normally.
- Foreign/adopted removals and `remove`/`removeMany` rows (not eagerly torn
  down) keep the full MO cleanup path.

Result: a linked-list clear now pays exactly **one** teardown walk (Step 4
paid two: eager `cleanupNodes` + the MO removed-subtree re-walk over ~11k
released metas per 1k rows).

Stack: Steps 1–5 + this change. reatom commit `9ba37d5` on
`perf/jsx-connect-opt`. Tests: 101 passed, 2 skipped.

## CPU geo-mean (01–09)

| | Baseline 1001.3.0 | Step 5 | Exp 1 | Δ vs Step 5 |
|---|---:|---:|---:|---:|
| Reatom | 43.59 | 49.04 | 51.87 | +2.83 |
| Vue | 40.24 | 45.13 | 47.53 | +2.40 |
| R/V ratio | 1.083 | 1.087 | 1.091 | — |

Machine drifted slower again (Vue +2.40 with unchanged code); drift-adjusted
Reatom geo residual vs Step 5: **+0.22 ≈ 0**.

## Reatom vs Vue (means)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs S5 | Δ script vs S5 | drift-adj script |
|---|---:|---:|---:|---:|---:|---:|---:|
| 01_run1k | 67.42 | 62.84 | 25.35 | 15.36 | -2.57 | -1.75 | -3.95 |
| 02_replace1k | 82.69 | 64.23 | 35.97 | 19.14 | +3.99 | +1.52 | -0.49 |
| 03_update10th1k_x16 | 32.94 | 31.49 | 2.39 | 4.27 | +6.15 | +0.44 | +0.45 |
| 04_select1k | 6.17 | 7.82 | 0.66 | 1.72 | +0.20 | +0.09 | +0.10 |
| 05_swap1k | 37.01 | 35.09 | 0.98 | 2.06 | +4.29 | +0.19 | +0.20 |
| 06_remove-one-1k | 27.02 | 32.01 | 0.49 | 3.67 | +0.07 | +0.09 | +0.06 |
| 07_create10k | 714.35 | 626.36 | 246.26 | 110.94 | +18.79 | +5.40 | -1.94 |
| 08_create1k-after1k_x2 | 82.93 | 65.77 | 26.69 | 13.13 | +8.26 | +2.75 | +1.03 |
| 09_clear1k_x8 | 40.51 | 26.93 | 37.50 | 23.84 | -0.17 | -0.17 | **-3.63** |
| 21_ready-memory | 1.21 | 1.37 | — | — | -0.00 | — | — |
| 22_run-memory | 6.47 | 4.26 | — | — | +0.02 | — | — |
| 25_run-clear-memory | 1.64 | 2.05 | — | — | -0.08 | — | — |
| 41_size-uncompressed | 30.60 | 63.70 | — | — | +0.10 | — | — |
| 42_size-compressed | 10.70 | 22.80 | — | — | +0.00 | — | — |
| 43_first-paint | 75.10 | 123.30 | — | — | +2.90 | — | — |

(drift-adj script = observed − Step5-Reatom × within-bench Vue script ratio)

## Create family + clear focus (script, R/V ratio)

| Bench | Step5 R/V script | Exp1 R/V script |
|---|---:|---:|
| 01_run1k | 1.91 | 1.65 |
| 02_replace1k | 1.90 | 1.88 |
| 07_create10k | 2.24 | 2.22 |
| 08_create1k-after1k_x2 | 1.95 | 2.03 |
| 09_clear1k_x8 | **1.73** | **1.57** |

`09_clear` R/V ratio trend: baseline 1.39 → S3 1.49 → S4 1.50 → S5 1.63 →
**Exp1 1.57 (script ratio back to the S3/S4 band)**. The clear path now does
the same single walk as pre-Step-4, just synchronously, so ratios return to
that band rather than below it.

## Verdict — KEPT as new base

- The targeted bench improved on the only mechanism the change touches:
  09 script −3.63 ms drift-adjusted; 02_replace (rebuild path) −0.49.
- Everything else within noise (01's −3.95 has no mechanism — Vue's 01 script
  15.36 is its max across all seven runs, inflating the residual).
- Geo residual ≈ 0 vs Step 5, and the change strictly removes work, so Exp 1
  becomes the base for Experiment 2.
