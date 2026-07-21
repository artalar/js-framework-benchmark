# Closure-style analysis: `@reatom/jsx` `src/index.ts` (V8 / JS-engine view)

**Source:** `/workspace/reatom/packages/jsx/src/index.ts` at `origin/v1001` `3606a78` (npm `1001.3.0`) — stock published code, no opt patches.
**Bench context:** js-framework-benchmark keyed table, 1k–10k rows. Per row: 1 reactive class getter, 1 label-atom child, 2 `on:click` handlers, per-node `meta.subscribes`/`unsubscribes` arrays. Prior conclusion (see `OPT_SUMMARY.md`): remaining create-script gap vs Vue is per-row atom/computed/`bind` construction, not MutationObserver walks. This report explains *where* those allocations come from at the closure level and which are worth changing.

---

## Executive summary

Per benchmark row, the stock library allocates roughly **36 function objects** (~32 of them retained for the row's lifetime), **~11 heap Contexts** that survive create (plus ~6 transient ones), **5 `Function.prototype.bind` exotic objects**, 4 meta objects with 8 arrays, one full computed atom (`classNameAtom`), and ~10 strings. Ballpark retained size ≈ **2.5–3.5 KB/row**, which is consistent with the observed `22_run-memory` gap (Reatom ~6.7 MB vs Vue ~4.2 MB ⇒ ~2.5 MB/1k rows of extra retention).

Three sites dominate:

1. **`setProp` allocates a heap Context on *every* call — including static attributes.** Verified via `--print-bytecode`: `CreateFunctionContext` is emitted in the prologue (bytecode offset 0) because `marked`/`onPropError`/`setter`/`name`/`listener` and all four parameters are context-allocated somewhere in the function. 6 of the 9 `setProp` calls per bench row are static strings (`class="col-md-1"`, `aria-hidden`) that create zero closures, yet each pays a Context allocation + 4 slot stores. These die young (scavenger), so this is allocation-rate/CPU, not retained memory.
2. **Each `on:` handler materializes 5 function objects + 2 Contexts + eager name strings** (user arrow, `jsxEvent` wrapper arrow, `bind()` bound function, subscribe thunk, mount-time removal arrow, plus `eventActionName`'s 3+ string allocations even with `DEBUG` off). Two handlers per row ⇒ ~10 retained function objects/row from events alone.
3. **Each reactive `class` getter builds a full computed atom at mount** (`reatomClassName` → `computed()` → `createAtom`/`castAtom`): ~7 function objects (atom target closure, `set` method, `subscribe.bind(target)`, `toString`, `toJSON`, the per-call `.extend()` arrow, the `parseClasses` arrow), plus `__reatom` object, a fresh 3-element middlewares array, a frame with `pubs`/`subs` arrays, a `named()` string, and a `defineProperty` descriptor. Then `.subscribe()` in core adds 4 more function objects. This is the single largest per-row block and matches the prior "atom/computed/bind construction" conclusion — but most of it lives in `@reatom/core`, not in `index.ts`.

---

## Hot-path inventory (per benchmark row unless noted)

| Site (line, `index.ts`) | Function objects created | Captures (context slots) | Lifetime | Cost rank |
|---|---|---|---|---|
| User `Row` arrows (bench code) | 3 (class getter, 2 click arrows) | shared Row Context: `row` | row lifetime (pinned via meta/listeners) | inherent |
| `setProp` static attr path (773–784 prologue; 6 calls/row) | 0 | Context still allocated: `dom, element, key, value, marked, onPropError, setter` | dies in nursery | **high (alloc rate)** |
| `setProp` `on:` branch (743–766; ×2/row) | 3 each: wrapper arrow (751), `bind()` bound fn (750–753), subscribe thunk (761) + removal arrow at mount (763) | function ctx: `element, value, key`; block ctx: `name, listener` | row lifetime (listener on DOM + `meta.subscribes`) | **high** |
| `setProp` reactive class (773–795) | 3: `onPropError` (774), `setter` (777), subscribe thunk (788) | one shared ctx: `dom, element, key, value, marked, onPropError, setter` | row lifetime (`meta.subscribes` + atom `frame.subs`) | **high** |
| `reatomClassName` at mount (utils.ts 7–10 → core `computed`) | ~7 (see summary) + 4 from core `subscribe` | `value` (user getter); core: `target, setup, name`; sub: `frame, errored, userCb, errorCb, parentFrame` | row lifetime | **highest** |
| `walkAtom` fast path for label (313–358) | 2 at render: `onError` (330), subscribe thunk (334); at mount: update arrow (335) + 4 from core `subscribe` | ctx: `state` (mutated), `marked`, `textNode`, `anAtom`, `dom`, `onError` | row lifetime | med-high |
| `eventActionName` (79–88; ×2/row) | 0 | — | strings retained by wrapper arrow | med (strings, eager) |
| `ensureMeta` (121–130; 4 nodes/row: tr, 2×a, label Text) | 0 | 1 object + 2 arrays each | row lifetime | med |
| `connectNode` lifecycle arrows (215; 4 subs/row) | 4 short-lived `() => meta.unsubscribes.push(subscribe())` | `meta`, `subscribe` | one MO batch | low-med (alloc rate: 4k/1k-rows) |
| `walkLinkedList` (415–511) | 2 per *list* (`cb`, subscribe thunk); DocumentFragment batches per update | `element, dom, lastVersion` (mutated) | list lifetime | negligible |
| `createLiveFragment.update` (535–563) | 1 per *fragment* (≈1 total in bench) | `dom, fragment, start, end, name, marked` | fragment lifetime | negligible |
| `walkTree` visitors in `connectNode`/`cleanupNodes`/`mount` (207–274, 982–999) | 1–2 per mutation *batch* | `symbol, nodesToMount`/`metaNodes` | one batch | negligible |
| Core `subscribe` per delivery (atom.ts 738) | 1 `_enqueue` arrow per state change delivered | `frameSnapshot, state, userCb` | one microtask | negligible for create; linear in update benches |
| `jsxEvent` action call per click (90–105 + core `actionMiddleware`) | 1 cleanup arrow + `{params, payload}` + array copy per call, ×2 (`log.label` is also an action) | — | one microtask | per-event only |

**Count per row:** render-time 14 function objects (incl. 2 bound) + 11 Contexts (≈6 die young); mount-time ~22 function objects (incl. 3 bound, ≈4 short-lived). Total ≈ **36 fn objects, ~11 retained Contexts, 5 bound functions**.

---

## V8 mechanics, tied to the code

**1. Contexts are allocated in the prologue, not at closure creation.**
Ignition emits `CreateFunctionContext` as the first bytecode when any variable in the function scope is captured by any inner function literal. In `setProp`, the parameters `dom/element/key/value` and the lets `marked/onPropError/setter` are captured by the arrows at lines 751, 761–763, 774–784, 788, 813, 815 — so *every* call allocates the Context and copies the 4 params into it, even the `key === 'children'` early return. Verified on a `setProp`-shaped repro:

```text
0 : CreateFunctionContext [0], [7]
3 : PushContext r1
5 : Ldar a0 / StaCurrentContextSlot [5]   ; dom → context
9 : Ldar a1 / StaCurrentContextSlot [4]   ; element → context
...
```

The `on:` branch's `let name`/`let listener` (745, 750) are block-scoped and captured, so that branch pays a second (block) Context. The same prologue rule applies to `walkAtom` (captures `state/marked/textNode/anAtom/dom`) and `walkAtomFragment` — but those only run for reactive children, where the closures are genuinely needed.

**2. Function objects are cheap-ish; contexts and bound functions add up.**
With pointer compression, a JSFunction is ~28–32 B (map, properties, elements, SFI, context, feedback cell, code). Closures instantiated from the same literal **share the SharedFunctionInfo and feedback cell**, so 1000 rows' worth of `setter` arrows share one feedback vector and one IC state — there is *no* per-row IC or compilation cost, only the object + context. A Context is ~(4 + slots) tagged words. `bind()` (used at 750 and in core `subscribe`'s return, atom.ts 775) creates a JSBoundFunction — an extra object plus a slightly slower call path (the Call builtin unpacks bound receiver/args; TurboFan can inline it, but the DOM event dispatch entry point is never inlined). Per row: 2 bound event listeners + `subscribe.bind(target)` in `castAtom` + 2 bound unsubscribes = 5.

**3. Lifetime: `meta.subscribes` pins render-scope Contexts until teardown.**
`unlink` (167–171) stores the subscribe thunk in `meta.subscribes` for the node's whole life (needed for reconnect; `cleanupNodes` 263–272 only releases them on real disconnect). Each thunk pins its `setProp`/`walkAtom` Context, which transitively pins `element`, `key`, `value` (the user getter → `row` → both row atoms). This is the designed ownership model and it's why closures created at render survive into `22_run-memory`. Nothing leaks — but every function object you avoid creating at render is a function object that doesn't sit in the heap snapshot.

**4. Hidden classes / ICs are mostly fine.**
`ensureMeta`'s object literal (122–129) has a single shape everywhere → monomorphic loads in `connectNode`/`cleanupNodes`. The symbol-keyed expando `(node)[metaSymbol()]` makes DOM wrapper maps transition once per node interface; the walk-time `(visited)[symbol]` load is polymorphic across `HTMLTableRowElement`/`HTMLAnchorElement`/`Text`/… — inherent to the design and already measured as non-dominant (prior MO-walk experiments). `set()` (826–894) does megamorphic keyed stores (`element[key] = value`) — inherent to a generic prop setter; every framework's generic path looks like this.

**5. Eager strings with `DEBUG` off.**
`eventActionName` (79–88) always runs at render: `nodeName.toLowerCase()` + 2 template literals per handler. `reatomClassName` always calls `named('classNameAtom')` → counter string per row. `jsxEvent` (92) then runs `/\._/.test(name)` + `log.label(...)` per click — `log.label` is itself an action, so each click pays two action frames (`actionMiddleware`, atom.ts 986–994: `[...state, {params, payload}]` copy + `_enqueue` cleanup closure each). Create-path cost is the strings; click-path cost is the double action machinery.

---

## Ranked recommendations

### High

**R1. Split `setProp` so the static path allocates no Context.**
Move everything from line 773 down (`marked`/`onPropError`/`setter` + the reactive branches) into a separate `setReactiveProp(dom, element, key, value)` and keep `setProp` capture-free (early returns, `on:` dispatch, static `setter(value)` → direct `set(...)` call). The `on:` branch should likewise live in its own function so its block Context doesn't ride along. Effect: removes ~6 Context allocations + ~24 slot stores per bench row from the create path (60k stores at 10k rows). Expected: **1–3% of create script** (alloc rate + prologue work), no run-memory change (those contexts already die young). Zero semantic risk.

**R2. Shared event dispatcher instead of per-handler `arrow + bind() + thunk`.**
Today each `on:` prop creates: wrapper arrow (751) + `bind()` bound function (750) + subscribe thunk (761) + removal arrow (763), and registers a distinct listener per (element, event). Replace with:
- store handlers on the meta: `meta.handlers ??= {}; meta.handlers[key] = value` (one record per element with handlers);
- register **one module-scope listener** per (element, type): `element.addEventListener(key, sharedDispatch)` where `sharedDispatch` is a single prebound function created once at module load: `let sharedDispatch = (event) => rootFrame.run(dispatchJsxEvent, event)` — all handlers already bind to `top().root.frame` (the fix noted at 746–749), which is a module-lifetime constant in single-context apps, so one prebound dispatcher preserves the exact frame semantics;
- `dispatchJsxEvent` reads `ensureMeta(event.currentTarget).handlers[event.type]` and calls `jsxEvent(name, event, node, handler)` as today (name computed here — see R5).
Unlink/`$spread` semantics survive: overwriting `meta.handlers[key]` replaces the stale handler (same as the current listener disposal), and `removeEventListener(key, sharedDispatch)` on teardown stays possible via one shared removal thunk shape. Saves ~4 function objects + 1 bound function + 1 block Context per handler ⇒ ~10 objects/row. Expected: **~0.2–0.4 MB run-memory per 1k rows** and a few % of create script (2 handlers/row × 10k rows = 80–100k avoided allocations in `07_create10k`).

**R3. Trim per-atom construction cost for `classNameAtom` (mostly a `@reatom/core` follow-up).**
The reactive class getter is the most expensive binding: `computed()` per row means `castAtom`'s `Object.assign` with a fresh `set` closure, `subscribe.bind(target)`, `toString`/`toJSON` arrows, a fresh `middlewares` array, plus `computed()`'s own per-call `.extend((target) => {...})` arrow (core/atom.ts 1355) and a `named()` string. Concrete shapes, in increasing invasiveness:
- (jsx, tiny) hoist `computed()`'s extend callback: it is identical for every computed — in core, define it once at module scope (atom.ts 1355).
- (jsx) skip `named('classNameAtom')` when a name isn't needed (prior Step 3 measured "real, small"); `createAtom` already skips `defineProperty` for falsy names.
- (core) move `toString`/`toJSON`/`set` to a shared delegate or install lazily; replace `subscribe.bind(target)` with a shared function reading its target from the callee — each removes one object *per atom ever created*, which at 3 atoms/row (label, selected, className) is 1k–30k objects per bench run.
Expected: this is the **largest remaining lever** for both create script and run-memory (matches the prior campaign's conclusion), but the wins land in core, not `index.ts`.

### Medium

**R4. Hoist `reatomClassName(value)` out of the subscribe thunk (788–790).**
The computed is currently constructed *inside* the thunk, so a disconnect→reconnect cycle silently builds a brand-new atom (old one becomes garbage, state resets). Creating it once per prop and capturing it in the thunk keeps reconnect cheap, drops the re-construction, and makes the memory profile flatter for list reordering scenarios. Neutral for the create bench (mount always follows render), fixes a subtle behavior quirk.

**R5. Build event action names lazily at dispatch time.**
`eventActionName` allocates ~3 strings per handler at render even with `DEBUG` off, and `jsxHName.current` is only correct at render time — so capture the *component name* (one string reference, no concat) and compose the full name inside `jsxEvent`/the dispatcher only when actually logging (the `/\._/` gate and `log.label` are the only consumers). ~6 string allocations/row off the create path; log output unchanged per click.

**R6. Remove the per-subscription `lifecycle` arrows in `connectNode` (215).**
`lifecycle('mount', visited, () => meta.unsubscribes.push(subscribe()))` allocates one arrow per subscription per connect — ~4k transient arrows for create1k, ~40k for create10k. Change `lifecycle` to accept the function and its argument (`lifecycle(phase, node, fn, arg)`) or inline the try/catch (V8 no longer deoptimizes functions containing try/catch). Pure alloc-rate win; scavenger-only, so expect small but non-zero create-script effect at 10k.

### Low

- **`setter`/`onPropError` fusion (773–784):** the pair shares one Context, so merging them into one closure that takes an ok/error flag saves only one JSFunction (~32 B) per reactive prop. Do it only if touching the code for R1 anyway.
- **`walkAtom` label path:** the mutable `state` capture and update arrow (335–354) are the minimal correct shape (needs `Object.is` against arbitrary previous states for the fragment-upgrade path). Leave as is; the real cost of a label is core `subscribe` (4 objects), not this arrow.

---

## What NOT to change

- **Module-scope `let fn = (…) => …` style** (`jsxElementKey`, `eventActionName`, `isSkipped`, `walk`, `set`, `setStyleProp`, …): each is allocated once per module load. Arrow-vs-`function` here has no engine cost difference that matters.
- **`walkTree` visitor closures** in `connectNode`/`cleanupNodes`/`mount.processMutations`: one per mutation *batch*, not per node. Prior experiments already showed MO-walk cost is secondary.
- **`walkLinkedList`'s `cb` (422) and its `lastVersion` mutable capture:** one Context per *list*. The `DocumentFragment` append batches (425, 437) are short-lived nursery objects — batching is a win, don't unbatch it. The capture of `element` is required.
- **`createLiveFragment.update` (540):** one closure per fragment; the bench has ~1. General apps have one per reactive non-primitive child — acceptable ownership shape.
- **`ensureMeta`'s literal:** single hidden class, monomorphic everywhere it's read. Don't split it or lazy-init fields individually (prior "slim meta" step already captured the available win); a `null`-vs-array polymorphism on `subscribes` would *pollute* the ICs in `connectNode`/`cleanupNodes`.
- **Closure-per-row user code (`() => selectRow(row)`)**: 3 arrows sharing one Context per row is the minimal JSX contract; frameworks that beat this (Solid) do it with compile-time delegation, not runtime style.
- **Feedback/IC worries about "thousands of closures":** V8 shares SFI + feedback vectors across closure instances from the same literal. There is no per-row IC pollution, warm-up, or code duplication — the cost is purely object + Context allocation and their retention. Don't restructure to classes "for the ICs".
- **`Object.is(state, (state = newState))` (337, 408):** allocation-free compare-and-update; keep.
- **`jsxEvent` being an action:** per-click action machinery (array copy + cleanup closure ×2 with `log.label`) is measurable per event but irrelevant to create/memory benches; it buys real observability. If ever revisited, gate `log.label` behind connected-logger detection rather than removing the action.

---

## Bottom line

Estimated ≈36 function objects, ≈11 retained Contexts, 5 bound functions, ≈2.5–3.5 KB retained per row. The create-script and run-memory gaps vs Vue are explained by (a) per-atom construction in core (`classNameAtom` + `castAtom` shape — R3), (b) per-handler event closure stacks (R2), and (c) prologue Context churn in `setProp` (R1). R1+R2 are implementable inside `packages/jsx/src/index.ts` without semantic change; R3 needs core.
