---
name: odoo-grill
description: Use before implementing a non-trivial Odoo feature or design, to pressure-test the approach before any code is written. Symptoms — "I'm about to build X", an underspecified plan, or a design with multi-company / upgrade implications. Interrogates the plan one question at a time, each with a recommended answer, walking the design tree. Not for quick fixes the user already scoped.
---

# Grill the plan

Before writing Odoo code, interrogate the plan — one question at a time, each carrying your recommended answer — and walk the design tree to its leaves. The grill produces decisions, not code; it pre-empts the rework that surfaces when a wrong assumption is only discovered mid-implementation. Implementation starts after the plan survives, not before.

If the codebase can answer a question, read the code instead of asking. Wait for the answer before the next question.

## Question lenses (walk the tree through these)

1. **Functional before technical.** What does the customer actually need to happen? Could a setting, an automated action, Studio, or an existing module deliver it with zero custom code? Why does this need a new module — or any code — at all?
2. **The Odoo dimensions.** Multi-company, multi-currency, multi-language, multi-warehouse, multi-website: which apply, and what happens at each boundary? Invent a concrete scenario per applicable dimension and make the plan resolve it.
3. **Record rules & access rights.** Which groups see this, and which do it? What record rules apply? What does a user *outside* the group experience — and what's exposed on portal?
4. **Upgrade safety.** Does every schema choice survive the next migration? Which xml_ids will customer data reference? What does the eventual upgrade of this code look like?
5. **Edge cases.** Invent scenarios that stress the concept's boundaries: 100× the data, in-flight records (draft / posted / done) at deploy time, the empty set, the duplicate. Make the plan account for each.

## Conduct

- **Cross-reference the source.** When a claim contradicts the module or core code, read it and quote it back: "`_compute_x` already sets this on `account.move` — which one wins?" The code is the tie-breaker, not the assertion in the plan.
- **Sharpen fuzzy terms.** "'account' — `res.partner`, `account.account`, or `res.users`?" An ambiguous noun hides two forked designs.
- **Stop when the remaining questions wouldn't change the design.** Grilling past that point is theatre.
- **Close with a short summary of what got decided.** Surface the durable decisions for the user — the ones future work should know about. They can be folded back into the skills via the learning loop; see `refining-odoo-skills` / `skills/_journal.md`.
