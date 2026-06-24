---
name: odoo-code-review
description: Use when reviewing Odoo code — yours before commit, Claude's after generation, or a teammate's PSDU PR. Catches shape-of-bad-code issues that lint won't, plus customer-readiness gaps. Invoke before claiming Odoo work is done.
---

# Odoo Code Review

Review for *shape*, not for lint — lint catches typos, review catches assumptions. These are review stances to carry, not a checklist to run. The file-type conventions live in the skill that owns the file (table below); this skill owns the review act itself — how to read a diff, what only a human-style read catches, and the customer bar.

## Route by file type — load the matching skill

For each changed file, load this skill **plus** the one its path maps to. The right-hand skill carries the shapes; this skill carries how to look.

| File / path | Also load |
|---|---|
| `models/`, `controllers/`, `wizard/` `*.py` | `odoo-python` |
| `__manifest__.py` | `odoo-module-development` |
| `tests/`, `test_*.py` | `odoo-test-runner` |
| `migrations/**` `*.py` | `odoo-migrations` |
| `views/`, `data/`, `report/` `*.xml` & `security/*.csv` | `odoo-xml-conventions` |
| `static/src/**` (js, xml, scss) | `odoo-js` |

## Reading a diff

### Two passes — don't mix them

**Pass 1 — file by file, line by line.** Walk each changed file on its own, line by line, never skimming for shape. Nitpick as the senior reviewer would — the dead `or`, the redundant `string=`, the `[0]` on an invariant, the per-row op tucked in a helper — and anchor every finding to its `file:line`. The shapes to hunt are in *What only this read catches*; a thematic "overall this is solid" summary is the failure mode, because it's how you skim past every one of them.

**Pass 2 — the whole change, as a system.** Only once every line is read, trace one real user action end to end — view or route → access checks → DB. This catches what no single file shows: user input meeting a `sudo`, a `limit` that disagrees with a column, the same job done two ways. Most of the dangerous bugs live here — budget for it.

### The bug is usually the code that isn't there

Lint can only read what the diff contains; a reviewer reads what the diff *should* contain and doesn't. A new model with no `ir.model.access` line, a feature with no demo data, a new `required` field with no migration, a schema change with no `__manifest__.py` version bump — none of these surface as a red line, so you go looking for the gap. The migration absence test is the sharpest form: a diff that changes a field/model shape, doesn't bump the version, and doesn't add `migrations/<version>/` is the bug — flag it before approving (→ `odoo-migrations` for which shapes owe a script). When no plan ran, carry grill's two questions yourself: does this hold under multi-company, and what happens on the customer's *upgrade*? (→ `odoo-grill`).

### Name the shape, not the fix

"Change this line" fixes one PR; "this is a stored compute that depends on runtime context" teaches the pattern, so the next PR doesn't reintroduce it. A review only compounds when the finding generalizes beyond the instance — so state the category, point at the owning skill, and let the author internalize the rule rather than just the patch.

## What only this read catches

### Style the linter is blind to

Three shapes recur and none trip a linter. **Repeated `vals` dicts built inline** should be a `_prepare_<x>_vals` method — a customer submodule extends behavior by overriding a method, never by patching a literal buried in a loop, so the missing seam is a real defect, not taste. **Dead code is a blocker, not a nitpick** — an unused file, a one-line wrapper that only calls `super`, an `ensure_one()` that guards nothing, an `a or b` whose `b` can never be reached, an `if x and y` where one half is always true, a comment that just restates the line it sits on, a field assigned identically in every branch *and* again after them: each is a maintenance tax and a lie about intent, and the customer inherits both. **Every error and log message names the record and the problem** — `raise UserError("Error")` or a bare `"Invalid"` is unactionable in a customer's log at 2am; the message has to say which record and what's wrong with it.

### Shapes that should trigger a deeper read

When one of these appears, that's the signal to pull the owning skill and read harder — don't wave it through:

- `.py`: a loop that writes or queries — *including one tucked inside a private helper, which doesn't pay the debt* — `search(...)[0]` or `filtered(...)[0]` leaning on an invariant declared elsewhere (prefer `sum(mapped(...))`), a stored compute reading `context`/time/`env.user`, a `.sudo()` with no preceding `has_group`, a `has_group` check doing work `ir.rule` should own, a self-less method doing external I/O on a model → `odoo-python`.
- `static/src`: a reassignment that should be `patch()`, the DOM reached imperatively, an un-`_t()`'d user-facing string → `odoo-js`.
- `tests/`: a test run on a polluted DB (false confidence), or named for a method instead of a behavior → `odoo-test-runner`.

### The customer-readiness bar

Done is not "it works" — it's "I'd hand this to the customer with no cover note." That means clear field labels, translated strings, demo data that actually demonstrates the feature, and a clean `-i` install on a fresh DB: if it can't install on your throwaway DB, it can't install on theirs. (Full wrap-up sequence: → `odoo-task-completion`.)
