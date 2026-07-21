# Opt step 4 — Clear / version-jump teardown fast-path

## Change summary

- On linked-list `clear` and version-jump rebuild, `cleanupNodes` runs on current children **before** `innerHTML = ''`
- Avoids relying solely on MutationObserver removed-subtree walks for bulk teardown

Builds on Steps 1–3. Patch: `reports/step4-clear-teardown.patch`

## CPU geo-mean (01–09)

| | Baseline | Step 3 | Step 4 | Δ vs Step 3 |
|---|---:|---:|---:|---:|
| Reatom | 43.59 | 43.45 | 47.48 | +4.03 |
| Vue | 40.24 | 40.19 | 44.00 | +3.81 |

## Reatom vs Vue

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs Step3 | Δ script vs Step3 |
|---|---:|---:|---:|---:|---:|---:|
| 01_run1k | 70.37 | 57.85 | 27.63 | 14.65 | 3.83 | 2.69 |
| 02_replace1k | 72.99 | 56.89 | 30.29 | 15.43 | 3.09 | 2.17 |
| 03_update10th1k_x16 | 23.39 | 25.54 | 1.33 | 3.33 | 0.19 | 0.04 |
| 04_select1k | 5.49 | 7.17 | 0.55 | 1.66 | 0.28 | 0.10 |
| 05_swap1k | 34.67 | 34.53 | 0.76 | 2.09 | 6.81 | -0.05 |
| 06_remove-one-1k | 26.05 | 29.65 | 0.42 | 3.30 | 3.13 | 0.10 |
| 07_create10k | 706.09 | 622.47 | 237.99 | 110.21 | 55.31 | 31.25 |
| 08_create1k-after1k_x2 | 75.69 | 62.75 | 24.89 | 12.46 | 7.63 | 2.69 |
| 09_clear1k_x8 | 38.49 | 25.64 | 35.29 | 22.61 | 3.78 | 3.71 |
| 21_ready-memory | 1.24 | 1.32 | — | — | 0.07 | — |
| 22_run-memory | 6.41 | 4.25 | — | — | -0.02 | — |
| 25_run-clear-memory | 1.66 | 2.04 | — | — | 0.04 | — |
| 41_size-uncompressed | 30.50 | 63.70 | — | — | 0.10 | — |
| 42_size-compressed | 10.70 | 22.80 | — | — | 0.00 | — |
| 43_first-paint | 72.40 | 121.40 | — | — | 2.00 | — |

## Clear focus

- `09_clear1k_x8`: Reatom total 38.49 / script 35.29 (Step3 34.71 / 31.59); Vue script 22.61

## Verdict

- Reatom geo 47.48 (Step3 43.45, Δ +4.03).
- Clear script Δ vs Step3: +3.71.
