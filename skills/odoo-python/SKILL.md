---
name: odoo-python
description: Use when writing or reviewing any Python in an Odoo module — models, computes, overrides, controllers, wizards. Carries PSAE review-derived principles for ORM correctness, performance, style, and security, plus references for exact patterns. Invoke before writing logic in models/, controllers/, or wizard/.
---

# Odoo Python

Write the Python an experienced PSAE reviewer wouldn't comment on — so the review is about the logic, not the nitpicks. These principles are distilled from real recurring review feedback on PSAE customer code.

## Principles

### ORM correctness — where the real bugs hide

- **A compute owns exactly one field — its own.** Never write to another field from inside a compute. Cross-writing makes recomputation order a lottery and the value non-deterministic. If field B depends on A, give B its own `@api.depends('A')` compute.
- **A stored compute is a promise that the value derives only from its declared `@api.depends`.** Reading context, time, `env.user`, or another model's unstored state breaks the promise — the next cron recompute will disagree with what you wrote. Drop `store=True` or drop the runtime input.
- **`@api.depends` names every real input and nothing else.** A missing dependency means stale data; a dependency on a non-stored field means pointless recomputes. Wrong depends is worse than no compute.
- **Reach for `related=` before hand-writing a compute** when the value is a plain hop (`partner_id.country_id`). A custom compute for a one-hop value is maintenance you signed up for with no reason.
- **No `hasattr` in Odoo code.** It almost always hides a missing module dependency or a `super()` mistake (`hasattr(super, ...)` inspects the builtin, not the proxy — it's always False). If you need to know a field exists, depend on the module that defines it.
- **Don't introspect `_fields` to dodge a dependency.** `partner._fields.get('currency_id')` to avoid depending on the module that adds it is a hack. Add the dependency.
- **Changing a field is a data event, not just a code edit.** Removing, renaming, retyping, or adding `required=True` to a field on a model that's already deployed will strand or break existing rows on upgrade. The moment you make that edit, invoke the **`odoo-migrations`** skill — the migration ships *with* the field change, not as an afterthought.

### Reach for the framework helper before hand-rolling

Odoo ships helpers for the things you're tempted to write by hand. The helper is shorter, correct, and survives version upgrades — reviewers ask "why not use `<helper>`?" by reflex.

- **Actions:** `records._get_records_action(...)` — not a hand-built `ir.actions.act_window` dict.
- **Record links in messages:** `record._get_html_link()` — not hand-assembled `<a href=...>`.
- **Notifications:** `record.message_post(...)` already notifies tagged partners in the log — don't re-implement it.
- **Activities:** `record.activity_schedule(...)` — not manually creating `mail.activity` records.
- **Currency:** `currency._convert(...)` already short-circuits same-currency — don't guard it with an `if`.
- **Float/money comparison:** `float_compare` / `float_is_zero` — never `==` / `<` on floats; rounding makes them lie.
- **Set one field with `=`; reserve `.write({...})` for multiple fields.**

If you catch yourself writing an `<a href>` string, an action dict, or a float `==`, stop — there's a helper. See `references/orm-patterns.md` for the exact signatures.

### Overrides — survive the framework changing under you

- **An override that ignores its signature takes `*args, **kwargs`.** When Odoo changes a standard method's signature, your override keeps forwarding correctly instead of silently dropping a new argument.
- **Guard, then `super()`.** If your override only acts in a special case, `return super()...` first for everything else. An early return beats wrapping the whole method body in an `if`.
- **Override `create`/`write` only when there's no cleaner hook.** Wanting to read context is not a reason — context already reaches the compute. Reserve CRUD overrides for when you genuinely must intercept persistence.

### Performance — the loop is the enemy

- **No query inside a loop.** `search` / `search_read` / `read_group` in a `for` is the single most repeated performance finding. Hoist the query out, fetch once, index the result in Python.
- **Batch writes and creates.** One `create([vals1, vals2, ...])` beats N creates. Set x2many with `Command.set/link`, never a loop of writes.
- **Aggregate with `_read_group`, not `search` + Python sum.** The database groups faster than you can, and it doesn't pull every record into memory to do it.
- **Index what you filter on.** A field that appears in domains, joins, or search defaults wants `index=True`.

### Style & hygiene — keep the diff clean

- **DRY the vals dicts.** Repeated `{...}` for move lines or record creation becomes a `_prepare_*_vals` helper. Submodules can then override the helper instead of duplicating your method.
- **Dead code is a blocker, not a nitpick.** Unused const files, one-line wrappers, `ensure_one()` that guards nothing — delete them. But never break PEP 8 or readability just to save lines.
- **Error and log messages name the record and the problem.** `"Error"` tells the support tech nothing. `_("Invoice %s has no journal", move.name)` tells them exactly where to look.
- **Don't mix quote styles in one file.** Pick `"` or `'` and stay consistent — the diff stays clean and the review skips the nitpick.
- **Let the method breathe — group with blank lines.** Inside a method, separate logical phases (guards → fetch/compute → write/act → return) with a single blank line, and keep related statements touching. A short method stays compact; the longer it grows the more it needs the breaks — but sparingly, a double blank line inside a body is an `E303` finding. (If it has many phases, it's probably two methods.)

### Security in Python code

- **Protected fields: check access, then `sudo()` to read.** Credential/secret fields carry `groups='base.group_system'`. To read them in logic, first assert the user's right (`has_group(...)` → raise `AccessError`), *then* `company.sudo()._get_credentials()`. Sudo without the access check defeats the point.
- **Webhook and payment controllers verify authenticity and are idempotent.** Check the HMAC/signature, and guard re-entry: `if tx.state not in ('done', 'cancel', 'error')`. A replayed webhook must never double-process. Don't silently redirect to a success page when the transaction isn't found — log it and show an error.
- **Imports inside an addon are absolute.** `from odoo.addons.<module>.tools.x import Y`, never bare `from tools.x import Y` — the bare form breaks depending on the working directory.

### Copy-paste vigilance

- **A cloned module is a liability until you've renamed everything.** When you copy a module to start a new one, the method prefixes, comments referencing the old provider, and vestigial files all come with it. Duplicate method names across modules collide. Rename `_aps_` → `_cc_avenue_` everywhere and fix the comments before you write a line of new logic.

## References (consult, don't memorize)

| Need... | Read |
|---|---|
| Exact ORM rules — compute fields, SQL constraints, company rules, Monetary, field prefixing, override snippets | `references/orm-patterns.md` |
| Performance patterns with code — batching, `_read_group`, indexes, upgrade-script SQL | `references/performance.md` |
| Calling an external API / device / webhook — Session, auth, pagination, error handling | `references/external-integration.md` |
| Imports, naming, attribute order, idioms, transactions, translation idiom, PEP8 exceptions | `references/python-conventions.md` |

For the stored-compute-is-deterministic principle at review time, see also `odoo-code-review`. For where Python files live in the module, see `odoo-module-development`.

## What good looks like

```python
# B has its own compute; A's compute touches only A.
amount_total = fields.Monetary(compute="_compute_amount_total", store=True)

@api.depends("line_ids.subtotal")
def _compute_amount_total(self):
    for order in self:
        order.amount_total = sum(order.line_ids.mapped("subtotal"))

# Override forwards unknown args, guards early.
def _get_next_date(self, last_call, *args, **kwargs):
    if self.frequency != "every_two_months":
        return super()._get_next_date(last_call, *args, **kwargs)
    return last_call + relativedelta(months=2)

# Group once with .grouped(), not a hand-rolled dict.
partners = self.env["res.partner"].search([("active", "=", True)])
by_country = partners.grouped("country_id")   # {country: partners}
```

## What bad looks like

```python
# Cross-writing fields from one compute — recompute order roulette.
@api.depends("line_ids")
def _compute_total(self):
    for order in self:
        order.amount_total = sum(order.line_ids.mapped("price"))
        order.tax_total = sum(order.line_ids.mapped("tax"))   # belongs in its own compute

# hasattr hiding a super() bug — always False, parent never runs.
if hasattr(super, "_onchange_type"):
    super()._onchange_type()

# Query inside a loop — N round-trips to Postgres.
for order in self:
    partner = self.env["res.partner"].search([("id", "=", order.partner_id.id)])
```
