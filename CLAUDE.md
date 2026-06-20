# Odoo Superpowers — Values & Conventions

Auto-loaded by Claude Code, Codex, and Gemini CLI. Describes *how we work on Odoo customer projects*, not what to do step-by-step — the skills under `skills/` carry the operational guidance.

## Who this is for

Omar Assouma — Odoo SA, PSDU (Professional Services Development Unit). Customer-facing work via `project.task` tickets on `www.odoo.com` (db: `openerp`). Internal login `omas@odoo.com`. Email task-IDs match `res_id=(\d+)`.

## Hard rules (machine-enforced)

- **`git push` is gated.** You may not push without my explicit approval in the current turn.
- **Python edits run `ruff format`** via PostToolUse hook. Don't skip it. If formatting fails, that's a signal to look at the file.

## Core values

- **We ship to customers.** Every change is read by someone who didn't write it, often under deadline pressure. Code should be obvious before it's clever.
- **Every change traces to a ticket.** Commits, branches, and PRs reference the `project.task` id. If you can't name the ticket, you don't have a justification.
- **Odoo idioms before clever Python.** The ORM, the view system, and the security model are the API. Reaching around them is almost always a mistake.
- **Throwaway DBs are the only credible test environment.** Local pollution hides real bugs and discovers fake ones.
- **Customer-readiness is a higher bar than "it works."** Migrations, security, demo data, and translations all count.

## Skills

Always invoke the relevant skill before acting, even if you "know" the topic — skills evolve from real feedback and what you remember may be stale. The entry point `using-odoo-superpowers` routes to the rest.

## When instructions conflict

User instructions (this file, direct messages, project-level CLAUDE.md) win over skills; skills win over default behavior. If a customer's repo CLAUDE.md says "don't use the test runner," follow the customer.

## The learning loop

Corrections are data — fold them back via `refining-odoo-skills`. Surface to me if `skills/_journal.md` hits 5+ unprocessed entries.

## What this plugin is not

- Not a substitute for the Odoo developer documentation. Look things up.
- Not for greenfield Odoo OCA-style community modules. PSDU work is customer-specific by nature.
- Not a license to skip code review. The `odoo-code-review` skill assists; it doesn't replace human review for anything customer-facing.
