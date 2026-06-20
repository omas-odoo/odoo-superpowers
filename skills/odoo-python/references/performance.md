# Performance Reference

Use when a method touches more than one record or runs in a cron/batch context. The single most-flagged class of issue in PSAE review.

## No query inside a loop

The number-one performance finding. Each `search` / `search_read` / `read_group` in a loop is a round-trip to Postgres.

```python
# BAD — no relation to navigate, so you search per row. N round-trips.
for order in orders:
    logs = self.env["x.log"].search([("order_ref", "=", order.name)])

# GOOD — one search for the whole set, then group in Python.
logs = self.env["x.log"].search([("order_ref", "in", orders.mapped("name"))])
logs_by_ref = logs.grouped("order_ref")
for order in orders:
    logs = logs_by_ref.get(order.name, self.env["x.log"])

# When a relation already exists, just navigate it — Odoo prefetches across the recordset
# (order.partner_id, order.line_ids); never search([("id", "=", ...)]) for those.
```

## Batch creates and writes

```python
# BAD — N inserts.
for vals in vals_list:
    self.env["sale.order.line"].create(vals)

# GOOD — one insert.
self.env["sale.order.line"].create(vals_list)

# x2many — use Command, not a write loop.
order.write({"line_ids": [Command.set(line_ids)]})
order.write({"line_ids": [Command.link(line_id)]})

# Account moves — batch reversal with per-record defaults.
moves._reverse_moves(default_values_list, cancel=True)
```

## Aggregate with `_read_group`

```python
# BAD — load every record, sum in Python.
orders = self.env["sale.order"].search(domain)
total = sum(orders.mapped("amount_total"))

# GOOD — the DB groups and sums.
groups = self.env["sale.order"]._read_group(
    domain,
    groupby=["partner_id"],
    aggregates=["amount_total:sum"],
)
```

## Index fields used in domains

```python
reference = fields.Char(index=True)          # filtered/searched often
partner_id = fields.Many2one("res.partner", index=True)   # joined often
```

Index fields that appear in domains, search defaults, or joins. Don't index everything — each index costs write speed.

## Upgrade scripts: always use the upgrade `util` library

In `migrations/`, reach for `odoo.upgrade.util` helpers — never an ORM `search → write → mapped` chain (it loads and recomputes everything), and never bare `cr.execute` when a `util` helper exists. `util` does the SQL safely: cache, stored recomputes, column existence, indirect deps.

```python
from odoo.upgrade import util

util.rename_field(cr, "sale.order", "old_name", "new_name")
util.recompute_fields(cr, "sale.order", ["amount_total"])
util.explode_execute(cr, query, table="sale_order")   # huge tables, chunked
```

Bare `cr.execute` only when no helper covers it — guarded with `util.column_exists` first. Invoke the **`odoo-migrations`** skill for the full conventions.
