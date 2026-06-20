---
name: odoo-python
description: Use when writing or reviewing any Python in an Odoo module — models, computes, overrides, controllers, wizards. Carries PSAE review-derived principles for ORM correctness, performance, style, and security, plus references for exact patterns. Invoke before writing logic in models/, controllers/, or wizard/.
---

# Odoo Python

Write the Python an experienced PSAE reviewer wouldn't comment on — so the review is about the logic, not the nitpicks. These are stances to carry, not a checklist to run: the exact signatures live in `references/`, pulled only when you need the fact. When a task skips the planning pass (`odoo-grill`), these reflexes carry the design alone.

## The stances

### Assume it already exists

Before hand-rolling anything non-trivial — an action, a record link, a chatter notification, money conversion, date math, a domain — your default is that Odoo already ships it; the burden is on you to prove it doesn't. Hand-rolled code is longer, wrong at the edges (timezones, rounding, same-currency), and rots when the framework moves. Catching yourself assembling an `<a href>`, an `ir.actions.act_window` dict, or a float `==` *is* the signal a helper exists — stop and find it. (Whether a whole *feature* already exists — a setting, an automated action, an app — is grill's question; carry it anyway when no plan ran. Signatures: `references/orm-patterns.md`.)

### Think in sets, not records — a loop is the smell

The ORM is plural: it prefetches, batches, and groups across the whole recordset. The moment you iterate to query, write, or sum, name the set-operation you're dodging — a `search` per row becomes one search with `in` then `.grouped()`; N `create`s become one `create(vals_list)`; a Python sum over `mapped` becomes `_read_group`; an x2many write-loop becomes `Command.set`. Index the fields your domains filter and join on. (`references/performance.md`.)

### Stored data is a promise to the future

A compute owns exactly one field — its own; writing a second field from inside it makes recompute order a lottery. A *stored* compute promises its value derives only from its declared `@api.depends` — read context, time, `env.user`, or another model's unstored state and the next cron recompute disagrees with what you wrote (drop `store=True`, or drop the runtime input). `@api.depends` names every real input and nothing else: a missing one means stale data, a non-stored one means pointless recomputes. The field's *shape* is the same kind of promise — renaming, retyping, dropping, or newly requiring it has to answer for the rows that already exist, so it travels *with* a migration, not after one (→ `odoo-migrations`). Reach for `related=` before hand-writing a one-hop compute.

### Work with the framework's grain, not around it

When there's no ready-made helper and you're unsure how to build something, the pattern is almost always already in core — find where standard Odoo does this, or something close, and follow it; you inherit its edge cases and its upgrade path for free. The failure mode is reaching *around* the framework instead: `hasattr` or `_fields` introspection to dodge a missing dependency (and `hasattr` on a `super` proxy is *always* False) when you should just depend on the module that defines the field; an override that drops `*args, **kwargs` and so breaks the day core changes the signature; bare imports inside an addon instead of `from odoo.addons.<module>...`. Guard then `super()` rather than wrapping the body in an `if`, and override `create`/`write` only to intercept persistence — never to read context, which already reaches the compute.

### Whose rights is this running as?

Every `sudo()` and every external entrypoint is a trust boundary — name it before you cross it. To read a protected field (`groups='base.group_system'`), assert the user's right first (`has_group(...)` → `AccessError`), *then* `sudo()`; sudo without the check defeats the point. Webhook and payment controllers verify the signature and are idempotent — guard re-entry (`if tx.state not in ('done', 'cancel', 'error')`) so a replay never double-processes, and never redirect to success when the transaction isn't found.

### Assume multi-company until the task says otherwise

Code as if more than one company shares the database. A `company_id` field doesn't isolate records — the `ir.rule` does; uniqueness and domains are usually per-company; never hardcode `base.main_company`. (Which dimensions truly apply — multi-currency, multi-warehouse, multi-website — is grill's full pass; carry the company reflex when no plan ran.)

## References (consult for the fact, don't memorize)

| Need the exact... | Read |
|---|---|
| compute / constraint / Monetary / domain / override / company-rule snippet | `references/orm-patterns.md` |
| batching / `_read_group` / index / upgrade-`util` pattern | `references/performance.md` |
| external API / device / webhook — session, auth, pagination, errors | `references/external-integration.md` |
| import order, naming, attribute order, transactions, translation, method shape | `references/python-conventions.md` |

At review time see also `odoo-code-review`; for where Python files live in the module, `odoo-module-development`.
