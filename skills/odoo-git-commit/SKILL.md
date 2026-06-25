---
name: odoo-git-commit
description: Use when committing Odoo work — before `git commit`, when a staged diff spans multiple modules or mixes a move with edits, or when writing a commit message or PR body. Covers splitting into one-logical-change commits, the `[TAG] module: summary` convention, move-then-modify, and terse human messages over AI essays. Invoke before any commit on a customer module.
---

# Odoo Git Commit

A commit is a unit of review and revert, not a save point — someone bisecting a customer bug six months from now reads it cold. So it carries exactly one idea, tagged the way the whole Odoo codebase tags it. Stances, not a checklist.

## The stances

### One commit, one logical change — split before you commit

The reflex this skill exists to enforce. A commit that bundles a feature, a refactor, and a whitespace sweep can't be reviewed (the reviewer can't separate intent from noise), can't be reverted (backing out the bug backs out the feature), and can't be cherry-picked into another branch. Before `git commit`, read the staged diff and ask: *is this one idea?* Touches two modules → two commits. A behavior change plus a reformat → two commits. Two unrelated bugfixes → two commits. Split by module first — each commit touches one module unless they're genuinely coupled — then by concern. Stage the paths for one change at a time (`git add <path>`), never `git add -A` over a working tree that grew three changes at once.

> A working tree with a new `sale_order` discount field, a typo fix in `account_move`, and a reindented `product_template` is **three** commits: `[ADD] sale_order: …`, `[FIX] account_move: …`, `[REF] product_template: reindent`. One `git add -A` buries all three in a diff no one can review or revert cleanly.

### Move and modify are two commits, never one

Rename or move a file *and* change its contents in the same commit and git records delete-plus-add — the diff shows a brand-new file instead of the three lines you actually touched, and the change becomes unreviewable. Do the pure move in its own `[MOV]` commit first (git then detects the rename at ~100% similarity), and put the content change on top. The same logic governs a large reformat: reformat in one commit, change behavior in the next, so the behavior diff stays small.

### `[TAG] module: summary` — name the change the way Odoo does

Every Odoo commit header has the same shape: `[TAG] module: summary`. `module` is the technical name (the directory), and the summary is lowercase, imperative, no trailing period, short enough to scan in a `git log --oneline`. The tag is the *kind* of change:

| Tag | When |
|---|---|
| `[FIX]` | bug fix |
| `[IMP]` | incremental improvement to existing behavior |
| `[ADD]` | new module or feature |
| `[REF]` | refactor — behavior unchanged |
| `[REM]` | remove code or a resource |
| `[MOV]` | move/rename only, no content change |
| `[REV]` | revert |
| `[UPG]` | migration / upgrade script (→ odoo-migrations) |
| `[I18N]` | translations |

The tag *is* the split test made visible: if you can't pick a single tag, the commit is doing more than one thing — go back and split it. `[REF]`, `[MOV]`, and `[IMP]` only stay meaningful when each lives in its own commit.

### The body explains why — terse and human, never an AI essay

The diff already shows *what* changed; the message exists for the *why* — the customer constraint, the symptom in the ticket, why this approach over the obvious one. Keep it terse and human: a senior's standard is "no point making an AI write an essay as a commit message." When an already-split commit still groups a few genuinely related edits, a one-line summary then `This commit includes the following:` with one bullet each is plenty — but if you're writing paragraphs, the commit is too big or the prose is padding. Every commit references its `project.task` id; that's the trace from the change back to why it exists.

### Commit freely; never push — that's mine to do

Staging and committing are local and reversible — do them as needed. `git push` is a different act: outward-facing and not yours to run. **Don't `git push`; I do that myself.** Stop at the commit, then hand me the exact push command to run — spell out the branch and remote (e.g. `git push origin my-branch`) so I can paste it straight in with the `!` prefix. Don't fold a push into a "commit this" request, and don't run it even when it looks like the obvious next step. (Machine-enforced too: the push hook will block it — CLAUDE.md hard rule.)

---

This skill owns commit shape and message format; → odoo-task-completion orchestrates the wrap-up that ends in a commit, → odoo-migrations for the `[UPG]` commit that ships a schema change.
