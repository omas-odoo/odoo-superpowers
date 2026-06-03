# OWL Components Reference

Use when writing an OWL component or widget. The principles live in `SKILL.md`; this is the exact-rules lookup, from recurring PSAE review feedback on POS and backend frontend code.

* [ ]  Getters vs state vs methods

A value you can *derive* from `state`/`props` is a getter. Getters recompute on every render, so they're never stale, and they cost nothing to store.

```js
// BAD — stored in setup(): wrong the moment state.code changes.
setup() {
    super.setup();
    this.isValid = this.state.code.length === 4;
}

// BAD — a no-arg method is a getter with worse ergonomics.
isValidCode() { return this.state.code.length === 4; }

// GOOD
get isValidCode() {
    return this.state.code.length === 4;
}
```

Use a getter for anything the template needs to compute — especially class/`t-if` conditions:

```js
get rowClass() {
    return {
        highlight: this.props.selected,
        "text-muted": this.props.disabled,
    };
}
```

```xml
<div t-att-class="rowClass"/>
```

Reach for `useState` only for values that *change over the component's life and must trigger a re-render*. Don't wrap props or derived values in state. In POS (v16+), the models (`Order`, `Orderline`, `Payment`) are already reactive and hooked to the render — you usually don't need extra state to mirror them; `t-model` on an input is enough.

## Module-level constants — define once, outside the class

A regex, a `Set`, a lookup table that never changes must not be redefined every time the component instantiates or every time a method runs.

```js
// BAD — rebuilt on every IdWidget instance.
setup() {
    this.REG = /^\d{3}-\d{4}-\d{7}-\d$/;
    this.lengths = [3, 4, 7, 1];
}

// GOOD — module scope, built once.
const TEST_REG = /^(\d{3})-?(\d{4})-?(\d{7})-?(\d)$/;
const SECONDARY_COLUMNS = new Set(["secondary_credit", "secondary_debit"]);

export class IdWidget extends CharField {
    setup() {
        super.setup();
        this.input = useRef("input");
    }
}
```

## Reaching the DOM — `useRef` + `t-ref`, never a query

`document.getElementById`, `$(...)`, and `this.el.querySelector` from the component root all start searching from too high and break when multiple instances of the component exist. Take a reference while OWL renders instead.

```js
import { useRef } from "@odoo/owl";

setup() {
    super.setup();
    this.inputRef = useRef("search");
}
focus() {
    this.inputRef.el?.focus();
}
```

```xml
<input t-ref="search" type="text"/>
```

For an `<input>`'s value, prefer `t-model` (`t-model.lazy` for change-on-blur) over reading `.el.value`.

## Lifecycle hooks — where side-effects go


| Need                                                         | Hook                                                                                          |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Async data the component needs before first render           | `onWillStart(async () => { this.data = await ... })`                                          |
| Work that needs the rendered DOM (3rd-party init, measuring) | `onMounted(() => ...)`                                                                        |
| Cleanup:`clearInterval`, remove listeners, destroy plugins   | `onWillDestroy(() => clearInterval(this.timer))`                                              |
| Set-up-with-teardown in one place                            | `useEffect(() => { const id = setInterval(...); return () => clearInterval(id); }, () => [])` |

```js
import { onWillStart, onWillDestroy } from "@odoo/owl";

setup() {
    super.setup();
    this.orm = useService("orm");
    onWillStart(this.loadData.bind(this));     // not in setup()'s body
    onWillDestroy(() => clearInterval(this.timer));
}
async loadData() {
    this.records = await this.orm.searchRead("my.model", [], ["name"]);
}
```

- **Async work never goes in `setup()`'s body, in `init`, or *after* mount.** Changing state already forces a re-render — doing DOM tweaks in `onMounted` after a state change is redundant.
- **Don't override the whole `setup()` to add one line.** Add a hook; hooks from multiple patches stack, a full `setup` override clobbers.

### `onWillUpdateProps` — the hook that stops props going stale

`onWillStart` runs **once, on mount**. A component that loads or derives something from a prop (`resId`, `record`, a domain) in `onWillStart` and nowhere else shows the *first* prop's data forever — the parent re-rendering it with a new prop changes nothing. `onWillUpdateProps((nextProps) => …)` is the "props changed" half; pair the two.

```js
import { onWillStart, onWillUpdateProps } from "@odoo/owl";

setup() {
    super.setup();
    this.orm = useService("orm");
    onWillStart(() => this.loadRecord(this.props.resId));
    onWillUpdateProps((nextProps) => {
        // this.props is still the OLD props here — read the new value from the argument.
        if (nextProps.resId !== this.props.resId) {
            return this.loadRecord(nextProps.resId);
        }
    });
}
```

- **The callback takes one argument — the *incoming* props.** There is no old-props argument; the old props are still on `this.props` until the callback finishes, so diff `nextProps` against `this.props` to skip redundant reloads.
- **It never fires on first render** (that's `onWillStart`) and **never fires when the new props equal the old** — OWL diffs them first (`arePropsDifferent`).
- **It may be `async` and is awaited before the re-render**, exactly like `onWillStart` (same 3s dev timeout). Heavy work here delays the patch; for props that change rapidly, debounce or load through a `KeepLast`.
- **Only side-effects belong here** — fetching, or syncing a local `useState` from the new props. A value you merely *derive* from props is a getter (recomputes every render), never an `onWillUpdateProps` body.

Real backend idioms: reload a model when the search domain/groupBy change (`model.load(getSearchParams(nextProps))`), mirror a critical prop into local state (`_syncStateWithProps(nextProps.value)`), or react only to a specific transition (`onWillUpdateProps(({ disabled }) => …)`). It is a **web/backend** hook — POS components almost never need it: the POS models are already reactive and re-render themselves, so you rarely feed a POS component data through a prop that must be re-fetched.

## Prop shapes

Declare new props by patching the shape, with a type and `optional` where appropriate (see `patching-services.md` for why `patch`):

```js
patch(ProductCard.props, {
    show_stock: { type: Boolean, optional: true },
    quant: {
        type: { overall: { type: Number, optional: true } },
        optional: true,
    },
});
```

There's no need to `export` the result of a props patch.

## Avoid jQuery and legacy widgets in OWL

OWL exists to remove jQuery. If you find yourself writing `$(...)`, `.on('click', ...)`, `useListener`, or making a legacy widget "talk" to an OWL component, stop and do it the OWL way: state + `t-on-*` + `useRef`. If you genuinely must touch a jQuery-era widget, scope queries with `this.$el.find(...)`, never bare `$(...)`.
