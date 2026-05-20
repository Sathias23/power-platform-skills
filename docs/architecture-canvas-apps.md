# Architecture — `canvas-apps`

> Plugin v2.1.0 — authors Power Apps Canvas Apps via a live MCP-driven coauthoring session.

## Executive summary

`canvas-apps` is the only plugin that **requires an external MCP server runtime authored by Microsoft and shipped as a .NET 10 NuGet package** (`Microsoft.PowerApps.CanvasAuthoring.McpServer`). The MCP server runs locally via `dnx` and connects to a *live Power Apps Studio coauthoring session* in the browser. Skills coordinate with the server to discover controls, APIs, and data sources, then generate or modify `.pa.yaml` files that the server validates and syncs back to the cloud.

**Critical runtime constraint:** the Power Apps Studio browser tab must remain open throughout the session. Closing it ends the coauthoring session, breaking `compile_canvas` and `sync_canvas` operations.

## Technology stack

| Category | Technology | Purpose |
|----------|-----------|---------|
| Authoring | Markdown SKILL.md + YAML frontmatter | Skill workflow |
| Execution | (no Node.js scripts in this plugin) | All execution happens via MCP tools |
| MCP server | `Microsoft.PowerApps.CanvasAuthoring.McpServer` via `dnx` | Authoring, validation, sync |
| Runtime prerequisite | **.NET 10 SDK** | Required to start the MCP server |
| Source format | PA YAML (`.pa.yaml`) | Canvas App representation |
| Downstream language | Power Fx | Formula language used inside `.pa.yaml` properties |

## Architecture pattern

**MCP-driven coauthoring with planner-then-parallel screen builders.** A unified `/canvas-app` skill auto-detects whether the user wants to CREATE (new app) or EDIT (existing app), runs a planner agent to draft the structure, then fans out parallel screen-builder agents to generate or modify individual screens. The MCP server validates each YAML file via `compile_canvas` and syncs the final state via `sync_canvas`.

## Skills inventory

| Skill | Purpose |
|-------|---------|
| `/canvas-app` | Create or edit a Canvas App — auto-detects mode |
| `/configure-canvas-mcp` | Register the Canvas Authoring MCP server with Claude Code (initial setup) |
| `/add-data-source` | Guide the user to add a data source / connection / API connector in Studio, then verify |
| `/generate-canvas-app` | **[DEPRECATED]** Redirects to `/canvas-app` |
| `/report-issue` | Bug report (shared cross-plugin skill) |

## Sub-agents

Agents are invoked by skills via the `Task` tool — they are not user-invocable.

| Agent | Invoked by | Role |
|-------|-----------|------|
| `canvas-app-planner` | `/canvas-app` | After the skill collects the approved plan: discovers controls (`list_controls`, `describe_control`), APIs (`list_apis`, `describe_api`), and data sources (`list_data_sources`, `get_data_source_schema`); writes `App.pa.yaml` (CREATE mode); writes `canvas-app-plan.md` for the screen-builders to consume. |
| `canvas-screen-builder` | `/canvas-app` (in PARALLEL across screens) | For CREATE: writes the YAML for one new screen based on the plan. For EDIT: applies targeted edits to one existing screen. |

**Orchestration:** Planner first (sequential), then all screen-builders in parallel. Final validation runs in the skill (not the agents) via `compile_canvas`.

## Source tree

```text
plugins/canvas-apps/
├── .claude-plugin/plugin.json
├── .mcp.json                          # canvas-authoring (dnx + .NET 10 nuget pkg)
├── AGENTS.md, CLAUDE.md (symlink), README.md
├── references/
│   ├── TechnicalGuide.md              # YAML syntax, control selection, Power Fx patterns
│   ├── DesignGuide.md                 # Aesthetic guidelines, anti-patterns
│   ├── QAChecks.md                    # Runtime anti-pattern checks for self-QA
│   └── PlanTemplates.md               # CREATE / EDIT plan document structures
├── agents/
│   ├── canvas-app-planner.md
│   └── canvas-screen-builder.md
└── skills/
    ├── canvas-app/SKILL.md
    ├── configure-canvas-mcp/SKILL.md
    ├── add-data-source/SKILL.md
    ├── generate-canvas-app/SKILL.md   # DEPRECATED — redirect to canvas-app
    └── report-issue/SKILL.md
```

## MCP server: `canvas-authoring`

Registered via `.mcp.json`:

```json
{
  "canvas-authoring": {
    "command": "dnx",
    "args": ["Microsoft.PowerApps.CanvasAuthoring.McpServer", "--yes", "--prerelease", "--source", "https://api.nuget.org/v3/index.json"]
  }
}
```

The `--prerelease` flag indicates the server is currently shipped as a prerelease NuGet package.

### MCP tools exposed

| Tool | Purpose |
|------|---------|
| `configure` | Configure the MCP server for a specific coauthoring session (env ID, app ID, cluster category) |
| `compile_canvas` | Validate canvas app YAML files in a directory using the Power Apps authoring service |
| `describe_api` | Get detailed info about a specific API/connector (operations, parameters) |
| `describe_control` | Get detailed info about a specific Power Apps control (properties, variants, metadata) |
| `get_data_source_schema` | Get the schema (columns + Power Fx types) for a data source in the session |
| `list_apis` | List all available APIs/connectors in the current authoring session |
| `list_controls` | List all available Power Apps controls in the current authoring session |
| `list_data_sources` | List all available data sources in the current authoring session |
| `sync_canvas` | Sync the current coauthoring session state from the server to a local directory |

## Data architecture

No persistent data owned by the plugin. The Canvas App's data sources are configured by the user in Studio (via `/add-data-source` guidance) and discovered by the planner via `list_data_sources` + `get_data_source_schema`. Power Fx formulas reference these sources directly in the generated `.pa.yaml`.

## Source format: `.pa.yaml`

Each Canvas App project has:
- `Src/App.pa.yaml` — app-level definition (OnStart, variables, navigation)
- `Src/<Screen>.pa.yaml` — one file per screen
- Property values are Power Fx formulas (single-line `=Formula` or multi-line block)

`TechnicalGuide.md` documents the YAML syntax in detail; `PlanTemplates.md` documents the CREATE/EDIT plan document structure consumed by `canvas-app-planner`.

## API design

The plugin exclusively communicates with the MCP server. No direct REST calls. No `pac` CLI usage (the MCP server handles auth + sync).

## Component overview (controls, not React)

"Components" here means **Power Apps controls** (Button, Gallery, Form, etc.), not React components. The planner agent enumerates them via `list_controls` and gets their property schemas via `describe_control` before writing YAML.

## Development workflow

```bash
# 1. Install .NET 10 SDK (one-time)
# https://dotnet.microsoft.com/download/dotnet/10.0

# 2. Local plugin development
claude --plugin-dir /path/to/plugins/canvas-apps

# 3. Set up the MCP server (first session only)
/configure-canvas-mcp

# 4. Open Power Apps Studio in a browser → start or open a Canvas App
# 5. Run the unified skill
/canvas-app
```

No tests. No CI workflow specific to this plugin.

## Deployment

There is no separate deployment step — the Canvas App is committed to the cloud as soon as `sync_canvas` succeeds against the live coauthoring session. The plugin itself is distributed via the marketplace (`/plugin install canvas-apps@power-platform-skills`).

## Testing strategy

| Layer | Tooling | Location |
|-------|---------|----------|
| Unit | (none — no scripts) | — |
| Eval | (none today) | — |
| Manual | Live Studio coauthoring session | After each `sync_canvas`, the user verifies behavior in Studio |
| Pre-commit validation | `compile_canvas` MCP tool | Run by the skill before `sync_canvas` |

## Key constraints

| Constraint | Impact |
|------------|--------|
| Power Apps Studio tab must stay open | Closing the tab kills the coauthoring session — `compile_canvas` and `sync_canvas` fail until reconnected |
| Agents run in parallel | `canvas-screen-builder` instances are siblings — they must not depend on each other's outputs |
| Planner runs first | Screen-builders depend on `App.pa.yaml` + `canvas-app-plan.md` written by the planner |
| YAML syntax errors fail validation | `compile_canvas` catches them; the skill must run validation before sync |

## Open risks / debt

- **MCP server is prerelease** (`--prerelease` flag). Breaking changes possible. Pin the version or test on each NuGet update.
- **No automated testing of skills.** Behavioural correctness is validated manually via Studio.
- **DEPRECATED `/generate-canvas-app` still exists.** Verify any cross-plugin docs (or external user docs) don't still reference it; the redirect message handles end-user invocation.
- **`.NET 10 SDK` dependency** is uncommon for this monorepo. Users on Windows / macOS / Linux must install it before any canvas-apps skill works. `/configure-canvas-mcp` should detect and guide.
