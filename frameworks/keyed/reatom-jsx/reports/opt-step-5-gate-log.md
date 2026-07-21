# Opt step 5 — Gate `jsxEvent` / `log.label` behind DEBUG

## Change summary

- `jsxEvent` calls `log.label` only when `peek(DEBUG)` is true
- `on:` handlers use `._${key}` names when DEBUG is off (skips `eventActionName` string work)

Builds on Steps 1–4. Patch: `reports/step5-gate-log.patch`

## CPU geo-mean progression

| Step | Reatom | Vue |
|---|---:|---:|
| Baseline npm 1001.3.0 | 43.59 | 40.24 |
| 1 eager connect | 49.72 | 47.41 |
| 2 peek props | 44.02 | 39.76 |
| 3 slim meta | 43.45 | 40.19 |
| 4 clear teardown | 47.48 | 44.00 |
| 5 gate log | 49.04 | 45.13 |

## Reatom vs Vue (Step 5)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ size vs Step4 |
|---|---:|---:|---:|---:|---:|
| 01_run1k | 69.99 | 56.68 | 27.10 | 14.21 | -0.39 |
| 02_replace1k | 78.70 | 62.03 | 34.45 | 18.09 | 5.71 |
| 03_update10th1k_x16 | 26.79 | 30.84 | 1.95 | 4.29 | 3.39 |
| 04_select1k | 5.97 | 7.56 | 0.57 | 1.76 | 0.48 |
| 05_swap1k | 32.71 | 34.26 | 0.79 | 2.09 | -1.96 |
| 06_remove-one-1k | 26.95 | 29.39 | 0.40 | 3.45 | 0.90 |
| 07_create10k | 695.55 | 609.25 | 240.86 | 107.66 | -10.53 |
| 08_create1k-after1k_x2 | 74.67 | 61.84 | 23.93 | 12.25 | -1.02 |
| 09_clear1k_x8 | 40.67 | 24.97 | 37.67 | 21.83 | 2.19 |
| 21_ready-memory | 1.21 | 1.37 | — | — | -0.03 |
| 22_run-memory | 6.44 | 4.26 | — | — | 0.03 |
| 25_run-clear-memory | 1.72 | 2.05 | — | — | 0.06 |
| 41_size-uncompressed | 30.50 | 63.70 | — | — | 0.00 |
| 42_size-compressed | 10.70 | 22.80 | — | — | 0.00 |
| 43_first-paint | 72.20 | 124.90 | — | — | -0.20 |

## Size focus

- uncompressed: 30.50 (Step4 30.50)
- compressed: 10.70 (Step4 10.70)

## Verdict

- Reatom geo 49.04 vs Step4 47.48 (+1.56); vs baseline +5.45.
- Runtime DEBUG gate does not remove `log` from the bundle (needs compile-time DCE); size delta is modest.
