# Reatom create/mount opt summary

**Baseline:** npm `@reatom` 1001.3.0 (CPU geo-mean ≈ 43.59)  
**Library:** local experiments were discarded; clone restored to `origin/v1001` (`3606a78`). Bench uses published npm packages again.  
**PR branch:** `cursor/reatom-jsx-opt-steps-19bd`

## Problem

Create/mount (01/02/07/08) is the real gap vs Vue/Solid — almost all **script**, not paint. Swap is not a real loss (script already fastest; paint/layout floor).

## What we tried (local only; not shipped upstream)

| Step / exp | Change | Verdict |
|---|---|---|
| 1 Eager connect + MO skip | Connect LL roots when parent connected; skip remount | **noise** (scaffolding) |
| 1b Meta registry | Walk ~4 subscribed nodes/row | **reverted** (broke fragments) |
| 2 Peek initial props | Apply class/atom props at render; skip first subscribe write | **real, small** (~1–2 ms create script) |
| 3 Slim meta / class | Lazy meta arrays; light class getters; unnamed bindings if `DEBUG` off | **real, small**; ready-mem −0.04 MB; run-mem still ~6.4 |
| 4 Clear teardown | `cleanupNodes` before `innerHTML=''` | **regress** (double walk with MO) |
| 5 Gate `log.label` | Only when `DEBUG` | **noise** (no size win without compile-time DCE) |
| Fable Exp 1 | `eagerlyDetached` — skip MO removal after eager teardown | **kept** (fixes Step 4) |
| Fable Exp 2 | Render-time connect registry + owned-append MO bypass | **kept** (create help; +0.5 MB mem) |
| Fable Exp 3 | Release child registries after merge | **kept** (mem 6.99→6.67) |

Absolute geos drifted with the machine (Vue alone ±9%); prefer within-run R/V ratios and script columns.

## Final 5-framework run (on the experimental stack)

| Framework | CPU geo 01–09 | vs baseline |
|---|---:|---:|
| solid | 37.58 | −0.32 |
| vue | 40.69 | +0.44 |
| **reatom-jsx** | **43.79** | **+0.20** |
| react-hooks | 56.91 | −0.16 |
| preact-hooks | 57.36 | −0.85 |

Create/clear **script** (experimental stack):

| Bench | Reatom | Vue | Solid |
|---|---:|---:|---:|
| 01_run1k | 26.1 | 13.5 | 5.4 |
| 07_create10k | 227.7 | 101.7 | 50.5 |
| 09_clear1k_x8 | 24.7 | 16.3 | 16.7 |

`22_run-memory`: Reatom ~6.7 MB · Vue ~4.2 · Solid ~3.1.

## Takeaways

1. Connect-path opts barely moved geo-mean vs 1001.3.0 (+0.20).
2. Real wins: peek props, slim meta, registry connect, fix double-clear walk — all small.
3. Remaining ~2× create-script gap vs Vue is mostly **per-row atom/computed/`bind` construction**, not MO child walking.
4. Runtime `DEBUG` gates do not shrink the bundle; need compile-time DCE for size.
5. Next leverage: fewer/lighter per-row bindings and closure/`bind` cost (see closure review if present).

## Closure follow-ups

Deep closure/V8 review of stock `packages/jsx/src/index.ts` (1001.3.0): see [`CLOSURE_ANALYSIS.md`](./CLOSURE_ANALYSIS.md).
Headline: ~36 function objects + ~11 retained contexts + 5 bound fns per row (~2.5–3.5 KB), matching the run-memory gap. Top levers:
1. Split `setProp` so static attrs skip the prologue `CreateFunctionContext` (bytecode-verified; 6 of 9 calls/row).
2. Shared module-scope event dispatcher replacing per-handler `arrow + bind() + thunk` stacks (~10 objects/row).
3. Trim per-atom construction (`castAtom` bound subscribe, toString/toJSON, `computed()`'s per-call extend arrow) — biggest lever, lives in core.

## Artifacts

Raw step/exp dumps lived under `reports/opt-step-*` / `opt-fable-*` during the campaign; this file is the collapsed summary. Full narrative also in the PR discussion / agent transcript.

## Closure-opt follow-up (tried + benched)

Implemented R1/R2/R4–R6 (+ tiny R3 hoist) on a local reatom branch and benched vs Vue.
See `CLOSURE_OPT_BENCH.md` and `CLOSURE_ANALYSIS.md`. Library clone tip for the experiment: `perf/jsx-closure-opt` (not pushed upstream).
