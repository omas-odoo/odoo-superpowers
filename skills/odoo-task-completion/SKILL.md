---
name: odoo-task-completion
description: Use when wrapping up a project.task ticket — after writing the code, before claiming done. Orchestrates the code-review pass, the test-runner pass, and a customer-readiness check. Invoke instead of saying "I think it's done."
---

# Odoo Task Completion

"Done" means proven and customer-ready, not "it runs on my machine." This is an orchestrator: it routes through the review and test gates rather than re-teaching them, then asks the one question only you can answer — would you hand this to the customer without a cover note? Stances, not a checklist.

## The gates

### Done is customer-ready, not "it runs"

It works on your machine; it has to work on the customer's — on a fresh install, after an upgrade, in their language. That gap *is* the job: demo data that demonstrates the feature, a security record for every new model, user-facing strings translated, and a migration travelling with every changed field (→ odoo-migrations). A green test that skips any of these is a higher-confidence wrong answer. (If no plan was pressure-tested up front, this is the last place to catch what grill would have — multi-company, the missing standard setting, the upgrade path; → odoo-grill.)

### Review and test are gates, not formalities

Two passes you owe before the word "done": read your own diff as if a stranger wrote it (→ odoo-code-review), and prove the behavior on a clean DB — never the dev DB you've polluted all week (→ odoo-test-runner). Order matters: a flaw caught at review never reaches the test run. Don't restate their checks here, invoke them. Most non-trivial tickets need more than one proof layer because `--test-enable` cannot see what the user sees, so pick the layers in odoo-test-runner. If verification fails you're not "almost done" — you're back in implementation.

### Every change traces to its task id, and "done" carries its evidence

A `project.task` closed with "fixed" is not closed: the comment is part of the deliverable, and a non-developer at the customer has to be able to act on it — and the commit message and PR body answer to the same bar, terse and human, not an AI essay no one reads — a one-line summary, then `This commit includes the following:` and one bullet per change. Saying "done" means linking the ticket, the branch, and the proof — test output, shell session, or screenshot. Without that, "done" is a claim, not a fact.

### A gate that keeps failing is data, not a personal failing

Stuck on the same gate for a third pass means it found something a skill should have warned you about earlier. Note the concrete case in `skills/_journal.md`; → refining-odoo-skills folds it into a sharper principle or a script fix.

This skill orchestrates → odoo-code-review then → odoo-test-runner; the customer-readiness stance reaches → odoo-migrations and → odoo-grill.
