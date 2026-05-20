# Project Documentation Index — `power-platform-skills`

> Master entry point for AI-assisted development on this repo. Generated 2026-05-20 via `/bmad-document-project`.

## Project Overview

- **Type:** Monorepo (plugin marketplace) with 5 active parts + 1 proposal
- **Primary Language:** Markdown (workflow content) + JavaScript (deterministic scripts)
- **Architecture:** Per-plugin skill/agent/script bundles, surfaced via `.claude-plugin/marketplace.json`
- **Distribution:** Merged to `main` → auto-update via marketplace manifest

## Quick Reference (per part)

### power-pages (v1.3.0)

- **Type:** Plugin (library)
- **Tech Stack:** Markdown SKILL.md + Node.js scripts; `playwright` + `microsoft-learn` MCP
- **Root:** `plugins/power-pages`
- **Skills:** 15 (the reference implementation; all patterns present)
- **Downstream:** Power Pages code sites (React / Vue / Angular / Astro SPAs) deployed via `pac pages upload-code-site`

### model-apps (v1.0.6)

- **Type:** Plugin (library)
- **Tech Stack:** Markdown SKILL.md + Playwright MCP launcher
- **Root:** `plugins/model-apps`
- **Skills:** 2 (genpage + report-issue)
- **Downstream:** React 17 + Fluent UI v9 single-file genux pages deployed via `pac model genpage`

### mcp-apps (v1.0.0)

- **Type:** Plugin (library)
- **Tech Stack:** Markdown-only (no scripts, no MCP, no agents)
- **Root:** `plugins/mcp-apps`
- **Skills:** 2 (generate-mcp-app-ui + report-issue)
- **Downstream:** Self-contained HTML widgets using MCP Apps protocol (CDN dependencies)

### canvas-apps (v2.1.0)

- **Type:** Plugin (library)
- **Tech Stack:** Markdown SKILL.md + .NET 10 SDK + Canvas Authoring MCP server
- **Root:** `plugins/canvas-apps`
- **Skills:** 5 (canvas-app, configure-canvas-mcp, add-data-source, [deprecated] generate-canvas-app, report-issue)
- **Downstream:** Canvas Apps (`.pa.yaml`) authored via live coauthoring session

### code-apps-preview (v1.0.0)

- **Type:** Plugin (library) — marketplace name `code-apps-preview` ⚠️, source dir `plugins/code-apps`
- **Tech Stack:** Markdown SKILL.md + `npx power-apps` CLI orchestration
- **Root:** `plugins/code-apps`
- **Skills:** 14 (1 router + 8 connector add-skills + create/deploy/list + report-issue)
- **Downstream:** Power Apps code apps (React + Vite + TS) deployed via `npx power-apps push`

## Generated Documentation

- [Project Overview](./project-overview.md) — High-level snapshot of the repo
- [Source Tree Analysis](./source-tree-analysis.md) — Annotated directory layout
- [Integration Architecture](./integration-architecture.md) — How plugins, MCP servers, hooks, and external services connect
- [Development Guide](./development-guide.md) — Setup, tests, skill authoring quick reference
- [Deployment Guide](./deployment-guide.md) — Marketplace shipping + per-plugin downstream deployment
- [Contribution Guide](./contribution-guide.md) — PR checklist, conventions, common review pitfalls
- [Component Inventory](./component-inventory.md) — Sub-agents, MCP tools, shared modules, samples
- [Project Parts (JSON)](./project-parts.json) — Machine-readable plugin metadata + integration points

### Per-plugin architecture

- [architecture — power-pages](./architecture-power-pages.md)
- [architecture — model-apps](./architecture-model-apps.md)
- [architecture — mcp-apps](./architecture-mcp-apps.md)
- [architecture — canvas-apps](./architecture-canvas-apps.md)
- [architecture — code-apps-preview](./architecture-code-apps-preview.md)

## Existing Documentation (preserved, not regenerated)

### Repo-level

- [README](../README.md) — User-facing install + plugin index
- [AGENTS.md](../AGENTS.md) — Repo-level dev guidelines (concise; symlinked to `CLAUDE.md`)
- [CONTRIBUTING.md](../CONTRIBUTING.md) — Microsoft CLA / legal preamble
- [SECURITY.md](../SECURITY.md) — Security disclosure policy
- [SUPPORT.md](../SUPPORT.md), [CODE_OF_CONDUCT.md](../CODE_OF_CONDUCT.md), [LICENSE](../LICENSE), [CODEOWNERS](../CODEOWNERS)

### Repo-level AI context

- [`_bmad-output/project-context.md`](../_bmad-output/project-context.md) — **100 numbered project rules**, canonical agent context for this repo

### power-pages

- [README](../plugins/power-pages/README.md)
- [AGENTS.md](../plugins/power-pages/AGENTS.md) (+ symlinked CLAUDE.md)
- [PLUGIN_DEVELOPMENT_GUIDE.md](../plugins/power-pages/PLUGIN_DEVELOPMENT_GUIDE.md) — **Five UX & reliability pillars** (must-read for skill authors)
- `references/` — 9 cross-skill reference docs (see `component-inventory.md` for the full list)

### model-apps

- [README](../plugins/model-apps/README.md)
- [AGENTS.md](../plugins/model-apps/AGENTS.md) (+ symlinked CLAUDE.md)
- `references/genpage-rules-reference.md`, `references/troubleshooting.md`
- `samples/1-9-*.tsx` — 9 reference patterns

### mcp-apps

- [README](../plugins/mcp-apps/README.md) — Only authoring doc; no AGENTS.md
- `references/mcp-apps-reference.md`, `references/design-guidelines.md`
- `samples/{flight-status,weather-refresh}-widget.html`

### canvas-apps

- [README](../plugins/canvas-apps/README.md)
- [AGENTS.md](../plugins/canvas-apps/AGENTS.md) (+ symlinked CLAUDE.md)
- `references/{TechnicalGuide,DesignGuide,QAChecks,PlanTemplates}.md`

### code-apps-preview

- [README](../plugins/code-apps/README.md)
- [AGENTS.md](../plugins/code-apps/AGENTS.md)
- `shared/{shared-instructions,connector-reference,development-standards,memory-bank,planning-policy,preferred-environment,version-check}.md`

### power-automate (proposal only)

- [PROPOSAL.md](../plugins/power-automate/PROPOSAL.md) — Not registered in marketplace

## Getting Started

1. **Pick what you're working on.** If you're contributing repo-level guidance or CI: start with [AGENTS.md](../AGENTS.md) (root). If you're adding a skill or agent: read the plugin's own `AGENTS.md` plus `plugins/power-pages/PLUGIN_DEVELOPMENT_GUIDE.md`.
2. **Set up locally.** See [Development Guide](./development-guide.md). No `npm install` at repo root.
3. **Run tests.** `node --test plugins/power-pages/scripts/tests/` is the only test suite today.
4. **Develop a single plugin in isolation.** `claude --plugin-dir /path/to/plugins/<plugin-name>`.
5. **Before opening a PR.** Follow the checklist in [Contribution Guide](./contribution-guide.md).

### For brownfield PRD / feature planning

Point the PRD workflow at this `index.md`. The five per-plugin architecture docs give per-plugin context. For UI-only features pick the relevant downstream framework architecture (React via `model-apps` / `code-apps` / `power-pages`; Power Fx via `canvas-apps`; HTML via `mcp-apps`).

### For migration / refactor planning

Start with [Source Tree Analysis](./source-tree-analysis.md) and [Integration Architecture](./integration-architecture.md). The repo-level `_bmad-output/project-context.md` enumerates the conventions that must not silently break.

### When in doubt about a rule

Defer to the most restrictive option. Cross-check the repo-level `AGENTS.md`, the plugin's own `AGENTS.md`, and `_bmad-output/project-context.md`. If they conflict, the plugin's own file wins for plugin-local concerns; the repo-level file wins for cross-plugin concerns.
