---
name: odoo-migrations
description: Use when writing or reviewing any migrations/ upgrade script, or when a diff changes a field or model on a module already deployed to a customer. Covers when you owe a migration, the util-first rule, pre/post/end timing, and verifying against a real pre-upgrade snapshot.
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

On PSAE/PS work (odoo.sh, Odoo SA) `odoo.upgrade.util` is available and is **the** way to write migrations. Reach for a `util` helper first — it handles the ORM cache, stored recomputes, column/table existence, FK integrity, and indirect dependencies that bare SQL silently breaks.

```python
from odoo.upgrade import util

util.rename_field(cr, "psae.task", "old_name", "new_name")   # also fixes ir_model_fields + FKs
util.remove_field(cr, "psae.task", "legacy_status")          # column + registry + indirect deps
util.rename_model(cr, "old.model", "new.model")
util.recompute_fields(cr, "psae.task", ["priority_score"])
```

Drop to bare `cr.execute(...)` **only** when no helper covers the case (an arbitrary backfill, a dedupe) — and even then guard with `util.column_exists` / `util.table_exists` first. A `DROP`/`ALTER`/`UPDATE` written directly, with no `util` and no guard, is a review blocker.

### `pre-` operates on the old schema; `post-` on the new

- **`pre-migration.py`** runs before the module's Python loads — old schema, no ORM. Renames, drops, dedupe-before-constraint, raw schema work.
- **`post-migration.py`** runs after install/update — new schema, ORM live. Backfills via ORM, recomputes, noupdate data updates.
- **`end-migration.py`** runs after *all* modules migrate — cross-module cleanup.

Rule of thumb: needs the new schema → `post-`. Operates on the old one → `pre-`.

### Always handle first-install

`migrate(cr, version)` receives `version=None` on a fresh install. Guard it — there's nothing to migrate from:

```python
def migrate(cr, version):
    if not version:
        return
    ...
```

### Verify against a snapshot of the OLD version — not a fresh install

A migration "tested" by installing on a clean DB isn't tested — that's just an install. A real test loads a dump of the *previous* version's data and upgrades it. See `references/migration-patterns.md` for the dump → load → `-u` loop.

## What good looks like

A diff that adds a required field, bumps `__manifest__.py`, adds `migrations/<v>/post-migration.py` that backfills existing rows, and was verified against a snapshot of the live DB.

## What bad looks like

- A new required field, no version bump, no `migrations/` dir → upgrade dies on NOT NULL.
- `cr.execute("ALTER TABLE ... DROP COLUMN ...")` + a hand-written `DELETE FROM ir_model_fields` — that's `util.remove_field` reimplemented badly, and it misses indirect deps.
- A `pre-migration.py` that calls ORM methods (the registry isn't built yet).
- "Tested" only by reinstalling on a clean DB.

For the full util catalog, per-shape code, version-bump convention, and the verification loop, see `references/migration-patterns.md`.
