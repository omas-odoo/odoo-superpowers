---
name: odoo-xml-conventions
description: Use when writing, reviewing, or migrating any .xml file in an Odoo module — views, actions, menus, security records, data files, QWeb templates. Holds Odoo's naming, formatting, inheritance, and cross-version syntax conventions. Invoke before adding records to views/, security/, data/, or report/ directories.
---

# Odoo XML Conventions

Write the backend XML an experienced reviewer wouldn't comment on — so the review is about the design, not the naming. These are stances to carry, not a checklist to run: the exact patterns — naming tables, attribute order, xpath and version syntax — live in `references/`, pulled only when you need the fact. When a task skips the planning pass (`odoo-grill`), these reflexes carry the design alone.

Scope is *backend* XML — view, action, menu, security, and data records. OWL/QWeb **client** templates under `static/src` are **`odoo-js`**'s domain — the inheritance instinct rhymes (extend, don't replace) but the directives and file conventions differ.

## The stances

### An external id is a contract, not a label

The XML id outlives the code that writes it: once a customer DB stores `account_move_view_form`, every `ref=`, every inheriting module, every Studio customization, and every migration script points at that string — renaming it isn't a rename, it's a break. So spend the naming thought up front, module-prefixed and descriptive, so the next person finds the record by *guessing* its id. The canonical patterns aren't aesthetics; they're the shared vocabulary reviewers and tooling (`web_studio`, OCA) parse. Legacy non-conforming ids you can't rename without a migration stay as-is — but new ids you add still follow the convention; don't let the exception spread. (Exact patterns: `references/xml-naming.md`.)

### Inherit a view — never redefine it

When you need a field on someone else's form, your default is an `xpath` into their view, not a fresh copy: a copy forks the UI and silently drops every *other* module's contributions the day core changes the original. Anchor the xpath on something stable — a named field, a named page — never a position index, which shifts the moment any module inserts above you. An extension keeps the original's id and earns the `.inherit.<module>` name suffix; a *primary* view is a genuinely new record and takes a new id with no suffix. Get the suffix wrong (underscores where dots belong, or missing entirely) and the override loads silently wrong. (xpath, primary-vs-extension, suffix mechanics: `references/xml-inheritance.md`, `references/xml-naming.md`.)

### The view is the UI — don't rebuild it in Python

Odoo's view layer already renders labels, formats, conditional visibility, and grouping; reaching into Python to assemble HTML, hardcode a layout, or compute what a `<filter>` or an `invisible=` expression *states* declaratively is fighting the framework — it won't theme, translate, or survive an upgrade. An empty `<search>` is the same mistake in miniature: it overrides the freebie Odoo generates with something worse, so give it the obvious search field, the lifecycle filters, and a group-by. Add a view type only when the model earns it — each one is surface area someone else maintains. (search and view-type forms: `references/xml-format.md`.)

### Declare data with the narrowest semantics it needs

`noupdate` is a promise about who owns a record after install: `noupdate="1"` says "the customer may edit this and my next module update must not stomp it" (config seed, parameters), plain records say "this is mine, re-asserted on every upgrade" (views, actions, security). Decide per-record intent, not per-file habit — wrap only the locked records in `<data noupdate="1">`, and if the *whole* file is locked put it on `<odoo>` directly rather than a redundant `<data>`. Keep one concept per file (views / menus / security / data each their own) so review and demo-data loading stay predictable. (noupdate and data-tag forms: `references/xml-format.md`.)

### Match the target version's era — a mismatch fails silent

XML doesn't reject an attribute from the wrong Odoo era, it *ignores* it: a v16 `attrs=` on a v17 DB simply never hides the field, no error to catch it. So before writing gating or inheriting a core view, know which major you target and which tag/attribute it actually renders — `<tree>` vs `<list>`, `attrs=`/`states=` vs inline `invisible=`/`readonly=`. When inheriting, your xpath must match the tag the target version renders, not the one you remember. (Era-by-era syntax: `references/version-changes.md`; the JS side of a port: `odoo-js`.)

## References (consult for the fact, don't memorize)

| Need the exact... | Read |
|---|---|
| id / name pattern for a view, action, menu, group, or rule | `references/xml-naming.md` |
| attribute order, `<menuitem>`/`<template>`, `<search>`, `noupdate`/`<data>` form | `references/xml-format.md` |
| xpath anchor, primary-vs-extension, `position=` values | `references/xml-inheritance.md` |
| pre/post-v17 gating, `<tree>`→`<list>`, `column_invisible` | `references/version-changes.md` |

At review time see also `odoo-code-review`; for where each XML file lives in the module, `odoo-module-development`.
