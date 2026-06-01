# XML View Inheritance Reference

Use when extending another module's view or creating a primary view based on one. Inheritance rules from the Odoo 19 Coding Guidelines.

## Standard inheritance

Two rules, and people almost always get the second one wrong:

1. **`id`: same as the original record.** Module prefix added at install time prevents collision. Don't invent a new id.
2. **`name`: `<original.dotted.name>.inherit.<current.module.dotted>`** — the current module's technical name with **underscores converted to dots**.

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

| Current module | Inherit suffix |
|---|---|
| `psae_base` | `.inherit.psae.base` |
| `psae_account_extra` | `.inherit.psae.account.extra` |
| `customer_acme_2024` | `.inherit.customer.acme.2024` |
| `sale` | `.inherit.sale` |

If your inheriting view doesn't install, or reviewers can't find your override, this is almost always the bug: the `name` field uses underscores where it should have dots, or is missing the module suffix entirely.

Why same `id`: when someone searches "where is `plant_nursery_view_form` modified?", they find every inheriting module instantly.

## Primary views

A primary view is a *new* view based on another (using `mode="primary"`). It needs its own `id` because it's a different record.

```xml
<record id="model_view_form_alternate" model="ir.ui.view">
    <field name="name">model.view.form.module2</field>
    <field name="inherit_id" ref="module1.model_view_form"/>
    <field name="mode">primary</field>
    <field name="arch" type="xml">
        ...
    </field>
</record>
```

The `name` does **not** get the `.inherit` suffix — primary views aren't overrides, they're new views.

## When to use which

| You want to... | Use |
|---|---|
| Add a field, button, or section to an existing view that everyone uses | Standard inheritance (same id, `.inherit.<detail>` name) |
| Build a different view of the same model for a specific action or context | Primary view (new id, no `.inherit` suffix) |
| Completely replace a view | Either, but prefer extending unless you genuinely need a parallel form |

## XPath against stable anchors

Inherit by targeting elements with `name` attributes or unique structural points — not by position.

```xml
<!-- Stable: targets a named field -->
<xpath expr="//field[@name='partner_id']" position="after">
    <field name="custom_field"/>
</xpath>

<!-- Stable: targets a notebook page by string -->
<xpath expr="//page[@name='details']" position="inside">
    ...
</xpath>

<!-- Fragile: breaks when fields are reordered -->
<xpath expr="//field[3]" position="after">
    ...
</xpath>
```

Position values: `inside` (default), `after`, `before`, `replace`, `attributes`.

## `position="attributes"`

When you only want to change an attribute on an existing field:

```xml
<xpath expr="//field[@name='partner_id']" position="attributes">
    <attribute name="readonly">1</attribute>
    <attribute name="string">Customer</attribute>
</xpath>
```

## Quick checklist before committing an inheriting view

- [ ] Same `id` as the original (if standard inheritance)
- [ ] New `id` and `mode="primary"` (if a parallel view)
- [ ] `name` ends in `.inherit.<current.module.with.dots>` (standard) or has no `.inherit` suffix (primary)
- [ ] `inherit_id` ref is `<module>.<id>` form
- [ ] xpath targets a named anchor, not a position
