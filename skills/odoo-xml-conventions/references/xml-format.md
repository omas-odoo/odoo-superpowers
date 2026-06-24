# XML Format Reference

Use when writing or reviewing any `<record>`, `<menuitem>`, `<template>`, or data file. Formatting rules from the Odoo 19 Coding Guidelines.

## No XML prolog

Start every data file with `<odoo>` directly. Never add the `<?xml version="1.0" encoding="utf-8"?>` declaration — XML already defaults to UTF-8 and version 1.0, so the prolog is pure noise repeated on every file.

## Record notation

For `<record>`-style declarations, follow this attribute order:

1. `id` attribute **first**, then `model`.
2. Inside fields: `name` first, then either inner value or `eval=`, then other attributes (widget, options, ...) ordered by importance.
3. Group records by model when possible (skip if action → menu → view dependencies force a different order).

```xml
<record id="view_id" model="ir.ui.view">
    <field name="name">view.name</field>
    <field name="model">object_name</field>
    <field name="priority" eval="16"/>
    <field name="arch" type="xml">
        <list>
            <field name="my_field_1"/>
            <field name="my_field_2" string="My Label" widget="statusbar" statusbar_visible="draft,sent,progress,done"/>
        </list>
    </field>
</record>
```

## Don't restate the field name

Odoo derives a field's label from its name (`partner_id` → "Partner", stripping `_id` and title-casing). A `string=` that just reproduces that derived label is dead noise — drop it, in both the view (`<field name="partner_id" string="Partner"/>`) and the model definition (`partner_id = fields.Many2one(..., string="Partner")`). Set `string=` only to *override* the label with something the field name can't express ("Landlord", "Ejari #").

## Custom tags (syntactic sugar)

Use these instead of the equivalent `<record>`:

| Tag          | Replaces                                                           |
| ------------ | ------------------------------------------------------------------ |
| `<menuitem>` | `<record model="ir.ui.menu">`                                      |
| `<template>` | `<record model="ir.ui.view">` when only `arch` is set (QWeb views) |

```xml
<menuitem
    id="model_name_menu_root"
    name="Main Menu"
    sequence="5"
/>
<menuitem
    id="model_name_menu_action"
    name="Sub Menu 1"
    parent="module_name.model_name_menu_root"
    action="model_name_action"
    sequence="10"
/>

<template id="portal_layout" inherit_id="portal.portal_layout">
    <xpath expr="..." position="...">
        ...
    </xpath>
</template>
```

## The `<data>` tag

Use `<data>` **only** to declare `noupdate="1"` data inside a file that also has updatable records.

If the **whole file** is non-updatable, drop the `<data>` tag and set `noupdate="1"` on `<odoo>` directly:
```xml
<!-- whole file is non-updatable -->
<odoo noupdate="1">
    <record id="..." model="...">
        ...
    </record>
</odoo>
```

```xml
<!-- mixed: most records updatable, a few not -->
<odoo>
    <record id="updatable_one" model="...">...</record>

    <data noupdate="1">
        <record id="locked_one" model="...">...</record>
    </data>
</odoo>
```

## Action records

Actions are user-visible — the `name` field shows in breadcrumbs. Give it a real display name, not a technical string.

```xml
<record id="model_name_action" model="ir.actions.act_window">
    <field name="name">Plants</field>
    <field name="res_model">plant.nursery</field>
    <field name="view_mode">list,form,kanban</field>
</record>
```

## Search views — never empty

A `<search>` with no content is worse than the default Odoo generates: it overrides the freebie
with nothing. Always give the obvious search field(s), the status/lifecycle filters, and a
`Group By` group:

```xml
<record id="plant_nursery_view_search" model="ir.ui.view">
    <field name="name">plant.nursery.view.search</field>
    <field name="model">plant.nursery</field>
    <field name="arch" type="xml">
        <search>
            <field name="name"/>
            <field name="partner_id" operator="child_of"/>
            <filter string="Draft" name="filter_draft" domain="[('state', '=', 'draft')]"/>
            <separator/>
            <group expand="0" string="Group By">
                <filter string="Partner" name="group_partner" context="{'group_by': 'partner_id'}"/>
            </group>
        </search>
    </field>
</record>
```

## Quick checklist before saving an XML file

- [ ] All `id` attributes match the model/role naming pattern (see `xml-naming.md`)
- [ ] All `name` fields mirror the id with dots replacing underscores (for ir.ui.view records)
- [ ] No `<?xml?>` prolog — the file starts at `<odoo>`
- [ ] No `string=` that just restates the field's derived label
- [ ] Records grouped by model
- [ ] No `<data>` tag wrapping the whole file just to use `noupdate=1` — put it on `<odoo>` instead
- [ ] Actions have human-readable names
- [ ] Used `<menuitem>` / `<template>` when applicable
- [ ] Any `<search>` view carries fields, filters, and a group-by — never empty
- [ ] Field/button gating uses the target version's syntax — inline `invisible=` (v17+) vs `attrs=`/`states=` (v16-); see `version-changes.md`
