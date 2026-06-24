---
name: odoo-module-development
description: Use when designing or writing any Odoo module — new models, inherited models, views, security records, migrations, or demo data. Invoke before creating __manifest__.py or any file under a module directory.
---

# Odoo Module Development

A customer-ready module is self-describing: the manifest, models, security, demo data, and translations each answer one question about what it does and who may use it. These are stances to carry, not a checklist to run — the exact layouts, manifest keys, and file names live in `references/module-structure.md`, pulled only when you need the fact. When a task skips the planning pass (`odoo-grill`), these reflexes carry the design alone.

## The stances

### The manifest is the module's cover letter

A reviewer who reads only `__manifest__.py` should already know what the module does, what it depends on, what it ships, and which Odoo version it targets — if they have to open `models/` to find out, the manifest is failing its one job. It also carries the paperwork that makes the work traceable and shippable to a customer: a `name` following the MMC convention, a `summary` that states the *business need* the module solves (never an echo of the `name` — "Units Management" on a module named Units Management says nothing), a real `description`, the `project.task` id the work traces to, and a customer license (not `LGPL`). (Exact keys and formats: `references/module-structure.md`.)

### One module, one responsibility — and `depends` is a contract

Customers install what they pick, so a module that does "project management *and* invoicing extensions" is really two modules wearing one manifest. Every `depends` entry is something you carry on your upgrade path forever — depend on the smallest thing that works, and never pull in `account` just to borrow one constant. The tell that the boundary is wrong: a `product_extra` on `sale + stock + pos + purchase` forces four apps on a customer who runs one. Split it — a base module on the shared dependency, then thin `*_sale` / `*_stock` satellites that each add their single app.

### Security is a deliverable, not an afterthought

Every model owns its access — an `ir.model.access.csv` line even for the model that's "only used internally," because "internal" is exactly the assumption that leaks data. Who-may-see-which-row is the record rule's job (`ir.rule`), never a Python `if`: ad-hoc checks silently skip `search`, `read_group`, and report exports (→ `odoo-code-review`). And every `sudo()` is a trust boundary you justify in a comment — assert the right, *then* sudo (→ `odoo-python`).

### `_inherit` over redefining — the framework is the extension API

Extending an existing model is `_inherit`; delegating to a parent record is `_inherits`; only a genuinely new thing earns a bare `_name`. Confuse them and you get fields that exist twice with security that applies once. The reflex underneath: the framework *is* the extension API — inherit the model, xpath the view, extend the method, rather than redefining or monkey-patching, so you keep core's edge cases and its upgrade path instead of forking them. (Override discipline on the Python side → `odoo-python`.)

### Store only what you must, and make each field a deliberate choice

Every stored field is data you migrate and keep consistent forever, so don't store what you can derive — a `check_in`/`check_out` boolean pair where one `punch_type` would do, a stored `punch_date` you could get by grouping the datetime. Stored values must be deterministic from their stored inputs (→ `odoo-python` owns this). For the fields that earn their place, decide required-ness instead of defaulting to optional: ask whether an empty value is genuinely meaningful — a transaction with no amount, a terminal with no company usually shouldn't be possible.

### Field and view design is a user-facing decision

Field names are read by users in tooltips and exports, not just by you — `partner_id` not `partner`, `is_active` not `flag`. And views describe data, not behavior: a `<field>` carrying a three-way `invisible`/`attrs` branch is logic that belongs in a computed field on the model, where it's testable and reusable. (XML naming, formatting, and inheritance → `odoo-xml-conventions`.)

### Demo data and translations are part of "done"

A module isn't done when it works — it's done when a customer installs it on a fresh DB and sees the feature work without typing anything. Demo data is what demonstrates it; every user-facing string goes through `_()`, and a customer module ships the `.po` for their locale. And if a change would strand existing rows on upgrade, you owe a migration before it ships, not after (→ `odoo-migrations` owns the when and how).

## References (consult for the fact, don't memorize)

| Need... | Read |
|---|---|
| where a file goes, what to name it, manifest keys / license / MMC name | `references/module-structure.md` |
| a change that would strand data on a deployed module | invoke `odoo-migrations` |
| the Python inside `models/` — ORM, computes, performance, security | invoke `odoo-python` |
| XML record ids, view inheritance, menu / action naming | invoke `odoo-xml-conventions` |
| reviewing the result before you call it done | invoke `odoo-code-review` |

If a reference contradicts existing PSDU code (customer legacy, project exception), follow the existing file and log the case in `skills/_journal.md`.
