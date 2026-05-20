# Integration Architecture

> How the 5 plugins in this monorepo connect — to each other, to AI agents, to MCP servers, and to Power Platform services.

## The big picture

`power-platform-skills` is a **plugin marketplace**, not a running application. Plugins don't make network calls to each other. What ties them together is:

1. The marketplace manifest (`.claude-plugin/marketplace.json`) that discovers them.
2. Shared conventions enforced by repo-level CI and root `AGENTS.md`.
3. Cross-plugin shared skills defined under `shared/skills/<name>/`.
4. A common deterministic-Node.js pattern for scripts and validators (most fully realized in `power-pages`).

```text
                     ┌──────────────────────────────────────────────┐
                     │ User invokes a slash-command / skill name    │
                     └────────────────────┬─────────────────────────┘
                                          │
                     ┌────────────────────▼─────────────────────────┐
                     │ AI agent host (Claude Code / Copilot CLI)    │
                     │  • Loads enabled plugins                     │
                     │  • Auto-discovers skills + agents            │
                     │  • Mounts MCP servers from .mcp.json         │
                     │  • Fires PostToolUse:Skill hooks             │
                     └────────────┬─────────────────┬───────────────┘
                                  │                 │
              ┌───────────────────┘                 └────────────────────┐
              │                                                          │
   ┌──────────▼──────────┐  Skill workflow (.md) drives the agent.       │
   │ plugins/<plugin>/   │  Agents invoked via Task tool.                │
   │ skills/<skill>/     │  Validators run as Node.js scripts.           │
   │ agents/*.md         │  Hooks dispatch via Skill matcher.            │
   │ scripts/*.js        │                                               │
   │ .mcp.json           ◄───┐                                           │
   └──────────┬──────────┘   │                                           │
              │              │                                           │
              │       ┌──────┴──────────────────────┐                    │
              │       │ MCP servers (per plugin)    │                    │
              │       │  • canvas-authoring (dnx)   │                    │
              │       │  • playwright (node)        │                    │
              │       │  • microsoft-learn (HTTP)   │                    │
              │       └─────────────────────────────┘                    │
              │                                                          │
              ▼                                                          ▼
   ┌────────────────────────┐                          ┌─────────────────────────┐
   │ External CLIs          │                          │ Power Platform services │
   │  • pac (universal)     ◄──────────────────────────┤  • Dataverse Web API    │
   │  • az (power-pages)    │                          │  • Power Pages REST API │
   │  • dotnet (canvas)     │                          │  • Power Apps Studio    │
   │  • node (universal)    │                          │  • Power Apps NPX CLI   │
   │  • npx (code-apps)     │                          └─────────────────────────┘
   └────────────────────────┘
```

## Integration points

### 1. Marketplace ↔ plugins

| Source | Target | Type | Details |
|--------|--------|------|---------|
| `.claude-plugin/marketplace.json` | `plugins/<plugin>/.claude-plugin/plugin.json` | Directory reference (`source: ./plugins/<plugin>`) | Marketplace lists active plugins by relative path. Names can differ from source dir (e.g., `code-apps-preview` → `plugins/code-apps`). |

### 2. Cross-plugin shared skills

| Source | Target | Type | Details |
|--------|--------|------|---------|
| `shared/skills/<skill>/<workflow>.md` | `plugins/<plugin>/skills/<skill>/SKILL.md` | Reference (thin wrapper) | The wrapper carries plugin-specific frontmatter + a one-line link to the workflow. The workflow is the single source of truth. |
| `shared/skills/<skill>/SKILL.template.md` | Multiple per-plugin wrappers | Templated generation | `{{PLUGIN_NAME}}` placeholder substituted per plugin. |

Currently: `report-issue` is the only cross-plugin shared skill, present in all 5 plugins.

### 3. Skill ↔ sub-agent (within a plugin)

Skills delegate specialized work to agents via the `Task` tool. Agents have their own context, allowed-tools, and system prompt.

| Plugin | Sub-agents | Invoked by |
|--------|-----------|-----------|
| `power-pages` | `data-model-architect`, `table-permissions-architect`, `webapi-integration`, `webapi-settings-architect` | `/integrate-webapi`, `/setup-datamodel`, `/integrate-backend`, etc. |
| `canvas-apps` | `canvas-app-planner`, `canvas-screen-builder` | `/canvas-app` (planner first, then screen-builders in PARALLEL) |
| `code-apps` | `code-app-architect` | `/create-code-app`, `/add-dataverse`, etc. |
| `model-apps` | (none — single-skill plugin) | N/A |
| `mcp-apps` | (none) | N/A |

**Orchestration patterns:**

- **Sequential** — `power-pages` skills spawn agents one at a time, present output for approval, then continue. The default.
- **Sequential-then-parallel** — `/integrate-webapi` (power-pages) processes the first table sequentially to create the shared `powerPagesApi.ts` client, then runs remaining tables in PARALLEL.
- **Parallel from the start** — `/canvas-app` (canvas-apps) invokes `canvas-app-planner` first, then runs `canvas-screen-builder` in PARALLEL across all screens.
- **Independent parallel** — `/integrate-webapi` Phase 6.3 runs `table-permissions-architect` and `webapi-settings-architect` in parallel (no shared state).

### 4. Skill ↔ MCP server

Each plugin's `.mcp.json` registers MCP servers that mount when the plugin is enabled. The skill workflow invokes MCP tools by name (`mcp__<server>__<tool>`).

| Plugin | MCP server | Transport | Tools used |
|--------|-----------|-----------|-----------|
| `power-pages` | `playwright` | Local node process (`scripts/launch-playwright-mcp.js`) | Browser-based site verification |
| `power-pages` | `microsoft-learn` | HTTP (`https://learn.microsoft.com/api/mcp`) | Docs search/fetch for Power Pages guidance |
| `model-apps` | `playwright` | Local node process | Browser verification of deployed pages |
| `canvas-apps` | `canvas-authoring` | `dnx Microsoft.PowerApps.CanvasAuthoring.McpServer` (.NET 10) | Authoring tools: `configure`, `list_controls`, `describe_control`, `list_apis`, `describe_api`, `list_data_sources`, `get_data_source_schema`, `compile_canvas`, `sync_canvas` |
| `code-apps` | (none) | — | Uses `npx power-apps` CLI directly |
| `mcp-apps` | (none) | — | Pure markdown skill — generates static HTML |

### 5. Skill ↔ deterministic Node.js scripts

LLMs compose intent; scripts execute deterministic operations. The most developed version of this pattern lives in `power-pages`:

| Layer | Examples | Purpose |
|-------|---------|---------|
| Shared helpers (`scripts/lib/`) | `validation-helpers.js` (`getAuthToken`, `makeRequest`, `approve`, `block`, `findProjectRoot`), `powerpages-config.js`, `powerpages-hook-utils.js`, `render-template.js` | Boilerplate for every script |
| File creation scripts | `create-table-permission.js`, `create-site-setting.js`, `create-environment-variable.js`, `generate-uuid.js` | YAML / config artifact generation |
| Dataverse / Azure API scripts | `dataverse-request.js`, `verify-dataverse-access.js`, `check-activation-status.js`, `clear-site-cache.js` | Authenticated REST calls |
| Validators | `validate-permissions-schema.js`, `lib/site-settings-validator.js`, `lib/table-permissions-validator.js`, `lib/web-roles-validator.js` | Schema + business-rule validation |
| Renderers | `render-createsite-plan.js`, `render-permissions-plan.js`, `render-audit-report.js`, etc. | HTML plan visualizations |

Other plugins use this pattern more lightly (or not at all — `mcp-apps` has no scripts).

### 6. Hook lifecycle (power-pages only)

`plugins/power-pages/hooks/hooks.json` registers a `PostToolUse` hook with matcher `Skill`. When a tracked skill completes:

```text
Skill completes → PostToolUse:Skill fires
              → run-skill-posttool-validation.js (dispatcher)
              → reads tool_input, maps skill name to validator script
                (via scripts/lib/powerpages-hook-utils.js)
              → spawns the validator (Node child process)
              → validator calls approve() or block(reason) from validation-helpers.js
              → exit code propagates back to the agent host
```

Hook scripts run on **every** Skill tool use, so they must be fast and silent (debug logging gated behind `process.env.DEBUG`).

Other plugins do not register hooks today.

### 7. Plugin ↔ Power Platform

All five plugins eventually call into Power Platform — but via different surfaces:

| Plugin | Authoring surface | Deployment surface | Auth |
|--------|--------------------|---------------------|------|
| `power-pages` | Dataverse Web API (`/api/data/v9.2/`) for table permissions, site settings, web roles; local code-site source files | `pac pages upload-code-site` | `az login` → `getAuthToken` mints Dataverse-scoped AAD token |
| `model-apps` | Single-file `.tsx` (React 17 + Fluent UI v9 + DataAPI) | `pac model genpage` | `pac auth` |
| `code-apps` (`code-apps-preview`) | React + Vite + TS source, `npx power-apps add-data-source` to generate typed services | `npx power-apps push` | `pac auth` (reused by `npx power-apps`) |
| `canvas-apps` | `.pa.yaml` files via Canvas Authoring MCP coauthoring session | `compile_canvas` + `sync_canvas` (server commits the session state) | Inherits browser coauthoring auth |
| `mcp-apps` | Standalone HTML | None — host loads the HTML directly via MCP App protocol | None |

### 8. Repo-level CI integration

`.github/workflows/` integrates with the plugin source as follows:

| Workflow | Triggers | What it runs |
|----------|----------|--------------|
| `ensure-skill-version-check.yml` | PRs touching power-pages SKILL.md | `node scripts/ensure-skill-version-check.js --check` |
| `validate-plugin-names.yml` | Any PR | Confirms `.claude-plugin/marketplace.json` ↔ `plugins/*/.claude-plugin/plugin.json` name consistency |
| `power-pages-script-tests.yml` | PRs touching `plugins/power-pages/` | `node --test plugins/power-pages/scripts/tests/` |
| `github-repo-stats.yml` | Schedule | Non-blocking repo telemetry |

## Shared assumptions across plugins

These conventions are repo-wide, enforced by `AGENTS.md` and (partly) CI:

- **Markdown-first authoring.** Workflows live in `.md` files. JS is for deterministic execution only.
- **`${CLAUDE_PLUGIN_ROOT}/...` for plugin-internal paths.** Never hardcode `plugins/<name>/...` from inside a SKILL.md.
- **Comma-separated `allowed-tools`** in frontmatter — never JSON array or YAML list.
- **Three-point user approval** in every skill (discovery, plan, deploy).
- **Phase-wise workflows** of 5–8 phases including a mandatory standalone Verify phase.
- **`TaskCreate` upfront** at Phase 1, one task per phase, marked in_progress / completed as phases run.
- **Shell-agnostic docs** — no PowerShell cmdlets, no shell-specific variable syntax in `bash` code blocks.

## What is NOT integrated

- The `power-automate` directory contains `PROPOSAL.md` only — it is **not** in `marketplace.json` and is not loaded by any agent host. The proposal documents a planned Dataverse Web API + `pac solution` authoring path, but no code exists yet.
- There is no shared cross-plugin script library at the repo root. Each plugin owns its own `scripts/` and `references/`. Only `shared/skills/` is the cross-plugin sharing point.
- There is no inter-plugin network protocol or shared state. Plugins coexist; they do not communicate at runtime.
