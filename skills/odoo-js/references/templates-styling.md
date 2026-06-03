# OWL Templates & Styling Reference

Use when writing or inheriting a QWeb/OWL template (`static/src/**/*.xml`) or SCSS. The principles live in `SKILL.md`; this is the exact-rules lookup, from recurring PSAE review feedback.

> This covers **OWL/QWeb client templates**. For backend *view* XML (`<record model="ir.ui.view">`, actions, menus, security) see the `odoo-xml-conventions` skill instead.

## Directives

### `t-out`, never `t-esc`

`t-esc` is deprecated — replace every instance. `t-out` is for rendering a *value*; a literal needs no directive.

```xml
<!-- BAD -->          <span t-esc="line.name"/>          <td t-out="'Total'"/>
<!-- GOOD -->         <span t-out="line.name"/>          <td>Total</td>
```

### Conditional classes — object form

Pass an object whose keys are classes and values are boolean expressions. Far clearer than a ternary that builds a class string, and it merges with a static `class`.

```xml
<!-- BAD -->
<div t-att-class="state.ok ? 'btn-primary' : 'btn-secondary'"/>
<!-- GOOD -->
<button class="btn"
        t-att-class="{ 'btn-primary': state.ok, 'btn-secondary': !state.ok }"/>
```

When the expression is non-trivial, move it to a getter and let the template read it: `t-att-class="badgeClass"`.

### `t-attf-*` for string interpolation

Use `t-attf-src` (etc.) when you're formatting a string; reserve `t-att-src` for a plain expression.

```xml
<img t-attf-src="/web/image/res.company/{{ props.company.id }}/logo"/>
```

### QWeb-JS allows `and` / `or`

In the JS flavour of QWeb you may write `and`/`or` instead of `&amp;&amp;`/`||` — easier to read and no entity escaping.

```xml
<t t-if="line.price >= 0 and line.quantity >= 0"> ... </t>
```

### No `data-*` attributes, no inline logic-by-DOM

A `data-*` attribute on an OWL node is almost always a sign you intend to read it back out of the DOM — use component state/props instead. Don't reference values via `this.` in templates where the bare name works (`props.x`, `state.x`, getters by name).

## Inheriting a template

```xml
<t t-name="my_module.OrderReceipt"
   t-inherit="point_of_sale.OrderReceipt" t-inherit-mode="extension">
    <xpath expr="//div[hasclass('receipt-header')]" position="after">
        <span t-out="props.data.company.arabic_name"/>
    </xpath>
</t>
```

- **Always set `t-inherit-mode`** (almost always `extension`).
- **Hide with `d-none`, not `t-if="False"` / removal / `position="replace"`.** Deleting a node that another module's xpath targets breaks that module silently. Add a class so the node stays in the tree:
  ```xml
  <xpath expr="//div[hasclass('total-orders')]" position="attributes">
      <attribute name="t-att-class" add="d-none" separator=" "/>
  </xpath>
  ```
- **`position="replace"` is banned on standard nodes** — target the parent and *add*, or change attributes. (Same rule as backend views.)
- **xpath on structure, not strings.** Use `hasclass('foo')` rather than `@class='foo'` (exact-match) and avoid predicating on directive text like `@t-if='...'`. Keep the expression as short as stays unambiguous.

## Building DOM from JS — render a template, don't concatenate

If you're assembling elements in JS (legacy/public widgets), define a template and render it rather than `createElement`/string-building:

```js
import { renderToElement } from "@web/core/utils/render";
const el = renderToElement("my_module.button", { name, classes });
container.append(el);
```
`new Option(name, id)` beats `document.createElement("option")` + assignments for the common `<option>` case.

## Styling

Order of preference: **existing Bootstrap utility class → a SCSS class → (never) inline `style`.**

```xml
<!-- BAD -->  <div style="position:absolute;bottom:0;opacity:.5;"/>
<!-- GOOD -->  <div class="position-absolute bottom-0 opacity-50"/>
```

- **Inline styles don't get cached** and cost refresh time — a real, repeated reviewer point, not a style nit.
- If you'll vary a property a lot (widths, etc.), generate utility classes with a SCSS loop instead of one class per value:
  ```scss
  @each $pc in (10, 25, 50, 75, 100) {
      .w-#{$pc} { width: percentage($pc / 100); }
  }
  ```
- **Bootstrap's grid is not loaded in POS.** `class="row"` / `col-sm-*` do nothing on POS screens — use flex utilities or your own SCSS there.
- Reserve `!important` for genuinely out-shadowing a Bootstrap rule — and leave a note when you do.
