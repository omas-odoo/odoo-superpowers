---
name: odoo-js
description: Use when writing or reviewing any JavaScript, OWL component, QWeb/OWL template, or SCSS under static/src in an Odoo module — POS overrides, backend widgets, website/public widgets, services. Carries PSAE review-derived principles for OWL correctness, the patch() discipline, template conventions, and frontend JS style, plus references for exact patterns. Invoke before writing anything under static/src/.
---
# Odoo JS

Write the frontend an experienced PSAE reviewer wouldn't comment on — so the review is about the feature, not the patch mechanics or the `let` that should have been a `const`. These principles are distilled from hundreds of real recurring review comments on PSAE customer code (POS overrides, backend widgets, website widgets).

The single most repeated finding is not a bug — it's *reaching around the framework*: querying the DOM instead of using state, copy-pasting a standard method instead of calling `super`, reassigning `X.props` instead of `patch`-ing it. OWL gives you a declarative tool for almost everything you're tempted to do imperatively. Use it.

## Principles

### Extend with `patch()`, and always reach `super`

- **Customisation is `patch()`, not reassignment or prototypal tricks.** `import { patch } from "@web/core/utils/patch"`. Never `X.props = {...X.props, foo}`, never `Registries.Component.extend(...)`, never `MainComponent.components = {...}`. The reviewer rewrites all of these to `patch(...)` by reflex — it's the only form that can be cleanly unpatched and that composes with other modules' patches.
- **Patch the right layer, separately.** `patch(X.prototype, {...})` for methods/`setup`; `patch(X, {...})` for static members like `template`; `patch(X.props.line.shape, {...})` for props; `patch(X.components, {...})` for sub-components. `props` is a *static* property — patching the prototype to add a prop is wrong.
- **Always find a way to call `super`.** Copy-pasting an inherited method's whole body to change three lines is the most dangerous frontend pattern: the day Odoo ships a hotfix to that method, every customer who isn't on your fork silently loses it. Guard for your special case, then `return super.method(...arguments)` for everything else. "I couldn't see how to call super" means look harder — introduce a temp flag, override a smaller hook, anything.
- **Never override a pure function to introduce a side-effect.** If `get_cashier()` returns a value and nothing else, don't override it to also mutate state. Put the side-effect in its own method and call it explicitly.
- **Change props/template before you reach for a whole new component.** Most "I extended `OrderReceipt` into a new component" diffs should be `patch(OrderReceipt.props, ...)` plus a `t-if` in the inherited template. A new `Component` subclass is a maintenance liability you rarely need.
- **Load POS master data through the Python loader, not an extra JS RPC.** Extra per-screen `orm.call`s add network round-trips the standard single-load avoids. Override the server-side loader (`_load_pos_data` / `_pos_data_process`) so your data rides the one RPC POS already makes.

### Think in state and getters, not the DOM

- **A value derived from state or props is a getter, not a stored field or a method.** `get total() { return ... }` recomputes on every render and never goes stale. Storing it in `this.x` in `setup()` means it's wrong the moment state changes; making it a no-arg method (`computeTotal()`) is just a getter with worse ergonomics. "make this a getter" is the most repeated component-level comment.
- **Compute template conditions in the component, not inline in XML.** A complex `t-if` or `t-att-class` expression belongs in a getter the template *reads* — `t-att-class="badgeClass"` with `get badgeClass()` in the JS. The template stays declarative and the logic stays testable.
- **The DOM is OWL's job — don't touch it.** No `document.getElementById`, no `$(...)`, no jQuery. To reach an element, `useRef("name")` + `t-ref="name"` in the template. To react to a click, `t-on-click` in the template, not `useListener` (deprecated) and not a manual `addEventListener`. Mixing legacy widgets and OWL components in one flow is always a smell.
- **Side-effects belong in lifecycle hooks, not in `setup()`'s body or after render.** Async data loading goes in `onWillStart`; DOM-dependent work in `onMounted`; cleanup (`clearInterval`, listeners) in `onWillDestroy` (or return a teardown from `useEffect`). Don't override the entire `setup()` to inject one line — add a hook instead; hooks stack, a full override doesn't.

### Services, ORM and RPC

- **Acquire dependencies with `useService`, and prefer the short hook.** `this.orm = useService("orm")`, `this.dialog = useService("dialog")` — not `this.env.services.orm` scattered through the methods. When a domain hook exists (`usePos()` → `this.pos`), use it instead of `this.env.services.pos`.
- **Call models through `orm`, not legacy `rpc({model, method, args})`.** `this.orm.call("res.partner", "write", [...])`. Reserve bare `rpc(route, params)` for actual controller routes.
- **A user-triggered RPC that can fail needs a visible failure.** Wrap it (`try/catch`, or `.catch(err => ({ err }))` then a guard) and surface an `AlertDialog`/`ErrorDialog` — never let a rejected promise vanish silently. For independent calls, fire them with `Promise.all` instead of awaiting in sequence.

### Templates are declarative — keep the logic out

- **`t-out`, never `t-esc`.** `t-esc` is deprecated; every reviewer flags it. (And `t-out` is for *values* — a literal `<td>Total</td>` needs no directive at all.)
- **Conditional classes use the object form.** `t-att-class="{ 'text-danger': hasError }"` — not a ternary that concatenates class strings. Multiple conditions read cleanly; a getter returning the object reads even better.
- **Hide standard elements with `d-none`, not `t-if="False"` or `position="replace"`.** Removing a node another module's xpath depends on breaks that module silently. Add a `d-none` class (or `invisible`) so the node stays in the tree. Same reason `position="replace"` is banned — target the parent and *add*.
- **xpath on structure, not on text.** Use `hasclass('foo')`, not `@class='foo'` (which is an exact-string match); avoid predicating on `@t-if='...'` string contents. Always set `t-inherit-mode` and keep the expression as short as it can be while staying unambiguous.
- **No inline styles, no `data-*` attributes on OWL templates.** Inline `style="..."` doesn't get cached and costs refresh time — move it to SCSS or a Bootstrap utility class. A `data-*` attribute on an OWL node almost always means you're about to read it back from the DOM, which means you should have used state.

### SCSS over style attributes

- **A repeated visual tweak is a class, and probably already a Bootstrap one.** `class="position-absolute bottom-0 opacity-50"` beats a new `.scss` rule beats an inline `style`. Reach for an existing Bootstrap utility first; write SCSS only when none fits; never inline. (And remember Bootstrap's grid isn't loaded in POS — `row`/`col-*` do nothing there.)

### Frontend JS the reviewer won't flag

These recur on nearly every PR. They're house conventions, not deep framework knowledge — get them right once and the review moves on to the logic.

- [ ]  **`const` by default; `let` only when you reassign; never `var`.** "Mutating an object held in a `const`" is fine — `const` blocks rebinding, not mutation. `var` is a finding every time.
- [ ]  **`camelCase` for variables and functions.** `PascalCase` is for classes/constructors only; `snake_case` is Python leaking into JS. Prefix a variable with `$` *only* when it holds a jQuery object.
- [ ]  **ES6 imports — `import { x } from "..."`.** Not `require(...)`, not `odoo.define(...)`. Import OWL hooks from `@odoo/owl` (not `owl.hooks`), `_t` from `@web/core/l10n/translation`, `patch` from `@web/core/utils/patch`.
- [ ]  **Pick the semantic array method.** `map` to transform, `filter` to select, `reduce` to fold, `some`/`every` for a test, `Map.groupBy`/`Object.groupBy` to bucket — over a hand-rolled `for`/`forEach`+`push`. A repeated lookup inside a loop is an O(n²) trap: index it into a `Map`/object once.
- [ ]  **Guard clauses over nested `if`/`else`.** Return early for the special cases; keep the happy path un-indented. Reviewers reword deep nesting into guards constantly.
- [ ]  **`===`/`!==`, and remember an empty array is *truthy* in JS.** `if (list)` does not check emptiness — that Python instinct is a bug here. Use `list.length`. Don't compare arrays/objects with `===` (it's reference identity); compare element-wise or via `Set`.
- [ ]  **Every user-facing string is wrapped in `_t()`.** Titles, bodies, notifications, action names. An un-wrapped string can't be translated; the reviewer adds `_t` and asks for the `.po` entry.
- [ ]  **No `debugger`, no stray `console.log`, no dead/commented code.** Leftover `debugger` is the second-most-repeated literal comment in the whole corpus. If you must leave commented code, leave a `// why` next to it.
- [ ]  **Let the code breathe — group with blank lines.** Separate a function's phases (guards → compute → side-effect → return) with a blank line, and keep statements that belong together touching. A 2–3 line helper stays compact and can sit right next to its siblings; but the longer a method gets, the more it needs the breaks — never drop a 30-line block as one unbroken wall.

## References (consult, don't memorize)


| Need...                                                                                             | Read                              |
| --------------------------------------------------------------------------------------------------- | --------------------------------- |
| OWL component anatomy — getters vs state,`useRef`/`t-ref`, lifecycle hooks, prop shapes            | `references/owl-components.md`    |
| The`patch()` rules (prototype/static/props), calling super, services, ORM/RPC, registries           | `references/patching-services.md` |
| QWeb/OWL template directives, conditional classes, inheritance (`t-inherit`, xpath, `d-none`), SCSS | `references/templates-styling.md` |
| JS style the reviewer enforces — const/naming/imports, array idioms, guards, async,`_t`            | `references/js-conventions.md`    |

For where frontend files live and how assets are declared in the manifest, see `odoo-module-development`. For XML *backend* views (not OWL templates), see `odoo-xml-conventions`. For the Python side of a POS data loader or controller, see `odoo-python`.
