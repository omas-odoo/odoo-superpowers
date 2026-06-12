---
name: odoo-xml-conventions
description: Use when writing, reviewing, or migrating any .xml file in an Odoo module — views, actions, menus, security records, data files, QWeb templates. Holds Odoo's naming, formatting, inheritance, and cross-version syntax conventions. Invoke before adding records to views/, security/, data/, or report/ directories.
---

# Odoo XML Conventions

XML in Odoo is the API customers see — view labels, action names, security IDs survive across upgrades and end up in their muscle memory. Get the conventions right once, then stop thinking about them.

> **Scope:** this skill covers *backend* XML — `ir.ui.view` records, actions, menus, security, data, reports. For OWL/QWeb **client templates** under `static/src` (`t-name`, `t-out`, `t-att-class`, `t-inherit`), use the **`odoo-js`** skill instead — the inheritance rules rhyme (no `position="replace"`, hide with `d-none`) but the directives and file conventions differ.

## Principles

### Conventions are not optional

- **XML IDs are forever.** Once a customer's DB has a record with id `account_invoice_view_form`, renaming it on your side breaks every reference to it. Pick the convention up front, follow it always.
- **Match the canonical names exactly.** The Odoo coding guidelines are not suggestions — they're what every reviewer expects to see, and what `studio`, `web_studio`, and OCA tooling rely on. The reference files in this skill mirror those guidelines verbatim.
- **One concept per file.** Backend views, templates, menus, data, and security each have their own file with their own suffix. Mixing them makes review harder and demo data unpredictable.

### Format > prose

For XML you don't need a principle for every detail — you need the template. The references give you copy-pasteable canonical forms. When in doubt, follow the form.

### Views earn their place

- **Never ship an empty `<search>`.** A search view with no fields, filters, or group-by is a worse default than the one Odoo generates for free — give the obvious search field, the status filters, and a sensible group-by.
- **Justify every view type.** `list` and `form` always; `kanban`, `pivot`, `gantt`, `calendar`, `graph` only when the model genuinely earns one — each is surface area someone else maintains. Don't add views "just in case." → `references/xml-format.md`

## When to consult which reference

| Writing... | Read |
|---|---|
| `<record>` for a view, action, group, or rule | `references/xml-format.md` and `references/xml-naming.md` |
| A view that inherits another | `references/xml-inheritance.md` |
| A menu or template tag | `references/xml-format.md` (custom tags section) |
| A `data/*.xml` file | `references/xml-format.md` (noupdate / data tag section) |
| A search view, or deciding which view types to add | `references/xml-format.md` (search / view-types section) |
| Gating fields/buttons, list vs tree, or any pre/post-v17 syntax | `references/version-changes.md` |

Read the relevant reference *before* writing the XML, not after. Renaming IDs after the fact creates migration debt for the customer.

## When the rules genuinely don't fit

PSDU customer work occasionally has bespoke needs (customer-specific prefix, legacy IDs you can't rename without a migration). In those cases:

1. Document the exception in the module's `README.md` or `CHANGELOG.md` — so the next person knows it's deliberate.
2. Don't propagate the exception to new files. Even if old IDs are non-conforming, new ones follow the convention.
3. Log it in `skills/_journal.md` so the `refining-odoo-skills` pass can decide if the principle needs sharpening.
