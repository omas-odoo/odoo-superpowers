---
name: odoo-code-review
description: Use when reviewing Odoo code — yours before commit, Claude's after generation, or a teammate's PSDU PR. Catches shape-of-bad-code issues that lint won't, plus customer-readiness gaps. Invoke before claiming Odoo work is done.
---

# Odoo Code Review

Review Odoo code the way an experienced PSDU dev would — for *shape*, not for lint. Lint catches typos; review catches assumptions.

## Principles

### Read for the shape, not the syntax

- **Code should read like Odoo wrote it.** If a reviewer has to ask "why this way?", the code is doing too much explaining. Match the framework's idioms before optimizing.
- **Look at what's *missing*, not just what's there.** No security file? No demo data? No migration for the new required field? Those are the bugs. Lint can't find absent code.
- **Trace one user action end-to-end.** Pick a button or a form save. Follow it from the view, through the model, through access checks, into the DB. If you can't follow it cleanly, the customer's bug report won't be reproducible either.

### ORM correctness

- **Stored computes depending on context, time, or `env.user` are bombs.** They give one value when written and a different one when re-computed. Either drop the `store=True` or drop the runtime dependency.
- **`search(... limit=1)` is fine; `search(...)[0]` is a `KeyError` waiting for the day the search returns nothing.**
- **Recordsets are not lists.** `for rec in records: rec.field = x` is N writes. `records.write({'field': x})` is one.
- **`self.ensure_one()` is for genuinely single-record methods — not a reflex.** If the operation could run over a recordset, batch it instead of forcing one-at-a-time. When you see `ensure_one()` on logic that isn't truly atomic-per-record, ask why it isn't batched.

### Security

- **Every `sudo()` answers a question** — same principle as in module-development. In review, if the answer isn't in a comment, request one.
- **`if self.env.user.has_group(...)`-style checks in business logic are a smell.** The right place is `ir.rule` or `ir.model.access`. Python checks miss aggregations and reports.
- **Look at the access CSV.** Read permissions for `base.group_public` on anything non-public is a finding. Even read.
- **A model method that doesn't use `self` and does external I/O is RPC-attackable.** Every public model method is reachable via RPC and server actions. Self-less external-call logic belongs in a module-level `utils.py`. Flag it when you see it on a model.
- **Secret/credential fields need `groups='base.group_system'`, and reads need access-check-then-`sudo()`.** A `.sudo()` read of a protected field with no preceding `has_group` check defeats the protection.

### Customer-readiness

- **The PR isn't done until you'd be comfortable handing it to the customer without a cover note.** Customer-facing means: clear field labels, translated strings, demo data that demonstrates the feature, a migration that won't blow up their DB.
- **Look at the diff against an `-i` install on a fresh DB.** If the module can't install cleanly on a clean DB, customers can't install it either.

### Migration triggers — the absence test

When the module already exists in a customer DB, a fresh install is irrelevant — what matters is what happens on **upgrade**. Scan the diff for these shapes; each one requires a `migrations/<version>/` script:

| Diff shape | Migration required because... |
|---|---|
| New `required=True` field on an existing model | Existing rows are NULL; upgrade fails on NOT NULL |
| Field type change | Old values may not match the new type |
| Renamed field or model | Old column/table still in DB; data orphaned |
| Removed field or model | FK references break silently |
| New `_sql_constraints` (UNIQUE, CHECK) | Existing rows may already violate it |
| Changed `ondelete` semantics on a Many2one | Customer's FK behavior changes under them |
| Edits to `noupdate="1"` XML records | Customer's customizations may collide |

The reviewer's job is to spot the *absence*. A diff that touches any of the above **and** doesn't bump `__manifest__.py` version **and** doesn't add `migrations/<new_version>/` is the bug, full stop. Flag it before approving.

For the actual script conventions (util-first patterns, pre/post/end timing, verification), invoke the **`odoo-migrations`** skill.

### Tests

- **A test that runs on a polluted DB is worse than no test** — it gives false confidence. Demand the test-runner skill was used.
- **Tests should describe behavior, not implementation.** `test_user_can_see_own_tickets` is a test. `test_compute_method` is not.

## What good looks like

A review that points at three things:
1. A *shape* — "this stored compute reads `env.context`, that's the problem."
2. A *missing* — "I don't see an `ir.model.access.csv` line for the new model."
3. A *customer concern* — "the field label `is_blocked` will appear in French as 'is_blocked' — wrap it in `_()` and add a translation."

## What bad looks like

- A review that's all "use `f'{x}'` instead of `.format()`." That's lint's job, not yours.
- A review that approves without running the module on a fresh DB. The reviewer is now the regression.
- A "LGTM" on a PR that adds `sudo()` with no comment. The next reviewer inherits the problem.

## When you find something

Don't just say "this is wrong." Name the *shape* — "stored compute depending on runtime context" — so the author learns the pattern, not just the fix. That's what makes review compound across reviews.
