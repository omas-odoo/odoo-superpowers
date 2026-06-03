# odoo-superpowers

Odoo SA development skills plugin for PSDU work. Mirrors the [superpowers](https://github.com/obra/superpowers) format so it works across Claude Code, Codex, Gemini CLI, and other agent harnesses.

## What's inside

- **10 skills** (principles, not rules) — entry skill plus module dev, migrations, Python, XML conventions, JS/OWL frontend, code review, test runner, task completion, and a self-refinement loop
- **Hooks** — block `git push` until you approve, auto-`ruff format` on Python edits, stub for Discord uncertainty capture
- **`settings/settings.json`** — a permissions template you copy into project `.claude/settings.json` so common read-only commands don't prompt
- **`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`** — values doc, auto-loaded by each harness

## Install (Claude Code)

This repo ships its own marketplace (`.claude-plugin/marketplace.json`), so add the
marketplace first, then install the plugin by name. No manual `git clone` needed.

```bash
# 1. register the marketplace (pick one source form)
/plugin marketplace add omas-odoo/odoo-superpowers                          # owner/repo shorthand
/plugin marketplace add https://github.com/omas-odoo/odoo-superpowers.git   # full git URL
/plugin marketplace add /path/to/odoo-superpowers                           # local checkout (dev)

# 2. install the plugin (format is <plugin>@<marketplace>)
/plugin install odoo-superpowers@odoo-superpowers-dev
```

Private repo? Users need GitHub access for the shorthand/URL forms to fetch it.

### Pre-register for a team

To make it always available (e.g. via a shared `settings.json`), skip the `marketplace add`
step and declare it in your Claude Code settings instead:

```json
{
  "extraKnownMarketplaces": {
    "odoo-superpowers-dev": {
      "source": { "source": "git", "url": "https://github.com/omas-odoo/odoo-superpowers.git" }
    }
  },
  "enabledPlugins": {
    "odoo-superpowers@odoo-superpowers-dev": true
  }
}
```

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
