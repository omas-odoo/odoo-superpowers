# XML Format Reference

Use when writing or reviewing any `<record>`, `<menuitem>`, `<template>`, or data file. Formatting rules from the Odoo 19 Coding Guidelines.

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

## Custom tags (syntactic sugar)

Use these instead of the equivalent `<record>`:

| Tag | Replaces |
|---|---|
| `<menuitem>` | `<record model="ir.ui.menu">` |
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

## Quick checklist before saving an XML file

- [ ] All `id` attributes match the model/role naming pattern (see `xml-naming.md`)
- [ ] All `name` fields mirror the id with dots replacing underscores (for ir.ui.view records)
- [ ] Records grouped by model
- [ ] No `<data>` tag wrapping the whole file just to use `noupdate=1` — put it on `<odoo>` instead
- [ ] Actions have human-readable names
- [ ] Used `<menuitem>` / `<template>` when applicable
