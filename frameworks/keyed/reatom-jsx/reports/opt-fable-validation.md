# Fable validation — opt steps 1–5 (Reatom jsx create/mount)

Scope: classify each step's measured deltas as **real / noise / regress**, using only
evidence that survives machine drift. Sources: `opt-step-{1..5}-*.md/json`,
raw per-iteration results in `webdriver-ts/results` (Step-5 run), and the current
`packages/jsx/src/index.ts` (Steps 1–5 landed).

## Method: what counts as evidence on this machine

- **Vue is the drift control.** Vue's code was identical in all six runs, yet its CPU
  geo-mean moved 40.24 → 47.41 → 39.76 → 40.19 → 44.00 → 45.13 (±9%). Any Reatom delta
  of similar relative size and direction as Vue's in the same transition is machine
  drift, not the step.
- **Within-run Reatom/Vue ratios** are the primary signal. Drift on this box is close
  to multiplicative (e.g. Step3→Step4, `09_clear` script: Reatom +11.7%, Vue +11.7% —
  exactly proportional).
- **Drift-adjusted residual** = observed − (previous Reatom × Vue-ratio for the same
  transition). Residuals below ~2 ms on 01/02/08/09 and ~7 ms on 07 are within noise
  (per-run SE from 15 iterations: 01 script ±0.6, 02 ±0.7, 07 ±3.4, 08 ±0.4, 09 ±1.3;
  Vue's 09 script swings 16.0–23.0 across runs, its worst-noise bench).
- **Size and ready-memory** are near-deterministic (SE ≈ 0) — small deltas there are real.

## Verdict table

| Step | Measured Δ geo (report) | Verdict | Real effect after drift correction |
|---|---|---|---|
| 1 eager connect + MO skip | +6.13 | **noise** | ≈ 0. Vue moved +7.16 in the same run. No create-script win (R−V script 01: +13.8, 07: +125.7 — same as the structural gap before). Keeps value as *scaffolding* for later connect algorithms, not as a measured win. |
| 2 peek initial props | −5.70 | **real (small)** | ~1–2 ms create-family script at best. Vue moved −7.64, so most of the −5.70 is drift-back. R−V script narrowed slightly (01: 13.8→12.9, 02: 14.7→14.2, 07: 125.7→112.8, 09: 8.4→7.8) but R/V script ratios stayed ~flat (01: 1.97→1.99, 07: 2.12→2.13). Mechanism is sound (first value applied at render; connect-time re-emission skipped by `Object.is`). |
| 3 slim meta + light class binding | −0.57 | **real (small)** | Only step where Reatom improved while Vue *worsened* in the same run (R −0.57 vs V +0.43; R/V geo 1.107→1.081). Drift-adjusted script residuals are consistently negative on the create family (01 −1.0, 02 −1.4, 07 −4.6, 08 −2.9). Ready-memory 1.21→1.17 MB is deterministic and real. Run-memory goal (~4.5–5 MB) not reached (6.44). |
| 4 clear/rebuild eager teardown | +4.03 | **regress** | Small but real regression on the clear path. Raw +4.03 is mostly drift (Vue +3.81), but the *targeted* bench got worse within-run: `09_clear` R/V total 1.489→1.501 (and script stayed 1.56), `02_replace` (which exercises the rebuild path) drift-adjusted script +2.1 ms. Mechanism confirms strictly-added work: `teardownLinkedListChildren` walks the whole subtree, then `innerHTML = ''` fires a MutationObserver removal record and `cleanupNodes` **walks the same ~11k-node subtree a second time** (metas are already released, but the walk itself is paid twice). |
| 5 gate `log.label` behind DEBUG | +1.56 | **noise** | ≈ 0. Vue +1.13 in the same run. The intended effect — bundle size — did **not** materialize: 41/42 sizes identical to Step 4 (30.5/10.7 KB), because a runtime `peek(DEBUG)` gate cannot dead-code-eliminate `log`; that needs a compile-time flag. The odd 09 movement (+3.6 drift-adjusted) cannot be caused by event-name gating (clear path runs no event code) — noise. Harmless to keep. |

## Cross-cutting observations

1. **The stack did not move the Reatom/Vue gap.** Within-run R/V geo ratio:
   baseline 1.083 → S1 1.049 → S2 1.107 → S3 1.081 → S4 1.079 → S5 1.087.
   Net ≈ unchanged (S1's 1.049 is driven by a single Vue paint outlier on
   `03_update`: Vue paint 27.8 there vs 20–24 in every other run).
2. **Create script gap is structural, not connect-walk-bound.** R−V script on
   01/02/07/08 stays ~13/14/110–130/11 ms across all runs. Per-row cost is dominated
   by 4 atom/computed constructions + 4 subscriptions + `bind()` closures per row, not
   by the ~6 extra non-meta DOM nodes the connect walk visits (~0.5–2 ms per 1k rows).
   Expectations for MO-algorithm experiments should be single-digit ms on 01/02/08 and
   low-tens on 07/09.
3. **`09_clear1k` is the one bench with a consistent adverse trend**: R/V total
   ratio 1.39 (base) → 1.30 → 1.40 → 1.49 → 1.50 → 1.63. Steps 3–4 are the plausible
   contributors (S4's double walk is mechanistically confirmed; S5's jump is likely
   noise given Vue-09 variance). This is the first target for the experiments: clear
   currently pays teardown-walk twice plus MO record processing for 1000 removed roots.
4. **Step-1's "registry half" remains the open create-side idea** — the eager-connect
   half alone only re-ordered work (sync instead of MO-async) rather than removing it.

## Per-step evidence detail

### Step 1 — eager connect + MO skip (noise)

- Both frameworks slowed ~15% vs the baseline run (R geo +6.13, V +7.16) — drift.
- Drift-adjusted totals baseline→S1: 01 −0.2, 02 +2.5, 07 +2.4, 08 +1.7, 09 −2.4 —
  all within noise bands. (Baseline JSON has no script split, so script residuals are
  not computable for this transition.)
- Design analysis: eager connect performs the same `connectNode` walk that MO would
  have done, just synchronously; the MO-side skip only avoids the duplicate *root*
  visit (`metaHasLiveUnsubs` early return), not the per-node work. No mechanism for a
  create-script win — consistent with the null measurement.

### Step 2 — peek initial props (real, small)

- Vue −7.64 says most of Reatom's −5.70 is drift-back from the slow S1 run.
- Within-run R−V script deltas (S1→S2): 01 −0.9, 02 −0.5, 07 −12.9, 08 +1.4, 09 −0.6.
  Only 07 exceeds noise, and its R/V ratio was flat (2.12→2.13) — so treat the true
  effect as small-positive rather than the headline −5.7.
- Mechanism removes real work: the initial `subscribe` emission no longer performs
  DOM writes at connect (skipped via `Object.is`), and class bindings apply once at
  render. Small per-binding win, applied ~4×/row.

### Step 3 — slim meta (real, small)

- The only transition where the two frameworks moved in *opposite* directions
  (R −0.57, V +0.43); R/V geo 1.107→1.081.
- Drift-adjusted script: 01 −1.03, 02 −1.35, 07 −4.59, 08 −2.90 (uniformly negative
  on the create family — a coherent pattern, unlike noise) — vs 09 +1.48 (within 09's
  noise, but note observation 3 above).
- 21_ready-memory 1.21→1.17 MB (deterministic metric — real). 22_run-memory
  unchanged (6.46→6.44), so the lazy-array release did not reduce steady-state row
  cost; the remaining ~2.2 MB gap to Vue lives in per-row subscriptions/closures.

### Step 4 — clear teardown (regress)

- Targeted bench got worse, not better: 09 script 31.59→35.29 while Vue 20.25→22.61
  (both +11.7% — drift), i.e. **zero improvement** from the change; ratio trend on 09
  kept climbing. 02_replace (rebuild = teardown + create in one click) shows
  +2.1 ms drift-adjusted script — the double-walk cost is visible where it should be.
- Code path: `cleanupNodes` on `element.childNodes` (full `walkTree`), then
  `innerHTML = ''` → MO removal record → `cleanupNodes` again over the same 1000
  roots (second full `walkTree`; per-node symbol lookup + meta checks, all no-ops).
- Verdict: revert, or keep the eager half only if the MO re-walk is suppressed
  (experiment target).

### Step 5 — gate log (noise)

- Size deltas: 0.0 / 0.0 KB — primary goal not achieved (runtime gate ≠ DCE; `log`
  and its label machinery stay in the bundle). A real size win needs a build-time
  define (e.g. `__DEV__`-style) in the packed build.
- Event-name gating saves string work only on row *render* (handler registration);
  amount (~2 strings/row) is below measurement noise. CPU deltas track Vue's drift.

## Bottom line

- Keep Steps 2, 3, 5 (real-small, real-small, harmless).
- Step 1 is neutral scaffolding — keep for the follow-up algorithms.
- Step 4 is a small real regression as landed — the experiments should either revert
  it or suppress the MutationObserver double-walk it introduced.
- The honest state after Steps 1–5: Reatom/Vue geo ratio ≈ 1.08–1.09, unchanged from
  baseline; the create-script gap (~2× Vue) is dominated by per-row reactive-binding
  construction, not by the connect algorithm.
