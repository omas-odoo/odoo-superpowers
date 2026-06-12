---
description: Review a GitHub PR against Odoo PSDU conventions — business need, file-by-file, then full-logic; before/after fixes; terminal-only, read-only
argument-hint: <pr-number | pr-url>
---

# Review a GitHub PR (Odoo PSDU)

You are reviewing the pull request identified by: **$ARGUMENTS**

This review is **read-only**. Print the report in the terminal. Do **not** post comments,
push, or write anything to GitHub.

## 1. Guard

If `$ARGUMENTS` is empty, print this and stop:

```
Usage: /odoo-review <pr-number | pr-url>
Example: /odoo-review 42
         /odoo-review https://github.com/owner/repo/pull/42
```

## 2. Fetch the PR (read-only)

`gh pr view` / `gh pr diff` accept a number, URL, or branch and resolve the repo from the
local remote. Run:

- `gh pr view "$ARGUMENTS" --json number,title,body,author,headRefName,baseRefName,files`
- `gh pr diff "$ARGUMENTS"`

If `gh` errors (not authenticated, PR not found, no repo), surface the **real error** and
stop — never fabricate a review. If the diff is empty, say so and stop.

**Scale check:** if the PR is large (many files), review **file by file** so no single
context holds the whole diff — fetch a file's slice with `gh pr diff "$ARGUMENTS" -- <path>`
when needed. If you ever truncate or skip a file, say so explicitly in the report; never let a partial review read as complete.

## 3. Establish the business need (do this first)

Before any code critique, work out **what this PR is for**, in plain language. Pull from the
PR title/body, the linked `project.task` (the ticket id / `task_ids`), and what the code
actually does. Write 2–4 simple sentences a non-developer could follow: the customer problem,
what a user can now do, and the shape of the solution. This frames every finding that follows.

## 4. Phase A — review file by file (the skill's Pass 1)

First, **map each changed file to its domain skill** using the **Route by file type** table in
the `odoo-code-review` skill (e.g. `models/sale_order.py` → `odoo-python`, `static/src/**` →
`odoo-js`). That table is the single source of truth — read it, don't restate it here.

Then, for **each changed file**, dispatch a review subagent (run them in parallel — a single
message with multiple Task tool calls; for a large PR, batch them). Give each agent that file's
diff and, **naming the skills explicitly in its prompt**, tell it to:

1. invoke the `odoo-code-review` skill **and** the domain skill you mapped for this file (e.g.
   `models/sale_order.py` → `odoo-python`), then
2. perform the skill's **Pass 1** on **only that file** — read it for its own shape against the
   skill's principles (ORM, security, migrations, customer-readiness, tests).

The skill owns *what* to look for — don't restate its checklist here; the agents get it by
invoking the skills you named. Each agent returns its findings in the **before/after** shape from §6.

## 5. Phase B — full-logic review (the skill's Pass 2)

After every file agent returns, run the skill's **Pass 2** on the change as one system.
Pass 2 needs to see **all files at once**, so it can't be a per-file agent — run it either in
the main session or via a single subagent given the **whole** diff. It invokes
`odoo-code-review` and traces the primary user action end to end across files (e.g. *portal
user → upload → validate → parse → product lookup → create order*), surfacing what file-by-file
can't: user input meeting `sudo`, limits/columns that disagree between files, duplicated logic,
and behavior that doesn't match the business need from §3.

Report these in the same before/after shape where a concrete code change applies; otherwise
state the issue, the risk, and the recommended direction.

## 6. Output — one report, in the terminal

Use this exact structure. Keep every **Why** to one or two plain sentences — explain the
*shape* and the impact simply, not in jargon.

```
# Review: <PR title> (#<number>)
<author> · <headRefName> → <baseRefName> · <N> files changed

## Business need
<2–4 plain sentences from §3>

---

## File-by-file

### `path/to/file.py`
**<short title / shape>** — `file:line` · <🔴 blocker | 🟠 warning | 🟡 nit>

Before
```python
<the current code>
```
After
```python
<the corrected code>
```
Why: <one or two simple sentences — what's wrong and what the fix buys>

<repeat per finding; if a file is clean, write "✅ No issues.">

---

## Full-logic review
<cross-file / end-to-end findings from §5, same Before/After + Why shape where code applies>

---

## Verdict
<Would you hand this to the customer as-is? The single most important thing to fix first.>
```

Rules for the before/after blocks:
- **Before** is the real code from the diff, unedited. **After** is the minimal corrected
  version — change only what the finding is about, keep surrounding style.
- One finding = one Before/After pair. Don't bundle unrelated changes.
- Omit empty severity groups. If the whole PR is clean, say so plainly rather than padding.
- Name the *shape* ("sudo search on user-controlled input") so the author learns the pattern,
  per the `odoo-code-review` skill — not just the line fix.
