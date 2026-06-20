# Migration Patterns Reference

Use when writing the actual code for a migration script. The stances (why, when, the absence test) live in `SKILL.md`; this is the exact-code lookup. Everything here is `util`-first (Odoo upgrade-util + PSAE practice) — bare `cr.execute` appears only where no helper exists, always guarded.

## When you owe a migration — what breaks without one

On a module already running at a customer, any of these shapes strands existing data unless a `migrations/<version>/` script handles it:

| Change | What breaks without a migration |
|---|---|
| New `required=True` field on an existing model | Existing rows are NULL → upgrade fails on NOT NULL |
| Field type change | Old values don't match the new type |
| Renamed field or model | Old column/table orphaned; new one empty |
| Removed field or model | FK references break, `ir_model_fields` orphans |
| New `_sql_constraints` / `models.Constraint` (UNIQUE, CHECK) | Existing rows may already violate it |
| Changed `ondelete` on a Many2one | FK behavior changes under the customer |
| Edited `noupdate="1"` data | Collides with customer customizations |

A fresh install hits none of these — only an upgrade does. The version bump in `__manifest__.py` is what triggers Odoo to run the matching `migrations/<version>/` directory.

## Where scripts live

```
my_module/
├── __manifest__.py             # version bumped, e.g. 19.0.1.0.0 → 19.0.1.0.1
└── migrations/
    └── 19.0.1.0.1/             # the version you're migrating TO
        ├── pre-migration.py    # old schema, no ORM
        ├── post-migration.py   # new schema, ORM live
        └── end-migration.py    # after all modules migrate
```

Directory name = the version you're upgrading **to**. Odoo runs every `migrations/<v>/` whose version falls between the previously-installed version and the new one.

## Script signature

```python
from odoo.upgrade import util

def migrate(cr, version):
    if not version:        # fresh install — nothing to migrate from
        return
    ...
```

## util helper catalog (reach for these first)

| Helper | Does |
|---|---|
| `util.rename_field(cr, model, old, new)` | Rename a field — column + `ir_model_fields` + FKs |
| `util.remove_field(cr, model, name)` | Drop a field — column + registry + indirect deps |
| `util.rename_model(cr, old, new)` | Full model rename — table, `ir_model`, `ir_model_data`, relations |
| `util.recompute_fields(cr, model, [fields], ids=None)` | Recompute stored fields safely |
| `util.column_exists(cr, table, col)` / `util.table_exists(cr, table)` | Guard before raw schema work |
| `util.replace_record_references` | Repoint references from one record to another |
| `util.explode_execute(cr, query, table=...)` | Run a heavy UPDATE chunked, for huge tables |
| `util.env(cr)` | A clean `Environment` when you must use the ORM |
| `util.add_to_migration_reports(...)` | Surface a note in the customer's upgrade report |

## Per-shape patterns

### Rename a field — one line

```python
# pre-migration.py
def migrate(cr, version):
    if not version:
        return
    util.rename_field(cr, "psae.task", "old_priority", "priority")
```

### Drop a field — one line

```python
# pre-migration.py
def migrate(cr, version):
    if not version:
        return
    util.remove_field(cr, "psae.task", "legacy_status")   # column + ir_model_fields + deps
```

Do **not** hand-roll `ALTER TABLE ... DROP COLUMN` + `DELETE FROM ir_model_fields` — that's `remove_field` reimplemented, and it misses indirect dependencies (views, server actions, related fields) that `util` cleans up.

### Rename a model — one line

```python
# pre-migration.py
def migrate(cr, version):
    if not version:
        return
    util.rename_model(cr, "psae.old.model", "psae.new.model")
```

### Recompute stored fields

```python
# post-migration.py  (new schema must be live)
def migrate(cr, version):
    if not version:
        return
    util.recompute_fields(cr, "psae.task", ["priority_score"])
```

### Backfill a new required field — bare SQL is the legitimate exception

No util helper does an arbitrary backfill, so a guarded `UPDATE` is correct here. The new column already exists by the time `pre-` runs.

```python
# pre-migration.py
def migrate(cr, version):
    if not version:
        return
    if util.column_exists(cr, "psae_task", "priority"):
        cr.execute("UPDATE psae_task SET priority = 'normal' WHERE priority IS NULL")
```

### Dedupe before a new UNIQUE constraint — bare SQL, guarded

```python
# pre-migration.py
def migrate(cr, version):
    if not version:
        return
    cr.execute("""
        DELETE FROM psae_task a USING psae_task b
         WHERE a.id > b.id AND a.reference = b.reference
    """)
```

## Manifest version bumping

```python
{
    "name": "PSAE Base",
    "version": "19.0.1.0.1",   # was 19.0.1.0.0 — the bump is what triggers the migration check
}
```

Convention `<odoo_version>.<major>.<minor>.<patch>`. Bump the version but forget the `migrations/<version>/` directory and the upgrade silently does nothing about your data.

## Porting a module to a new major version

The judgment (research order, mechanical-pass-first, schema-changed-only, `[UPG]` tag) lives in `SKILL.md`; these are the exact commands.

### Mechanical codemod pass — run first

```bash
odoo-bin upgrade_code <module>   # Odoo's own codemods, e.g. 18.1-00-sql-constraint.py
```

Run this before any hand-rewriting — it applies the renames/rewrites the platform already knows about. Then hand-fix what it leaves, using the version-sensitive sections of `odoo-python` / `odoo-xml-conventions` / `odoo-js`.

### Manifest version on a port — reset, don't patch-bump

```python
{
    "version": "19.0.1.0.0",   # TARGET major, reset to .1.0.0 — NOT a +1 patch bump
}
```

A port resets to `<TARGET>.0.1.0.0`; an in-version data migration patch-bumps the last segment (see "Manifest version bumping" above).

### Check what Odoo SA's upgrade already handles before writing a script

```bash
grep -r "<field_or_model>" $ODOO_SRC/upgrade/migrations/          # Odoo SA scripts
grep -r "<field_or_model>" $ODOO_SRC/upgrade-specific/scripts/    # PSAE-specific scripts
```

If the platform already renames/merges it, don't duplicate it in your `migrations/` script.

## Verification — against a snapshot of the OLD version

```bash
# Dump the live (sanitized) customer DB at the CURRENT version
odoo-bin db dump <customer_clone> /tmp/before.zip

# Load it into a throwaway DB
odoo-bin db load tmp_test_migration /tmp/before.zip --force

# Upgrade with your new code
odoo-bin -d tmp_test_migration -u <module> --stop-after-init

# Spot-check the migrated data
odoo-bin -d tmp_test_migration shell
```

If the upgrade errors, the migration is wrong. If it passes but the data looks off, it's incomplete. A clean-DB reinstall proves nothing — the data has to come from the previous version. See `odoo-test-runner` for the verification layers.
