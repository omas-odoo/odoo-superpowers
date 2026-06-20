---
name: odoo-migrations
description: Use when writing or reviewing any migrations/ upgrade script, when a diff changes a field or model on a module already deployed to a customer, or when porting a module to a new Odoo major version. Covers when you owe a migration, the util-first rule, pre/post/end timing, the research-then-mechanical-pass order for version ports, and verifying against a real pre-upgrade snapshot.
---

# Odoo Migrations

A migration is the only thing between your code change and a customer's data on upgrade day — get it wrong and their DB doesn't load. These are stances to carry, not a checklist to run: the exact `util` signatures, the script layout, and the verification loop live in `references/`, pulled only when you need the fact. When a task skips the planning pass (`odoo-grill`), these reflexes carry the design alone.

## The stances

### A schema change is a data event — you owe a migration for the rows that already exist

Fresh install is irrelevant for a deployed module; upgrade day is. The moment a diff on a customer-running module changes a field's or model's *shape* — newly `required`, retyped, renamed, removed, newly constrained (`UNIQUE`/`CHECK`), or changed `ondelete` — the rows already in their DB no longer fit, and the upgrade fails on a NOT NULL / FK / constraint or silently corrupts. Editing `noupdate="1"` data is the same event: it collides with what the customer already customized. **The absence test, carried into review:** a diff with any of those shapes that doesn't bump `__manifest__.py` *and* doesn't add `migrations/<new_version>/` is the bug — flag it before approving. (Shape-by-shape "what breaks" table in `references/migration-patterns.md`.)

### The migration ships WITH the schema change, not after it

The script, the manifest version bump, and the schema diff are one commit, not a follow-up. The version bump is the *only* thing that makes Odoo look for `migrations/<version>/` — bump without the directory and the upgrade silently does nothing about the data; add the directory without bumping and it never runs. A port carries the `[UPG]` tag (`[UPG][TASK_ID] module: migrate SOURCE → TARGET`) and resets the manifest to the target major; an in-version data fix patch-bumps the last segment.

### The upgrade `util` library does the SQL you'd get wrong by hand

On PSAE/PS work `odoo.upgrade.util` is **the** way to write migrations — reach for a helper before any raw SQL. A field or model is never just a column or table: it's referenced by views, server actions, related fields, `ir_model_data` xml_ids, and the customer's own studio customizations — each reference a contract you can't silently break. Rename or drop the column by hand and every one of those dangles; `util`'s helpers repoint them, fix the FKs, clear the ORM cache, and trigger the stored recomputes — the integrity work you'd forget. Drop to guarded `cr.execute` **only** where no helper covers the case (an arbitrary backfill, a dedupe), and guard it with `util.column_exists` / `table_exists` first. A bare `DROP`/`ALTER`/`UPDATE` with no helper and no guard is a review blocker. (Helper catalog and per-shape one-liners in `references/migration-patterns.md`.)

### The ORM is the wrong tool inside a migration — think in rows, not recordsets

The ORM loads records, runs computes, fires constraints, and recomputes on every write — exactly what you don't want while moving data under a half-migrated schema. So migrations think in SQL (via `util`), not recordsets. This is also why timing splits by schema: **`pre-`** runs before the module's Python loads (old schema, no ORM) — renames, drops, dedupe-before-constraint, raw schema work; **`post-`** runs after the update (new schema, ORM live) — backfills, recomputes, `noupdate` data; **`end-`** runs after *all* modules migrate — cross-module cleanup. Needs the new schema → `post-`; operates on the old → `pre-`. And `migrate(cr, version)` gets `version=None` on a fresh install — guard it and `return`, there's nothing to migrate from.

### Verify against a snapshot of the OLD version, not a fresh install

A migration "tested" by installing on a clean DB isn't tested — that's just an install, and the data it would operate on never existed. A real test loads a dump of the *previous* version's data, upgrades it with your new code, and spot-checks the result: the upgrade errors → the migration is wrong; it passes but the data looks off → it's incomplete. (Dump → load → `-u` loop in `references/migration-patterns.md`; verification layers in `odoo-test-runner`.)

### Porting a version: research what's now standard, then let the codemod do the mechanical pass

A feature the target version now ships as standard is a **drop, not a port** — so find out before you port anything. Research cheapest-source-first: your own knowledge of the target → official docs → release notes between source and target → grep `$ODOO_SRC` only as a last resort; agree KEEP / DROP / REWRITE per feature with the handler before touching code. Never delete an override just because its hook vanished — the business requirement remains, so relocate the logic. Then run Odoo's own codemods (`odoo-bin upgrade_code`) for the mechanical renames *before* hand-rewriting from memory, and hand-fix what's left with the version-sensitive sections of `odoo-python`, `odoo-xml-conventions`, and `odoo-js` (where upgrades break silently). A clean port often needs no `migrations/` script at all — write one only when the schema actually changed, and check what Odoo SA's upgrade already handles before duplicating a rename the platform does for free.

## References (consult for the fact, don't memorize)

| Need the exact... | Read |
|---|---|
| "what breaks" shape table — when you owe a migration | `references/migration-patterns.md` |
| script signature, directory layout, manifest version bump | `references/migration-patterns.md` |
| `util` helper catalog + per-shape one-liners (rename/remove field, rename model, recompute, backfill, dedupe) | `references/migration-patterns.md` |
| port commands — `upgrade_code`, version reset, what-SA-already-handles grep | `references/migration-patterns.md` |
| dump → load → `-u` verification loop | `references/migration-patterns.md` |

The field-shape-is-a-promise stance lives in `odoo-python`; the hand-rewrite during a port uses `odoo-python` / `odoo-xml-conventions` / `odoo-js`; verify with `odoo-test-runner`; at review time `odoo-code-review`.
