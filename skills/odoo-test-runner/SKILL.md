---
name: odoo-test-runner
description: Use when verifying an Odoo development before claiming "it works," or when writing the tests themselves. Spins up a fresh DB, installs the module, and proves the feature behaves as the ticket asked via `--test-enable`, the shell, or the UI — and carries the conventions for the tests: class choice, `@tagged`, `assertRecordValues`, the demo-user password trap, tours. Invoke whenever tempted to say "it works" without proving it on a clean install, or before adding files under tests/.
---

# Odoo Test Runner

Testing is verification. The goal isn't to run `--test-enable` and check a box — it's to prove the development does what the ticket asked, on a DB that looks like a customer's. Two halves: **run** what you wrote on a clean install, and **write** tests an experienced reviewer wouldn't rework.

## Running — prove it on a fresh DB

### Testing means proving the feature works

- The loop is: **build → install on a fresh DB → verify the behavior → drop the DB.**
- "Works on my machine" doesn't count unless your machine is a fresh DB with the module installed cleanly. The dev DB you've been using all week has state the customer's DB won't.
- The acceptance criteria come from the ticket. If you can't restate them in one sentence before you start verifying, you don't yet know what you're proving.

### A fresh DB every time, dropped after

- **Pollution looks like correctness.** A leftover record from yesterday's test makes today's search return `1` instead of `0`, and the verification passes for the wrong reason.
- **Customer DBs are fresh relative to your module.** The closest local equivalent is a throwaway DB with only your dependencies installed — name it `tmp_test_<module>_*` so orphan cleanups stay trivial.
- **Drop after, always** — even if the run failed. Keeping the DB "for inspection" creates the next pollution. Chain the drop with `;` (not `&&`) so it runs regardless; for shell/UI runs the DB has to outlive the install, so drop it by hand.

### Three layers — match to the risk

Unit tests (`--test-enable`) for logic that breaks unnoticed — computes, constraints, security, migrations, anything with branches. Shell (`odoo-bin shell`) for ORM data-shape questions — did the field compute right, does this domain return those records — faster than clicking. UI (`:8069`) for what the customer actually clicks — views, wizards, reports, screenshots. Not exclusive: a new field with a compute and a form view wants all three; a pure migration wants only the unit test.

### `odoo-bin` owns the whole loop — never `createdb`/`dropdb`

`odoo-bin db init` / `db drop` handle the **filestore** alongside the DB; raw `createdb`/`dropdb` leave orphan filestore dirs that pollute the next run. One binary creates, installs, verifies, and drops — no shell wrappers, no bespoke scripts. Connection settings (`db_user`, `addons_path`, …) live in `~/.odoorc`, so a command passes only what differs per-run. Exact commands, flags, the `--test-tags` filter syntax, and the one-shot invocation → `references/odoo-bin-commands.md`.

### When verification fails

- **Read the first failure, not the last.** Test failures cascade; the first trace is the cause, the rest are noise.
- **Don't loosen the test.** Matching the assertion to the buggy output is worse than the original bug.
- **Re-run on a fresh DB after fixing** — your fix might lean on dev-DB state the customer won't have.
- **A surprise from shell/UI verification earns a unit test before the fix** — that's how a regression becomes impossible.
- "Verified" means every claim in the ticket has a concrete check behind it — a test, a shell output, or a screenshot — and the suite passes on a fresh DB.

## Writing tests

### Pick the lightest class that does the job

- **`TransactionCase` is the default** — one transaction per test, rolled back; `setUpClass` fixtures persist across the class and roll back at the end. Step up to **`HttpCase`** only when you need the HTTP layer — controllers, sessions, tours.
- **`SavepointCase` is gone** — `TransactionCase` absorbed it. Don't reach for it in v16+; the shared-fixture behaviour is already built in.

### Tag every test, and tag it `post_install`

- **`@tagged("post_install", "-at_install")`** on custom tests — run once, after all modules install, not twice. A `-at_install` with no `post_install` never runs at all.
- Add a module tag (e.g. `"psae_my_module"`) so `--test-tags` can target just your suite.

### `setUpClass` for fixtures, `setUp` for mutable state

Expensive shared fixtures go in `setUpClass` (built once); per-test state a test might mutate goes in `setUp` (rebuilt each test). Always `super()` first.

### Assert record values, and the *specific* exception

- **`assertRecordValues` over chained `assertEqual`** — one call asserts every field and gives a readable diff on failure. Field-by-field is fine for one or two.
- **Assert on stable record values, not UI labels** — labels are translatable and drift.
- **`assertRaises(UserError)` / `assertRaisesRegex`, never `assertRaises(Exception)`** — the broad catch accepts too much and masks the next bug.

### Test discovery is manual — `tests/__init__.py`

`tests/` needs its own `__init__.py` importing every test file, one per line, alphabetically — without it the runner finds nothing and the suite silently passes empty. The module-root `__init__.py` never imports `tests`; Odoo discovers them itself.

### Tours drive the UI from an `HttpCase`

A tour is three coupled pieces: a Python `HttpCase.start_tour(...)` entry, a JS tour registered in `web_tour.tours`, and the JS file declared in the **`web.assets_tests`** bundle (test-only, not production). A tour always needs `HttpCase` — never `TransactionCase`. The JS side follows `odoo-js` conventions.

### The demo-user password trap

Demo users' **login is not their password.** `start_tour(..., login="admin")` authenticates with the password *equal to the login*, but on a fresh/dev DB `admin`/`demo`/`portal` often have an empty password — so the tour silently fails to authenticate. Set the passwords before any test run that logs in as them.

Exact `@tagged` syntax, the `setUpClass`/`assertRecordValues`/`assertRaises` snippets, the full Python+JS+manifest tour example, the password SQL, and the pitfall list → `references/writing-tests.md`.

## References (consult, don't memorize)

| Need...                                                                                                                                                  | Read                              |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `odoo-bin` commands and flags — `db init`/`drop`/`duplicate`, `-i`/`-u`, `--test-enable`, `--test-tags` syntax, `shell`, the UI server, `--dev`, the one-shot loop | `references/odoo-bin-commands.md` |
| Writing the tests themselves — class choice, `@tagged`, `setUp`/`setUpClass`, `assertRecordValues`, `assertRaises`, `tests/__init__.py`, tours, demo passwords      | `references/writing-tests.md`     |

For the JS half of a tour (registering the tour, step `trigger`/`run` shapes), see `odoo-js`. For wrapping up the whole ticket — review + test + customer-readiness — see `odoo-task-completion`; for the review pass itself, `odoo-code-review`.
