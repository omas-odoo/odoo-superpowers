---
name: odoo-xml-conventions
description: Use when writing or reviewing any .xml file in an Odoo module — views, actions, menus, security records, data files, QWeb templates. Holds Odoo's naming, formatting, and inheritance conventions. Invoke before adding records to views/, security/, data/, or report/ directories.
---

# Odoo XML Conventions

XML in Odoo is the API customers see — view labels, action names, security IDs survive across upgrades and end up in their muscle memory. Get the conventions right once, then stop thinking about them.

## Principles

### Conventions are not optional

- **XML IDs are forever.** Once a customer's DB has a record with id `account_invoice_view_form`, renaming it on your side breaks every reference to it. Pick the convention up front, follow it always.
- **Match the canonical names exactly.** The Odoo coding guidelines are not suggestions — they're what every reviewer expects to see, and what `studio`, `web_studio`, and OCA tooling rely on. The reference files in this skill mirror those guidelines verbatim.
- **One concept per file.** Backend views, templates, menus, data, and security each have their own file with their own suffix. Mixing them makes review harder and demo data unpredictable.

### Format > prose

For XML you don't need a principle for every detail — you need the template. The references give you copy-pasteable canonical forms. When in doubt, follow the form.

## When to consult which reference

| Writing... | Read |
|---|---|
| `<record>` for a view, action, group, or rule | `references/xml-format.md` and `references/xml-naming.md` |
| A view that inherits another | `references/xml-inheritance.md` |
| A menu or template tag | `references/xml-format.md` (custom tags section) |
| A `data/*.xml` file | `references/xml-format.md` (noupdate / data tag section) |

Read the relevant reference *before* writing the XML, not after. Renaming IDs after the fact creates migration debt for the customer.

## What good looks like

- A view file where every record's `id` matches the `name` (with dots for underscores), every action has a real human name, and the file groups records by model.
- A security file whose group and rule IDs follow `<module>_group_<role>` and `<model>_rule_<group>` — so a reviewer scanning the CSV can predict the XML IDs.
- An inheriting view that keeps the original ID (module prefix prevents collision) and has a `.inherit.<detail>` suffix on its `name`.

## What bad looks like

- View IDs like `view_form_1`, `my_form`, or anything that doesn't include the model name.
- A single `<data>` file containing both updatable and non-updatable records mixed together.
- An inherited view with a brand-new ID like `partner_form_extended_v2` — reviewers can't find the original at a glance.
- Action `<field name="name">Stuff</field>` — the name is what users see in the breadcrumb. Make it meaningful.

## When the rules genuinely don't fit

PSDU customer work occasionally has bespoke needs (customer-specific prefix, legacy IDs you can't rename without a migration). In those cases:

1. Document the exception in the module's `README.md` or `CHANGELOG.md` — so the next person knows it's deliberate.
2. Don't propagate the exception to new files. Even if old IDs are non-conforming, new ones follow the convention.
3. Log it in `skills/_journal.md` so the `refining-odoo-skills` pass can decide if the principle needs sharpening.
