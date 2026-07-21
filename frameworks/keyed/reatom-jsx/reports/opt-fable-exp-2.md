# Fable experiment 2 — render-time registry connect + MO bypass for owned appends

## Algorithm

Implements the deferred "registry half" of Step 1 (the earlier attempt broke
live-fragment / shared-element / pre-mount-upgrade tests) plus the eager-only
idea, on top of Steps 1–5 + Exp 1:

- **Registry capture at render.** Every jsx-created container gets a
  `registryKey` expando: `EMPTY_REGISTRY` sentinel when created by `h`,
  `Node[]` of descendant-or-self nodes carrying subscriptions/ref hooks, or
  `null` (tombstone = must fall back to walkTree). `unlink` and the two
  `meta.mount` sites enroll their node; `walk`'s append sites merge child lists
  upward; `walkAtomFragment` lifts the detached buffer's list onto the live
  fragment (`[start, ...content]`, matching document connect order).
- **Registry connect.** `connectNode` iterates the root's captured list (~4
  meta nodes per bench row) instead of `walkTree` over every DOM child (~10),
  with the same subscribe-parent-first / ref-child-first two-pass order.
  **One-shot**: the registry is consumed on first connect, so reconnects fall
  back to the DOM walk and post-mount mutations can never leave stale entries.
  Entries detached by a mid-connect emission (primitive Text upgrade) are
  skipped via `isConnected`; replacement content connects through MO.
- **Safety fallbacks (what broke the first registry attempt).** Foreign
  (unstamped) element children, tombstoned containers, detached linked-list
  flushes (`registryKey = null` on the container), `Bind` on non-jsx elements,
  and every reconnect keep the full walkTree path.
- **Eager-only MO bypass.** `flushAppendBatch` connects the batch children
  (exactly the nodes MO will report — fragment rows are already flattened)
  synchronously and marks them in an `eagerlyConnected` WeakSet;
  `processMutations` consumes the mark and skips the addition record without
  any meta/`isConnected` work. `mount` does the same for its own append. MO
  keeps handling foreign/adopted nodes.

Stack: Steps 1–5 + Exp 1 + this. reatom commit `a4dc2c8` on
`perf/jsx-connect-opt`. Tests: 101 passed, 2 skipped.

## CPU geo-mean (01–09)

| | Baseline | Step 5 | Exp 1 | Exp 2 | Δ vs Exp 1 |
|---|---:|---:|---:|---:|---:|
| Reatom | 43.59 | 49.04 | 51.87 | 45.21 | -6.66 |
| Vue | 40.24 | 45.13 | 47.53 | 42.48 | -5.05 |
| R/V ratio | 1.083 | 1.087 | 1.091 | **1.064** | — |

Machine drifted back fast (Vue −5.05 with unchanged code). Drift-adjusted
Reatom geo residual vs Exp 1: **−1.14** — the best within-run R/V ratio of all
seven runs, but treat the magnitude as small-real, not −6.66.

## Reatom vs Vue (means)

| Bench | Reatom total | Vue total | Reatom script | Vue script | Δ total vs E1 | Δ script vs E1 | drift-adj script |
|---|---:|---:|---:|---:|---:|---:|---:|
| 01_run1k | 70.63 | 58.31 | 27.35 | 13.90 | +3.21 | +1.99 | +4.40 |
| 02_replace1k | 75.78 | 62.89 | 30.30 | 16.83 | -6.91 | -5.67 | -1.33 |
| 03_update10th1k_x16 | 25.49 | 27.33 | 1.98 | 3.62 | -7.45 | -0.41 | -0.04 |
| 04_select1k | 5.65 | 6.72 | 0.57 | 1.55 | -0.52 | -0.08 | -0.02 |
| 05_swap1k | 29.40 | 30.11 | 0.75 | 1.78 | -7.61 | -0.23 | -0.09 |
| 06_remove-one-1k | 24.60 | 25.90 | 0.30 | 2.89 | -2.42 | -0.19 | -0.08 |
| 07_create10k | 701.82 | 578.62 | 230.19 | 106.33 | -12.53 | -16.07 | -5.84 |
| 08_create1k-after1k_x2 | 70.74 | 61.46 | 22.53 | 11.71 | -12.19 | -4.16 | -1.27 |
| 09_clear1k_x8 | 28.51 | 24.10 | 25.61 | 20.98 | -11.99 | -11.89 | -7.39* |
| 21_ready-memory | 1.20 | 1.34 | — | — | -0.02 | — | — |
| 22_run-memory | **6.99** | 4.26 | — | — | **+0.52** | — | — |
| 25_run-clear-memory | 1.62 | 2.05 | — | — | -0.02 | — | — |
| 41_size-uncompressed | 31.60 | 63.70 | — | — | +1.00 | — | — |
| 42_size-compressed | 11.00 | 22.80 | — | — | +0.30 | — | — |
| 43_first-paint | 76.00 | 127.30 | — | — | +0.90 | — | — |

\* Vue's 09 mean (20.98) is inflated by two warmup-contaminated iterations
(52.4/42.0 ms; its median is 16.8), so the mean-based −7.39 overstates. By
medians, Reatom 09 script 25.9 vs Vue 16.8 → R/V 1.54, i.e. Exp 1's clear
improvement carried forward plus drift, not a further clear win from Exp 2.

## Create family R/V script ratios across runs

| Bench | S3 | S5 | Exp 1 | Exp 2 |
|---|---:|---:|---:|---:|
| 01_run1k | 1.91 | 1.91 | 1.65* | 1.97 |
| 02_replace1k | 1.83 | 1.90 | 1.88 | **1.80** |
| 07_create10k | 2.08 | 2.24 | 2.22 | 2.16 |
| 08_create1k-after1k_x2 | 1.88 | 1.95 | 2.03 | 1.92 |
| 09_clear1k_x8 | 1.56 | 1.73 | 1.57 | 1.22 (1.54 med) |

\* Exp 1's 01 ratio was an outlier (Vue's 01 script hit its across-runs max).

## Verdict — kept provisionally, but with a real memory regression

- Create-script: a small real win — 07 −5.8 ms drift-adjusted, 02/08 −1.3 each;
  01 within its historical band (+4.4 adj is against Exp 1's outlier run).
- **22_run-memory 6.47 → 6.99 MB (+0.52 MB) is real** (memory metrics are
  near-deterministic). Cause: intermediate merge arrays (`td`/`a` level) stay
  referenced by their nodes' `registryKey` expandos for the row's lifetime —
  only the row root's registry is consumed at connect. ~5 small arrays per row
  ≈ 0.5 MB per 1k rows.
- Bundle +1.0/+0.3 KB from the registry code.
- Decision: keep the algorithm, fix the retention in Experiment 3 (release a
  child's registry once it is merged into its parent; later merges of a moved
  child then tombstone the new parent, preserving the walkTree fallback), and
  re-bench before choosing the final stack.
