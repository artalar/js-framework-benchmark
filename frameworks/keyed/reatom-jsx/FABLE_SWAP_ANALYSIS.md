# Reatom-JSX vs Vue/Solid: swap1k paradox and other suspicious numbers

Analysis of js-framework-benchmark results for `keyed/reatom-jsx` (`@reatom/jsx` + `@reatom/core` v1001.3.0) vs `vue` v3.6.0-alpha.2, `solid` v1.9.3, `preact-hooks` v10.27.1, `react-hooks` v19.2.0. Data sources: `/workspace/webdriver-ts/results/*.json`, raw Chrome traces in `/workspace/webdriver-ts/traces/`, installed package sources under `/workspace/frameworks/keyed/*/node_modules/`, plus two fresh Playwright microbenchmarks run for this report.

---

## 1. Executive answer: is the O(1) swap actually slower in JS? No.

**Reatom's swap script is the fastest of the three; the total looks "equal to Vue" because 90%+ of the measured time is browser layout/paint that is identical for any framework performing the same two DOM moves, and Vue's slightly lower mean comes from a single outlier run.**

Three independent lines of evidence:

1. **Harness script bucket (4× CPU throttle):** Reatom 0.89 ms mean vs Vue 1.76 ms vs Solid 1.46 ms. Reatom's script is ~2× faster than Vue's — the O(1) linked-list swap vs Vue's O(n) keyed diff is visible exactly where it should be.
2. **Fresh script-only microbench (this VM, unthrottled, 200 iterations, click → all microtasks flushed):**

   | framework | median | p10 | p90 | `insertBefore` calls per swap |
   |---|---:|---:|---:|---:|
   | reatom-jsx | **0.020 ms** | 0.015 | 0.035 | 2 |
   | vue | 0.100 ms | 0.080 | 0.145 | 2 |
   | solid | 0.080 ms | 0.070 | 0.100 | 2 |

   Reatom is ~5× faster in pure script, and DOM-mutation instrumentation (`Node.prototype.insertBefore` etc. wrapped) confirms **all three frameworks execute exactly 2 `insertBefore` calls and nothing else** — the resulting browser work is byte-identical.
3. **Trace anatomy** (run 0 of each, times under 4× throttle): after the click handler ends, the pipeline is the same for both:

   | phase | reatom | vue |
   |---|---:|---:|
   | click `EventDispatch` (handler) | 0.68 | 1.81 |
   | `UpdateLayoutTree` (style recalc) | 3.23 | 4.12 |
   | `Layout` | 11.78 | 11.42 |
   | `PrePaint` | 3.12 | 2.85 |
   | `HitTest` (counted in neither bucket) | 1.28 | 1.01 |
   | `Paint` | 5.69 | 5.79 |
   | `Layerize` + `Commit` | 1.26 | 1.07 |
   | **total (click → Commit end)** | **28.9** | **30.2** |

So the ~26 ms "paint" is a **fixed rendering floor** produced by moving two `<tr>`s inside a 1000-row `.table-striped` table: Bootstrap's `.table-striped>tbody>tr:nth-of-type(odd)` rule (verified in `/workspace/css/bootstrap/dist/css/bootstrap.min.css`) makes Blink do structural sibling invalidation from the earliest mutated position (row 1) — a near-full style recalc plus a full table layout pass — regardless of who issued the two `insertBefore` calls.

**Why Vue's mean (29.37) edges out Reatom's (29.75):** Vue's run 10 is an outlier — total 23.7 ms with paint 10.9 ms (its trace shows `Layout` took 2.7 ms instead of the typical ~11.4 ms; the browser got away with a cheaper layout on that frame). Excluding it, Vue's mean is 29.78 vs Reatom 29.75 — a dead tie. Medians tell the same story: **Reatom 29.2 < Vue 29.3 < Solid 30.2**, with stddevs of 1.7–2.2 ms. A 0.4 ms mean difference on a 29 ms paint-dominated benchmark with that variance is noise.

---

## 2. Measurement methodology caveats

How `webdriver-ts` computes the numbers (all in `/workspace/webdriver-ts/src/timeline.ts`):

- **total** = `commit.end - click.ts` (`computeResultsCPU`, line 247): from the start of the click `EventDispatch` to the end of the first `Commit` trace event that follows the last script/layout event. It therefore includes idle gaps, `HitTest` (~1–1.3 ms here), and scheduling latency that belong to neither bucket — which is why `script + paint < total` in every run.
- **script** = merged union of intervals for `EventDispatch, EvaluateScript, FunctionCall, TimerFire, FireAnimationFrame, RunMicrotasks, V8.Execute…` (`traceJSEventNames`, lines 83–93). Note this includes the click dispatch overhead of the synthetic Playwright click, not just framework code.
- **paint** = merged union of `UpdateLayoutTree, Layout, Commit, Paint, Layerize, PrePaint` (`tracePaintEventNames`, lines 95–103). So the "paint" column is really **style recalc + layout + paint + compositing**, and for swap it is dominated by *layout*, not rasterization.
- **CPU throttling** (`benchmarksCommon.ts` lines 105–111, applied via `Emulation.setCPUThrottlingRate` in `forkedBenchmarkRunnerPlaywright.ts` lines 98–102): benchmarks 03, 04, 05, 09 run at **4×**, 06 at **2×**; 01, 02, 07, 08 unthrottled. Throttling multiplies *all* main-thread work — the swap's "26 ms paint" is ~6.5 ms real time, and Reatom's 0.89 ms script is ~0.22 ms real.
- **Memory** = `performance.measureUserAgentSpecificMemory()` after a forced major GC (`forkedBenchmarkRunnerPlaywright.ts` lines 194–198) — a fair, GC-stabilized retained-size measure.
- 15 iterations per benchmark, mean reported. With paint stddev ≈ 1.7–4.0 ms on swap, differences under ~2 ms in the mean are not meaningful; a single outlier (like Vue's run 10) shifts the mean by ~0.4 ms.

**Implication:** for paint-floor benchmarks (swap, select, remove), the total is a poor discriminator of framework quality once every implementation reaches the "minimal DOM ops" plateau. Script and median are the right columns to read.

---

## 3. Swap path walkthrough: Reatom vs Vue

### Reatom (O(1) end to end)

1. Bench handler `swapRows` (`frameworks/keyed/reatom-jsx/src/main.tsx` lines 113–120) reads `head`/`tail` neighbors directly off the linked-list state — no array scan — and calls `rows.swap(first, second)`.
2. `swapLL` (`@reatom/core/dist/index.js` lines 3126–3166) rewires 8 `LL_PREV`/`LL_NEXT` symbol pointers — pure O(1) pointer surgery, one `{kind:"swap", a, b}` change record pushed.
3. `reatomMap`'s computed (`@reatom/core/dist/index.js` ~line 3379) translates the change record to the mapped `<tr>` nodes via a `WeakMap` lookup — O(1).
4. `walkLinkedList`'s swap branch (`@reatom/jsx/dist/index.js` lines 397–402) executes:
   ```js
   let [aNext, bNext] = [change.a.nextSibling, change.b.nextSibling];
   if (bNext) element.insertBefore(change.a, bNext); else element.append(change.a);
   if (aNext) element.insertBefore(change.b, aNext); else element.append(change.b);
   ```
   → exactly 2 DOM ops.
5. The `mount` MutationObserver (`@reatom/jsx/dist/index.js` lines 712–741) fires one microtask: moved nodes are still `isConnected`, so `cleanupNodes` gets an empty list and `connectNode` walks just the two moved `<tr>` subtrees (~20 node visits), finds `meta.unsubscribes.length !== 0` and `meta.mounted === true`, and does nothing. Cheap, but it *is* extra work Vue/Solid don't have.

### Vue (O(n) diff, same 2 DOM ops)

1. Bench handler `swapRows` (`frameworks/keyed/vue/src/App.vue` lines 51–60) swaps `_rows[1]` and `_rows[998]` in the array and re-assigns the `shallowRef`.
2. The component re-renders: `v-for` produces 1000 vnodes (`v-memo="[label, id === selected]"` makes 998 of them cheap cached clones).
3. `patchKeyedChildren` (`@vue/runtime-core/dist/runtime-core.esm-bundler.js` lines 5814–5981): prefix/suffix scans strand a middle window of ~998 vnodes; it builds a `keyToNewIndexMap` (Map of ~998 entries), fills `newIndexToOldIndexMap` (998-slot array), patches each pair, runs `getSequence` (longest-increasing-subsequence) over the window, and finally `move`s the ~2 nodes not in the LIS via `insertBefore`.
4. No post-mutation observer pass.

**Net:** Vue burns O(n) script (two 1000-element scans, a Map, an LIS) to *find* the same 2 moves Reatom knows by construction. That's the 0.10 ms vs 0.02 ms script difference — real, in Reatom's favor, and invisible under a ~6.5 ms (real-time) layout/paint floor.

---

## 4. Script vs paint breakdowns (means of 15 runs)

### 05_swap1k (4× throttle)

| framework | total | script | paint | total median |
|---|---:|---:|---:|---:|
| reatom-jsx | 29.75 | **0.89** | 26.56 | **29.2** |
| vue | 29.37¹ | 1.76 | 24.50¹ | 29.3 |
| solid | 30.21 | 1.46 | 26.28 | 30.2 |
| preact-hooks | 53.45 | 23.91 | 26.60 | 52.2 |
| react-hooks | 188.63 | 28.81 | 156.83 | 190.9 |

¹ Vue's means include the run-10 outlier (total 23.7, paint 10.9). Preact pays O(n) diff without memoization (script 24) but still only ~26 paint; React 19's forward-only move heuristic physically moves ~996 rows, so its *paint* explodes to 157 — a nice demonstration that paint tracks DOM-op count, and that Reatom/Vue/Solid all sit on the same 2-op plateau.

### The "suspicious" Reatom benches — the gap is always script, never paint

| bench | metric | reatom | vue | solid | reatom−vue total gap | script gap | paint gap |
|---|---|---:|---:|---:|---:|---:|---:|
| 01_run1k | total / script / paint | 69.05 / 26.20 / 42.15 | 55.69 / 13.35 / 41.65 | 48.77 / 5.55 / 42.43 | +13.4 | **+12.9** | +0.5 |
| 02_replace1k | | 71.34 / 28.66 / 41.99 | 60.10 / 15.95 / 43.45 | 56.88 / 10.51 / 45.63 | +11.2 | **+12.7** | −1.5 |
| 07_create10k | | 659.87 / 201.03 / 453.05 | 568.50 / 100.41 / 463.57 | 505.24 / 50.90 / 449.33 | +91.4 | **+100.6** | −10.5 |
| 08_create1k-after1k | | 69.57 / 21.74 / 46.65 | 59.06 / 11.54 / 46.29 | 54.01 / 6.92 / 45.79 | +10.5 | **+10.2** | +0.4 |
| 09_clear1k_x8 (4×) | | 26.41 / 23.23 / 2.42 | 18.99 / 16.24 / 2.06 | 20.26 / 16.67 / 1.95 | +7.4 | **+7.0** | +0.4 |

Reatom's paint numbers are statistically identical to Vue's and Solid's on every benchmark. **Every point of Reatom's CPU deficit lives in the script column of the create/clear family.** Conversely, on the fine-grained-update family Reatom is at or near the top: 03_update 24.59 (vs Solid 24.33, Vue 25.81), 04_select **5.54 (best overall)**, 06_remove 23.49 (best overall).

---

## 5. Suspiciously high Reatom numbers, with root-cause hypotheses

### 5.1 Create family (01, 02, 07, 08): mount-time connect pass — *the* real regression

Microbench (this VM, unthrottled, script only = click + all queued microtasks, median of 25):

| framework | create 1k script | clear 1k script |
|---|---:|---:|
| reatom-jsx | **15.1 ms** | **2.35 ms** |
| vue | 6.9 ms | 0.58 ms |
| solid | 3.6 ms | 0.47 ms |

Splitting Reatom's create into phases: **sync click handler 0.41 ms, MutationObserver microtask 11.87 ms.** The element construction itself (Reatom's `h()`, `walkLinkedList` batched `DocumentFragment` append) is competitive; nearly all the deficit is the post-insertion connect pass:

- `mount` (`@reatom/jsx/dist/index.js` lines 712–741) installs a `MutationObserver` (`childList: true, subtree: true`) whose callback runs `connectNode` on every added root.
- `connectNode` (lines 229–247) does a full `walkTree` DOM traversal of the inserted batch — ~10 node visits per row (`<tr>`, 4 `<td>`, 2 `<a>`, `<span>`, text nodes), i.e. ~10,000 visits for run1k and ~100,000 for create10k — checking `visited[metaSymbol]` on each, though only ~4 nodes per row carry meta.
- For each meta node it runs the deferred `subscribe()` thunks queued by `unlink` (lines 192–196): per row that is (a) the `class={() => …}` subscription, which **first creates and evaluates a `reatomClassName` computed at connect time** (`setProp`, lines 605–608) and then performs an initial `tr.className = ''` DOM write (setter receives `undefined` → `set` line 651 coerces to `''`); (b) the label Text-node subscription, whose synchronous first emission is discarded by the `Object.is` guard but still connects the atom graph; (c)+(d) two event-listener re-registration thunks (no-op re-`addEventListener` + teardown closure allocation).

So per 1000 rows Reatom pays ~4000 graph connections + 1000 computed creations + 1000 wasted `className` writes + a 10k-node DOM walk, all inside one microtask. Vue does none of this (listeners and props are set during patch); Solid compiles the wiring away. This one architectural spot explains the entire 01/02/07/08 gap and is the biggest lever on the CPU geomean (43.59 → the ~13 ms script excess on 01 alone costs several geomean points).

### 5.2 09_clear1k_x8: teardown walk

Symmetric cause: `element.innerHTML = ""` is fast, but the MutationObserver then delivers all 1000 removed `<tr>`s and `cleanupNodes` (lines 253–274) re-walks every removed subtree, popping ~4 unsubscribes per row and resetting metas. Measured microtask cost 2.34 ms vs Vue's whole clear at 0.58 ms. Real but modest (~+7 ms at 4× throttle in the harness). Note this teardown is *necessary* work for a subscription-based design (else atoms would retain detached DOM), but it's O(all DOM nodes) rather than O(subscribed nodes).

### 5.3 22_run-memory: 6.36 MB vs Vue 4.26 / Solid 3.11

Subtracting 21_ready-memory baselines (reatom 1.22, vue 1.32, solid 1.01), the per-1000-row retained cost is **reatom 5.14 MB vs vue 2.94 MB vs solid 2.10 MB** — ~2.2 KB/row more than Vue. Plausible inventory per row: 2 atoms (`label`, `selected`) with full `__reatom` records; 1 `reatomClassName` computed atom; ~4 JSX meta objects (`{boundary, subscribes[], unsubscribes[], mount, unmount, mounted}` — 2 arrays each) on `<tr>`, the label text node, and both `<a>`s; 2 bound event closures wrapping `jsxEvent`; linked-list symbol pointers on *both* the source node and the mapped `<tr>`; `reatomMap`'s WeakMap entry; unsubscribe closures. That's real per-row overhead of the fine-grained design, not a leak — 25_run-clear-memory (1.62 vs vue 2.05, react 2.49) shows cleanup releases everything correctly.

### 5.4 42_size-compressed: 10.4 KB vs Solid 4.5 / Preact 5.7

The built bundle (`frameworks/keyed/reatom-jsx/dist/assets/index-Cvkzi9Yd.js`) is 29,986 bytes raw / ~10.4 KB compressed. Tree-shaking works — `ErrorBoundary`, form bindings (`bindFieldModel`), `addCallHook` are absent from the bundle — so this is the irreducible cost of shipping the full runtime reactive-graph engine (frames, computed, actions) + linked list + JSX runtime, vs Solid's compile-away approach. One avoidable slice: `jsxEvent` (`@reatom/jsx/dist/index.js` lines 143–155) unconditionally calls `log.label(...)` per DOM event, pulling the `LOG` action machinery (`@reatom/core` `initLog`, lines 2696–2712) into every production bundle. Not a bug; a design trade-off plus a small trim opportunity.

### 5.5 Not suspicious after inspection

- **05_swap1k** — statistical tie; Reatom wins script and median (section 1).
- **03/04/06** — Reatom is at or near best; notably the MutationObserver only watches `childList`, so pure text/attribute updates (update, select) bypass it entirely, which is why Reatom's fine-grained path shows no such tax.

---

## 6. Concrete next optimizations, ranked by expected impact

1. **[Library, high impact — targets 01/02/07/08, the CPU geomean]** Kill or shrink the MO connect pass for bulk insertions. Options: (a) during `h()`/`walk`, collect meta-bearing nodes into a per-subtree registry so `connectNode` iterates ~4 metas/row instead of walking ~10 DOM nodes/row; (b) in `walkLinkedList`, when the host `element.isConnected`, connect new children's subscriptions eagerly right after the batched append (synchronously, in the same task) and let the MO handle only foreign/adopted nodes. Expected: cut the 11.9 ms microtask to a few ms; on the harness ≈ −10 ms on 01/02/08 and −80…100 ms on 07.
2. **[Library, medium]** Apply initial reactive prop values at render time instead of first-connect: the `class={() => …}` path currently defers computed creation + first evaluation + a guaranteed `className` DOM write (even for `undefined` → `''`) into the connect pass. Evaluate once via `peek` during `setProp` and skip the first synchronous `subscribe` emission. Saves 1 computed evaluation + 1 DOM write per row on create, and removes the pointless `class=""` mutation.
3. **[Library, medium — targets 22_run-memory]** Slim per-row plumbing: a lightweight "subscription" record for simple prop bindings instead of a full named computed atom per binding; lazily allocate the two meta arrays (most nodes need only `unsubscribes`); consider event delegation in `@reatom/jsx` to drop 2 listeners + 2 bound closures per row. Expected: pull run-memory from 6.4 toward ~4.5 MB.
4. **[Library, low-medium — targets 09]** Fast-path teardown for `clear`/version-jump: `walkLinkedList` already knows the whole list is going away; unsubscribe from the meta registry (idea 1a) instead of re-walking every removed DOM subtree.
5. **[Library, low — targets 42]** Gate `log.label` in `jsxEvent` behind `DEBUG`/dev builds so `LOG` machinery tree-shakes out of production bundles; audit other always-reachable debug surfaces.
6. **[Bench app, negligible]** `main.tsx` is already near-optimal (linked-list head/tail swap, batched `createMany`, `DEBUG.set(false)`). The `update` implementation's `rows.find` full scan is idiomatic and 03 is already competitive. Nothing to fix.
7. **[Measurement]** For swap/select/remove, report medians and treat sub-2 ms mean deltas as noise; consider more iterations or outlier trimming (Vue's run 10 alone reordered the swap ranking). Do **not** spend effort "optimizing" Reatom's swap — it is already the fastest script path in the field.

---

## 7. Verdict: real regression vs measurement artifact

| Observation | Verdict |
|---|---|
| Reatom swap (29.75) "slower" than Vue (29.37) | **Artifact.** Paint-floor benchmark (~90% layout/paint identical across frameworks, both do exactly 2 `insertBefore`); Vue's mean rests on one outlier run; medians and script both favor Reatom. O(1) swap is genuinely the fastest script implementation measured (0.02 ms vs Vue's 0.10 ms). |
| 01/02/07/08 create-family totals ~15–20% above Vue | **Real library regression.** Entirely script-side; caused by the MutationObserver-driven connect pass (full DOM walk + deferred subscribe + connect-time class evaluation/write per row). Paint is identical to Vue/Solid. Fixable per §6.1–6.2. |
| 09_clear1k +39% vs Vue | **Real, smaller.** Teardown walk over all removed DOM nodes. Fixable per §6.4. |
| 22_run-memory 6.36 MB (+49% vs Vue) | **Real.** ~2 KB/row of atoms/metas/closures — a fine-grained-reactivity cost, reducible per §6.3. No leak (25_run-clear-memory is healthy). |
| 42_size 10.4 KB (vs Solid 4.5) | **Real but by design.** Runtime graph engine ships in the bundle; tree-shaking already works; small trims available (§6.5). |
| 03/04/06 (update/select/remove) | **No issue — Reatom is at or near best in class**; 04_select is the best result of all five frameworks. |

**Bottom line:** the O(1) swap delivers exactly what it promises — the least script work and the fewest possible DOM operations — but on this benchmark the win is capped by a ~6.5 ms (real-time) browser layout/paint floor shared by every good implementation. The numbers that actually cost Reatom its geomean standing are the create/mount path (a genuine, fixable connect-pass overhead) and per-row memory, not the swap.

---

### Appendix: microbench method

- Static server over `/workspace`, Chromium via Playwright (from `webdriver-ts/node_modules`), production dist builds of each framework.
- Swap script-only: create 1k rows, 20 warmup swaps, then 200× `t0; button.click(); await null ×4 (flush microtask chains incl. Vue scheduler + Reatom MutationObserver); t1`. DOM-op counts captured by wrapping `insertBefore/appendChild/removeChild/append` for one swap.
- Create/clear script-only: same flush pattern, 5 warmup cycles, 25 measured cycles; Reatom additionally split into sync-handler vs microtask phase.
- Caveats: unthrottled, headless, different absolute scale from the harness (which adds 4× throttle on some benches and counts click-dispatch overhead in "script") — used only for *relative* framework comparison and phase attribution; relative ratios match the harness (e.g. create script reatom/vue ≈ 2.2 in microbench vs 1.96 in harness).
