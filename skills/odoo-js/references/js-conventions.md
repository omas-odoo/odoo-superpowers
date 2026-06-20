# Frontend JS Conventions Reference

Use for the JS style PSAE reviewers enforce on nearly every frontend PR. The principles live in `SKILL.md`; this is the exact-rules lookup. None of this is deep — it's the set of nits that, left in, make the review about style instead of the feature.

## Declarations: `const` by default, never `var`

```js
const res = super.getDisplayData(...arguments);   // not reassigned → const
let total = 0;                                     // reassigned in a loop → let
for (const line of order.lines) { total += line.price; }
```

- `var` is a finding every time — it's function-scoped and hoists. Use `const`/`let`.
- `const` blocks *rebinding*, not *mutation* — `const o = {}; o.x = 1;` is fine, so a `const` holding an object/array you mutate is correct.
- Don't introduce a variable just to return it: `return super.foo(...arguments);` not `const x = super.foo(...arguments); return x;`.

## Naming

- `camelCase` for variables and functions.
- `PascalCase` only for classes/constructors and component handles.
- No `snake_case` (that's Python) for new identifiers — though when you're matching an existing standard field/method name, keep it.
- Prefix with `$` *only* a variable holding a jQuery object — never a plain `HTMLElement`.

## Imports — ES6 only

```js
import { patch } from "@web/core/utils/patch";
import { useService } from "@web/core/utils/hooks";
import { _t } from "@web/core/l10n/translation";
import { onWillStart, useState, useRef } from "@odoo/owl";
```

Not `require("web.core")`, not `odoo.define(...)`, not `const { onWillStart } = owl`. Import OWL hooks from `@odoo/owl`.

A file under `static/src` is treated as an ES module by the asset system. On versions/bundles that still need it, an `/** @odoo-module **/` header at the top of the file is what opts it in (and `/** @odoo-module alias=... **/` keeps a legacy import name working during a migration).

## Pick the semantic array method

```js
// transform → map
return res.map((data) => ({ ...data, disabled: data.disabled || check_user }));
// extract THEN fold — map feeds reduce; a reduce callback that also extracts gets "map + reduce"
const total = lines.map((l) => l.qty).reduce((sum, qty) => sum + qty, 0);
// flatten nesting → flatMap
const payments = orders.flatMap((o) => o.payment_ids);
// test → some / every
const allValid = entries.every(([, v]) => v <= 1);
// bucket → Map.groupBy / Object.groupBy
const byTag = Map.groupBy(items, (i) => i.tag.id);
// build an object → Object.fromEntries
const filtered = Object.fromEntries(Object.entries(data).filter(([, v]) => v));
```

- Don't hand-roll `forEach` + `push` when `map`/`filter`/`reduce` says it directly.
- Removing records from a live recordlist? Iterate a snapshot — `[...order.payment_ids].forEach((l) => order.removePaymentline(l))` — the live list shrinks mid-iteration and skips elements.
- A repeated lookup inside a loop is an **O(n²)** trap — index once into a `Map`/object (`Object.fromEntries(rows.map(r => [r.virtual_id, r]))`) then look up in O(1).
- Membership/dedup → `Set` (`new Set(a).has(x)`), not `array.includes` in a loop.

## Guard clauses over nesting

```js
// BAD                                   // GOOD
if (confirmed) {                         if (!confirmed) {
    if (valid) {                             return;
        doThing();                       }
    }                                    if (!valid) {
}                                            return;
                                         }
                                         doThing();
```

Return early for the special/error cases; keep the happy path at the lowest indentation. Apply De Morgan's to flip an `if` so you can return early.

## Whitespace & grouping — let the code breathe

Vertical space is how a reader finds the seams. Separate a function's logical phases with a blank line; keep the statements that belong together touching.

```js
// BAD — one unbroken wall; you have to read every line to find the parts.
applyReward(reward) {
    if (!reward) { return; }
    const lines = this.getLines();
    const qty = this.freeQty(reward);
    const pct = (qty / this.totalQty(lines)) * 100;
    lines.forEach((l) => l.setDiscount(pct));
    this.notify();
}

// GOOD — guard, then compute, then act; grouped with blank lines.
applyReward(reward) {
    if (!reward) {
        return;
    }

    const lines = this.getLines();
    const pct = (this.freeQty(reward) / this.totalQty(lines)) * 100;

    lines.forEach((l) => l.setDiscount(pct));
    this.notify();
}
```

- **The longer the function, the more it needs the breaks.** A 2–3 line getter stays compact; a 30-line method dumped as unbroken text is a review comment every time — split it into setup / guards / computation / side-effects / return.
- **Group, don't atomise.** Reserve blank lines for *logical* groups, not every statement. Small, related one-line helpers can sit directly under each other without a blank line between each; over-spacing a tiny function is as noisy as cramming a big one.

## Equality and truthiness — the Python traps

- Use `===` / `!==`. `==` coerces and is a finding.
- **An empty array is truthy in JS.** `if (list)` does *not* mean "non-empty" — use `list.length`. (Python instinct; real bug here.)
- `===` on arrays/objects is *reference* identity, not value equality. Compare element-wise (`a.length === b.length && a.every((x, i) => x.id === b[i].id)`) or via `Set` intersection.
- Lean on `?.` and `??`/`??=`: `this.props.partner?.id ?? ""` beats a defensive ternary chain.

## Async

- **No `await` before `return`** when you don't use the result — `return super.foo(...args);` returns the promise directly.
- **No async work in `init`/constructors or in `setup()`'s body** — use `onWillStart` (see `owl-components.md`).
- An unawaited promise that can reject swallows its error — `await` it (with handling) or attach `.catch`. Don't fire-and-forget a failing RPC.

## Translation

Every string shown to a user goes through `_t()` — titles, bodies, notifications, action `name`s:

```js
this.dialog.add(AlertDialog, {
    title: _t("Access Denied"),
    body: _t("Only cashiers with advanced privileges can proceed."),
});
```

An unwrapped string can't be translated; ensure the term exists in the module's `.po` files. `_lt` (lazy) was removed around v18 — on ≤17 it still exists for module-level strings evaluated at import time.

## Hygiene

- Remove every `debugger` and stray `console.log` before pushing — leftover `debugger` is one of the most repeated review comments in the corpus.
- No dead or commented-out code. If a commented block must stay, put a `// why it's here` next to it.
- Don't pull in `underscore` (`_.each`, `_.isEmpty`, `_.indexBy`) for things native JS now does (`forEach`/`Array.isArray`+`.length`/`Object.fromEntries`) — underscore is being phased out.
