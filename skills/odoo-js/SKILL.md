---
name: odoo-js
description: Use when writing, reviewing, or migrating any JavaScript, OWL component, QWeb/OWL template, or SCSS under static/src in an Odoo module — POS overrides, backend widgets, website/public widgets, services. Carries PSAE review-derived principles for OWL correctness, the patch() discipline, template conventions, frontend JS style, and cross-version upgrade gotchas, plus references for exact patterns. Invoke before writing anything under static/src/.
---

# Odoo JS

Write the frontend an experienced PSAE reviewer wouldn't comment on — so the review is about the feature, not the patch mechanics or the `let` that should have been a `const`. These are stances to carry, not a checklist to run: the exact forms live in `references/`, pulled only when you need the fact. When a task skips the planning pass (`odoo-grill`), these reflexes carry the design alone.

The single most repeated finding isn't a bug — it's *reaching around the framework*: querying the DOM instead of reading state, copy-pasting a standard method instead of calling `super`, reassigning `X.props` instead of `patch`-ing it. OWL gives you a declarative tool for almost everything you reach for imperatively; the stances below are mostly that one instinct applied.

## The stances

### Extend with `patch()`, and always reach `super`

Customisation is `patch()` — never reassignment, `Component.extend`, or the old `Registries` API — because it's the only form that unpatches cleanly and composes with other modules' patches; and `X.prototype.method.call(this)` is not a back door to "the original," since `patch()` mutated the prototype, so it hits the patched method anyway while *looking* like a bypass. Patch the layer you mean: prototype (methods, `setup`), the class itself (static members like `template`), and the props object are separate objects, so patching the prototype to add a static prop misses. Above all, find a way to call `super`: copy-pasting an inherited method's whole body to change three lines is the most dangerous frontend pattern, because the day Odoo hotfixes that method every customer not on your fork silently loses it — guard your special case, then `return super.method(...arguments)` for the rest. Don't override a pure getter to bolt on a side-effect (put the mutation in its own method), and change props plus the template before reaching for a whole new `Component` subclass — that subclass is a maintenance liability you rarely need. (Exact forms, super in async patches, version-renamed methods: `references/patching-services.md`.)

### Think in state and getters, not the DOM

A value you can *derive* from state, props, or record fields is a getter — not a value frozen into `this.x` in `setup()` (stale the moment state changes) and not a no-arg method (a getter with worse ergonomics). Getters recompute on every render and never go stale; "make this a getter" is the single most repeated component-level comment, and it applies to patched POS models (`PosOrder`, `PosOrderline`) exactly as to screen components. Push a complex `t-if`/`t-att-class` expression into a getter too, so the template stays declarative and the logic stays testable. The DOM itself is OWL's to own: no `document.getElementById`, no `$(...)`, no jQuery — reach an element with `useRef`+`t-ref` and react to events with `t-on-click`, never a manual `addEventListener`, because a query from the component root starts too high and breaks once two instances exist. And side-effects belong in lifecycle hooks, not `setup()`'s body — async load in `onWillStart`, DOM-dependent work in `onMounted`, cleanup in `onWillDestroy`; never override the whole `setup()` to inject one line, since hooks from multiple patches stack while a full override clobbers. (`references/owl-components.md`.)

### Put service-needing logic where its dependencies live

Acquire each dependency once with `useService` (or the domain hook like `usePos()` when one exists), not `this.env.services.x` scattered through methods — and check what the core component already assigns before re-acquiring, or it's dead code (`TicketScreen` already exposes `this.dialog`). Services never travel as arguments: a method on a POS *record* that needs `orm`, notifications, or `env.utils` is in the wrong layer, and passing them in from the caller (`order.applySettlement({ orm, pos })`) just hides that. The fix is layering — records keep pure data mutations; shared service-needing business logic goes on the store (`patch(PosStore.prototype)`, where `this.data`, `this.models`, `this.env` already live, as core `pos_loyalty` does it); the component stays a thin UI orchestrator passing plain data. Call models through `orm`, reserving bare `rpc(route, params)` for actual controller routes. A user-triggered call that can fail needs a *visible* failure — surface an error dialog, never let a rejected promise vanish — and fire independent calls with `Promise.all` rather than awaiting in sequence. Load POS master data through the Python loader so it rides POS's one RPC, not a per-screen `orm.call`. (`references/patching-services.md`.)

### Keep templates declarative, styles in classes

The template renders state; it neither computes nor stores. Use `t-out`, never the deprecated `t-esc`. Conditional classes take the object form (`{ 'text-danger': hasError }`), not a ternary that concatenates class strings — and a getter returning that object reads even better. Hide a standard element with `d-none`, never `t-if="False"` or `position="replace"`: deleting a node another module's xpath depends on breaks that module silently, so add the class via `position="attributes"` and insert your variant `position="after"` (a *component* tag can't take `d-none` — its class becomes a prop — so use `t-if` there instead). xpath on structure, not text: `hasclass('foo')` over an exact-string `@class='foo'`, always with `t-inherit-mode` set. No inline `style=` and no `data-*` on OWL templates — inline style isn't cached and costs refresh time, and a `data-*` attribute almost always means you're about to read it back out of the DOM, which means it should have been state. For styling, reach for an existing Bootstrap utility first, write SCSS only when none fits, never inline (and Bootstrap's grid isn't loaded in POS — `row`/`col-*` do nothing there). (`references/templates-styling.md`.)

### The Odoo-specific JS reflexes

General JS style is the formatter's job and a competent baseline; these are the few reflexes that are *Odoo-specific* or are traps the framework sets. Files are ES6 modules — `import { x }`, never `require` or `odoo.define` (an `/** @odoo-module **/` header opts a file into the module system on versions that still need it). Prefix a variable with `$` only when it actually holds a jQuery object, never a plain `HTMLElement`. Every user-facing string passes through `_t()`, or it can't be translated. Extract with `map` *then* fold with `reduce` — one job per stage; a single callback that does both draws the stock "map + reduce" comment — and index a repeated lookup into a `Map` rather than nesting an O(n²) search. The trap carried over from Python: an empty array is *truthy* in JS, so `if (list)` doesn't test emptiness — use `list.length`. (`references/js-conventions.md`.)

### JS upgrades break silently — read both versions first

JS is where upgrades break *quietly*: import paths, registry categories, component names, and bundle names all move between major versions with no error — just a feature that stops working — so read the component in *both* the old and the target source before touching any JS file in a migration (→ `odoo-migrations`). The recurring traps: `publicWidget` is gone in v19 → the `Interaction` class (the `events` hash becomes `dynamicContent`/`addListener`, async work moves from `start()` into `willStart()`, and there's no `super`). A core method you wrap may have switched jQuery→native and now *throws* where it used to no-op — v18 `$('sel').append(x)` on an empty match vs v19 `querySelector('sel').append` throwing on null — so guard the container before delegating through; asset/patch order won't fix it. And deep state mutation may be missed by OWL 3's batched scheduler (v19): reassign `this.state.config = { ...this.state.config, key: x }` rather than writing `this.state.config.key = x` in place.

## References (consult for the fact, don't memorize)

| Need the exact... | Read |
|---|---|
| OWL component anatomy — getters vs state, `useRef`/`t-ref`, lifecycle hooks, prop shapes | `references/owl-components.md` |
| `patch()` rules (prototype/static/props), calling super, services, ORM/RPC, registries | `references/patching-services.md` |
| QWeb/OWL template directives, conditional classes, inheritance (`t-inherit`, xpath, `d-none`), SCSS | `references/templates-styling.md` |
| JS style — const/naming/imports, array idioms, guards, async, `_t` | `references/js-conventions.md` |

For where frontend files live and how assets are declared in the manifest, see `odoo-module-development`. For XML *backend* views (not OWL templates), see `odoo-xml-conventions`. For the Python side of a POS data loader or controller, see `odoo-python`.
