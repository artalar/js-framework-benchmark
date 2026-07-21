# Closure-opt bench — Reatom vs Vue

## Changes (local reatom `perf/jsx-closure-opt` @ `8a9441a`)

- **R1** Split `setProp`: static attrs take a capture-free path (no V8 Context prologue)
- **R2** Shared per-root event dispatcher; handlers on `meta.handlers`; names at dispatch time
- **R3 (tiny)** Hoist `computed().extend` callback in core; skip default `named('classNameAtom')`
- **R4/peek** Hoist `reatomClassName` + peek/apply initial reactive props (Object.is skip)
- **R5** Lazy event action names at click time
- **R6** Module-scope lifecycle helpers (no per-subscribe arrows in `connectNode`)

Tests: `@reatom/jsx` 101 passed / 2 skipped; `@reatom/core` unit 417 passed.

## CPU geo-mean (01–09)

| | npm 1001.3.0 baseline | Closure-opt | Δ |
|---|---:|---:|---:|
| Reatom | 43.59 | 43.61 | +0.02 |
| Vue (same run) | 40.24 | 40.63 | +0.38 |
| R/V ratio | 1.083 | 1.073 | — |

## Reatom vs Vue (means)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs baseline |
|---|---:|---:|---:|---:|---:|
| 01_run1k | 66.71 | 56.35 | 23.99 | 13.63 | -2.35 |
| 02_replace1k | 74.03 | 60.53 | 30.01 | 16.07 | 2.69 |
| 03_update10th1k_x16 | 24.07 | 26.66 | 1.59 | 3.59 | -0.52 |
| 04_select1k | 5.68 | 6.64 | 0.60 | 1.40 | 0.15 |
| 05_swap1k | 29.34 | 29.49 | 0.85 | 1.75 | -0.41 |
| 06_remove-one-1k | 23.52 | 25.61 | 0.33 | 2.86 | 0.03 |
| 07_create10k | 675.50 | 582.09 | 220.17 | 105.09 | 15.63 |
| 08_create1k-after1k_x2 | 68.22 | 59.55 | 21.80 | 11.39 | -1.35 |
| 09_clear1k_x8 | 26.56 | 19.09 | 23.95 | 16.21 | 0.15 |
| 21_ready-memory | 1.20 | 1.35 | — | — | -0.02 |
| 22_run-memory | 6.47 | 4.22 | — | — | 0.11 |
| 25_run-clear-memory | 1.63 | 2.05 | — | — | 0.01 |
| 41_size-uncompressed | 30.40 | 63.70 | — | — | 0.80 |
| 42_size-compressed | 10.70 | 22.80 | — | — | 0.30 |
| 43_first-paint | 72.10 | 115.40 | — | — | 6.20 |

## Create / clear / memory focus

| Bench | Baseline Reatom total | Closure total | Closure script | Vue script |
|---|---:|---:|---:|---:|
| 01_run1k | 69.05 | 66.71 | 23.99 | 13.63 |
| 02_replace1k | 71.34 | 74.03 | 30.01 | 16.07 |
| 07_create10k | 659.87 | 675.50 | 220.17 | 105.09 |
| 08_create1k-after1k_x2 | 69.57 | 68.22 | 21.80 | 11.39 |
| 09_clear1k_x8 | 26.41 | 26.56 | 23.95 | 16.21 |
| 22_run-memory | 6.36 | 6.47 | — | — |

## Verdict

- Reatom geo **43.61** vs baseline **43.59** (+0.02); Vue same-run **40.63** (baseline 40.24).
- R/V geo ratio 1.073 vs baseline 1.083.
- `01_run1k` script 23.99; `07_create10k` script 220.17 (still ~2.1× Vue).
- `22_run-memory` 6.47 MB (baseline 6.36).
- Size uncompressed/compressed 30.4/10.7 KB.
- Absolute Δ-vs-baseline still noisy; prefer within-run script and R/V ratio.
- Remaining create gap is still dominated by per-row atom/`computed` construction in core (castAtom shape), not event/setProp Contexts.
