---
name: refining-odoo-skills
description: Use when the user corrects you on Odoo specifics, when you notice a skill missed a real case, or when skills/_journal.md has accumulated 5+ entries. Turns concrete corrections into sharper principles in the actual skill files. Invoke after sessions where the user pushed back on Odoo-specific advice.
---

# Refining Odoo Skills

Skills aren't static. When the user corrects you or you surprise yourself, that's data — feed it back so the next session is sharper. Without this loop, the plugin goes stale within weeks.

## Principles

### Improve, don't accumulate — the #1 discipline

This is the principle that overrides everything else in this skill:

**Default to sharpening an existing principle, not adding a new one.** Every refinement pass must hold the skill files at roughly the same size as it found them. The instinct to "add a bullet covering this case" is what kills skill plugins — within months they become 800-line tomes that the model skims, the human can't audit, and the principles dilute into noise.

Practical consequences:

1. **Read every relevant skill file before proposing any edit.** You cannot improve what you haven't loaded.
2. **For each piece of feedback, find the existing principle that comes closest. Quote it.** If you can't find one, that's evidence the feedback might warrant a new principle — but only after the search.
3. **Prefer SHARPENING the existing wording over adding a parallel bullet.** A clearer phrasing of an existing principle that now also covers the new case beats two principles that overlap.
4. **MERGE on contact.** If your proposed edit creates a principle that overlaps an existing one, fold them into one sharper bullet.
5. **DELETE when superseded.** If feedback reveals a principle was wrong or now misleads, rewrite or remove it — don't add a contradicting principle beside it.
6. **NO-OP is a valid outcome.** If feedback is too vague, one-off, or already perfectly covered, mark it `[done]` with rationale and propose no edit at all.

**New principles are allowed — when truly justified.** A piece of feedback that doesn't generalize from any existing principle, names a recurring failure mode, and would prevent future cases of the same shape: that's a new principle. But the bar is higher than the bar to *sharpen* one. Default = sharpen. Escalate = add. Always = keep tight.

If a refinement pass ends with the skill files visibly longer than before, you've probably added when you should have sharpened. Length is a smell, not a metric — the question isn't "how much did the file grow," it's "could I have replaced an existing principle instead of stacking a new one beside it?"

### The 7 steps, applied

Each step is a question to ask, in order. Don't skip; don't reorder.

#### 1. Identify what went wrong (or right)

Start from a *specific* moment. Not "Claude is bad at security." Concrete:
> "On 2026-05-22, Claude wrote `@api.depends_context('uid')` on a stored compute. Customer DB blew up on next cron run."

Vague entries can't be acted on. If the entry doesn't have a date, a file, and a verbatim phrase or snippet, it's not ready yet.

#### 2. Ask: Why?

The failure is a symptom. Find the underlying cause.
> Surface: "Claude doesn't know `depends_context`."
> Deeper: "Claude treats stored vs non-stored computes as interchangeable."

Stop at the level that *generates the most other failures if left alone*. Too shallow = you patch one case. Too deep = you write a philosophy essay nobody applies.

#### 3. Zoom out to the pattern

Would this apply beyond this one case?
> "Compute fields are one instance. The general shape is: persistence semantics differ from runtime semantics. Same applies to `related=` fields, `properties`, and to denormalized cache fields."

If the pattern is "Odoo 19 changed the API for X," that's *not* a principle — that's a fact, and it belongs in CLAUDE.md or the framework docs, not a skill.

#### 4. Check against existing principles

Open the relevant SKILL.md. Is this already covered?
- *Already covered, well:* Add the new example as a "what bad looks like" bullet under the existing principle.
- *Already covered, vaguely:* Sharpen the existing principle's wording using the new case.
- *Not covered:* Add a new principle. Check overlap with neighbors first; merge if possible.

The goal is *fewer, sharper principles*, not more. Don't grow the skill file unbounded.

#### 5. Write it as a principle, not a rule

Describe how to think, not what to do.

❌ Rule: "Never use `depends_context` on stored fields."
✅ Principle: "Stored values must be deterministic from stored inputs — context is runtime, storage is forever."

The rule patches one API. The principle prevents the next ten variations.

#### 6. Put it where it belongs

Section matters. A model-design principle goes in `odoo-module-development`. A review-time check goes in `odoo-code-review`. A test-credibility issue goes in `odoo-test-runner`.

If you're not sure, ask: *when does the model need to apply this?* While writing → module-development. While reviewing → code-review. While verifying → test-runner.

#### 7. Edit and commit

Open the file, make the edit, keep it tight. Merge overlapping principles into one. Delete principles that the new sharper one supersedes. Commit with a message that names the source — "Sharpen storage-determinism principle (journal 2026-05-22)" — so future-you can audit the loop.

Then mark the journal entry as processed (e.g., prefix `[done]`) so it doesn't get reprocessed.

### How often to run

- **Reactively:** when the user corrects you, write the journal entry immediately, then run the loop on that entry within the same session if it's a sharp lesson.
- **Proactively:** when `skills/_journal.md` hits 5+ unprocessed entries. Batch them, look for common patterns (often 5 entries collapse into 2 principles), and process together.

### What not to do

- **Don't expand a skill to "cover everything."** Skills that grow without pruning become unusable — models skim them, principles dilute. Pruning is part of refinement. See the "Improve, don't accumulate" principle above — it overrides any temptation here.
- **Don't write a principle from a single weird case.** Step 3 (zoom out) is the test. If it doesn't generalize, it's a footnote at best, not a principle.
- **Don't fold customer-specific facts into the skill.** "ACME Corp uses Odoo 17.2 with these custom modules" goes in that customer's repo CLAUDE.md, not here.
- **Don't propose an edit before reading the file you'd edit.** If you can't quote the principle you're sharpening or replacing, you don't yet know it exists. Read first.
- **Don't restate what the model already knows.** Skills add framework-specific and team-specific knowledge — not Python 101. A bullet teaching `dict.get()`, comprehensions, or "use `.grouped()` not a hand-rolled dict" is only worth keeping if the Odoo-specific angle is the point. Generic language competence is assumed; stating it dilutes the signal and burns context.

## What good looks like

A commit that:
- Removes one vague bullet from `odoo-code-review`.
- Adds one sharper bullet covering the same case plus three others.
- Marks 3 journal entries as `[done]` because all three collapse to that one bullet.

## What bad looks like

- A commit that adds 6 bullets and removes none.
- A "principle" that names a specific Odoo version or API. That's a fact, not a principle.
- The journal sitting untouched for two months. The plugin is now lying about being maintained.
