---
name: odoo-test-runner
description: Use when verifying an Odoo development before claiming "it works," or when writing the tests themselves. Spins up a fresh DB, installs the module, and proves the feature behaves as the ticket asked via `--test-enable`, the shell, or the UI — and carries the conventions for the tests: class choice, `@tagged`, `assertRecordValues`, the demo-user password trap, tours. Invoke whenever tempted to say "it works" without proving it on a clean install, or before adding files under tests/.
---

# Odoo Test Runner

A green test on the dev DB you've run all week is a lie — it passed against records, half-applied columns, and hand-fixed data the customer's DB won't have. A throwaway DB, installed clean and dropped after, is the only signal you can believe that the development does what the ticket asked. These are stances to carry, not a checklist to run: the exact `odoo-bin` invocations and test-class patterns live in `references/`. When a task skips planning (`odoo-grill`), these reflexes carry verification alone.

## Running — make the run credible

### Fresh DB, or don't trust the green

Pollution looks like correctness: a leftover record from yesterday makes today's search return `1` instead of `0`, and the check passes for the wrong reason. A pass on your long-lived dev DB therefore proves nothing about a clean install. The only run you can believe is a throwaway DB carrying just your module's dependencies, built fresh and dropped after — *even when the run failed*, because the DB you keep "for inspection" is the next run's pollution. After a fix, re-run on a fresh DB: the fix itself may lean on dev-DB state the customer won't have. One binary owns the whole loop (`odoo-bin db init`/`drop`) because dropping the DB takes its filestore with it — raw `dropdb` orphans the filestore and re-pollutes. (Invocation, flags, `~/.odoorc` → `references/odoo-bin-commands.md`.)

### Prove the ticket, not the checkbox

`--test-enable` going green proves the code runs without crashing — not that it does what was asked. Before verifying, restate the acceptance criteria in one sentence; if you can't, you don't yet know what you're proving. "Verified" means every claim in the ticket has a concrete artifact behind it — a passing test, a shell output, or a screenshot — and the suite is green on a fresh DB. Anything less is "it didn't crash."

### The new behavior earns a new test

A suite that still passes after your change proves you didn't break the *old* behavior — not that you delivered the new one. The bug you fixed earns a test that fails before the fix and passes after; the feature you built earns a test asserting its new contract. Leaning on an existing test that happens to still pass is how a feature ships untested. A surprise found while poking in the shell or UI earns a unit test *before* you fix it — that's the move that turns a one-off into a regression that can never return.

### Match the layer to the risk

Unit (`--test-enable`) for logic that breaks unnoticed — computes, constraints, security, migrations, anything with branches. Shell (`odoo-bin shell`) for data-shape questions — did the field compute, does this domain return those records — faster than clicking. UI (`:8069`) for what the customer actually clicks — views, wizards, reports. Not exclusive: a new computed field on a form wants all three; a pure migration wants only the unit test.

### Isolate the module under test

Install only your module and its dependencies, never `-i all` — a failure then traces to your code, not a neighbor's. Name the DB `tmp_test_<module>_*` so orphan cleanups stay trivial, and tag your suite so `--test-tags` runs just it instead of letting a hundred unrelated tests drown your signal.

### When it fails, fix the code — not the test

Failures cascade, so read the *first* trace; the rest are noise. Never relax the assertion to match the buggy output — a test bent to accept the bug is worse than the bug, because it now hides every future one too. (For the debugging discipline itself, lean on `systematic-debugging`.)

## Writing — tests that run and mean something

### Pick the lightest class — and let it roll back

`TransactionCase` is the default: one transaction per test, rolled back, so every test starts from the same clean state — that roll-back is the fresh-DB discipline enforced *inside* the suite, and a stray `commit()` breaks it. Step up to `HttpCase` only for the HTTP layer — controllers, sessions, tours. `SavepointCase` is gone (merged into `TransactionCase`, v16+) — don't import it.

### A test that never runs is a green lie

Several silent no-ops pass while testing nothing: `@tagged("-at_install")` with no `post_install` never executes; a new test file not imported in `tests/__init__.py` leaves the runner discovering nothing and "passing" empty; and a tour logging in as `admin`/`demo`/`portal` silently fails to authenticate, because a demo user's login is *not* their password and a fresh DB leaves it empty — set the passwords first. Each looks green; none proves anything. (Class table, `@tagged`, the `tests/__init__.py` rule, the full Python+JS+manifest tour wiring, and the password SQL → `references/writing-tests.md`.)

### Assert what won't drift, and the *specific* failure

Assert stable record values (`assertRecordValues` for many fields at once), never UI labels — labels are translatable and move under you. Assert the *specific* exception (`assertRaises(UserError)`), never `Exception` — the broad catch passes on the next, unrelated bug too.

## References (consult, don't memorize)

| Need the exact... | Read |
|---|---|
| `odoo-bin` command or flag — `db init`/`drop`/`duplicate`, `-i`/`-u`, `--test-enable`, `--test-tags` syntax, `shell`, the UI server, `--dev`, the one-shot loop | `references/odoo-bin-commands.md` |
| test-writing pattern — class table, `@tagged`, `setUp`/`setUpClass`, `assertRecordValues`, `assertRaises`, `tests/__init__.py`, tours, demo passwords | `references/writing-tests.md` |

For the JS half of a tour (registering the tour, step `trigger`/`run` shapes), see `odoo-js`. For wrapping up the whole ticket — review + test + customer-readiness — see `odoo-task-completion`; for the review pass itself, `odoo-code-review`.
