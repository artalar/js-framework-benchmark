# Opt step 2 — Peek/apply initial reactive props

## Change summary

In `setProp` for `class`/`className` (object/function), atom props, and function props:

- Create the binding (`reatomClassName` / `computed`) at render time
- `peek` + apply the first value onto the DOM during create
- On `subscribe`, skip writes when `Object.is` matches the already-applied state (still clears `data-reatom-error` on equal re-emit after a prop error)

Builds on Step 1 eager connect. Patch: `reports/step2-peek-initial-props.patch`

## CPU geo-mean (01–09)

| | Baseline | Step 1 | Step 2 | Δ vs Step 1 |
|---|---:|---:|---:|---:|
| Reatom | 43.59 | 49.72 | 44.02 | -5.70 |
| Vue | 40.24 | 47.41 | 39.76 | -7.64 |

## Reatom vs Vue (means)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs Step1 | Δ script vs Step1 |
|---|---:|---:|---:|---:|---:|---:|
| 01_run1k | 68.04 | 54.61 | 26.00 | 13.08 | -3.72 | -2.09 |
| 02_replace1k | 71.68 | 57.86 | 29.68 | 15.51 | -6.66 | -3.36 |
| 03_update10th1k_x16 | 23.09 | 26.38 | 1.31 | 3.47 | -2.06 | -0.71 |
| 04_select1k | 6.34 | 6.50 | 0.62 | 1.36 | -0.19 | -0.08 |
| 05_swap1k | 29.23 | 29.32 | 0.87 | 1.53 | -7.38 | 0.05 |
| 06_remove-one-1k | 23.39 | 25.53 | 0.33 | 2.78 | -4.46 | -0.15 |
| 07_create10k | 667.61 | 561.99 | 212.65 | 99.86 | -69.36 | -24.99 |
| 08_create1k-after1k_x2 | 72.03 | 57.58 | 23.93 | 11.25 | -6.11 | -0.50 |
| 09_clear1k_x8 | 26.42 | 18.92 | 23.82 | 16.03 | -7.84 | -7.59 |
| 21_ready-memory | 1.21 | 1.37 | — | — | 0.00 | — |
| 22_run-memory | 6.46 | 4.21 | — | — | 0.06 | — |
| 25_run-clear-memory | 1.63 | 2.04 | — | — | 0.01 | — |
| 41_size-uncompressed | 30.10 | 63.70 | — | — | 0.20 | — |
| 42_size-compressed | 10.60 | 22.80 | — | — | 0.10 | — |
| 43_first-paint | 70.00 | 118.70 | — | — | -6.70 | — |

## Create-family + clear focus

| Bench | Step1 Reatom | Step2 Reatom | Step1 script | Step2 script | Vue script |
|---|---:|---:|---:|---:|---:|
| 01_run1k | 71.76 | 68.04 | 28.09 | 26.00 | 13.08 |
| 02_replace1k | 78.34 | 71.68 | 33.04 | 29.68 | 15.51 |
| 07_create10k | 736.97 | 667.61 | 237.64 | 212.65 | 99.86 |
| 08_create1k-after1k_x2 | 78.14 | 72.03 | 24.43 | 23.93 | 11.25 |
| 09_clear1k_x8 | 34.26 | 26.42 | 31.41 | 23.82 | 16.03 |
| 22_run-memory | 6.39 | 6.46 | — | — | — |

## Verdict

- Reatom geo 44.02 vs Step1 49.72 (-5.70); Vue geo 39.76.
- `01_run1k` script 26.00 (Step1 28.09, Δ -2.09).
- `07_create10k` script 212.65 (Step1 237.64, Δ -24.99).
