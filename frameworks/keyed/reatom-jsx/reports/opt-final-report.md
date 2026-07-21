# Reatom create/mount opt — final report

## Stack shipped

Local `@reatom/jsx` (reatom `6be904d`) vendored into `frameworks/keyed/reatom-jsx`:

1. Eager connect after linked-list append + MO skip of live roots
2. Peek/apply initial reactive class & props; skip redundant first subscribe writes
3. Lazy meta arrays + light simple class getters + unnamed bindings when `DEBUG` off
4. Eager teardown on clear / version-jump (later refined by Exp 1)
5. Gate `log.label` / `eventActionName` behind `jsx.DEBUG`
6. **Fable Exp 1:** `eagerlyDetached` WeakSet — skip MO removal walks for already-torn-down clear roots
7. **Fable Exp 2:** render-time connect registry + `eagerlyConnected` MO bypass for owned appends
8. **Fable Exp 3:** release child registry arrays after merge (memory fix)

Meta-registry (Step 1 half) initially broke fragment tests; Fable Exp 2 delivered a corrected registry.

## Fable validation (Steps 1–5)

| Step | Verdict | Notes |
|---|---|---|
| 1 eager connect | noise | scaffolding; Vue drifted similarly |
| 2 peek props | real (small) | ~1–2 ms create script |
| 3 slim meta | real (small) | only step R↑ while V↓; ready-memory tiny win |
| 4 clear teardown | regress | double walk (eager + MO); fixed by Exp 1 |
| 5 gate log | noise | size Δ 0 without compile-time DCE |

Details: `reports/opt-fable-validation.md`, `opt-fable-exp-{1,2,3}.md`

## CPU geo-mean progression (Reatom vs Vue, steps/exps)

| Milestone | Reatom | Vue |
|---|---:|---:|
| Baseline npm 1001.3.0 | 43.59 | 40.24 |
| Step 1 | 49.72 | 47.41 |
| Step 2 | 44.02 | 39.76 |
| Step 3 | 43.45 | 40.19 |
| Step 4 | 47.48 | 44.00 |
| Step 5 | 49.04 | 45.13 |
| Fable exp 1 | 51.87 | 47.53 |
| Fable exp 2 | 45.21 | 42.48 |
| Fable exp 3 | 44.89 | 41.37 |
| **Final 5-fw run** | **43.79** | **40.69** |

## Final five-framework results (means)

| Bench | reatom-jsx | preact-hooks | react-hooks | solid | vue |
|---|---:|---:|---:|---:|---:|
| 01_run1k | 68.43 | 58.47 | 57.90 | 47.45 | 56.55 |
| 02_replace1k | 71.87 | 65.85 | 63.80 | 52.84 | 59.89 |
| 03_update10th1k_x16 | 25.51 | 44.91 | 28.51 | 24.65 | 26.71 |
| 04_select1k | 5.46 | 24.57 | 8.43 | 6.44 | 6.66 |
| 05_swap1k | 28.83 | 52.45 | 194.52 | 29.46 | 29.69 |
| 06_remove-one-1k | 23.10 | 35.20 | 24.67 | 23.16 | 25.95 |
| 07_create10k | 675.51 | 603.17 | 770.87 | 496.39 | 579.38 |
| 08_create1k-after1k_x2 | 70.11 | 67.69 | 64.22 | 54.55 | 59.62 |
| 09_clear1k_x8 | 27.37 | 20.99 | 29.67 | 20.33 | 19.05 |
| 21_ready-memory | 1.22 | 1.09 | 1.68 | 1.00 | 1.35 |
| 22_run-memory | 6.68 | 3.79 | 4.92 | 3.10 | 4.22 |
| 25_run-clear-memory | 1.64 | 1.31 | 2.44 | 1.23 | 2.03 |
| 41_size-uncompressed | 31.60 | 14.60 | 190.30 | 11.50 | 63.70 |
| 42_size-compressed | 11.00 | 5.70 | 51.40 | 4.50 | 22.80 |
| 43_first-paint | 68.50 | 51.20 | 285.70 | 47.90 | 122.60 |

## Final CPU geo-mean (01–09)

| Framework | Final | Baseline | Δ |
|---|---:|---:|---:|
| reatom-jsx | 43.79 | 43.59 | +0.20 |
| preact-hooks | 57.36 | 58.21 | -0.85 |
| react-hooks | 56.91 | 57.06 | -0.16 |
| solid | 37.58 | 37.90 | -0.32 |
| vue | 40.69 | 40.24 | +0.44 |

## Create / clear script (final)

| Bench | Reatom script | Vue script | Solid script |
|---|---:|---:|---:|
| 01_run1k | 26.11 | 13.51 | 5.37 |
| 02_replace1k | 28.55 | 15.79 | 10.11 |
| 07_create10k | 227.67 | 101.73 | 50.53 |
| 08_create1k-after1k_x2 | 22.18 | 11.31 | 6.74 |
| 09_clear1k_x8 | 24.66 | 16.25 | 16.71 |

## Takeaways

- Final Reatom CPU geo-mean **43.79** vs baseline **43.59** (+0.20); vs Vue **40.69**, Solid **37.58**.
- Absolute cross-run totals are noisy; prefer within-run ratios and script columns.
- Connect-path opts (eager + registry + skip double clear walk) help create/clear script modestly; remaining ~2× create-script gap vs Vue is mostly per-row atom/computed/bind construction in core, not MO walk.
- `22_run-memory` still ~6.5–7 MB vs Vue ~4.3 MB after slim-meta / registry release.

## Artifacts

- Steps: `opt-step-{1..5}-*.md`
- Fable: `opt-fable-validation.md`, `opt-fable-exp-{1,2,3}.md`
- This file + `opt-final-report.json`
- Vendor tarballs + patches under `frameworks/keyed/reatom-jsx/`
