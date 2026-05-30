# odoo-superpowers

Odoo SA development skills plugin for PSDU work. Mirrors the [superpowers](https://github.com/obra/superpowers) format so it works across Claude Code, Codex, Gemini CLI, and other agent harnesses.

## What's inside

- **6 skills** (principles, not rules) — entry skill plus module dev, code review, test runner, task completion, and a self-refinement loop
- **Hooks** — block `git push` until you approve, auto-`ruff format` on Python edits, stub for Discord uncertainty capture
- **`settings/settings.json`** — a permissions template you copy into project `.claude/settings.json` so common read-only commands don't prompt
- **`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`** — values doc, auto-loaded by each harness

## Install (Claude Code)

```bash
# from the parent dir of this repo
/plugin install ./odoo-superpowers
```

Or add to your Claude Code marketplace and `/plugin install odoo-superpowers`.

## Install (Codex)

Copy or symlink this directory into Codex's plugins folder. Codex reads `.codex-plugin/plugin.json`.

## Install (Gemini CLI)

Add the plugin directory; Gemini reads `gemini-extension.json` and auto-loads `GEMINI.md`.

## Layout

```
.claude-plugin/        Claude Code manifest
.codex-plugin/         Codex manifest
hooks/                 PreToolUse + PostToolUse hooks
skills/                One folder per skill, each with SKILL.md
settings/              Permissions template to copy into projects
CLAUDE.md              Values doc (PSDU conventions)
AGENTS.md              Symlink → CLAUDE.md
GEMINI.md              Gemini context file
```

## Philosophy

**Principles, not rules.** Each SKILL.md tells the model *how to think*, not *what to do*. Rules go stale; principles transfer.

**Learning loop.** When the model surprises you (good or bad), log it in `skills/_journal.md`. Periodically run `refining-odoo-skills` to fold the journal back into the skills. The plugin gets sharper every week.

**Cross-harness.** Same skill files work in Claude Code, Codex, Gemini CLI, and any agent that loads markdown context files.
