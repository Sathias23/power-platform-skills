# Component Inventory

> Catalog of reusable building blocks in this repo: sub-agents, MCP tools, shared Node.js modules, reference docs, samples, and external CLIs. This is not a UI component library — the repo ships Markdown skills and Node.js scripts, not React/Vue components.

## Sub-agents (`agents/` directories)

Sub-agents are invoked by skills via the `Task` tool. They run with their own context, allowed-tools, and system prompt.

| Plugin | Agent | Mode | Allowed-tools | Purpose |
|--------|-------|------|---------------|---------|
| `power-pages` | `data-model-architect` | Plan mode (read-only) | Read, Bash, Glob, Grep, EnterPlanMode | Discover existing Dataverse tables, analyze requirements, propose ER diagram. No table creation. |
| `power-pages` | `table-permissions-architect` | Plan → create | Read, Write, Edit, Bash, Glob, Grep, EnterPlanMode | Propose table permissions plan with HTML visualization; create YAML on approval. |
| `power-pages` | `webapi-settings-architect` | Plan → create | Read, Grep, Glob, Bash, EnterPlanMode, ExitPlanMode | Query Dataverse for exact column names; propose validated Web API site settings. |
| `power-pages` | `webapi-integration` | Code-writing | Read, Write, Edit, Grep, Glob, Bash, mcp__microsoft-learn__* | Implement per-table client + types + service + hooks. Sequential first, then parallel. |
| `canvas-apps` | `canvas-app-planner` | Code-writing | Read, Write, TaskCreate, TaskUpdate, TaskList, mcp__canvas-authoring__* | Discover controls/APIs/data sources via MCP; write `App.pa.yaml` + plan document. |
| `canvas-apps` | `canvas-screen-builder` | Code-writing | Read, Write, Edit, TaskCreate, TaskUpdate | Create or modify one screen `.pa.yaml`. Runs in parallel with siblings. |
| `code-apps-preview` | `code-app-architect` | All tools | * | Architecture decisions, data model design, connector selection, build/deploy troubleshooting. |

## MCP tools (per server)

### `canvas-authoring` (canvas-apps)

Runs locally via `dnx` against the .NET 10 NuGet package.

| Tool | Purpose |
|------|---------|
| `configure` | Configure the MCP server for a specific coauthoring session (env ID, app ID, cluster category) |
| `compile_canvas` | Validate `.pa.yaml` files using the Power Apps authoring service |
| `describe_api` | Get detailed info about a specific API/connector |
| `describe_control` | Get detailed info about a specific Power Apps control |
| `get_data_source_schema` | Get columns + Power Fx types for a data source |
| `list_apis` | List all available APIs/connectors in the session |
| `list_controls` | List all available controls in the session |
| `list_data_sources` | List all available data sources in the session |
| `sync_canvas` | Sync session state from the server to a local directory |

### `microsoft-learn` (power-pages)

Remote HTTP MCP at `https://learn.microsoft.com/api/mcp`.

| Tool | Purpose |
|------|---------|
| `microsoft_docs_search` | Up to 10 concise content chunks (max 500 tokens each) — for breadth |
| `microsoft_code_sample_search` | Up to 20 official code samples — for practical examples |
| `microsoft_docs_fetch` | Full doc page → markdown — for depth |

### `playwright` (power-pages, model-apps)

Local node MCP launched via `scripts/launch-playwright-mcp.js` (each plugin has its own copy).

Tools exposed include `browser_navigate`, `browser_click`, `browser_type`, `browser_snapshot`, `browser_screenshot`, `browser_evaluate`, `browser_network_requests`, etc. — used for runtime verification of deployed Power Pages sites and model-driven generative pages.

## Shared Node.js modules

### `plugins/power-pages/scripts/lib/`

The most developed shared-helper module in the repo. Always grep here before writing new code.

| Module | Exports | Used by |
|--------|---------|---------|
| `validation-helpers.js` | `runValidation()`, `getAuthToken()`, `makeRequest()`, `approve()`, `block()`, `findPath()`, `findProjectRoot()` | Every validator and Dataverse-calling script |
| `powerpages-config.js` | Loaders for `.powerpages-site/*.yml` (table permissions, site settings, web roles) | Validators, audit scripts, integration scripts |
| `powerpages-hook-utils.js` | `getTrackedSkillFromToolInput()`, `getValidatorScript()` | Hook dispatcher only |
| `powerpages-schema-validator.js` | Schema validation for permission/setting YAML | Permissions/settings validators |
| `powerpages-validation-utils.js` | Common validation utilities | Other validators |
| `render-template.js` | `__PLACEHOLDER__` substitution | Site scaffolding |
| `site-settings-validator.js` | Site setting YAML validation | Hooks + `/integrate-webapi` |
| `table-permissions-validator.js` | Table permission YAML validation | Hooks + `/integrate-webapi` |
| `web-roles-validator.js` | Web role YAML validation | Hooks + `/create-webroles` |
| `detect-browser.js` | System browser detection for Playwright launcher | `launch-playwright-mcp.js` |

### `plugins/power-pages/scripts/` (file creation + APIs)

| Script | Purpose |
|--------|---------|
| `generate-uuid.js` | Centralized UUID generation — never duplicate |
| `create-table-permission.js` | Generate table permission YAML with UUIDs + field ordering |
| `create-site-setting.js` | Generate site setting YAML |
| `create-environment-variable.js` | Generate environment variable YAML |
| `update-skill-tracking.js` | Record skill usage in site settings |
| `dataverse-request.js` | Generic authenticated Dataverse REST helper |
| `verify-dataverse-access.js` | Verify Dataverse connectivity + permissions |
| `check-activation-status.js` | Query Power Platform admin API for activation |
| `clear-site-cache.js` | Clear site cache via admin API |
| `list-custom-actions.js` | List Dataverse custom actions |
| `create-azure-keyvault.js`, `list-azure-keyvaults.js`, `store-keyvault-secret.js` | Azure Key Vault integration |
| `render-createsite-plan.js`, `render-backend-plan.js`, `render-permissions-plan.js`, `render-cloudflow-plan.js`, `render-serverlogic-plan.js`, `render-audit-report.js`, `render-data-model-plan.js` | HTML plan visualizations for review checkpoints |
| `validate-permissions-schema.js` | Standalone permissions schema validator |
| `check-version.js` | Compare local plugin version to `origin/main` |
| `launch-playwright-mcp.js` | Playwright MCP launcher (uses `detect-browser.js`) |

### `plugins/model-apps/scripts/`

| Script | Purpose |
|--------|---------|
| `launch-playwright-mcp.js` | Same launcher pattern as power-pages |

### `plugins/code-apps/shared/`

Not Node.js modules — these are **plugin-root markdown shared docs** that the architect agent and skills reference at runtime:

| Doc | Purpose |
|-----|---------|
| `shared-instructions.md` | Cross-cutting (Windows CLI, safety, planning, memory bank) |
| `connector-reference.md` | Connector patterns + generated service usage |
| `development-standards.md` | Versioning, theme, build workflow, TS strict mode |
| `memory-bank.md` | Memory bank format + usage |
| `planning-policy.md` | Plan mode policy |
| `preferred-environment.md` | Environment selection priority |
| `version-check.md` | Version check instructions |

## Reference docs (per plugin)

These are linked from SKILL.md files via `${CLAUDE_PLUGIN_ROOT}/references/...`. Never duplicate their content into individual skills.

### `plugins/power-pages/references/`

| Doc | Purpose |
|-----|---------|
| `framework-conventions.md` | SPA framework-specific conventions (React / Vue / Angular / Astro) |
| `dataverse-prerequisites.md` | Dataverse setup prerequisites |
| `datamodel-manifest-schema.md` | Data model manifest schema |
| `permissions-plan-data-format.md` | Format spec for permissions plan data |
| `table-permission-analysis-guide.md` | Analysis guidance for table permissions |
| `webapi-core-client.md` | Core Web API client patterns |
| `webapi-service-patterns.md` | Service-layer patterns |
| `odata-common.md` | Common OData patterns |
| `skill-tracking-reference.md` | **CRITICAL** — skill name → tracking name mapping table |

### `plugins/model-apps/references/`

| Doc | Purpose |
|-----|---------|
| `genpage-rules-reference.md` | Code-gen rules, DataAPI types, layout patterns, common errors |
| `troubleshooting.md` | Common issues and fixes |

### `plugins/mcp-apps/references/`

| Doc | Purpose |
|-----|---------|
| `mcp-apps-reference.md` | MCP Apps API, Fluent UI components, CDN patterns |
| `design-guidelines.md` | Visual design defaults, theme tokens |

### `plugins/canvas-apps/references/`

| Doc | Purpose |
|-----|---------|
| `TechnicalGuide.md` | YAML syntax, control selection, layout strategies, Power Fx patterns |
| `DesignGuide.md` | Aesthetic guidelines, anti-patterns, design process |
| `QAChecks.md` | Runtime anti-pattern checks for self-QA |
| `PlanTemplates.md` | CREATE and EDIT plan document structures for `canvas-app-planner` |

### `plugins/code-apps/skills/*/references/`

Skill-local references in addition to plugin-root `shared/`:

| Skill | Reference |
|-------|-----------|
| `create-code-app` | `prerequisites-reference.md`, `troubleshooting.md` |
| `add-dataverse` | (Dataverse-specific references) |
| `add-sharepoint` | (SharePoint-specific references) |

## Samples (downstream code examples)

### `plugins/mcp-apps/samples/`

| Sample | Pattern demonstrated |
|--------|----------------------|
| `flight-status-widget.html` | Static data widget — receives JSON once at load |
| `weather-refresh-widget.html` | Interactive widget — uses `callServerTool` for live refresh |

### `plugins/model-apps/samples/`

Nine reference `.tsx` files demonstrating Fluent UI v9 patterns for model-driven generative pages:

1. `1-account-grid.tsx` — Tabular data display
2. `2-wizard-multi-step.tsx` — Multi-step wizard
3. `3-poa-revocation-wizard.tsx` — Real-world wizard variant
4. `4-account-crud-dataverse.tsx` — Full CRUD against Dataverse
5. `5-file-upload.tsx` — File upload flow
6. `6-navigation-sidebar.tsx` — Layout with sidebar
7. `7-comprehensive-form.tsx` — Complex form patterns
8. `8-responsive-cards.tsx` — Responsive card layout
9. `9-data-caching.tsx` — Client-side data caching

## External CLIs invoked by plugins

| CLI | Plugins | Use |
|-----|---------|-----|
| `pac` (Power Platform CLI) | All | Auth, model commands, deployment, solution operations |
| `az` (Azure CLI) | `power-pages` | Dataverse AAD token minting, Key Vault operations |
| `dotnet` / `dnx` | `canvas-apps` | Runs the Canvas Authoring MCP server (NuGet package) |
| `node` | All | Runs deterministic scripts and validators |
| `npx degit` | `code-apps` | Scaffold new code-app projects from `microsoft/PowerAppsCodeApps/templates/vite` |
| `npx power-apps` (`@microsoft/power-apps-cli`) | `code-apps` | Push, add-data-source, list-connections — installed per-app, not globally |

## Cross-plugin shared skills

Single source of truth under `shared/skills/`:

| Shared skill | Source-of-truth workflow | Per-plugin wrappers |
|--------------|--------------------------|----------------------|
| `report-issue` | `shared/skills/report-issue/report-issue-workflow.md` + `SKILL.template.md` | One thin SKILL.md per active plugin (5 wrappers today) |

## Hooks (power-pages only)

| Hook | Location | Purpose |
|------|----------|---------|
| `PostToolUse:Skill` | `plugins/power-pages/hooks/hooks.json` → `run-skill-posttool-validation.js` | Dispatches to per-skill validator scripts via `scripts/lib/powerpages-hook-utils.js` |

No other plugin registers hooks today.

## CI workflows (`.github/workflows/`)

| Workflow | Purpose |
|----------|---------|
| `ensure-skill-version-check.yml` | Run `node scripts/ensure-skill-version-check.js --check` to enforce the version-check line on every power-pages SKILL.md |
| `validate-plugin-names.yml` | Marketplace ↔ plugin.json name consistency |
| `power-pages-script-tests.yml` | `node --test plugins/power-pages/scripts/tests/` |
| `github-repo-stats.yml` | Non-blocking repo telemetry |
