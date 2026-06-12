---
name: odoo-js
description: Use when writing, reviewing, or migrating any JavaScript, OWL component, QWeb/OWL template, or SCSS under static/src in an Odoo module — POS overrides, backend widgets, website/public widgets, services. Carries PSAE review-derived principles for OWL correctness, the patch() discipline, template conventions, frontend JS style, and cross-version upgrade gotchas, plus references for exact patterns. Invoke before writing anything under static/src/.
---
# Odoo JS

Write the frontend an experienced PSAE reviewer wouldn't comment on — so the review is about the feature, not the patch mechanics or the `let` that should have been a `const`. These principles are distilled from hundreds of real recurring review comments on PSAE customer code (POS overrides, backend widgets, website widgets).

The single most repeated finding is not a bug — it's *reaching around the framework*: querying the DOM instead of using state, copy-pasting a standard method instead of calling `super`, reassigning `X.props` instead of `patch`-ing it. OWL gives you a declarative tool for almost everything you're tempted to do imperatively. Use it.

## Principles

### Extend with `patch()`, and always reach `super`

- **Customisation is `patch()`, not reassignment or prototypal tricks.** Reassigning `X.props`, `Component.extend(...)`, `MainComponent.components = {...}` — the reviewer rewrites all of these to `patch(...)` by reflex. It's the only form that unpatches cleanly and composes with other modules' patches. And `X.prototype.method.call(this)` is not a path to "the original" — `patch()` mutates the prototype, so that hits the patched method anyway while *looking* like it bypasses the chain; write `this.method()` or reach `super`.
- **Patch the layer you mean — they're separate objects — and one patched class per file.** The prototype for methods and `setup`, the class itself for static members like `template`, the props object for props. Patching the prototype to add a prop is wrong — `props` is static. Name the file after the core file it patches: a `PosOrderline` patch doesn't live in `pos_order.js`. (Exact forms in references.)
- **Always find a way to call `super`.** Copy-pasting an inherited method's whole body to change three lines is the most dangerous frontend pattern: the day Odoo ships a hotfix to that method, every customer not on your fork silently loses it. Guard your special case, then `return super.method(...arguments)` for the rest. "I couldn't see how to call super" means look harder.
- **Never override a pure function to introduce a side-effect.** If a getter returns a value and nothing else, don't override it to also mutate state. Put the side-effect in its own method and call it explicitly.
- **Change props/template before you reach for a whole new component.** Most "I extended `OrderReceipt` into a new component" diffs should be `patch(OrderReceipt.props, ...)` plus a `t-if` in the inherited template. A new `Component` subclass is a maintenance liability you rarely need.
- **Load POS master data through the Python loader, not an extra JS RPC.** Override the server-side loader so your data rides the one RPC POS already makes, instead of per-screen `orm.call`s.

### Think in state and getters, not the DOM

- **A value derived from state, props, or record fields is a getter — not a stored field, not a no-arg method.** `get total()` recomputes on every render and never goes stale; storing it in `this.x` in `setup()` is wrong the moment state changes. Applies to patched POS models exactly as to components — `_getTierProgram()` on `PosOrder` draws the same comment as on a screen. "make this a getter" is the most repeated component-level comment.
- **Compute template conditions in the component, not inline in XML.** A complex `t-if` or `t-att-class` expression belongs in a getter the template *reads*. The template stays declarative and the logic stays testable.
- **The DOM is OWL's job — don't touch it.** No `document.getElementById`, no `$(...)`, no jQuery. Reach an element with `useRef` + `t-ref`; react to a click with `t-on-click`, never a manual `addEventListener`. Mixing legacy widgets and OWL components in one flow is always a smell.
- **Side-effects belong in lifecycle hooks, not `setup()`'s body.** Async load in `onWillStart`, DOM-dependent work in `onMounted`, cleanup in `onWillDestroy`. Don't override the entire `setup()` to inject one line — add a hook; hooks stack, a full override doesn't.

### Services, ORM and RPC

- **Acquire dependencies with `useService`, and prefer the domain hook.** `useService("orm")`, not `this.env.services.orm` scattered through the methods; when a hook like `usePos()` exists, use it. When patching `setup`, check what the core component already assigns — `TicketScreen` already exposes `this.dialog`; re-acquiring it is dead code.
- **Services never travel as arguments — put the logic where its dependencies live.** A method on a POS *record* (`PosOrder`, `PosOrderline`) that needs `orm`, notifications, or `env.utils` is in the wrong layer; passing them in from the caller (`order.applySettlement({ orm, pos, ... })`) just hides that. Models keep pure data mutations; shared service-needing business logic goes on the store (`patch(PosStore.prototype)` — `this.data.call`, `this.models`, `this.env` are already there, as core `pos_loyalty` does it); the component stays a thin UI orchestrator passing plain data.
- **Call models through `orm`, not legacy `rpc({model, method, args})`.** Reserve bare `rpc(route, params)` for actual controller routes.
- **A user-triggered RPC that can fail needs a visible failure.** Wrap it and surface an error dialog — never let a rejected promise vanish silently. Fire independent calls with `Promise.all` instead of awaiting in sequence.

### Templates are declarative — keep the logic out

- **`t-out`, never `t-esc`** — `t-esc` is deprecated and every reviewer flags it.
- **Conditional classes use the object form** — `t-att-class="{ 'text-danger': hasError }"`, not a ternary that concatenates class strings. A getter returning the object reads even better.
- **Hide standard elements with `d-none`, never `t-if="False"` or `position="replace"`.** Removing a node another module's xpath depends on breaks that module silently — add the class via `position="attributes"` and insert your variant `position="after"`. A *component* tag can't take `d-none` (class becomes a prop) — add `t-if="!cond"` onto the existing node instead.
- **xpath on structure, not text** — `hasclass('foo')` over `@class='foo'` (exact-string match); always set `t-inherit-mode` and keep the expression as short as it can be while staying unambiguous.
- **No inline `style=`, no `data-*` attributes on OWL templates.** Inline style isn't cached and costs refresh time. A `data-*` attribute almost always means you're about to read it back out of the DOM — which means you should have used state.

### SCSS over inline styles

- **A repeated visual tweak is a class — probably already a Bootstrap utility.** Reach for an existing utility first, write SCSS only when none fits, never inline. (Bootstrap's grid isn't loaded in POS — `row`/`col-*` do nothing there.)

### House JS conventions the reviewer enforces

These recur on nearly every PR — house style, not deep framework knowledge. Get them right once and review moves on to the logic: **`const` by default, `let` only when you reassign, never `var`**; **`camelCase`** (PascalCase for classes only, `$`-prefix only for a jQuery object); **ES6 `import { x }`**, never `require` or `odoo.define`; **the semantic array method, one job per stage** (`map`/`filter`/`flatMap`/`reduce`/`some`) over a hand-rolled loop — extract with `map` *then* fold with `reduce`, never both in one callback ("map + reduce" is a stock comment) — index a repeated lookup into a `Map` instead of nesting an O(n²) search, and snapshot (`[...list]`) a live recordlist before removing from it; **guard clauses** over nested `if`/`else`; **strict `===`**, remembering an empty array is *truthy* (`if (list)` doesn't test emptiness — use `list.length`); **`_t()` on every user-facing string**; **no `debugger`, no stray `console.log`, no dead code**. → `references/js-conventions.md` for the rationale and edge cases.

### JS upgrades break silently — read both versions first

- **JS is where upgrades break quietly.** Import paths, registry categories, component names, and bundle names all move between major versions, usually with no error — just a feature that stops working. Before changing any JS file in a migration, read the component in *both* the old and the target source to see how it moved.
- **`publicWidget` is gone in v19 → `Interaction`.** Every file importing `@web/legacy/js/public/public_widget` migrates to the `Interaction` class: the `events` hash becomes `dynamicContent`/`addListener`, async work moves from `start()` into `willStart()`, and there is no `super` to call.
- **A core method you wrap may have switched jQuery→native and now *throws*.** v18 `$('sel').append(x)` no-oped on an empty match and searched the whole document; v19 `this.el.querySelector('sel').append(x)` throws on null and only searches the interaction's root. An override that delegates to such a method inherits the throw — guard the container before calling through. Asset/patch order won't fix it.
- **Deep state mutation may not re-render in OWL 3 (v19).** `this.state.config.key = x` can be missed by the batched scheduler — reassign (`this.state.config = { ...this.state.config, key: x }`) instead.

## References (consult, don't memorize)

| Need...                                                                                                                                                    | Read                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| OWL component anatomy — getters vs state, `useRef`/`t-ref`, lifecycle hooks, prop shapes                                                                    | `references/owl-components.md`    |
| The `patch()` rules (prototype/static/props), calling super, services, ORM/RPC, registries                                                                 | `references/patching-services.md` |
| QWeb/OWL template directives, conditional classes, inheritance (`t-inherit`, xpath, `d-none`), SCSS                                                         | `references/templates-styling.md` |
| JS style the reviewer enforces — const/naming/imports, array idioms, guards, async, `_t`                                                                    | `references/js-conventions.md`    |

For where frontend files live and how assets are declared in the manifest, see `odoo-module-development`. For XML *backend* views (not OWL templates), see `odoo-xml-conventions`. For the Python side of a POS data loader or controller, see `odoo-python`.
