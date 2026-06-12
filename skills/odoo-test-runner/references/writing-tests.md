# Writing Odoo Tests Reference

Read when writing or reviewing anything under a module's `tests/` directory — the exact class
choice, `@tagged` syntax, fixture split, assertions, discovery file, tour wiring, and the
demo-user password trap. The SKILL body carries the judgment; this file carries the patterns.

## Test class choice

| Class | When | Notes |
|---|---|---|
| `TransactionCase` | **Default.** Any ORM test. | One transaction per test, rolled back. `setUpClass` data persists across the class (savepoint per test), rolled back at class teardown. |
| `HttpCase` | Controllers, sessions, **tours** — anything needing the HTTP layer. | Slower; spins up a real HTTP server. |
| ~~`SavepointCase`~~ | **Removed.** | Merged into `TransactionCase` (v16+). Its shared-fixture behaviour is now the default — don't import it. |

Only step up to `HttpCase` when you actually need request/response or a tour.

```python
from odoo.tests.common import TransactionCase, tagged
```

## `@tagged` conventions

```python
@tagged("post_install", "-at_install")
class TestMyModule(TransactionCase):
    ...
```

- `post_install` — run after all modules are installed (where most custom tests belong).
- `-at_install` — exclude from the at-install run, so it isn't executed twice.
- A `-at_install` with **no** `post_install` never runs — a silent no-op.
- Add a module tag for targeted runs:

```python
@tagged("post_install", "-at_install", "psae_my_module")
# → odoo-bin ... --test-tags /psae_my_module
```

## `setUp` vs `setUpClass`

```python
class TestSaleOrder(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Heavy fixtures shared (and rolled back) across every test in this class
        cls.partner = cls.env["res.partner"].create({"name": "Test Partner"})
        cls.product = cls.env["product.product"].create({"name": "Test Product"})

    def setUp(self):
        super().setUp()
        # Per-test state a test may mutate — rebuilt each test
        self.order = self.env["sale.order"].create({"partner_id": self.partner.id})
```

Expensive shared fixtures → `setUpClass` (built once). Per-test mutable state → `setUp`.
Always `super()` first. Putting shared fixtures in `setUp` re-creates them per test and slows
the suite.

## Assertions

```python
# Prefer assertRecordValues — asserts every field at once, clear diff on failure
self.assertRecordValues(self.order, [{
    "state": "sale",
    "amount_total": 100.0,
    "partner_id": self.partner.id,
}])

# Field-by-field is fine for one or two fields
self.assertEqual(self.order.state, "sale")
```

`assertRecordValues` takes a recordset and a list of dicts (one per record, in order). Assert on
record values, not UI-visible labels — labels are translatable and drift.

### `assertRaises`

```python
from odoo.exceptions import UserError, ValidationError

with self.assertRaises(UserError):
    self.order.action_confirm_with_invalid_state()

# Match a specific message fragment
with self.assertRaisesRegex(ValidationError, "negative quantity"):
    self.order.line_ids[0].product_uom_qty = -1
```

Always assert the *specific* exception type. `with self.assertRaises(Exception)` accepts too
much and hides the next bug.

## `tests/__init__.py`

The `tests/` directory **must** have its own `__init__.py` importing every test file, one per
line, alphabetically:

```python
# tests/__init__.py
from . import test_account_move
from . import test_sale_order
from . import test_stock_picking
```

- No encoding header. One import per line. Alphabetical.
- Without it, the runner discovers no test classes and the suite silently passes empty.
- The module-root `__init__.py` **never** imports `tests` — Odoo handles test discovery itself.
  (Full `__init__.py` rules: `odoo-python`.)

## Tours — `HttpCase.start_tour`

A tour is a JS script that drives the UI step-by-step, launched from a Python `HttpCase`. Three
coupled pieces:

### 1. Python entry

```python
from odoo.tests import HttpCase, tagged

@tagged("post_install", "-at_install")
class TestMyTour(HttpCase):
    def test_my_workflow_tour(self):
        self.start_tour("/odoo/sales", "my_module.sale_order_tour", login="admin")
```

Always `HttpCase`, never `TransactionCase`.

### 2. JS tour definition (`static/src/js/tours/sale_order_tour.js`)

```javascript
import { registry } from "@web/core/registry";

registry.category("web_tour.tours").add("my_module.sale_order_tour", {
    test: true,
    url: "/odoo/sales",
    steps: () => [
        { trigger: ".o_kanban_button_new", content: "Click Create", run: "click" },
        {
            trigger: "input.o_input[name='partner_id']",
            content: "Enter customer",
            run: "edit Test Customer",
        },
        // ... more steps
    ],
});
```

JS conventions (imports, registry, naming) follow `odoo-js`.

### 3. Asset bundle (`__manifest__.py`)

```python
"assets": {
    "web.assets_tests": [
        "my_module/static/src/js/tours/sale_order_tour.js",
    ],
},
```

**`web.assets_tests`, not the main web bundle** — tour files ship only in test runs, never
production.

## Demo-user password trap

`start_tour(url, tour, login="admin")` authenticates with the password *equal to the login*
(`authenticate("admin", "admin")`). On a fresh or dev DB, `admin`/`demo`/`portal` frequently
have an **empty** password — so authentication silently fails and the tour never starts. Set the
passwords before a test run that logs in as these users:

```sql
UPDATE res_users SET password = 'admin'  WHERE login = 'admin';
UPDATE res_users SET password = 'demo'   WHERE login = 'demo';
UPDATE res_users SET password = 'portal' WHERE login = 'portal';
```

Raw SQL plaintext is fine here — Odoo re-hashes on the next successful login. Run it against the
throwaway DB after install, before `--test-enable`.

## Common pitfalls

- `TransactionCase` for a tour — tours always need `HttpCase`.
- `@tagged("-at_install")` with no `post_install` — the test never runs.
- Forgetting to import a new test file in `tests/__init__.py` — it silently doesn't run.
- Asserting on translatable UI labels instead of stable record values.
- Reaching for `SavepointCase` — removed; use `TransactionCase`.
- Side-effecting test data without rolling back — `TransactionCase` rolls back automatically,
  but only if you never `commit()`.
