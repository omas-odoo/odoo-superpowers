# Skill Journal

Append-only log of corrections, surprises, and "I should have known that" moments.
Process via `refining-odoo-skills` when this file has 5+ unprocessed entries.

Format per entry:

```
## YYYY-MM-DD — short title

**What:** concrete case, verbatim snippet or phrase
**Why it matters:** the underlying pattern, not the surface
**Likely target skill:** odoo-module-development | odoo-code-review | odoo-test-runner | other
**Status:** unprocessed
```

Mark `Status: [done] processed in commit abc1234` once folded into a skill.

---

## 2026-05-30 — promoted migrations to its own skill + fixed a self-contradiction

**What:** Migrations lived as a *reference inside* `odoo-module-development` (`references/migrations.md`), not a skill — inconsistent with how `.py` (odoo-python) and `.xml` (odoo-xml-conventions) each own a sibling skill. Worse, the file contradicted itself: its per-shape examples used raw `cr.execute` (drop column + hand-`DELETE FROM ir_model_fields`) while its own bottom section said "always use util, never bare cr.execute when a helper exists." User: "why do we have migrations in module development, we already have a separate skill for that no? also it should use the util."
**Why it matters:** (1) Triggering — "writing an upgrade script" should invoke a dedicated skill, not dig through module-development's references. (2) The raw-SQL examples were the exact anti-pattern we'd just banned, sitting in our own "good example" slot — an internal contradiction, which refining-odoo-skills explicitly warns against.
**Likely target skill:** new `odoo-migrations` (promoted) + rewire references everywhere
**Status:** [done] processed 2026-05-30 — created `odoo-migrations/SKILL.md` (triggers/absence-test, util-first, pre/post/end, first-install guard, snapshot verification) + `references/migration-patterns.md` (util catalog, per-shape code ALL rewritten util-first: rename_field/remove_field/rename_model/recompute_fields one-liners; bare SQL kept only for backfill/dedupe, guarded). Rewired odoo-code-review, odoo-module-development (SKILL bullet + refs table), odoo-python/performance.md, using-odoo-superpowers, SessionStart hook, both bot review prompts to point at the new skill.

## 2026-05-30 — refined from Odoo R&D review of omas-odoo's hr_attendance_zkteco PRs

**What:** Mined 41 reviewer comments on omas-odoo's `hr_attendance_zkteco` PRs in odoo/enterprise — reviewers were xmo-odoo (Xavier Morel, core framework) and tivisse (R&D lead). Core-Odoo-grade feedback on real shipped integration code. Conventions we lacked:
1. Self-less methods doing external I/O belong in module-level `utils.py`, NOT model methods — every public model method is RPC-callable (tivisse #30). Security.
2. Decide field required-ness deliberately; "empty transactions shouldn't be possible" (tivisse #14/16/18/24, xmo #66).
3. Don't store derivable/redundant fields — collapse check_in/check_out → one punch_type; drop punch_date (group datetime by date) (tivisse #20/28).
4. Protected fields: check access (has_group → AccessError) THEN `.sudo()` to read (tivisse #6/10).
5. Exception hierarchy rigor — don't catch redundant `requests` subclasses, don't write branches that all `return None`, logs must carry the error (xmo #42/56).
6. External HTTP → configured `requests.Session`, headers set once, `json=` implies content-type, follow pagination `next` links, take one URL not server_url+endpoint, no unused params, resolve auth in the call that uses it (xmo #44/48/52/58/60).
7. Framework helpers: `activity_schedule` not manual mail.activity creation (tivisse #34).
**Why it matters:** This is the highest-signal source yet — core devs reviewing PSAE integration code. Exposed a whole uncovered domain (external integrations) that PSAE does constantly (zkteco, almadeed, payment gateways).
**Likely target skill:** odoo-python (SKILL security + framework-helpers + new external-integration.md) + odoo-module-development (model design) + odoo-code-review (RPC red flag) + python-conventions (exceptions)
**Status:** [done] processed 2026-05-30 — created `odoo-python/references/external-integration.md` (Session, auth, pagination, exception hierarchy, utils.py, no-unused-params, stdlib); added utils.py-RPC + access-then-sudo bullets to odoo-python Security; added `activity_schedule` to framework-helper list; added deliberate-required-ness + minimize-fields to odoo-module-development model design; sharpened python-conventions Exceptions (redundant subclasses, informative logs); added RPC-exposure + sudo red flags to odoo-code-review; fixed a stale hand-rolled-dict example in odoo-python "what good looks like" to use `.grouped()`.

## 2026-05-30 — refined odoo-python from abwa-odoo's real PR review comments

**What:** Mined 119 inline review comments by abwa-odoo (senior PSAE reviewer) across 30 recent psae-* repos. Recurring conventions our skill missed or had wrong:
1. `=` for one field, `.write({...})` for many ("dont do `write` use `=`", ×2).
2. `fields.Json` instead of `json.dumps/loads` into a Text field (×2).
3. `float_compare` / `float_is_zero` for money/float — never `==`/`<` (comments 5, 63).
4. New `models.Constraint(...)` syntax (v18+), NOT the legacy `_sql_constraints = [...]` list (comment 67) — our orm-patterns.md had the old syntax.
5. "Reach for the framework helper" — `_get_records_action`, `_get_html_link`, `message_post`, `currency._convert` (×5+ "why not use X?").
6. `ensure_one()` is over-used — abwa pushes against it and for batching ("why self.ensure_one?" ×3, "not recommended when not atomic, batch where possible").
7. Deliberate `cr.commit()` in integration/sync loops (comments 15, 21) — nuances the hard "never commit" rule.
**Why it matters:** This is the highest-signal source we have — a senior's actual nitpicks ARE the convention. #4 was a correctness bug (old syntax on v19). #6 contradicted our existing "ensure_one is documentation" framing.
**Likely target skill:** odoo-python (SKILL + orm-patterns) + odoo-code-review (ensure_one)
**Status:** [done] processed 2026-05-30 — orm-patterns.md: switched SQL constraints to `models.Constraint`, added float-comparison / currency `_convert` / `=`-vs-`.write` / `_search`-for-filtered-compute / `fields.Json` sections. odoo-python SKILL.md: added "Reach for the framework helper before hand-rolling" subsection. odoo-code-review: sharpened the ensure_one bullet (batch, don't reflex). python-conventions.md: added the integration-loop commit exception.

## 2026-05-27 — added odoo-python skill from PSAE style guide

**What:** Plugin had `odoo-xml-conventions` for `.xml` but no equivalent for `.py`. PSAE-specific Python/ORM review feedback (compute-field cross-writes, no `hasattr`, `*args/**kwargs` overrides, no-query-in-loop, batching, Monetary/SQL-constraint/company-rule patterns, copy-paste vigilance) lived only in a loose CLAUDE.md style guide, captured nowhere in the skills.
**Why it matters:** Symmetry and triggering accuracy — `.py` work now has a skill that owns it, mirroring how `.xml` work routes to xml-conventions. Filled a real gap rather than accumulating; the upstream `python-conventions.md` reference moved from odoo-module-development into the new skill so all Python lives in one place.
**Likely target skill:** new `odoo-python` (+ small edits to odoo-module-development for manifest/structure PSAE bits)
**Status:** [done] processed 2026-05-27 — created `odoo-python/SKILL.md` + `references/orm-patterns.md` + `references/performance.md`; moved `python-conventions.md` in; repointed odoo-module-development references table; added split-by-dependency / OEEL-1 / MMC / one-import-per-line; registered in using-odoo-superpowers + SessionStart hook + bot review prompts.

## 2026-05-27 — skills were restating baseline Python the model already knows

**What:** `python-conventions.md` had an "Idioms" section teaching `dict.get()`, comprehensions, truthy collections, `dict()` over `.clone()`. `performance.md` showed a hand-rolled `{p.id: p for p in ...}` dict where Odoo's `.grouped()` is idiomatic. User: "whats the point of all of this it would already know this" + "we'd prefer `.grouped()`".
**Why it matters:** Skills exist to add framework/team knowledge the model lacks, not to re-teach language basics it has cold. Restating Python 101 dilutes signal and burns context. The right fix is twofold: delete the generic, and where a generic operation HAS an Odoo-idiomatic form (`.grouped()`, `mapped`, `_read_group`), state only that form.
**Likely target skill:** odoo-python references + refining-odoo-skills (authoring anti-pattern)
**Status:** [done] processed 2026-05-27 — removed the generic Idioms block from python-conventions.md (kept only Odoo-specific ORM idioms incl. `.grouped()`); rewrote performance no-query-in-loop example to navigate relations + use `.grouped()`; added "Don't restate what the model already knows" to refining-odoo-skills What-not-to-do.

## 2026-05-30 — upgrade scripts must use the upgrade `util` library, not raw SQL or openupgradelib

**What:** Both performance.md ("prefer raw SQL over ORM chains") and migrations.md ("openupgradelib is the standard PSDU helper") were wrong for PSAE. User: "upgrade scripts should always utilize upgrade util." The correct tool is `odoo.upgrade.util` (Odoo SA's official upgrade-util, available on odoo.sh), used via `from odoo.upgrade import util`.
**Why it matters:** Bare `cr.execute` silently breaks the ORM cache, stored recomputes, FK integrity, and indirect deps. `util.rename_field` / `remove_field` / `rename_model` / `recompute_fields` / `explode_execute` handle all of that. "Raw SQL is faster" was the wrong frame — `util` IS how you do the SQL safely. And openupgradelib is the OCA/community lib; PSAE is Odoo SA → official upgrade-util, not OCA.
**Likely target skill:** odoo-python/references/performance.md + odoo-module-development/references/migrations.md
**Status:** [done] processed 2026-05-30 — reframed both: util helpers first, bare cr.execute only when no helper exists and guarded with column_exists/table_exists; replaced the openupgradelib section with `odoo.upgrade.util`; bare unguarded DROP/ALTER/UPDATE is now a review blocker.

## 2026-05-23 — migration coverage was a single bullet; reviewer had no trigger list

**What:** `odoo-code-review` and `odoo-module-development` each had one passing mention of migrations ("if you add a required field, you owe a migration"). No reference for *how* to write one, no checklist of *what diff shapes* trigger the need, no signature documentation, no openupgradelib pointer. A reviewer reading the SKILL.md couldn't say "this diff needs a migration and is missing one."
**Why it matters:** PSDU work updates existing customer modules constantly. The most common production-breaking PR is a new required field with no migration script — the customer's upgrade fails on the NOT NULL constraint and they're stuck. The pattern generalizes beyond required fields: type changes, renames, drops, new UNIQUE constraints, ondelete changes, noupdate XML edits — all of them silently leave data inconsistent if there's no migration. The reviewer skill needs an explicit "absence test" because *missing* migrations is the bug, and lint can't find them.
**Likely target skill:** odoo-code-review (the reviewer needed an explicit trigger table) + odoo-module-development (needed a reference with the actual conventions: directory layout, `migrate(cr, version)` signature, pre/post/end timing, openupgradelib, manifest version bumping, verification via dump+load+upgrade)
**Status:** [done] processed 2026-05-23 — created `odoo-module-development/references/migrations.md` (full reference: triggers, script layout, signature, pre/post/end semantics, 6 common SQL patterns, openupgradelib, version bumping, verification loop); added "Migration triggers — the absence test" subsection to `odoo-code-review/SKILL.md` with the diff-shape trigger table; updated `odoo-module-development/SKILL.md` references map to point at migrations.md.

## 2026-05-23 — "testing" in PSDU means verification, not just `--test-enable`

**What:** `odoo-test-runner` SKILL.md framed testing narrowly as running unit tests via `--test-enable`. But PSDU "testing" is the full verification loop after writing a development: install on a fresh DB, prove the feature works (tests if present, shell for data shape, UI for flows), then drop. The skill conflated "automated tests" with "verification."
**Why it matters:** Unit tests can't see what the user sees. A view change, a new wizard button, a portal access rule — none of these are caught by `--test-enable` alone. Reframing the skill around *verification with three matching layers* (tests / shell / UI) makes the model pick the right tool for what was actually built, instead of mechanically running `--test-enable` and calling it done.
**Likely target skill:** odoo-test-runner (the skill's framing was wrong) + odoo-task-completion (Gate 2 needed broadening to match)
**Status:** [done] processed 2026-05-23 — rewrote SKILL.md around the three-layer verification model; added shell-based and UI-based loops alongside the test-enable loop; added `odoo-bin shell` and the bare-server invocation to references/odoo-bin-commands.md; renamed task-completion "Gate 2: Tests" → "Gate 2: Verification" with explicit layer-picking guidance.

## 2026-05-23 — test runner should use `odoo-bin` directly, not a shell wrapper

**What:** Initial `odoo-test-runner` skill shipped a `scripts/run_tests.sh` that wrapped `createdb` / `odoo-bin` / `dropdb`. The agent had to read the script to know what was actually running, and the script reinvented behavior `odoo-bin db init` / `db drop` already provide (including filestore handling).
**Why it matters:** Wrappers around well-designed CLIs add misdirection without adding capability. They also bypass features the CLI gets right — `odoo-bin db drop` cleans the filestore, raw `dropdb` doesn't. The principle generalizes: prefer the upstream tool over local shims unless the shim adds something real.
**Likely target skill:** odoo-test-runner (write-time concern — the model needs to know which command to invoke, not how to navigate a wrapper)
**Status:** [done] processed 2026-05-23 — deleted `scripts/run_tests.sh`, rewrote SKILL.md to call `odoo-bin db init` / `-i --test-enable` / `db drop` directly, added `references/odoo-bin-commands.md` with the scoped CLI cheat sheet (only commands relevant to local DB + install + test).

## 2026-05-23 — inherited view `name` suffix uses dots not underscores

**What:** Initial `xml-naming.md` and `xml-inheritance.md` mentioned the `.inherit.<module>` suffix on inheriting views but didn't make the underscore-to-dot conversion explicit. If module is `psae_base`, name suffix is `.inherit.psae.base`, NOT `.inherit.psae_base`. This is reportedly the most common XML mistake.
**Why it matters:** The dotted form mirrors the rest of the `name` field (which is the original record's name with dots). Underscores in the module suffix break the visual pattern reviewers scan for and confuse the model into emitting non-conforming names.
**Likely target skill:** odoo-xml-conventions (write-time concern — the model needs this while writing the XML, not after)
**Status:** [done] processed in scaffold update 2026-05-23 — added explicit table mapping module names to suffixes in both xml-naming.md and xml-inheritance.md; added two new entries to the common-mistakes table.

## 2026-06-03 — created odoo-js skill from ~800 frontend review comments

**What:** PSDU had no skill for `static/src` work — OWL components, `patch()` overrides, QWeb/OWL templates, SCSS — even though it's a large share of customer POS/website code. Mined 799 inline review comments left by ayo-odoo (Alaa Youssef) and abma-odoo on JS/OWL/SCSS files across 113 `odoo-ps` repos (search API for PRs they commented on / reviewed, core API for the inline comments; filtered to `.js/.ts/.jsx/.tsx`, `.scss/.css`, and `.xml` under `static/`). Clustered by theme; the signal was dense and consistent across both reviewers.
**Why it matters:** The recurring findings are almost all *reaching around the framework*, not logic bugs: reassigning `X.props` instead of `patch(X.props.line.shape, ...)`; copy-pasting a standard method instead of calling `super`; querying the DOM (`$()`, `getElementById`, `data-*`) instead of state/`useRef`/`t-on`; storing a derived value instead of using a getter; `t-esc` (deprecated) over `t-out`; inline styles; hiding standard nodes with `position="replace"`/`t-if="False"` instead of `d-none` (breaks other modules' xpaths); plus the house style nits (`const` not `var`, camelCase, ES6 imports, `_t()` on every user string, no leftover `debugger`). None of this is in odoo-python (Python-only) or odoo-xml-conventions (backend views) — it needed its own skill.
**Likely target skill:** new `odoo-js` (created)
**Status:** [done] processed 2026-06-03 — created `odoo-js/SKILL.md` (principles: patch-and-super discipline, state/getters over DOM, services/ORM/RPC, declarative templates, SCSS, frontend style) + four references (`owl-components.md`, `patching-services.md`, `templates-styling.md`, `js-conventions.md`). Registered in `using-odoo-superpowers` (routing + split QWeb-templates note off `odoo-xml-conventions`), added a "Frontend (OWL/JS)" dimension to `odoo-code-review`, added a scope note to `odoo-xml-conventions`, bumped README skill count 6→10.

## 2026-06-12 — refined odoo-js from abma-odoo's review of psae-cafe-najjar PR #12

**What:** 16 JS/XML inline comments on tier-discount POS code (review 4475579688). Gaps the skill missed: `applySettlement({ orm, pos, ... })` on the PosOrder *model* taking services as arguments ("bro find a better way to handle this"); `_getTierProgram()` no-arg methods on a patched model ("make this as getter" — skill only said state/props on components); `PosOrder.prototype.removeOrderline.call(this, ...)` (patch() mutates the prototype, so it's the patched method anyway — a no-op disguise); summing via a single `reduce` callback ("map + reduce", ×5 — js-conventions.md even showed that exact form as the GOOD example); PosOrderline patch living in `pos_order.js` ("move this part to separate file"); re-acquiring `useService('dialog')` when core TicketScreen already assigns it; `position="replace"` on a *component* tag where `d-none` can't work (class becomes a prop → `t-if` via `position="attributes"`). Plus one trap the reviewer's own literal suggestion would introduce: `order.payment_ids.forEach((l) => order.removePaymentline(l))` without snapshotting iterates a live recordlist that shrinks mid-loop.
**Why it matters:** The deep pattern is layering — models hold data, the store holds service-needing logic, components orchestrate UI; passing services as arguments is the symptom of logic in the wrong layer. The rest were existing principles stated too narrowly (components-only getters, element-only d-none) or contradicted by our own reference example.
**Likely target skill:** odoo-js
**Status:** [done] processed 2026-06-12 — SKILL.md: new "services never travel as arguments" principle (store-level home, pos_loyalty precedent); sharpened patch bullet (prototype.call trap), layer bullet (one patched class per file), getter bullet (record fields + patched models), useService bullet (check core setup first), d-none bullet (position="attributes" + component-tag t-if), semantic-array clause (one job per stage, flatMap, live-recordlist snapshot). references/js-conventions.md: fixed contradicting reduce example to map-then-reduce, added flatMap + snapshot lines.
