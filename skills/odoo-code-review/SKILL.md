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

### Read it twice — each file alone, then the change as a system

Pass one takes each changed file on its own terms: does this model / view / template read like Odoo wrote it? You're judging well-formedness, not correctness. Pass two — only once every file is read — traces one real user action end to end, from the view or route through the access checks into the DB. The two passes find different bugs and must not be mixed: a file can be flawless Odoo and still be wrong in concert with another — user input meeting a `sudo`, a `limit` that disagrees with a column, the same job done two ways. The dangerous bugs almost all live in pass two; budget for it.

### The bug is usually the code that isn't there

Lint can only read what the diff contains; a reviewer reads what the diff *should* contain and doesn't. A new model with no `ir.model.access` line, a feature with no demo data, a new `required` field with no migration, a schema change with no `__manifest__.py` version bump — none of these surface as a red line, so you go looking for the gap. The migration absence test is the sharpest form: a diff that changes a field/model shape, doesn't bump the version, and doesn't add `migrations/<version>/` is the bug — flag it before approving (→ `odoo-migrations` for which shapes owe a script). When no plan ran, carry grill's two questions yourself: does this hold under multi-company, and what happens on the customer's *upgrade*? (→ `odoo-grill`).

### Name the shape, not the fix

"Change this line" fixes one PR; "this is a stored compute that depends on runtime context" teaches the pattern, so the next PR doesn't reintroduce it. A review only compounds when the finding generalizes beyond the instance — so state the category, point at the owning skill, and let the author internalize the rule rather than just the patch.

## What only this read catches

### Style the linter is blind to

Three shapes recur and none trip a linter. **Repeated `vals` dicts built inline** should be a `_prepare_<x>_vals` method — a customer submodule extends behavior by overriding a method, never by patching a literal buried in a loop, so the missing seam is a real defect, not taste. **Dead code is a blocker, not a nitpick** — an unused file, a one-line wrapper that only calls `super`, an `ensure_one()` that guards nothing: each is a maintenance tax and a lie about intent, and the customer inherits both. **Every error and log message names the record and the problem** — `raise UserError("Error")` or a bare `"Invalid"` is unactionable in a customer's log at 2am; the message has to say which record and what's wrong with it.

### Shapes that should trigger a deeper read

When one of these appears, that's the signal to pull the owning skill and read harder — don't wave it through:

- `.py`: a loop that writes or queries, `search(...)[0]`, a stored compute reading `context`/time/`env.user`, a `.sudo()` with no preceding `has_group`, a `has_group` check doing work `ir.rule` should own, a self-less method doing external I/O on a model → `odoo-python`.
- `static/src`: a reassignment that should be `patch()`, the DOM reached imperatively, an un-`_t()`'d user-facing string → `odoo-js`.
- `tests/`: a test run on a polluted DB (false confidence), or named for a method instead of a behavior → `odoo-test-runner`.

### The customer-readiness bar

Done is not "it works" — it's "I'd hand this to the customer with no cover note." That means clear field labels, translated strings, demo data that actually demonstrates the feature, and a clean `-i` install on a fresh DB: if it can't install on your throwaway DB, it can't install on theirs. (Full wrap-up sequence: → `odoo-task-completion`.)
