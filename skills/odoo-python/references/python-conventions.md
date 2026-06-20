# Python Conventions Reference

Use when writing Python in `models/`, `controllers/`, `wizard/`, `report/`, or `tests/`. Lookup-only — conventions from the Odoo 19 Coding Guidelines.

## Imports

Three groups, blank line between, alphabetical within each:

```python
# 1. stdlib
import base64
import re
from datetime import datetime

# 2. odoo core (ASCII-betical)
from odoo import Command, _, api, fields, models
from odoo.fields import Domain
from odoo.tools.safe_eval import safe_eval as eval

# 3. odoo addons (rare; only if necessary)
from odoo.addons.web.controllers.main import login_redirect
```

## Naming

| Symbol | Convention | Example |
|---|---|---|
| Model technical name | singular, dotted | `res.partner`, `sale.order` |
| Transient (wizard) | `<base>.<action>` | `account.invoice.make` |
| Report model | `<base>.report.<action>` | `sale.report.summary` |
| Python class | PascalCase | `class AccountInvoice(models.Model):` |
| Model variable (recordset reference) | PascalCase | `Partner = self.env['res.partner']` |
| Common variable | snake_case | `partner_ids`, `total_amount` |
| Record id holder | suffix `_id` / `_ids` | `partner_id = partners[0].id` |
| `Many2one` field | suffix `_id` | `user_id` |
| `One2many` / `Many2many` field | suffix `_ids` | `sale_order_line_ids` |

## Method name patterns

| Purpose | Pattern |
|---|---|
| Compute | `_compute_<field>` |
| Search (for non-stored) | `_search_<field>` |
| Default | `_default_<field>` |
| Selection options | `_selection_<field>` |
| Onchange | `_onchange_<field>` |
| Constraint | `_check_<constraint>` |
| User-triggered action | `action_<name>` — start with `self.ensure_one()` |

## Model class attribute order

1. Private attributes (`_name`, `_description`, `_inherit`, `_order`, ...)
2. Default methods + `default_get`
3. Field declarations
4. SQL constraints, indexes
5. Compute / inverse / search methods (in same order as fields)
6. Selection methods
7. `@api.constrains` and `@api.onchange`
8. CRUD overrides (`create`, `write`, `unlink`, `read`, `name_search`, `_search`)
9. Action methods
10. Business methods

Skeleton:

```python
class Event(models.Model):
    # 1. Private attrs
    _name = 'event.event'
    _description = 'Event'

    # 2. Defaults
    def _default_name(self):
        ...

    # 3. Fields
    name = fields.Char(default=_default_name)
    seats_reserved = fields.Integer(store=True, compute='_compute_seats')

    # 5. Compute
    @api.depends('seats_max', 'registration_ids.state')
    def _compute_seats(self):
        ...

    # 7. Constraints / onchanges
    @api.constrains('seats_max')
    def _check_seats_limit(self):
        ...

    # 8. CRUD
    @api.model
    def create(self, vals_list):
        ...

    # 9. Actions
    def action_validate(self):
        self.ensure_one()
        ...

    # 10. Business
    def mail_user_confirm(self):
        ...
```

## ORM idioms (the Odoo-specific ones worth stating)

```python
# Navigate relations; don't search for what you can reach. Odoo prefetches.
emails = partners.mapped('email')
hot = partners.filtered(lambda p: p.is_company)

# Group a recordset by a key — use .grouped(), not a hand-rolled {rec.key: rec}.
lines_by_product = order.line_ids.grouped('product_id')   # {product: lines}

# Context: never mutate, use with_context.
records.with_context(lang='fr_FR').do_stuff()
records.with_context(**extra).do_stuff()  # merge; native ctx wins for unset keys
```

(Generic Python idioms — comprehensions, `.get()`, truthy collections, building
dicts in one go — are assumed. This reference only states what's Odoo-specific.)

## Transactions

**Never call `cr.commit()` or `cr.rollback()` yourself** unless you own the cursor. The framework manages the transaction for RPC, cron, and tests. Manual commits cause partial writes → data corruption, stuck documents, polluted test DBs.

Exception: you explicitly created your own cursor — then you handle errors, rollback, and close it.

You do **not** need to commit in: `_auto_init()`, reports, `TransientModel` methods.

Deliberate exception — external-integration / sync jobs: a long loop talking to an outside system may `cr.commit()` per processed record so a mid-run failure doesn't lose completed work. Only with an explicit comment saying why, and never inside a normal RPC / compute / onchange path.

## Exceptions

Catch specific exceptions, narrow scope. Never bare `except:` or `except Exception:` unless you genuinely want to swallow everything (you don't). Don't catch a subclass next to its parent (dead code), don't write several `except` branches that do the same thing, and always log the error object — `_logger.warning("X failed: %s", e)`, never `"error"`. The `requests` exception hierarchy specifically: see `external-integration.md`.

If you need to recover from framework exceptions, use a savepoint:

```python
try:
    with self.env.cr.savepoint():
        do_stuff()
except SpecificError:
    ...
```

Limit: <64 savepoints per transaction (Postgres slows down past that). For batch processing, cap batch size or move to a cron job.

## Translation

Use `_()` for *static* strings only. Field values translate via the field's `translate=True` flag, not via `_()`.

```python
_ = self.env._

# good
error = _('This record is locked!')
error = _('Record %s cannot be modified!', record)
error = _('Record %(name)s cannot be modified.', name=record.display_name)

# bad — translates after formatting (breaks translation)
error = _('Record %s cannot be modified!' % record)
# bad — formats outside translation (no fallback)
error = _('Record %s cannot be modified!') % record
# bad — string concatenation
error = _("'" + question + "'")
# bad — translating field values manually
error = _("Product %s is out of stock!") % _(product.name)
```

Prefer `%(name)s` over `%s` when multiple variables — easier for translators.

## Method shape

- One responsibility per method. Split as soon as it does two things.
- Prefer many small methods over few large ones — submodules can override them surgically.
- Don't hardcode domains or filters in business actions; extract to `_get_<thing>_domain()` and `_filter_<thing>()`.

```python
# Extendable
def action(self):
    partners = self.env['res.partner'].search(self._get_partner_domain())
    emails = partners.filtered(lambda p: p._filter_partners()).mapped('email')
```

Trade off against readability — over-extracting hurts too.

## Whitespace & grouping

Group a method's phases (guards / fetch / compute / write / return) with single blank lines; a double blank line inside a body is `E303`. The longer the method, the more it needs the breaks — but past two or three phases it's probably two methods (see *Method shape*). `ruff` normalizes the rest.

## PEP8 exceptions

These are explicitly ignored in Odoo style:
- `E501` line too long
- `E301` expected 1 blank line, found 0
- `E302` expected 2 blank lines, found 1

`ruff` is configured to match in your project's `pyproject.toml`.

## Always

- Readability beats conciseness.
- Docstrings on methods. Comments on tricky logic.
