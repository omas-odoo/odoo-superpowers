---
description: Review a GitHub PR against Odoo PSDU conventions — multi-agent, terminal-only, read-only
argument-hint: <pr-number | pr-url>
---

# Review a GitHub PR (Odoo PSDU)

You are reviewing the pull request identified by: **$ARGUMENTS**

This review is **read-only**. Print findings in the terminal. Do **not** post comments,
push, or write anything to GitHub.

## 1. Guard

If `$ARGUMENTS` is empty, print this and stop:

```
Usage: /odoo-review <pr-number | pr-url>
Example: /odoo-review 42
         /odoo-review https://github.com/owner/repo/pull/42
```

## 2. Fetch the PR (read-only)

`gh pr diff` and `gh pr view` accept a number, a full URL, or a branch, and resolve the
repo from the local remote. Run:

- `gh pr view "$ARGUMENTS" --json number,title,body,author,headRefName,baseRefName,files`
- `gh pr diff "$ARGUMENTS"`

If `gh` errors (not authenticated, PR not found, no repo), surface the **real error** and
stop. Never fabricate a review for a PR you couldn't fetch. If the diff is empty, say so
and stop.

## 3. Fan out — one subagent per review dimension

Dispatch the following review agents **in parallel** (a single message with multiple Task
tool calls). Give each agent the PR title, the changed-file list, and the full diff. Each
agent MUST invoke the `odoo-code-review` skill first, then review **only its assigned
dimension**. Each returns a list of findings; an empty list is a valid answer.

Each finding must be: `severity` (blocker / warning / nit), `shape` (the named pattern,
e.g. "stored compute depending on runtime context"), `file:line`, `why` (one line), `fix`
(one line).

1. **ORM & shape** — stored computes depending on context/time/`env.user`; `search(...)[0]`
   vs `limit=1`; per-record writes that should be batched; reflexive `ensure_one()`; code
   that doesn't read like Odoo wrote it.
2. **Security** — `sudo()` without a justifying comment; `has_group(...)` checks in business
   logic that belong in `ir.rule`/`ir.model.access`; the access CSV (public-group reads);
   self-less model methods doing external I/O (RPC-attackable); secret fields without
   `groups='base.group_system'`.
3. **Migrations — the absence test** — scan the diff for: new `required=True` field, field
   type change, renamed field/model, removed field/model, new `_sql_constraints`, changed
   `ondelete`, edits to `noupdate="1"` records. ANY of these **without** a matching
   `migrations/<version>/` script **and** a `__manifest__.py` version bump is a blocker.
4. **Customer-readiness** — untranslated user-facing strings (missing `_()`); unclear field
   labels; missing/empty demo data for the new feature; anything that would break a clean
   `-i` install on a fresh DB.
5. **Tests** — tests named after implementation not behavior; tests that assume a polluted
   DB; new model/flow with no test at all.

## 4. Synthesize — one report, in the terminal

Collect all findings, drop duplicates (same file:line + shape), and print a single report:

```
# Review: <PR title> (#<number>)
<author> · <headRefName> → <baseRefName> · <N> files changed

## Blockers
- **<shape>** — `file:line`
  <why>. Fix: <fix>.

## Warnings
- ...

## Nits
- ...

## Missing / absence
- <e.g. "new required field `x` on `model` but no migrations/<ver>/ script">
```

If a section is empty, omit it. End with a one-line verdict: whether you'd hand this PR to
the customer as-is, and the single most important thing to fix first. Name *shapes*, not
just fixes — so the author learns the pattern, per the `odoo-code-review` skill.
