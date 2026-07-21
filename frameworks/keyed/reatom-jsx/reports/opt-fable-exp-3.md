# Fable experiment 3 — release child registry arrays after merge

## Algorithm

Fixes the one real regression Exp 2 introduced (22_run-memory +0.52 MB) while
keeping its registry-connect algorithm:

- `registryMerge` now **releases the child's array** (sets the child's
  `registryKey` expando to `undefined`) once its entries are copied into the
  parent list. Intermediate per-`td`/`a` arrays become transient garbage
  instead of living as long as the row.
- Correctness for shared/moved elements is preserved: a later merge of the
  same (already-released) child sees an unstamped node and **tombstones its
  new parent**, so both hosts of a shared element fall back to the walkTree
  connect (the `multiple render shared element` semantics are unchanged).
- The row root's own array was already one-shot (consumed by `connectNode`),
  so after this change no per-row arrays survive connect.

Stack: Steps 1–5 + Exp 1 + Exp 2 + this. reatom commit `6be904d` on
`perf/jsx-connect-opt`. Tests: 101 passed, 2 skipped.

## CPU geo-mean (01–09)

| | Baseline | Step 5 | Exp 1 | Exp 2 | Exp 3 | Δ vs Exp 2 |
|---|---:|---:|---:|---:|---:|---:|
| Reatom | 43.59 | 49.04 | 51.87 | 45.21 | 44.89 | -0.32 |
| Vue | 40.24 | 45.13 | 47.53 | 42.48 | 41.37 | -1.11 |
| R/V ratio (means) | 1.083 | 1.087 | 1.091 | 1.064* | 1.085 | — |
| R/V ratio (medians) | — | 1.092 | — | 1.080 | 1.089 | — |

\* Exp 2's mean ratio was flattered by two warmup-contaminated Vue 09
iterations (52.4/42.0 ms) inflating Vue's geo; median-based ratios are the
robust comparison.

## Reatom vs Vue (means)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs E2 | Δ script vs E2 | drift-adj script |
|---|---:|---:|---:|---:|---:|---:|---:|
| 01_run1k | 69.73 | 56.47 | 26.07 | 13.30 | -0.91 | -1.28 | -0.10 |
| 02_replace1k | 73.09 | 59.93 | 28.95 | 15.90 | -2.69 | -1.35 | +0.33 |
| 03_update10th1k_x16 | 24.57 | 27.17 | 1.61 | 3.56 | -0.92 | -0.37 | -0.34 |
| 04_select1k | 5.73 | 7.07 | 0.49 | 1.54 | +0.08 | -0.08 | -0.08 |
| 05_swap1k | 33.31 | 31.36 | 0.97 | 1.75 | +3.91 | +0.22 | +0.23 |
| 06_remove-one-1k | 23.91 | 26.56 | 0.44 | 2.90 | -0.69 | +0.14 | +0.14 |
| 07_create10k | 661.15 | 565.27 | 215.47 | 100.91 | -40.67 | -14.73 | -2.99 |
| 08_create1k-after1k_x2 | 70.45 | 60.91 | 22.20 | 11.71 | -0.29 | -0.33 | -0.33 |
| 09_clear1k_x8 | 27.80 | 19.04 | 25.11 | 16.30 | -0.71 | -0.51 | +5.21* |
| 21_ready-memory | 1.24 | 1.36 | — | — | +0.05 | — | — |
| 22_run-memory | **6.67** | 4.26 | — | — | **-0.32** | — | — |
| 25_run-clear-memory | 1.68 | 2.05 | — | — | +0.06 | — | — |
| 41_size-uncompressed | 31.60 | 63.70 | — | — | +0.00 | — | — |
| 42_size-compressed | 11.00 | 22.80 | — | — | +0.00 | — | — |
| 43_first-paint | 67.00 | 125.80 | — | — | -9.00 | — | — |

\* Artifact of Exp 2's inflated Vue 09 mean; by medians the 09 R/V script
ratio is **1.54 in both runs** — the clear path is untouched by this change.

## Memory focus

- 22_run-memory: Exp 1 6.47 → Exp 2 6.99 → **Exp 3 6.67 MB**. The release fix
  recovers 0.32 of the 0.52 MB. The remaining ~0.2 MB is the structural cost
  of the registry design: one `registryKey` expando slot on every jsx node
  (~10/row) plus the `enrolled` meta field.
- 25_run-clear-memory stable (1.62–1.68 across runs).

## Verdict — KEPT; registry stack (Exp 1+2+3) is the final candidate

Net position of the full registry stack vs the Exp 1 base (all drift-adjusted):

- 07_create10k script ≈ −9 ms (−5.8 in Exp 2, −3.0 more in Exp 3);
  02/08 ≈ −1.3 each; 01 flat.
- 09_clear kept Exp 1's improvement (R/V script 1.54–1.57 vs Step 5's 1.73).
- Cost: +0.20 MB run-memory, +1.0 KB bundle (uncompressed) / +0.3 KB gzip.
- Median-based R/V geo ratio 1.089 vs Step 5's 1.092 — a small but consistent
  gain; the structural create gap (~2× Vue script, per-row reactive-binding
  construction) remains the dominant term and is out of scope for
  connect-algorithm work.

If run-memory were weighted above CPU, the fallback is the Exp 1 tip
(reatom `9ba37d5`), which keeps the clear-path win at zero memory cost.
