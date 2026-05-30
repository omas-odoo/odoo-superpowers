---
name: odoo-test-runner
description: Use after writing a development and before claiming "it works." Spins up a fresh DB, installs the module, and verifies the feature behaves as the ticket asked — via `--test-enable`, the shell, the UI, or all three depending on what's being verified. Invoke whenever tempted to say "it works" without proving it on a clean install.
---

# Odoo Test Runner

Testing is verification. The goal isn't to run `--test-enable` and check a box — it's to prove the development does what the ticket asked, on a DB that looks like a customer's.

## Principles

### Testing means proving the feature works

- The loop is: **build → install on a fresh DB → verify the behavior → drop the DB.**
- "Works on my machine" doesn't count unless your machine is a fresh DB with the module installed cleanly. The dev DB you've been using all week has state the customer's DB won't.
- The acceptance criteria come from the ticket. If you can't restate them in one sentence before you start verifying, you don't yet know what you're proving.

### Three layers — use whichever fits the feature

| Layer | What it's for | Required when... |
|---|---|---|
| **Unit tests** (`--test-enable`) | Fast, repeatable checks of logic that's easy to break unnoticed | Computes, constraints, security rules, migrations, anything with branches |
| **Shell verification** (`odoo-bin shell`) | ORM-level questions: did the field get the right value? Does this domain return the right records? Faster than the UI for data shape | New fields, search domains, defaults, mapped/filtered chains |
| **UI walkthrough** (the web client on `:8069`) | Buttons, flows, view changes, anything user-facing the customer will actually click | New views, wizards, reports, anything visible in a screenshot |

These are not mutually exclusive. A new field with a compute and a form view probably wants all three. A pure migration script probably only needs unit tests. Match the layer to the risk.

### Why a fresh DB every time

- **Pollution looks like correctness.** A leftover record from yesterday's test makes today's search return `1` instead of `0`, and the verification passes for the wrong reason.
- **Customer DBs are fresh relative to your module.** When the customer installs your code, it runs against *their* data, not yours. The closest local equivalent is a throwaway DB with only your dependencies installed.
- **Drop after, always.** Keeping the DB "for inspection" creates the next pollution. If you need to inspect, re-run with the same name once and own that fact.

### Use `odoo-bin`, never `createdb` / `dropdb` directly

- `odoo-bin db init` / `db drop` handle the **filestore** alongside the DB. Raw `createdb` / `dropdb` leave orphan filestore directories that pollute the next run.
- One binary owns the whole loop: create, install, verify, drop. No shell wrappers. No bespoke scripts. Run `odoo-bin` directly via the Bash tool.

### The canonical loop

For a module under your `--addons-path`:

```bash
# 1. Create the throwaway DB
odoo-bin db init tmp_test_<module>_$(date +%s) --force --without-demo

# 2a. Install (always) + run unit tests if present
odoo-bin -d <db> -i <module> --test-enable --stop-after-init --log-level=test

# 2b. (Optional) Verify via shell
odoo-bin -d <db> shell

# 2c. (Optional) Verify via UI — start the server, open http://localhost:8069
odoo-bin -d <db>

# 3. Drop (always — even if 2a/2b/2c failed)
odoo-bin db drop <db>
```

In one Bash call for the test-enable path:

```bash
DB=tmp_test_<module>_$(date +%s) && \
  odoo-bin db init "$DB" --force --without-demo && \
  odoo-bin -d "$DB" -i <module> --test-enable --stop-after-init --log-level=test ; \
  odoo-bin db drop "$DB"
```

`&&` between init and test — fail fast if init fails. `;` before drop — drop **always** runs.

For shell or UI verification, **don't** chain the drop in one shot — you need the DB to stick around between the install and the verification. Manage the drop manually:

```bash
DB=tmp_test_<module>_$(date +%s)
odoo-bin db init "$DB" --force --without-demo
odoo-bin -d "$DB" -i <module> --stop-after-init   # install, no tests yet
odoo-bin -d "$DB" shell                             # or: odoo-bin -d "$DB" for the UI
odoo-bin db drop "$DB"                              # when you're done
```

For the exact flag reference, consult `references/odoo-bin-commands.md`.

### Verifying via the shell

`odoo-bin shell` boots Odoo as a Python REPL with `env` pre-populated. The shell runs **inside a transaction** — changes roll back when you exit unless you `env.cr.commit()`.

```python
# Did the new field compute correctly?
>>> partner = env['res.partner'].browse(1)
>>> partner.psae_status
'active'

# Does this domain return what we expect?
>>> env['psae.task'].search([('priority', '=', 'high')])
psae.task(7, 12)

# Trigger an action and inspect the result
>>> task = env['psae.task'].browse(7)
>>> task.action_assign()
>>> task.assignee_id.name
'Customer Acme'
```

Use the shell when the question is "what does the data look like after install?" — much faster than clicking through the UI.

### Verifying via the UI

```bash
odoo-bin -d <db>   # leaves the server running on :8069
```

Log in as `admin / admin` (the defaults from `db init`). Walk through the feature exactly as the customer would. Take a screenshot if the ticket asked for one — most PSDU tickets do.

### Naming the throwaway DB

- Prefix with `tmp_test_` — orphan cleanups (`psql -l | grep tmp_test_`) become trivial.
- Include the module name so parallel runs from different modules don't collide.
- Suffix with a timestamp (`$(date +%s)`) for collision-avoidance within one module.

### Configuration belongs in `~/.odoorc`

Pass only what differs per-run on the command line. Connection settings (`db_user`, `db_host`, `db_port`, `addons_path`) live in `$HOME/.odoorc`:

```ini
[options]
db_user = odoo
db_host = localhost
db_port = 5432
addons_path = /Users/assouma/work/odoo/addons,/Users/assouma/work/psae
```

With that in place, the commands drop `--addons-path` and `--db_*` flags entirely. If `~/.odoorc` is wrong, fix it once — don't paper over it.

### When verification fails

- **Read the first failure, not the last.** Test failures cascade; the first trace is the cause, the rest are noise.
- **Don't loosen the test.** Matching the assertion to the buggy output is worse than the original bug.
- **Re-run on a fresh DB after fixing.** Your fix might depend on state that won't exist in the customer's environment.
- **If shell or UI verification surprised you, write a unit test for it before fixing the code.** That's how regressions become impossible.

### What counts as "verified"

- Every claim in the ticket has a concrete check behind it: a test, a shell output, or a screenshot.
- The unit tests that exist all pass on a fresh DB.
- The feature behaves the same on a fresh DB as it does on your dev DB. If those diverge, you have a hidden dependency on dev-DB state.

## What good looks like

> *Ticket #58231 — link helpdesk tickets to project tasks.*
>
> - Unit tests: `odoo-bin db init / -i project_task_helpdesk_link --test-enable / db drop` → 14/14 green
> - Shell check: `env['project.task'].browse(1).helpdesk_ticket_ids` returns 3 tickets, all owned by the right customer
> - UI walkthrough: opened a task as a portal user — the new notebook page shows tickets the user has access to, hides others. Screenshot in ticket.

## What bad looks like

- `pytest` against a long-lived dev DB. Odoo tests need the registry; `pytest` alone doesn't bootstrap it.
- "I clicked the button and it worked" with no clean-install proof. The customer's DB ≠ yours.
- Skipping the drop "just this once." That DB outlives the session and pollutes tomorrow's run.
- Running with `--with-demo` "for realism." Demo data is noise that masks real bugs. Install demo only when you specifically need to see the feature on demo records.
- Claiming verification on `--test-enable` alone when the ticket was about a view change. Tests can't see what the user sees.
