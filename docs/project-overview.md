# Project Overview — `power-platform-skills`

> Microsoft's official plugin marketplace for Power Platform development with Claude Code and GitHub Copilot CLI.

## What this is

`power-platform-skills` is a **monorepo plugin marketplace**, not a runnable application. It hosts Claude Code / Copilot CLI plugins that help developers build on the Microsoft Power Platform. Each plugin exposes a set of skills, sub-agents, and Node.js helper scripts that an AI agent uses to scaffold, integrate, validate, and deploy Power Platform artifacts.

Users install it via:

```bash
# One-line installer
curl -fsSL https://raw.githubusercontent.com/microsoft/power-platform-skills/main/scripts/install.js | node

# Or, inside a Claude Code session
/plugin marketplace add microsoft/power-platform-skills
/plugin install power-pages@power-platform-skills
```

The marketplace ships from `main` — there is no build step, no container, no package publish. The repo IS the distribution.

## Tech stack at a glance

| Category | Choice | Notes |
|----------|--------|-------|
| Authoring format | Markdown SKILL.md with YAML frontmatter | All skill workflows |
| Scripting language | Node.js (built-in modules only) | Determined-execution layer; zero `npm install` at repo root |
| Test runner | `node --test` | Zero dependencies. No Jest / Vitest / Mocha |
| AI agent hosts | Claude Code, GitHub Copilot CLI | Both supported by the marketplace manifest |
| MCP servers | `playwright` (local node), `microsoft-learn` (HTTP), `canvas-authoring` (.NET 10 via `dnx`) | Per-plugin via `.mcp.json` |
| External CLIs | `pac`, `az`, `dotnet`, `node`, `npx` | Used by various plugins; `scripts/install.js` installs `pac` |
| CI | GitHub Actions (4 workflows) | Version-check enforcement, plugin name validation, `node --test`, repo stats |
| Distribution | `.claude-plugin/marketplace.json` + auto-update | Pulled from `main` by users' agent hosts |

## Repository classification

- **Type:** Monorepo (plugin marketplace)
- **Active parts:** 5 plugins (see below) + 1 proposal-only directory
- **Primary language:** Markdown (workflow content) + JavaScript (deterministic scripts)
- **Architecture:** Per-plugin skill/agent/script bundles, surfaced via a shared marketplace manifest

## Active plugins

| Plugin | Version | Marketplace name | What it does |
|--------|---------|------------------|--------------|
| `power-pages` | 1.3.0 | `power-pages` | Build & deploy Power Pages code sites (React / Vue / Angular / Astro SPAs) with Dataverse + Web API + permissions + server logic + cloud flows |
| `model-apps` | 1.0.6 | `model-apps` | Build & deploy Power Apps generative pages (React 17 + Fluent UI v9 + DataAPI) for model-driven apps |
| `mcp-apps` | 1.0.0 | `mcp-apps` | Generate self-contained HTML widgets using the MCP Apps protocol (CDN-loaded; no build step) |
| `canvas-apps` | 2.1.0 | `canvas-apps` | Author Power Apps Canvas Apps via the Canvas Authoring MCP server (.NET 10 SDK required) |
| `code-apps` | 1.0.0 | `code-apps-preview` ⚠️ | Build & deploy Power Apps code apps (React + Vite + TS) with Power Platform connectors |

> ⚠️ `code-apps` is the only plugin where the marketplace name differs from the source directory. Preserve this mapping when editing `marketplace.json` or `plugin.json`.

## Proposed plugins (not active)

| Plugin | Status |
|--------|--------|
| `power-automate` | Proposal only — `PROPOSAL.md`. Planned authoring surface: Dataverse Web API (`/workflows`). Packaging: `pac solution clone/pack`. Not registered in `marketplace.json`. |

## Repository structure (summary)

```text
power-platform-skills/
├── .claude-plugin/marketplace.json   # Source of truth for active plugins
├── .github/workflows/                 # 4 CI workflows
├── plugins/                           # 5 active + 1 proposal
│   ├── power-pages/                   # Reference implementation — has all patterns
│   ├── model-apps/, mcp-apps/, canvas-apps/, code-apps/
│   └── power-automate/                # PROPOSAL.md only
├── shared/skills/                     # Cross-plugin shared workflows
│   └── report-issue/                  # Single source of truth + per-plugin wrappers
├── scripts/                           # Repo-level tooling
│   ├── install.js                     # Marketplace installer (curl | node)
│   └── ensure-skill-version-check.js  # CI: enforces version-check line on every power-pages SKILL.md
├── evals/                             # Per-plugin eval suites (mcp-apps, model-apps, power-pages)
├── AGENTS.md / CLAUDE.md              # Repo-level dev guidelines (CLAUDE.md is a symlink)
└── README.md                          # User-facing install + plugin index
```

Full annotated layout: `source-tree-analysis.md`.

## Conventions enforced repo-wide

These are baseline rules every plugin must follow (from the root `AGENTS.md`):

- **`allowed-tools` is a comma-separated string** in SKILL.md frontmatter (never a YAML list or JSON array).
- **Phase structure is 5–8 phases** with a mandatory standalone Verify phase.
- **`TaskCreate` upfront** at Phase 1; one task per phase.
- **Three-point user approval** (discovery, plan, deploy).
- **Shell-agnostic docs** (no PowerShell cmdlets in `bash` code blocks).
- **`${CLAUDE_PLUGIN_ROOT}/...` paths** in SKILL.md (never hardcoded `plugins/<name>/...`).
- **DRY** — shared helpers per plugin (most fully realized in `power-pages/scripts/lib/`); always grep before writing new code.
- **`node --test` only** — no Jest / Vitest / Mocha.
- **Every new/modified script ships with tests in the same PR.**

The fullest restatement of these rules is in `_bmad-output/project-context.md` (100 numbered rules) which serves as the canonical agent context.

## Document index

Detailed documentation generated by `/bmad-document-project`:

- **[Source Tree Analysis](./source-tree-analysis.md)** — Annotated directory layout, per-plugin
- **[Integration Architecture](./integration-architecture.md)** — How the plugins, MCP servers, hooks, and external services connect
- **[Development Guide](./development-guide.md)** — Setup, tests, skill authoring quick reference
- **[Deployment Guide](./deployment-guide.md)** — How the marketplace ships and how each plugin's downstream artifacts deploy
- **[Contribution Guide](./contribution-guide.md)** — PR checklist, conventions, common review pitfalls
- **[Component Inventory](./component-inventory.md)** — Sub-agents, MCP tools, shared modules, downstream UI patterns
- **[Project Parts (JSON)](./project-parts.json)** — Machine-readable plugin metadata

### Per-plugin architecture

- **[Power Pages](./architecture-power-pages.md)** — Reference implementation; full pattern set
- **[Model Apps](./architecture-model-apps.md)** — Single-skill lifecycle plugin
- **[MCP Apps](./architecture-mcp-apps.md)** — Pure-markdown plugin; no scripts
- **[Canvas Apps](./architecture-canvas-apps.md)** — MCP-driven coauthoring with .NET 10
- **[Code Apps (preview)](./architecture-code-apps-preview.md)** — Connector-centric; NPX CLI orchestration

## Getting started

1. **Browse the plugin index in `README.md`** to pick which plugins you need.
2. **Install** via `curl | node` or `/plugin marketplace add microsoft/power-platform-skills`.
3. **Develop locally** by cloning and using `claude --plugin-dir /path/to/plugins/<plugin-name>`.
4. **Contribute** — read `AGENTS.md` (root), the plugin-specific `AGENTS.md`, and `plugins/power-pages/PLUGIN_DEVELOPMENT_GUIDE.md` for the five UX/reliability pillars before submitting PRs.

For brownfield work, point any PRD or feature-plan workflow at `docs/index.md` as the AI retrieval entry point.
