# XML ID Naming Reference

Source: Odoo 19 Coding Guidelines. Use this when picking an `id` for any `<record>`, `<menuitem>`, `<template>`, group, or rule.

## The patterns

| Element | Pattern | Example |
|---|---|---|
| Menu (root) | `<model_name>_menu` | `plant_nursery_menu` |
| Menu (sub) | `<model_name>_menu_<detail>` | `plant_nursery_menu_orders` |
| View | `<model_name>_view_<type>` | `plant_nursery_view_form`, `plant_order_view_kanban` |
| **Inherited view (id)** | **same as original record's id** | `plant_nursery_view_form` (when inheriting `plant_nursery.plant_nursery_view_form`) |
| Action (primary) | `<model_name>_action` | `plant_nursery_action` |
| Action (extra) | `<model_name>_action_<detail>` | `plant_nursery_action_archive` |
| Window action by view | `<model_name>_action_view_<type>` | `plant_order_action_view_kanban` |
| Group | `<module_name>_group_<role>` | `plant_nursery_group_user`, `plant_nursery_group_manager` |
| Record rule | `<model_name>_rule_<group>` | `plant_order_rule_user`, `plant_order_rule_company`, `plant_order_rule_public` |

View types for the `<view_type>` slot: `form`, `list`, `kanban`, `search`, `pivot`, `graph`, `calendar`, `activity`, `gantt`, `cohort`, `dashboard`.

Common group roles: `user`, `manager`, `admin`, `portal`, `public`.

Common rule groups: `user` (matches `_group_user`), `manager`, `company` (multi-company), `public`.

## `name` field convention

For `ir.ui.view` records, the `name` field mirrors the `id` with **dots replacing underscores**:

```xml
<record id="plant_nursery_view_form" model="ir.ui.view">
    <field name="name">plant.nursery.view.form</field>
    ...
</record>

<record id="plant_nursery_view_kanban" model="ir.ui.view">
    <field name="name">plant.nursery.view.kanban</field>
    ...
</record>
```

For action records, the `name` is the **user-facing display name** (not a dotted technical string):

```xml
<record id="plant_nursery_action" model="ir.actions.act_window">
    <field name="name">Plants</field>
    ...
</record>
```

## Inheritance naming (the most-missed rule)

When you inherit a view from another module, **two things change** — and people get the second one wrong almost every time:

1. **`id` stays the same** as the original record. The module prefix added at install time prevents collision. Don't invent a new id like `plant_nursery_view_form_extended_v2`.
2. **`name` gets a suffix:** `<original.dotted.name>.inherit.<current.module.dotted>`

The trap: **the current module's name uses dots, not underscores.** If your module is `psae_base`, the suffix is `.inherit.psae.base` — not `.inherit.psae_base`.

### The formula

```
name = <original record's `name` field, verbatim> + ".inherit." + <current module's technical name, with underscores → dots>
```

### Worked example

You're in module `psae_base`. You're inheriting `plant_nursery.plant_nursery_view_form`. The original view's `name` field is `plant.nursery.view.form`.

```xml
<!-- in addons/psae_base/views/plant_nursery_views.xml -->
<record id="plant_nursery_view_form" model="ir.ui.view">
    <field name="name">plant.nursery.view.form.inherit.psae.base</field>
    <field name="inherit_id" ref="plant_nursery.plant_nursery_view_form"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='partner_id']" position="after">
            <field name="psae_custom_field"/>
        </xpath>
    </field>
</record>
```

| If current module is... | Suffix is... |
|---|---|
| `psae_base` | `.inherit.psae.base` |
| `psae_account_extra` | `.inherit.psae.account.extra` |
| `customer_acme_2024` | `.inherit.customer.acme.2024` |
| `sale` | `.inherit.sale` |

If your `inherit_id` ref points to a record but the `name` doesn't end in `.inherit.<your.module>`, that's the bug. Fix the name.

### Primary views are different

A primary view (`mode="primary"`) is a *new* record based on another. It needs a new `id` and the `name` does **not** get the `.inherit` suffix. See `xml-inheritance.md` for the full distinction.

## Worked example: a single model's XML records

```xml
<!-- views/plant_nursery_views.xml -->

<!-- views -->
<record id="plant_nursery_view_list" model="ir.ui.view">
    <field name="name">plant.nursery.view.list</field>
    <field name="model">plant.nursery</field>
    <field name="arch" type="xml">
        <list>...</list>
    </field>
</record>

<record id="plant_nursery_view_form" model="ir.ui.view">
    <field name="name">plant.nursery.view.form</field>
    <field name="model">plant.nursery</field>
    <field name="arch" type="xml">
        <form>...</form>
    </field>
</record>

<!-- action -->
<record id="plant_nursery_action" model="ir.actions.act_window">
    <field name="name">Plants</field>
    <field name="res_model">plant.nursery</field>
    <field name="view_mode">list,form</field>
</record>

<!-- menus -->
<menuitem id="plant_nursery_menu_root" name="Nursery" sequence="5"/>
<menuitem
    id="plant_nursery_menu_action"
    name="Plants"
    parent="plant_nursery_menu_root"
    action="plant_nursery_action"
    sequence="10"
/>
```

```xml
<!-- security/plant_nursery_groups.xml -->
<record id="plant_nursery_group_user" model="res.groups">
    <field name="name">Plant Nursery / User</field>
</record>

<record id="plant_nursery_group_manager" model="res.groups">
    <field name="name">Plant Nursery / Manager</field>
    <field name="implied_ids" eval="[(4, ref('plant_nursery_group_user'))]"/>
</record>
```

```xml
<!-- security/plant_nursery_security.xml -->
<record id="plant_nursery_rule_user" model="ir.rule">
    <field name="name">Plant Nursery: users see own records</field>
    <field name="model_id" ref="model_plant_nursery"/>
    <field name="domain_force">[('create_uid', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('plant_nursery_group_user'))]"/>
</record>
```

## Common mistakes

| Wrong | Right | Why |
|---|---|---|
| `view_form_1` | `plant_nursery_view_form` | Include model name; type goes at the end |
| `plant_nursery_form_view` | `plant_nursery_view_form` | Convention is `_view_<type>`, not `_<type>_view` |
| `action_plant_nursery` | `plant_nursery_action` | Convention is `<model>_action`, not `action_<model>` |
| `group_plant_user` | `plant_nursery_group_user` | Module name before `_group_`, not after |
| `<field name="name">Plant Nursery Form</field>` | `<field name="name">plant.nursery.view.form</field>` | View names are dotted technical strings; action names are human strings |
| Inherited view: new id like `plant_nursery_view_form_psae` | Same id as original: `plant_nursery_view_form` | Module prefix prevents collision; same id makes overrides discoverable |
| Inherited view name: `plant.nursery.view.form.inherit.psae_base` | `plant.nursery.view.form.inherit.psae.base` | Module name in the suffix uses **dots**, not underscores — this is the most common XML mistake |
| Inherited view name: `plant.nursery.view.form.inherit` (no module) | `plant.nursery.view.form.inherit.psae.base` | The module suffix is required — tells reviewers which module is doing the override |
