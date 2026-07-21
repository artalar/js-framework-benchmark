# Opt step 1 — Eager connect + MO skip

## Change summary

In `@reatom/jsx` `walkLinkedList` / `mount`:

- After linked-list append/rebuild flush, if the parent `element.isConnected`, **eagerly** `connectNode` on appended roots (subscriptions + refs run before MO).
- `connectNode` / `processMutations` **skip** roots that are already `mounted` with live `unsubscribes` (avoids double-walk of the same bulk insert).
- **Meta registry** (iterate ~4 subscribed nodes/row instead of walking ~10 DOM children) was attempted and **reverted**: it broke live-fragment / shared-element / pre-mount atom-upgrade tests. Deferred to a later experiment after Step 5 / Fable pass.

Patch: `frameworks/keyed/reatom-jsx/reports/step1-eager-connect.patch`

## CPU geo-mean (01–09)

| | npm 1001.3.0 baseline | Step 1 | Δ |
|---|---:|---:|---:|
| Reatom | 43.59 | 49.72 | +6.13 |
| Vue | 40.24 | 47.41 | +7.16 |

## Reatom vs Vue (means)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Reatom paint | Vue paint | Δ total vs baseline |
|---|---:|---:|---:|---:|---:|---:|---:|
| 01_run1k | 71.76 | 57.99 | 28.09 | 14.25 | 42.99 | 43.05 | 2.71 |
| 02_replace1k | 78.34 | 63.91 | 33.04 | 18.39 | 44.52 | 44.80 | 7.00 |
| 03_update10th1k_x16 | 25.15 | 34.99 | 2.03 | 4.79 | 20.87 | 27.78 | 0.55 |
| 04_select1k | 6.53 | 7.62 | 0.70 | 1.69 | 4.68 | 4.61 | 1.00 |
| 05_swap1k | 36.61 | 38.06 | 0.81 | 2.34 | 33.36 | 32.79 | 6.86 |
| 06_remove-one-1k | 27.85 | 29.73 | 0.48 | 3.50 | 26.00 | 24.79 | 4.37 |
| 07_create10k | 736.97 | 632.83 | 237.64 | 111.90 | 494.17 | 516.39 | 77.09 |
| 08_create1k-after1k_x2 | 78.14 | 64.93 | 24.43 | 13.11 | 52.41 | 50.65 | 8.57 |
| 09_clear1k_x8 | 34.26 | 26.33 | 31.41 | 22.99 | 2.28 | 2.73 | 7.85 |
| 21_ready-memory | 1.21 | 1.37 | — | — | — | — | -0.02 |
| 22_run-memory | 6.39 | 4.26 | — | — | — | — | 0.03 |
| 25_run-clear-memory | 1.63 | 2.03 | — | — | — | — | 0.00 |
| 41_size-uncompressed | 29.90 | 63.70 | — | — | — | — | 0.30 |
| 42_size-compressed | 10.50 | 22.80 | — | — | — | — | 0.10 |
| 43_first-paint | 76.70 | 127.40 | — | — | — | — | 10.80 |

## Create-family + clear focus (script)

| Bench | Baseline Reatom total | Step1 Reatom total | Step1 script | Vue script |
|---|---:|---:|---:|---:|
| 01_run1k | 69.05 | 71.76 | 28.09 | 14.25 |
| 02_replace1k | 71.34 | 78.34 | 33.04 | 18.39 |
| 07_create10k | 659.87 | 736.97 | 237.64 | 111.90 |
| 08_create1k-after1k_x2 | 69.57 | 78.14 | 24.43 | 13.11 |
| 09_clear1k_x8 | 26.41 | 34.26 | 31.41 | 22.99 |

## Memory

- **21_ready-memory**: Reatom 1.21 MB (baseline 1.22), Vue 1.37 MB
- **22_run-memory**: Reatom 6.39 MB (baseline 6.36), Vue 4.26 MB
- **25_run-clear-memory**: Reatom 1.63 MB (baseline 1.62), Vue 2.03 MB

## Verdict

- CPU geo-mean moved **+6.13** vs npm 1001.3.0.
- `01_run1k` total 71.76 (baseline 69.05), script 28.09.
- `07_create10k` total 736.97 (baseline 659.87), script 237.64.
- Expected ~10 ms script drop on 01/02/08 may need the registry half of Step 1 (deferred); eager connect alone mainly removes duplicate MO work.

## Notes on absolute vs baseline

Both Reatom (**+6.1**) and Vue (**+7.2**) CPU geo-means rose vs the earlier npm-1001.3.0 suite on this machine — treat absolute Δ-vs-baseline as noisy. Prefer **within-run Reatom vs Vue** (Reatom geo 49.72 vs Vue 47.41; create script still ~2× Vue). Eager connect alone did not deliver the hoped ~10 ms create-script cut; registry half remains deferred.
