---
name: odoo-task-completion
description: Use when wrapping up a project.task ticket — after writing the code, before claiming done. Orchestrates the code-review pass, the test-runner pass, and a customer-readiness check. Invoke instead of saying "I think it's done."
---

# Odoo Task Completion

Done means proven. Tests green, style clean, and you'd be comfortable handing the PR to the customer without a cover note.

## Principles

### What "done" means

- **Done is a higher bar than "it works."** It works on your machine; it has to work on the customer's. The difference between the two is exactly the gap this skill closes.
- **Three gates, in order: review, test, customer-read.** Skipping any of them creates a class of bug the next gate can't catch.
- **The ticket comment is part of the deliverable.** A `project.task` closed with "fixed" is not closed. It needs to name what changed and why the customer should care.

### The three gates

#### Gate 1: Code review (invoke `odoo-code-review`)

Read your own diff as if a stranger wrote it. Specifically check:
- Are there `sudo()` calls without comments?
- Are there stored computes with runtime dependencies?
- Is there a security file for every new model?
- Are user-facing strings wrapped in `_()`?

If any answer is "no" or "not sure," fix it before Gate 2.

#### Gate 2: Verification (invoke `odoo-test-runner`)

Prove the feature works on a fresh DB. Not on the dev DB you've been using all week. Not "I'll verify it when I push." Now.

Pick the layer that matches what you built:
- **Logic / computes / security / migrations** → unit tests (`--test-enable`)
- **Data shape / domains / new fields** → shell verification (`odoo-bin shell`)
- **Views / wizards / flows** → UI walkthrough

Most non-trivial tickets need at least two layers. Don't pretend one is enough — `--test-enable` cannot see what the user sees.

If verification fails, you're not in Gate 2 anymore — you're back in implementation.

#### Gate 3: Customer-readiness check

Ask yourself, out loud or in writing:
- Would the customer understand what changed if they read the field labels?
- If they install on a fresh DB, does demo data show them the feature works?
- If they upgrade an existing DB with this module, will the migration run cleanly?
- Is the ticket comment something a non-developer at the customer can act on?

Any "no" means more work, not a smaller commit.

### When all three gates pass

Then say "done" — and link the ticket, the branch, and the verification evidence (test output, shell session, or screenshot). "Done" without that evidence is a claim, not a fact.

## What good looks like

> *Task #58231 closed.*
>
> - Branch: `19.0-58231-link-helpdesk-to-task-odoo-gram`
> - Code review (self): clean, see notes in PR
> - Tests: `odoo-bin db init → -i project_task_helpdesk_link --test-enable --stop-after-init → db drop` → 14/14 green
> - Customer-readiness: demo data present, migration tested against staging snapshot, field labels reviewed
> - Ticket comment: written in customer-readable language, attached screenshot

## What bad looks like

- "Tests pass" with no link to the test run output.
- A diff with 12 files, no security update, and a 3-word commit message.
- A ticket closed with "fixed in commit abc123" and nothing else.
- Running through the gates in the wrong order — tests before review means you're testing un-reviewed code.

## If a gate keeps failing

If you're stuck on the same gate for a third pass, that's a signal — not a personal failing. Note it in `skills/_journal.md` ("Gate 2 keeps failing because I forget to install dependencies in the tmp DB"). The `refining-odoo-skills` skill will eventually turn that into either a sharper principle in one of the skills, or a script change.
