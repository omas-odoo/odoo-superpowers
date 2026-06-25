---
name: using-odoo-superpowers
description: Use at the start of any Odoo PSDU task to know which Odoo-specific skill to reach for. Covers when to invoke odoo-module-development, odoo-code-review, odoo-test-runner, odoo-task-completion, and refining-odoo-skills. Invoke before writing any Odoo model, view, or migration.
---

# Using Odoo Superpowers

You're shipping a module to a customer, not a script to a notebook. Reach for the Odoo skills before generic Python ones.

## Principles

### When to invoke which skill

- **`odoo-grill`** — before you write any code, to pressure-test the plan or design. Interrogates an Odoo plan one question at a time, each with a recommended answer, so the design holds up before module-development starts.
- **`odoo-module-development`** — before you write a new model, security file, or demo data. Anything that ends up in a `__manifest__.py`. Carries a reference for module structure.
- **`odoo-migrations`** — when writing an `migrations/` upgrade script, or when a diff changes a field/model on a module already deployed to a customer. Owns the util-first patterns and the "you owe a migration" absence test.
- **`odoo-python`** — before writing or reviewing any `.py` logic: models, computes, overrides, controllers, wizards. Carries PSAE principles for ORM correctness, performance, style, and references for exact patterns.
- **`odoo-xml-conventions`** — before writing or reviewing any *backend* `.xml` file: views, actions, menus, security records, data files. Carries references for XML naming, format, and inheritance. (For OWL/QWeb *client* templates under `static/src`, reach for `odoo-js`.)
- **`odoo-js`** — before writing or reviewing anything under `static/src`: OWL components, services, `patch()` overrides, QWeb/OWL templates, SCSS. Carries PSAE principles for OWL correctness, the patch discipline, template conventions, and frontend JS style.
- **`odoo-code-review`** — after writing Odoo code and before claiming it's done. Also when reviewing a teammate's PSDU PR.
- **`odoo-test-runner`** — before saying "tests pass." Pollution in a long-lived dev DB makes green tests lie.
- **`odoo-git-commit`** — before any `git commit` on customer code. Owns the splitting discipline (one logical change per commit, split by module, move-then-modify), the `[TAG] module: summary` convention, and the terse-human message bar.
- **`odoo-task-completion`** — when wrapping up a `project.task` ticket. Orchestrates review + test + the "would I show this to the customer" check.
- **`refining-odoo-skills`** — when the user corrects you on something Odoo-specific, or when the journal has accumulated entries. Skills that don't evolve go stale.

### Principles vs references — the two layers

Each skill has a `SKILL.md` (principles — how to think) and optionally a `references/` folder (lookup — exact conventions, copy-pasteable templates). Read the SKILL.md first; pull from references only when you need the fact. References live next to the skill that owns them, not at the plugin root.

### How to think about Odoo work

- **Trace every change to a ticket.** Customer work means a `project.task` id. If you can't name it, you're missing context — ask before writing.
- **Odoo idioms first.** ORM `search`/`read_group`/`mapped` beat raw SQL. `_inherit` beats monkey-patching. Views before computed HTML. When you're reaching around the framework, stop and re-read the docs.
- **The framework is the API contract.** Customers upgrade Odoo versions. If your code only works because of a private method's current behavior, it's a time bomb.
- **Customer-readiness > "it works on my machine."** Demo data, security records, translations, and migrations are part of the deliverable, not afterthoughts.

### Cross-harness behavior

This plugin works in Claude Code, Codex, and Gemini CLI. Skill invocation differs by harness:

- **Claude Code:** use the `Skill` tool with the skill name (no leading slash).
- **Codex:** use the `skill` tool the same way.
- **Gemini CLI:** skills activate via `activate_skill`; this entry skill plus `CLAUDE.md` are loaded automatically via `GEMINI.md`.

If you're unsure which harness you're in, the file format is identical — read the SKILL.md directly as a last resort, but prefer the tool if available.

### The learning loop

Every correction is data. If the user tells you "we don't do it that way in PSDU," or you catch yourself doing something un-Odoo-like:

1. Append an entry to `skills/_journal.md` with date, the concrete case, and what you should have done.
2. When the journal hits 5+ entries, suggest running `refining-odoo-skills` to fold the lessons back into the actual skill files.

Don't let the journal become a graveyard. Surface it.
