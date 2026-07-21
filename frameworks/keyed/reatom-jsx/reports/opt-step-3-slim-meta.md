# Opt step 3 — Slim per-row meta / class bindings

## Change summary

- Lazy `meta.subscribes` / `meta.unsubscribes` (allocate on first `unlink`, release to `undefined` on teardown)
- Simple class getters use a light `computed` (primitive fast-path) instead of full `reatomClassName`/`parseClasses` object walk
- Skip named prop-binding strings when `jsx.DEBUG` is off

Builds on Steps 1–2. Patch: `reports/step3-slim-meta.patch`

## CPU geo-mean (01–09)

| | Baseline | Step 2 | Step 3 | Δ vs Step 2 |
|---|---:|---:|---:|---:|
| Reatom | 43.59 | 44.02 | 43.45 | -0.57 |
| Vue | 40.24 | 39.76 | 40.19 | +0.43 |

## Reatom vs Vue

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs Step2 | Δ script vs Step2 |
|---|---:|---:|---:|---:|---:|---:|
| 01_run1k | 66.55 | 53.75 | 24.94 | 13.07 | -1.49 | -1.06 |
| 02_replace1k | 69.90 | 56.86 | 28.11 | 15.39 | -1.78 | -1.57 |
| 03_update10th1k_x16 | 23.21 | 25.79 | 1.29 | 3.54 | 0.12 | -0.02 |
| 04_select1k | 5.21 | 6.22 | 0.45 | 1.26 | -1.14 | -0.16 |
| 05_swap1k | 27.87 | 28.61 | 0.81 | 1.71 | -1.37 | -0.05 |
| 06_remove-one-1k | 22.92 | 25.08 | 0.32 | 2.79 | -0.47 | -0.01 |
| 07_create10k | 650.78 | 554.66 | 206.74 | 99.24 | -16.83 | -5.91 |
| 08_create1k-after1k_x2 | 68.06 | 60.11 | 22.19 | 11.80 | -3.97 | -1.73 |
| 09_clear1k_x8 | 34.71 | 23.31 | 31.59 | 20.25 | 8.29 | 7.77 |
| 21_ready-memory | 1.17 | 1.37 | — | — | -0.04 | — |
| 22_run-memory | 6.44 | 4.26 | — | — | -0.02 | — |
| 25_run-clear-memory | 1.62 | 2.03 | — | — | -0.01 | — |
| 41_size-uncompressed | 30.40 | 63.70 | — | — | 0.30 | — |
| 42_size-compressed | 10.70 | 22.80 | — | — | 0.10 | — |
| 43_first-paint | 70.40 | 122.80 | — | — | 0.40 | — |

## Memory focus

- **21_ready-memory**: Reatom 1.17 (Step2 1.21, baseline 1.22), Vue 1.37
- **22_run-memory**: Reatom 6.44 (Step2 6.46, baseline 6.36), Vue 4.26
- **25_run-clear-memory**: Reatom 1.62 (Step2 1.63, baseline 1.62), Vue 2.03

## Verdict

- Reatom geo 43.45 (Step2 44.02, Δ -0.57).
- `22_run-memory` 6.44 MB (Step2 6.46; target ~4.5–5 vs Vue 4.26).
