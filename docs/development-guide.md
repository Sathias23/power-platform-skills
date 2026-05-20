# Development Guide

> Setting up, building, testing, and contributing to the `power-platform-skills` marketplace and its plugins.

## Prerequisites

| Tool | Required for | Notes |
|------|--------------|-------|
| **Node.js** (LTS) | All plugins — scripts, validators, tests | The repo deliberately has no `package.json`; Node is invoked directly via `node` / `node --test` |
| **Claude Code** OR **GitHub Copilot CLI** | Consuming the plugins as a marketplace user | Use `claude --plugin-dir` for local plugin development |
| **`pac` CLI** (Power Platform CLI) | All plugins that deploy Power Platform artifacts | `scripts/install.js` installs this automatically |
| **`az` CLI** (Azure CLI) | `power-pages` Dataverse + Azure Key Vault flows | Plain `az login` is preferred; `--allow-no-subscriptions` ONLY on `az login` itself |
| **.NET 10 SDK** | `canvas-apps` only — needed by Canvas Authoring MCP server | Download: https://dotnet.microsoft.com/download/dotnet/10.0 |

There is **no root-level build or transpile step**. All plugin content is plain Markdown + Node.js — no bundler, no TypeScript, no `node_modules` at the repo root.

## Repo setup

```bash
git clone https://github.com/microsoft/power-platform-skills.git
cd power-platform-skills
```

That's it. No `npm install`. No `dotnet restore`. Plugins are consumed by an AI agent at runtime.

## Local plugin development

Test a single plugin in an isolated agent session:

```bash
claude --plugin-dir /path/to/power-platform-skills/plugins/power-pages
claude --plugin-dir /path/to/power-platform-skills/plugins/canvas-apps
claude --plugin-dir /path/to/power-platform-skills/plugins/model-apps
claude --plugin-dir /path/to/power-platform-skills/plugins/code-apps
claude --plugin-dir /path/to/power-platform-skills/plugins/mcp-apps
```

> The marketplace name for `code-apps` is `code-apps-preview` (see `marketplace.json`). The source directory is `plugins/code-apps`.

## Running tests

The only test runner used in this repo is **Node's built-in `node --test`**. Jest, Vitest, Mocha, etc. are explicitly not used.

```bash
# Power Pages script tests (the only plugin with a tests/ directory today)
node --test plugins/power-pages/scripts/tests/
```

CI runs this on PRs that touch `plugins/power-pages/` via `.github/workflows/power-pages-script-tests.yml`.

**Coverage expectation:** Every new or modified script ships with a corresponding `*.test.js` file under the same plugin's `scripts/tests/`. Validators are not an exception.

**Live-service tests** (Dataverse / Azure) must stay opt-in via explicit flags (e.g., `--validate-dataverse-relationships`). Default test runs must pass with zero connectivity.

## Repo-level checks before pushing

```bash
# Enforce the version-check line on every power-pages SKILL.md
node scripts/ensure-skill-version-check.js          # auto-fix missing lines
node scripts/ensure-skill-version-check.js --check  # CI: fail if any are missing
```

CI runs the `--check` variant via `.github/workflows/ensure-skill-version-check.yml`.

`.github/workflows/validate-plugin-names.yml` additionally enforces consistency between the marketplace manifest and per-plugin `plugin.json` names.

## Skill authoring quick reference

Authoring a new skill or modifying an existing one:

1. **Frontmatter** — Required keys: `name`, `description`, `user-invocable`, `allowed-tools` (**comma-separated string only**), `model` (`opus` or `sonnet`). Do NOT add `hooks:` to frontmatter.
2. **Version-check line** (power-pages only) — Immediately after the closing `---`:

   ```markdown
   > **Plugin check**: Run `node "${CLAUDE_PLUGIN_ROOT}/scripts/check-version.js"` — if it outputs a message, show it to the user before proceeding.
   ```

3. **Phase structure** — 5–8 phases: Prerequisites → Discover → Plan → Implement → **Verify** (mandatory standalone) → Deploy/Summarize.
4. **Task tracking** — `TaskCreate` all phase tasks upfront at Phase 1. One task per phase. `subject` / `activeForm` / `description` required on each.
5. **Three-point approval** — `AskUserQuestion` after discovery, after plan, before deploy. Work autonomously between checkpoints.
6. **Shell-agnostic** — No PowerShell cmdlets in code blocks. Use ` ```bash ` (or plain ` ``` `) for `pac`, `az`, `dotnet`, `node`. Angle-bracket `<placeholder>` style for variables.
7. **Path resolution** — Reference shared assets via `${CLAUDE_PLUGIN_ROOT}/...` (never hardcode `plugins/<name>/...` from inside a SKILL.md).
8. **Skill tracking** — Final phase records usage via the pointer pattern:

   ```markdown
   > Reference: ${CLAUDE_PLUGIN_ROOT}/references/skill-tracking-reference.md
   ```

   New skills must also be added to the mapping table in `references/skill-tracking-reference.md`.

See `plugins/power-pages/PLUGIN_DEVELOPMENT_GUIDE.md` for the full five-pillar standard (engaging loading, action verification, sub-agent architecture, deterministic execution, transparency & approval).

## Working with shared modules (DRY)

Before writing a new helper, **always grep the plugin's shared module dirs first**:

| Plugin | Shared module location |
|--------|------------------------|
| `power-pages` | `scripts/lib/` (validation-helpers, powerpages-config, schema-validator, hook-utils, etc.) + `references/` |
| `model-apps` | `references/` only |
| `mcp-apps` | `references/` only |
| `canvas-apps` | `references/` only |
| `code-apps` | `shared/` (at plugin root, not under `scripts/`) + per-skill `references/` |

When adding shared logic: put it in the plugin's shared module — never copy it into a skill-specific directory.

## Cross-plugin shared skills

Workflows that apply to multiple plugins live in `shared/skills/<skill-name>/`:

- `<workflow>.md` — Single source of truth for the workflow text
- `SKILL.template.md` — Frontmatter template with `{{PLUGIN_NAME}}` placeholder
- Each consuming plugin gets a **thin wrapper** at `plugins/<plugin>/skills/<skill-name>/SKILL.md` — frontmatter + one-line reference to the shared workflow. Never duplicate workflow content.

Currently: `report-issue` is the only cross-plugin shared skill.

When updating a shared workflow, update every per-plugin wrapper in the same PR.

## Common commands cheat sheet

```bash
# Run all power-pages script tests
node --test plugins/power-pages/scripts/tests/

# Run a single test file
node --test plugins/power-pages/scripts/tests/validation-helpers.test.js

# Verify version-check lines (CI mode)
node scripts/ensure-skill-version-check.js --check

# Install the marketplace via the official one-liner
curl -fsSL https://raw.githubusercontent.com/microsoft/power-platform-skills/main/scripts/install.js | node

# Add the marketplace from within a Claude Code session
/plugin marketplace add microsoft/power-platform-skills

# Install a specific plugin
/plugin install power-pages@power-platform-skills
```

## Git workflow

- **Branch:** Work off `main`. PRs target `main`.
- **Commits:** Terse, conventional style — match the existing `git log`. No mandated prefix scheme. Subject under ~72 chars.
- **Commit cadence:** During skill workflows that produce artifacts, commit after every significant milestone (per page, per component, per phase). Don't bundle a whole feature into one final commit.
- **PR hygiene:**
  - Touch a script → ship `node:test` coverage in the same PR.
  - Touch a SKILL.md → grep the plugin for stale phase numbers and contradicting guidance before submitting.
  - Touch a shared workflow under `shared/skills/<name>/` → update every per-plugin wrapper in the same PR.
  - Touch the marketplace → update both `.claude-plugin/marketplace.json` AND the affected `plugins/<name>/.claude-plugin/plugin.json`.
