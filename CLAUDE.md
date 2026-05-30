# Odoo Superpowers — Values & Conventions

This document is auto-loaded by Claude Code, Codex, and Gemini CLI (via symlink / GEMINI.md). It describes *how we work on Odoo customer projects*, not what to do step-by-step. The skills under `skills/` carry the operational guidance.

## Who this is for

Omar Assouma — Odoo SA, PSDU (Professional Services Development Unit). Customer-facing work via `project.task` tickets on `www.odoo.com` (db: `openerp`). Internal login `omas@odoo.com`. Email task-IDs match `res_id=(\d+)`.

## Core values

- **We ship to customers.** Every change is read by someone who didn't write it, often under deadline pressure. Code should be obvious before it's clever.
- **Every change traces to a ticket.** Commits, branches, and PRs reference the `project.task` id. If you can't name the ticket, you don't have a justification.
- **Odoo idioms before clever Python.** The ORM, the view system, and the security model are the API. Reaching around them is almost always a mistake.
- **Throwaway DBs are the only credible test environment.** Local pollution hides real bugs and discovers fake ones.
- **Customer-readiness is a higher bar than "it works."** Migrations, security, demo data, and translations all count.

## Skills

Skills live under `skills/`. Each one has a `SKILL.md` written as principles, not checklists. The entry point — `using-odoo-superpowers` — explains when to reach for each.

**Always invoke the relevant skill before acting**, even if you "know" the topic. Skills evolve from real feedback; what you remember from last week may already be stale.

## Hooks (machine-enforced rules)

- **`git push` is gated.** You may not push without my explicit approval in the current turn. This is a rule, not a principle — rules belong here.
- **Python edits run `ruff format` automatically** via PostToolUse hook. Don't skip it. If formatting fails, that's a signal to look at the file.

## When skills and instructions conflict

User instructions (this file, direct messages, project-level CLAUDE.md) win over skills. Skills win over default behavior. If a customer's repo CLAUDE.md says "don't use the test runner," follow the customer.

## The learning loop

Every correction is data. When I correct you on Odoo specifics, or you catch yourself doing something un-Odoo-like, log it in `skills/_journal.md` with date + concrete example + why-it-matters. Periodically the `refining-odoo-skills` skill batches the journal into actual skill updates.

Do not let the journal become a graveyard. If it has 5+ unprocessed entries, surface that to me.

## What this plugin is not

- Not a substitute for the Odoo developer documentation. Look things up.
- Not for greenfield Odoo OCA-style community modules. PSDU work is customer-specific by nature.
- Not a license to skip code review. The `odoo-code-review` skill assists; it doesn't replace human review for anything customer-facing.
