# Patching, Services & RPC Reference

Use when extending standard Odoo frontend code or calling the server. The principles live in `SKILL.md`; this is the exact-rules lookup, from recurring PSAE review feedback.

## `patch()` — the only sanctioned extension mechanism

```js
import { patch } from "@web/core/utils/patch";
```

Never reassign a class member, never use `Component.extend`, never the old `Registries` API. `patch` is the only form that composes with other modules and can be unpatched.

### Patch the layer you mean — they're separate objects

```js
patch(PosStore.prototype, { _loadPosData() { ... } });   // instance methods, setup
patch(OrderReceipt, { template: "my_module.OrderReceipt" });  // static members
patch(Orderline.props.line.shape, { barcode: { type: String, optional: true } });  // props
patch(MainComponent.components, { Dropdown });            // sub-components
patch(INTERVAL_OPTIONS, { hour: { description: _t("Hour") } });  // plain exported objects
```

`props` and `components` are **static** properties. Patching `X.prototype` to add a prop is wrong — patch `X.props` (or the nested `.shape`) directly. This is also fewer lines than spreading:

```js
// BAD
Orderline.props.line.shape = { ...Orderline.props.line.shape, barcode: {...} };
// GOOD
patch(Orderline.props.line.shape, { barcode: { type: String, optional: true } });
```

### Always call super

```js
// GOOD — guard for the special case, super for everything else.
patch(PaymentScreen.prototype, {
    async validateOrder(...args) {
        if (!this.currentOrder.needsToken) {
            return super.validateOrder(...args);   // standard path intact
        }
        const token = await this.askToken();
        if (!token) {
            return;
        }
        this.currentOrder.confirmation_token = token;
        return super.validateOrder(...args);
    },
});
```

- Copy-pasting the standard method's whole body to tweak a few lines means the customer loses every future Odoo hotfix to that method. Don't.
- When the change is *inside* a method you can't cleanly wrap, override a **smaller** hook instead (e.g. `compute_fixed_price` rather than the caller), or set a temporary flag (`this.pos.tmpForceX = true; const res = super...(); this.pos.tmpForceX = null;`) the smaller override reads.
- In an **async** patch, capture super before the first `await` if you'll need it after: `const _super = super.foo; await x(); return _super.call(this, ...args);` (older v16 `patch` used `this._super(...arguments)` — match the version you're on).
- Note v16 vs v17+ method names differ (`_onClickPay` → `onClickPay`, `export_as_JSON` removed in v18, `_lt` removed ~v18). Override the method that actually exists in the target version.

### Don't override a pure function to add a side-effect

```js
// BAD — get_cashier() is a getter; this hides a mutation in it.
patch(PosStore.prototype, {
    get_cashier() {
        const c = super.get_cashier();
        this.advancedSet = new Set(this.config.advanced_employee_ids);  // side-effect!
        return c;
    },
});
// GOOD — build the set once where it belongs, expose a clean checker.
patch(PosStore.prototype, {
    _loadPosData(...args) {
        super._loadPosData(...args);
        this.advancedSet = new Set(this.config.advanced_employee_ids);
    },
    getCheckUser() {
        return this.advancedSet.has(this.get_cashier().id);
    },
});
```

### Prefer props/template changes over a new component

```js
// BAD — new component subclass to add an A4 receipt variant.
export default class CustomReceipt extends OrderReceipt { ... }
// GOOD
patch(OrderReceipt.props, { isA4: { type: Boolean, optional: true } });
```

```xml
<xpath expr="//div[1]" position="attributes">
    <attribute name="t-att-class">{ 'pos-receipt-a4': props.isA4 }</attribute>
</xpath>
```

## Services

```js
import { useService } from "@web/core/utils/hooks";

setup() {
    super.setup();
    this.orm = useService("orm");
    this.dialog = useService("dialog");
    this.notification = useService("notification");
    this.pos = usePos();          // domain hook — prefer over this.env.services.pos
}
```

Acquire each service once in `setup`; don't sprinkle `this.env.services.x` through the methods. When a dedicated hook exists (`usePos`), use it.

## ORM vs RPC

```js
// Model methods → orm.
const partners = await this.orm.searchRead("res.partner", [["active", "=", true]], ["name"]);
await this.orm.call("res.partner", "write", [[id], { national_id: value }]);
await this.orm.write("calendar.event", [id], { state: "done" });

// Bare controller routes → rpc.
import { rpc } from "@web/core/network/rpc";
const info = await rpc(`/shop/country_info/${countryId}`, { address_type: this.addressType });
```

Don't use the legacy `this.rpc({ model, method, args })` form for model calls — use `orm`.

### Error handling is not optional

A user-triggered call that can fail must show the failure, never swallow it:

```js
const result = await this.orm
    .call("restaurant.table", "set_active_cashier", [[table.id], this.cashier.id])
    .catch((err) => ({ err }));            // turn rejection into a value you can guard
if (result.err) {
    this.dialog.add(AlertDialog, {
        title: _t("Connection lost"),
        body: result.err instanceof ConnectionLostError
            ? _t("Check your internet connection.")
            : result.err.data.message,
    });
    return false;
}
```

Independent calls run concurrently, not in sequence:

```js
// BAD — each awaits the previous.
const a = await rpc("/a"); const b = await rpc("/b");
// GOOD
const [a, b] = await Promise.all([rpc("/a"), rpc("/b")]);
```

## Loading POS data — server loader, not extra JS RPCs

Adding a per-screen `orm.call` to fetch master data adds a network round-trip the standard single-load already avoids. Extend the server-side loader so your data rides POS's one RPC:

```python
# pos config / session python side
def _load_pos_data_models(self, config_id):
    return super()._load_pos_data_models(config_id) + ["stock.picking.type"]
```

```js
// models.js
patch(PosStore.prototype, {
    async _processData(loadedData) {
        await super._processData(loadedData);
        this.pickingTypeById = Object.fromEntries(
            loadedData["stock.picking.type"].map((t) => [t.id, t]),
        );
    },
});
```

(Method names vary by version; the rule is constant — load through the existing pipeline, hash once, don't loop-fetch.)


