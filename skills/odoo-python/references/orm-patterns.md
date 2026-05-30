# ORM Patterns Reference

Source: recurring PSAE review feedback. Consult when writing model logic — computes, fields, constraints, overrides. The principles live in `SKILL.md`; this is the exact-rules lookup.

## Compute fields

### One compute, one field

```python
# BAD — _compute_total writes two fields; recompute order is undefined.
@api.depends("line_ids.price", "line_ids.tax")
def _compute_total(self):
    for order in self:
        order.amount_untaxed = sum(order.line_ids.mapped("price"))
        order.amount_tax = sum(order.line_ids.mapped("tax"))

# GOOD — each field owns its compute.
@api.depends("line_ids.price")
def _compute_amount_untaxed(self):
    for order in self:
        order.amount_untaxed = sum(order.line_ids.mapped("price"))

@api.depends("line_ids.tax")
def _compute_amount_tax(self):
    for order in self:
        order.amount_tax = sum(order.line_ids.mapped("tax"))
```

### Stored computes are `compute_sudo` by default

Don't set it redundantly:

```python
total = fields.Monetary(compute="_compute_total", store=True)              # already compute_sudo
total = fields.Monetary(compute="_compute_total", store=True, compute_sudo=True)  # redundant — drop it
```

### `related` over a hand-written one-hop compute

```python
# BAD — a compute for a plain hop.
country_id = fields.Many2one("res.country", compute="_compute_country")
def _compute_country(self):
    for rec in self:
        rec.country_id = rec.partner_id.country_id

# GOOD
country_id = fields.Many2one("res.country", related="partner_id.country_id", store=True)
```

## Monetary fields

`Monetary` defaults `currency_field` to `currency_id` (or `x_currency_id`). Only set it when you genuinely need a different currency:

```python
amount = fields.Monetary()                                   # uses currency_id — fine
amount_company = fields.Monetary(currency_field="company_currency_id")  # different currency — justified
```

## SQL constraints

Use the **new `models.Constraint` syntax** (Odoo 18+), not the legacy `_sql_constraints` list. One attribute per constraint, named `_<name>`:

```python
name = fields.Char(required=True)
company_id = fields.Many2one("res.company", required=True)

_unique_name = models.Constraint("UNIQUE(name, company_id)", "Name must be unique per company.")
_check_discount = models.Constraint("CHECK(discount <= 1)", "Discount cannot exceed 100%.")
```

Two rules reviews repeatedly enforce:
- **Uniqueness is usually per company** — `UNIQUE(name, company_id)`, not `UNIQUE(name)`.
- **The constrained columns must be `required=True`** — Postgres allows duplicate NULLs, so a nullable column silently lets duplicates through.

When a value has a natural minimum/maximum (a percentage, a count), add the `CHECK` constraint — reviewers ask for it by default.

## Float and monetary comparisons

Never compare floats or monetary values with `==`, `<`, `>` directly — rounding makes them lie. Use `float_compare` / `float_is_zero` with the field's precision.

```python
from odoo.tools import float_compare, float_is_zero

# BAD
if record.amount == 0: ...
if credit_sum < debit_sum: ...

# GOOD
if float_is_zero(record.amount, precision_rounding=record.currency_id.rounding): ...
if float_compare(credit_sum, debit_sum, precision_digits=precision) == -1: ...
```

## Currency conversion

Use `currency._convert(...)` — it already short-circuits when source and target currency match, so don't guard it with `if line.currency_id != company.currency_id` yourself.

```python
amount = line.currency_id._convert(
    from_amount=debit or credit,
    to_currency=line.company_id.currency_id,
    company=line.company_id,
    date=fields.Date.today(),
)
```

## Assign one field with `=`, many with `.write()`

In a method or compute on a recordset, assigning a single field reads cleaner as `record.field = value`. Reserve `.write({...})` for setting **multiple** fields at once (or when writing to a recordset you didn't loop over).

```python
# one field — use =
for record in self:
    record.state = "done"

# several fields at once — use write
record.write({"state": "done", "date_done": fields.Date.today(), "user_id": uid})
```

## Non-stored computes that users filter on need a `_search_`

A non-stored computed field can't be searched unless you give it a search method. If users will filter or group by it, add `_search_<field>` — otherwise the domain silently returns nothing.

## `fields.Json` over manual serialization

If a field holds structured data, use `fields.Json` — not a `Text`/`Char` field with `json.dumps`/`json.loads` scattered around it. The framework handles serialization and you stop leaking `json.*` calls through the model.

## Company-dependent models

Add a company `ir.rule`, don't just bolt `company_id` on the model. The field alone doesn't isolate records across companies — the rule does.

```xml
<record id="my_model_rule_company" model="ir.rule">
    <field name="name">my.model: multi-company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
</record>
```

Never hardcode `base.main_company` (or `ref('base.main_company')`) in defaults or logic — it breaks any multi-company database. Let init hooks and `company_id` defaults resolve the company at runtime.

## Field naming to avoid clashes

Generic field names (`mail_template_id`, `state`, `reference`) risk colliding with a future standard field. Prefix custom fields with the module/provider:

```python
dhl_mail_template_id = fields.Many2one("mail.template")   # not mail_template_id
```

Add a `domain` where it constrains the choices meaningfully.

## Override patterns

### Forward unknown args

```python
def _get_most_frequent_accounts_for_partner(self, *args, **kwargs):
    result = super()._get_most_frequent_accounts_for_partner(*args, **kwargs)
    # ... adjust result ...
    return result
```

### Guard, then super

```python
def _get_next_date(self, last_call, *args, **kwargs):
    if self.frequency != "every_two_months":
        return super()._get_next_date(last_call, *args, **kwargs)
    return last_call + relativedelta(months=2)
```

### Don't override `create` just to read context

Context is available in the compute and in `default_get`. Override `create` only when you must intercept persistence itself.

## Prefer a computed (stored) field over scattered state-setting

When a value can be derived, a computed stored field (even with no `@api.depends`, set in an inverse or in `create`) beats writing the same state from three different methods. One source of truth.
