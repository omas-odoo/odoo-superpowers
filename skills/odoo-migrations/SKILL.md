---
name: odoo-migrations
description: Use when writing or reviewing any migrations/ upgrade script, when a diff changes a field or model on a module already deployed to a customer, or when porting a module to a new Odoo major version. Covers when you owe a migration, the util-first rule, pre/post/end timing, the research-then-mechanical-pass order for version ports, and verifying against a real pre-upgrade snapshot.
---

# Odoo Migrations

A migration is the only thing standing between your code change and a customer's data on upgrade day. Get it wrong and their DB doesn't load. The rules here are non-negotiable because the failure mode is "customer is down."

## Principles

### You owe a migration when the change would strand existing data

A fresh install is irrelevant for a deployed module — what matters is **upgrade**. If your diff matches any of these shapes and the module is already running at a customer, you owe a `migrations/<version>/` script:

| Change | What breaks without a migration |
|---|---|
| New `required=True` field on an existing model | Existing rows are NULL → upgrade fails on NOT NULL |
| Field type change | Old values don't match the new type |
| Renamed field or model | Old column/table orphaned; new one empty |
| Removed field or model | FK references break, `ir_model_fields` orphans |
| New `_sql_constraints` / `models.Constraint` (UNIQUE, CHECK) | Existing rows may already violate it |
| Changed `ondelete` on a Many2one | FK behavior changes under the customer |
| Edited `noupdate="1"` data | Collides with customer customizations |

**Reviewer's absence test:** a diff that touches any shape above, *and* doesn't bump `__manifest__.py` version, *and* doesn't add `migrations/<new_version>/`, is the bug — flag it before approving.

### Always use the upgrade `util` library

On PSAE/PS work (odoo.sh, Odoo SA) `odoo.upgrade.util` is available and is **the** way to write migrations. Reach for a `util` helper first — it handles the ORM cache, stored recomputes, column/table existence, FK integrity, and indirect dependencies (views, server actions, related fields) that bare SQL silently breaks. Helper catalog and per-shape patterns in `references/migration-patterns.md`.

Drop to bare `cr.execute(...)` **only** when no helper covers the case (an arbitrary backfill, a dedupe) — and even then guard with `util.column_exists` / `util.table_exists` first. A `DROP`/`ALTER`/`UPDATE` written directly, with no `util` and no guard, is a review blocker.

### `pre-` operates on the old schema; `post-` on the new

- **`pre-migration.py`** runs before the module's Python loads — old schema, no ORM. Renames, drops, dedupe-before-constraint, raw schema work.
- **`post-migration.py`** runs after install/update — new schema, ORM live. Backfills via ORM, recomputes, noupdate data updates.
- **`end-migration.py`** runs after *all* modules migrate — cross-module cleanup.

Rule of thumb: needs the new schema → `post-`. Operates on the old one → `pre-`.

### Always handle first-install

`migrate(cr, version)` receives `version=None` on a fresh install — guard it and `return`, there's nothing to migrate from. Signature in `references/migration-patterns.md`.

### Verify against a snapshot of the OLD version — not a fresh install

A migration "tested" by installing on a clean DB isn't tested — that's just an install. A real test loads a dump of the *previous* version's data and upgrades it. See `references/migration-patterns.md` for the dump → load → `-u` loop, and `odoo-test-runner` for the verification layers.

### Research what became standard before you port — cheapest source first

Before porting a feature, find out whether the target version already does it as standard. A feature now covered by standard is a **drop, not a port**. Research in cost order: your own knowledge of what became standard in TARGET → official docs → release notes between SOURCE and TARGET → grep `$ODOO_SRC` only as a last resort. Agree KEEP / DROP / REWRITE per feature with the handler before touching code. Never delete an override just because its hook is gone — the business requirement remains; relocate the logic.

### Run the mechanical pass before hand-rewriting

Odoo ships its own codemod scripts — run `odoo-bin upgrade_code` first and let it do the mechanical renames, then hand-fix what's left rather than rewriting from memory. Hand rewrites use the per-file-type skills' version-sensitive sections: `odoo-python` for models, `odoo-xml-conventions` for views, `odoo-js` for the frontend (where upgrades break silently). Exact command and the manifest version reset in `references/migration-patterns.md`.

### A `migrations/` script only when the schema actually changed

A clean port often needs no migration script at all — write one only when the schema actually changed (renamed/removed field or model, new constraint). The same util-first rule applies. Before writing it, check what Odoo SA's own upgrade already handles and don't duplicate a rename the platform does for you.

### Commit version ports under `[UPG]`

A port carries the `[UPG]` tag, not `[IMP]`/`[REF]`: `[UPG][TASK_ID] module: migrate SOURCE → TARGET`. The manifest `version` resets to the target major (`odoo-module-development` for the bump).
