---
name: odoo-module-development
description: Use when designing or writing any Odoo module — new models, inherited models, views, security records, migrations, or demo data. Invoke before creating __manifest__.py or any file under a module directory.
---

# Odoo Module Development

A module should be self-describing — the manifest, models, views, security, and demo data each answer one question about what it does and who can use it.

## Principles

### Module shape

- **The manifest is the cover letter.** A reviewer reading only `__manifest__.py` should know what the module does, what it depends on, what it ships, and which Odoo version it targets. If they have to open `models/` to find out, the manifest is failing its job.
- **One module, one responsibility.** If you're tempted to add a second domain ("project management _and_ invoicing extensions"), that's two modules. Customers pick what they install.
- **Depends are contracts.** Every entry in `depends` is something you rely on. Adding `account` because you import a constant from it means `account` is now in your upgrade path forever. Depend on the smallest thing that works.
- **Split by dependency, don't fan out one module's depends.** A single `product_extra` that depends on `sale + stock + pos + purchase` forces a customer who only runs Sales to install all four. Make a base module on `product`, then thin `*_sale`, `*_stock`, `*_pos` modules that each depend on the base plus their one app.
- **The manifest carries the paperwork.** Beyond purpose and depends: a meaningful `name` following MMC convention (e.g. `Phoenix Fashion | Product`), a real `summary`/`description`, the `project.task` id(s) the work traces to, and `license` `OEEL-1` (not `LGPL`) for customer code.

### Model design

- **Stored values must be deterministic from stored inputs.** Context, time, and current user are _runtime_ — storage is _forever_. If a compute reads `self.env.context`, it can't be stored without a migration nightmare.
- **`_inherit` is for extending. `_inherits` is for delegation. `_name` is for new.** Mixing them up creates fields that exist twice and security that applies once.
- **Field names tell the user, not the developer, what's going on.** `partner_id` not `partner`. `is_active` not `flag`. The user sees this string in tooltips and exports.
- **Computes that are also stored need `@api.depends`.** If you can't list the dependencies, the field shouldn't be stored.
- **Decide required-ness deliberately.** Don't leave fields optional by default — ask "should an empty value be possible here?" A transaction with no amount, a terminal with no company: usually those should be `required=True`. Reviewers flag missing required-ness constantly.
- **Minimize fields — don't store what you can derive.** Two fields where one suffices (a `check_in` and `check_out` boolean instead of one `punch_type`), a stored value you could compute or group on the fly (a `punch_date` when you can group the datetime by date): collapse them. Every stored field is data to migrate and keep consistent.

### Views

- **Views describe data, not behavior.** Buttons trigger methods on the model; the view doesn't own the logic. If a view has `<field>` with a long `attrs=` block branching three ways, the model needs a computed field.

For XML naming, formatting, and inheritance details, invoke the **`odoo-xml-conventions`** skill — it owns those rules.

### Security

- **Every model needs an `ir.model.access.csv` line, even internal ones.** "It's only used internally" is how data leaks happen.
- **Every `sudo()` answers a question.** Why does this bypass access rules? Write the answer in a comment on the same line. If you can't justify it, you don't need it.
- **Record rules go in `ir.rule`, not in Python `if`-statements.** The ORM enforces them; ad-hoc checks miss `search()`, `read_group()`, and report exports.

### Migrations & customer-readiness

- **If a change would strand existing data on upgrade, you owe a migration** — invoke the **`odoo-migrations`** skill, which owns the when/how (util-first, pre/post/end, verification).
- **Demo data is part of the module.** A customer who installs your module should see something useful without typing.
- **Translations are not optional.** All user-facing strings go through `_()`. Customer-facing modules ship `.po` files for the customer's locale.

## References (consult, don't memorize)

The principles above tell you *how to think*. The references below hold the *exact conventions* — file layouts, method-name patterns, import order, idiomatic snippets. Read them when you need the fact, not the judgment.

| Need... | Read |
|---|---|
| Where does this file go? What do I name it? | `references/module-structure.md` |
| A change needs a data migration on a deployed module | invoke the **`odoo-migrations`** skill |
| How to write the Python inside models/ — ORM, computes, performance, style | invoke the **`odoo-python`** skill |
| XML record id, view inheritance, menu/action naming | invoke the **`odoo-xml-conventions`** skill |

If a reference doesn't match what you're seeing in PSDU code (customer legacy, project-specific exception), follow the existing code in that file and log the case in `skills/_journal.md`.
