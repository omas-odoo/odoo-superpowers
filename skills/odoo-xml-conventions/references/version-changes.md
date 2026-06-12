# Cross-Version XML Reference

Use when writing or migrating view XML that must match a specific Odoo major — field/button
gating and the list-view tag both changed at v17. Mixing eras usually fails *silent*: the
wrong-era attribute is ignored, not rejected, so the field simply never hides.

## Field/button gating — the v17 break

Up to v16, conditional visibility/editability is expressed as domain dicts in `attrs`, and
button availability via `states`. From v17, both are inline Python-like boolean expressions
placed directly on the field/button.

### v16 and below

```xml
<field name="partner_id"
    attrs="{'invisible': [('state', '=', 'done')],
            'readonly': [('state', 'not in', ['draft', 'sent'])]}"/>

<button name="action_confirm" states="draft"/>
```

### v17 and above

```xml
<field name="partner_id"
    invisible="state == 'done'"
    readonly="state not in ('draft', 'sent')"/>

<button name="action_confirm" invisible="state != 'draft'"/>
```

The inline form is a Python expression over field values, **not** a list of domain tuples.
`required` follows the same shape: `required="state == 'sale'"`.

## `<tree>` → `<list>` (v17+)

The root list-view arch tag was renamed. v16 and below: `<tree>...</tree>`. v17 and above:
`<list>...</list>`. The `ir.ui.view` `type` is `list` in both eras — only the tag changed — so
an inherited xpath written against `//tree` won't match a v17 `<list>`, and vice versa. When
inheriting a core list view, check which tag the target version actually renders before writing
the xpath.

## List-column hiding — `column_invisible` (v17+)

On a list column, conditional hiding uses `column_invisible`, distinct from the form-field
`invisible`:

```xml
<list>
    <field name="amount" column_invisible="parent.state == 'draft'"/>
</list>
```

`column_invisible` hides the whole column; `invisible` on a list field hides the cell per-row.
On a **form** field, use plain `invisible`. (Pre-v17, the equivalent was
`attrs="{'column_invisible': ...}"` — same migration as every other `attrs`.)

## Detecting the target version

When unsure which era a DB is on, grep a known core view in the target source — `attrs=` vs
inline `invisible=` on the same field answers it immediately:

```bash
grep -rn "attrs=" $ODOO_SRC/odoo-<version>/odoo/addons/sale/views/sale_order_views.xml | head
```

No `attrs=` hits in a recent core view means you're on the inline (v17+) era. For the JS/OWL
side of a cross-version migration (import paths, registries, `publicWidget` → `Interaction`),
see the **odoo-js** skill.
