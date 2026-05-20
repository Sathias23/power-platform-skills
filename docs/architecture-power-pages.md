# Architecture — `power-pages`

> Plugin v1.3.0 — the most feature-complete plugin in this repo and the reference implementation for every pattern documented in the root `AGENTS.md`.

## Executive summary

`power-pages` is a Claude Code / Copilot CLI plugin for **creating, deploying, and managing Power Pages code sites** (single-page applications built with React, Vue, Angular, or Astro). It is the only plugin in the repo with: a hook-based validation lifecycle, a `scripts/lib/` shared-helper module, sub-agents that use plan mode, and an MCP server combination (local Playwright + remote `microsoft-learn`).

**Server-rendered frameworks (Next.js, Nuxt, Remix, SvelteKit) are explicitly NOT supported.**

## Technology stack

| Category | Technology | Purpose |
|----------|-----------|---------|
| Authoring | Markdown SKILL.md + YAML frontmatter | All skill workflows |
| Execution | Node.js (built-in modules only) | Deterministic scripts and validators |
| Testing | `node --test` (zero deps) | Coverage for every script under `scripts/tests/` |
| MCP — local | Playwright (via `scripts/launch-playwright-mcp.js`) | Browser verification of deployed sites |
| MCP — remote (HTTP) | `microsoft-learn` (`https://learn.microsoft.com/api/mcp`) | Docs search/fetch grounded in official Microsoft sources |
| External CLI | `pac` (Power Platform CLI) | Site provisioning, deployment, model commands |
| External CLI | `az` (Azure CLI) | Dataverse AAD token minting; Key Vault operations |
| Hooks | Native Claude Code `PostToolUse` hook | Auto-validation after Skill completion |

**Downstream target stack** (what users' Power Pages sites are built with): React 18+, Vue 3, Angular 17+, or Astro. SPA only.

## Architecture pattern

**Phase-wise skill orchestration with deterministic execution.** LLMs compose intent through the SKILL.md workflow; Node.js scripts execute deterministic operations. Specialized agents (`agents/*.md`) handle complex sub-tasks via the `Task` tool with isolated contexts.

### The five pillars (from `PLUGIN_DEVELOPMENT_GUIDE.md`)

Every skill must satisfy all five:

1. **Engaging loading experience** — Task-based progress tracking via `TaskCreate` / `TaskUpdate`. All phase tasks created upfront at Phase 1.
2. **Action verification** — Dedicated standalone Verify phase. Uses a different code path than implementation (validators, globs, builds).
3. **Sub-agent architecture** — Complex multi-table operations delegate to specialist agents in isolated contexts.
4. **Deterministic execution** — File creation, YAML generation, UUID generation, Dataverse API calls all go through shared Node.js scripts.
5. **Transparency & user approval** — Three-point approval: after discovery, after plan, before deploy. Autonomous between checkpoints.

## Skills inventory (14 functional + report-issue)

| Skill | Purpose | Sub-agents used | Hook validator |
|-------|---------|------------------|----------------|
| `/create-site` | Scaffold a new Power Pages code site (8 phases) | (none) | (none) |
| `/setup-datamodel` | Create Dataverse tables/columns/relationships for a site | `data-model-architect` | (none) |
| `/add-sample-data` | Seed Dataverse tables with sample records | (none) | (none) |
| `/setup-auth` | Configure Entra ID login + protected routes | (none) | (none) |
| `/create-webroles` | Configure web roles for site users | (none) | Yes — web-roles validator |
| `/integrate-backend` | Router: chooses Web API / Server Logic / Cloud Flow | Delegates to sub-skills | (none) |
| `/integrate-webapi` | Implement Power Pages Web API + permissions for a table set | `webapi-integration`, `table-permissions-architect`, `webapi-settings-architect` | (none) |
| `/add-server-logic` | Create server-side JS endpoints running on Power Pages runtime | (none) | Yes — server-logic validator |
| `/add-cloud-flow` | Register/integrate Power Automate cloud flows | (none) | (none) |
| `/audit-permissions` | Audit existing table permissions for security issues | (none) | (none) |
| `/add-seo` | Add robots.txt, sitemap.xml, meta tags, favicon | (none) | (none) |
| `/deploy-site` | Build + upload + activate a code site via `pac` | (none) | (none) |
| `/activate-site` | Provision/activate a Power Pages site via REST API | (none) | (none) |
| `/test-site` | Runtime browser-based testing via Playwright | (none) | (none) |
| `/report-issue` | Bug reports — shared with all plugins | (none) | (none) |

## Sub-agents

| Agent | Mode | Tools | Role |
|-------|------|-------|------|
| `data-model-architect` | Plan mode (read-only) | Read, Bash, Glob, Grep, EnterPlanMode | Discovers existing Dataverse tables, analyzes requirements, proposes ER diagram. No table creation. |
| `table-permissions-architect` | Plan mode → create | Read, Write, Edit, Bash, Glob, Grep, EnterPlanMode | Analyzes site, proposes table permissions plan with HTML visualization, creates YAML on approval. |
| `webapi-settings-architect` | Plan mode → create | Read, Write, Edit, Bash, Glob, Grep, EnterPlanMode | Queries Dataverse for exact column logical names, proposes Web API site settings with validated names. |
| `webapi-integration` | Code-writing (per table) | Read, Write, Edit, Bash, Glob, Grep | Implements client + types + service + hooks for one Dataverse table. Sequential first to create shared client, then parallel for remaining tables. |

## Source tree

See `source-tree-analysis.md` for the annotated layout. Key directories:

```text
plugins/power-pages/
├── .mcp.json                  # playwright (local), microsoft-learn (HTTP)
├── hooks/hooks.json           # PostToolUse:Skill → validator dispatcher
├── agents/                    # 4 sub-agents (3 plan-mode, 1 code-writing)
├── references/                # 9 cross-skill reference docs
├── scripts/
│   ├── lib/                   # Shared helpers — grep here before writing new code
│   ├── tests/                 # node:test coverage (13 test files)
│   └── *.js                   # ~25 file-creation, API, render, validation scripts
└── skills/                    # 15 skills (14 functional + report-issue wrapper)
```

## Data architecture

The plugin itself does not own data — it manipulates Dataverse via the Web API.

### Site configuration artifacts (YAML)

| Artifact | Path inside a code-site project | Producer |
|----------|----------------------------------|----------|
| Table permissions | `.powerpages-site/table-permissions/*.yml` | `create-table-permission.js` (via `table-permissions-architect`) |
| Web API site settings | `.powerpages-site/site-settings/*.yml` | `create-site-setting.js` (via `webapi-settings-architect`) |
| Web roles | `.powerpages-site/web-roles/*.yml` | `/create-webroles` skill |
| Environment variables | `.powerpages-site/environment-variables/*.yml` | `create-environment-variable.js` |
| Skill tracking | site setting (records skill usage) | `update-skill-tracking.js` |

All artifacts use `__PLACEHOLDER__` token substitution during scaffolding. The `.gitignore` file ships as `gitignore` (no dot) and is renamed during scaffolding.

### Dataverse access pattern

```text
SKILL.md workflow → Node.js script
                  → import { getAuthToken, makeRequest } from scripts/lib/validation-helpers.js
                  → az login (user prerequisite)
                  → getAuthToken() mints Dataverse-scoped AAD token
                  → makeRequest() executes authenticated REST call
                  → token refresh every ~20 records / 3-4 tables / 60s during bulk ops
```

`dataverse-request.js` is the generic helper for ad-hoc REST. Higher-level scripts (`verify-dataverse-access.js`, `check-activation-status.js`, `list-custom-actions.js`) wrap specific endpoints.

## API design (Power Platform REST consumption)

The plugin consumes — does not expose — these surfaces:

| Service | API | Used by |
|---------|-----|---------|
| Dataverse Web API | `/api/data/v9.2/` | Table permissions, site settings, web roles, environment variables, custom actions |
| Power Platform admin API | (provider-specific) | Site activation status, cache clearing |
| Azure Key Vault | (data-plane and management) | `create-azure-keyvault.js`, `list-azure-keyvaults.js`, `store-keyvault-secret.js` |
| Power Pages REST API | `/api/v1/websites/...` | Site provisioning (`/activate-site` skill) |

## Component overview

This plugin produces no UI components of its own (it is a code authoring/deployment tool). The components it generates inside the user's code-site project depend on the framework chosen (React / Vue / Angular / Astro), and follow conventions documented in `references/framework-conventions.md`.

## Development workflow

See `docs/development-guide.md`. Plugin-specific notes:

- Tests: `node --test plugins/power-pages/scripts/tests/`
- Lint: (none — no formal linter)
- CI: `power-pages-script-tests.yml` runs on every PR touching this plugin
- Version check: `node scripts/ensure-skill-version-check.js --check` enforces the version-check line on every SKILL.md

## Deployment architecture (the plugin itself)

The plugin is distributed via the marketplace (`/plugin install power-pages@power-platform-skills`). Each user gets a copy in their Claude Code config; the marketplace auto-update mechanism pulls new commits from `main`.

The per-skill version-check (`scripts/check-version.js`) compares the local plugin version against `origin/main` at the start of every skill run.

## Testing strategy

| Layer | Tooling | Location | Coverage expectation |
|-------|---------|----------|----------------------|
| Unit (scripts) | `node --test` | `scripts/tests/*.test.js` | One `*.test.js` per source file; every new/modified script ships with tests in the same PR |
| Schema validators | Same as unit | `scripts/tests/validate-*.test.js` | Boundary cases tested verbatim against the SKILL.md constraint |
| Hook scripts | `node --test` | `scripts/tests/run-skill-posttool-validation.test.js` (when added) | Hooks gate debug behind `process.env.DEBUG`; only true errors go to stderr unconditionally |
| Eval (behavioural) | Eval framework | `evals/power-pages/<skill>/evals.json` | 11 skill-level eval suites today: activate-site, add-sample-data, add-seo, audit-permissions, create-site, create-webroles, deploy-site, integrate-webapi, setup-auth, setup-datamodel, test-site |
| Dataverse live tests | Opt-in only | Gated behind `--validate-dataverse-relationships` | Must NOT run by default; CI must not require Dataverse connectivity |

## Anti-patterns (called out repeatedly in the plugin's own AGENTS.md)

These have triggered repeat PR feedback:

1. **Phase cross-references break silently** — When renumbering phases, grep the whole skill dir AND `references/` for the old phase numbers.
2. **Validators must match the exact constraint** — Rule "no exports at all" → block both `module.exports` AND bare `exports.foo = …`. Rule "try/catch required" → assert BOTH exist. Test boundary cases.
3. **Hook scripts run on every Skill tool use** — Unconditional `process.stderr.write` becomes noise. Gate behind `process.env.DEBUG`.
4. **Template placeholders in `<script>` blocks need special care** — `render-template.js` injects values as-is (no encoding). Don't declare JS variables with `"__PLACEHOLDER__"`; read from DOM or use `JSON.stringify`.
5. **Contradictions within a single skill** — Re-read every SKILL.md end-to-end before submitting.

## Open risks / debt

- **`/integrate-backend` is a thin router skill** that delegates to `/integrate-webapi`, `/add-server-logic`, or `/add-cloud-flow`. Keep its decision tree synchronized with the underlying skills' phase counts.
- **`generate-canvas-app` parallel doesn't exist here** — but the `/generate-canvas-app` in `canvas-apps` is deprecated and redirects to `/canvas-app`. If any cross-plugin doc references it, update.
