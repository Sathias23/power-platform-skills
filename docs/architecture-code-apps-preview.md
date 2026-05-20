# Architecture — `code-apps-preview` (source dir: `plugins/code-apps`)

> Plugin v1.0.0 — builds and deploys Power Apps code apps (React + Vite + TypeScript) connected to Power Platform via connectors. **Marketplace name differs from source directory: `code-apps-preview` ↔ `plugins/code-apps`. Preserve this mapping.**

## Executive summary

`code-apps-preview` is the connector-centric plugin. Apps run in a sandboxed Power Apps runtime where **direct HTTP calls (`fetch`, `axios`, Graph API) do not work** — all external data access must go through Power Platform connectors. The plugin's many `add-*` skills wrap the official Power Apps NPX CLI (`npx power-apps add-data-source`) to register connectors and generate typed TypeScript services.

## Technology stack

| Category | Technology | Purpose |
|----------|-----------|---------|
| Authoring | Markdown SKILL.md + YAML frontmatter | Skill workflows |
| Execution | (no Node.js scripts in this plugin) | All execution shells out to `npx power-apps` |
| External CLI | `npx power-apps` (`@microsoft/power-apps-cli`) | Scaffolding, connector registration, build, deploy |
| External CLI | `pac` (Power Platform CLI) | Auth (reused by `npx power-apps`) |
| External CLI | `npx degit` | Scaffolding new projects from `microsoft/PowerAppsCodeApps/templates/vite` |
| Downstream stack | React 18, Vite, TypeScript (strict mode), Fluent UI v9, theme tokens | Generated code-app projects |

**No `scripts/lib/`, no node tests in the plugin itself, no hooks.** The plugin is a pure orchestrator of the official Power Apps CLI.

## Architecture pattern

**Router skill + per-connector skills.** `/add-datasource` is a router that asks the user what they need and routes to a specific `add-*` skill. Each specific skill calls `npx power-apps add-data-source -a <connector>` with the right arguments and runs the architect agent for any code integration work. `/create-code-app` is the bootstrap skill (scaffold + init + first deploy).

## Skills inventory (13 functional + report-issue)

| Skill | Purpose |
|-------|---------|
| `/create-code-app` | Scaffold (via `npx degit`), configure, and deploy a new code app |
| `/deploy` | Build + deploy an existing code app via `npx power-apps push` |
| `/list-connections` | List Power Platform connections to find connection IDs |
| `/add-datasource` | Router — picks the right `add-*` skill based on user intent |
| `/add-dataverse` | Add Dataverse tables (creates them if needed) with generated TS models + services |
| `/add-sharepoint` | Add SharePoint Online connector (can also create lists) |
| `/add-teams` | Add Microsoft Teams connector |
| `/add-excel` | Add Excel Online (Business) connector |
| `/add-azuredevops` | Add Azure DevOps connector |
| `/add-onedrive` | Add OneDrive for Business connector |
| `/add-office365` | Add Office 365 Outlook connector |
| `/add-mcscopilot` | Add Microsoft Copilot Studio connector |
| `/add-connector` | Generic fallback for any other Power Platform connector |
| `/report-issue` | Bug report (shared cross-plugin skill) |

## Sub-agents

| Agent | Invoked by | Role |
|-------|-----------|------|
| `code-app-architect` | Most skills | Architecture decisions, data model design, connector selection, build/deploy troubleshooting |

## Source tree

```text
plugins/code-apps/                       # ⚠ Marketplace name: "code-apps-preview"
├── .claude-plugin/plugin.json           # name: "code-apps-preview"
├── AGENTS.md, README.md                 # No CLAUDE.md symlink in this plugin
├── agents/
│   └── code-app-architect.md
├── shared/                              # ⚠ NOT scripts/lib — this plugin uses plugin-root shared/
│   ├── shared-instructions.md           # Cross-cutting (Windows CLI, safety, planning, memory bank)
│   ├── connector-reference.md           # Connector patterns + generated service usage
│   ├── development-standards.md         # Versioning, theme, build workflow, TS strict mode
│   ├── memory-bank.md                   # Memory bank format and usage
│   ├── planning-policy.md               # Plan mode policy
│   ├── preferred-environment.md         # Environment selection priority
│   └── version-check.md                 # Version check instructions
└── skills/
    ├── create-code-app/
    │   ├── SKILL.md
    │   └── references/                  # Skill-specific (in addition to plugin-root shared/)
    │       ├── prerequisites-reference.md
    │       └── troubleshooting.md
    ├── deploy/SKILL.md
    ├── list-connections/SKILL.md
    ├── add-datasource/SKILL.md          # Router
    ├── add-connector/SKILL.md           # Generic fallback
    ├── add-dataverse/
    │   ├── SKILL.md
    │   └── references/                  # Dataverse-specific reference docs
    ├── add-sharepoint/
    │   ├── SKILL.md
    │   └── references/                  # SharePoint-specific reference docs
    ├── add-teams/SKILL.md
    ├── add-excel/SKILL.md
    ├── add-azuredevops/SKILL.md
    ├── add-onedrive/SKILL.md
    ├── add-office365/SKILL.md
    ├── add-mcscopilot/SKILL.md
    └── report-issue/SKILL.md
```

## Key concepts

### Connector-first principle

Power Apps code apps run in a sandboxed runtime. Direct HTTP calls do not work. **All external data access must go through a Power Platform connector.** This is enforced by the runtime; the skills enforce it at authoring time by always routing data fetches through generated services.

### Generated services

`npx power-apps add-data-source` produces typed TypeScript artifacts:

```
src/generated/models/{Name}Model.ts       # TypeScript interfaces for entities
src/generated/services/{Name}Service.ts   # CRUD methods (typed, awaitable)
```

The skill always uses these generated services — never `fetch`, never `axios`.

### NPX CLI

The Power Apps CLI ships as `@microsoft/power-apps-cli` and is installed locally via `npm install` as part of the app template. All commands run from inside the project directory via `npx power-apps <verb>`:

```bash
npx power-apps push                              # Deploy app
npx power-apps add-data-source -a <connector>    # Register connector + generate services
npx power-apps list-connections                  # List existing connections
```

**No PowerShell wrapper needed** — runs natively on all platforms in bash.

### Scaffolding via `degit`

Always use `npx degit` to scaffold new projects — **never `git clone`** and never manual file creation. This avoids dragging git history into the user's project.

```bash
npx degit microsoft/PowerAppsCodeApps/templates/vite <folder> --force
```

### Memory bank

`shared/memory-bank.md` defines a memory bank pattern unique to this plugin — used by `code-app-architect` to retain context across skill invocations within the same code-app project. Stored in the user's project directory, not the plugin.

## Data architecture

No persistent data owned by the plugin. Data is accessed at runtime via generated connector services. The plugin's `add-dataverse` skill can also **create** Dataverse tables (via `pac`), generating both the table and the TS models in one flow.

## API design

The plugin orchestrates these external surfaces:

| Surface | Purpose |
|---------|---------|
| `npx power-apps` | Connector registration, build, deploy |
| `pac` | Authentication, Dataverse table creation |
| Power Platform connector runtime | Data access at app runtime |

## Component overview

UI components inside the generated code app are based on **Fluent UI v9 + theme tokens**, per `shared/development-standards.md`. The plugin does not ship any reusable components — it generates them as needed via the architect agent.

## Development standards (from `shared/development-standards.md`)

- **TypeScript strict mode** — `strict: true` in `tsconfig.json`
- **Versioning** — Per-app `package.json` version, bumped via the deploy skill
- **Theme** — Fluent UI v9 theme tokens; no hardcoded colors
- **Build workflow** — Vite for dev + build; `npx power-apps push` for deploy

## Development workflow

```bash
# Local plugin development
claude --plugin-dir /path/to/plugins/code-apps

# Skill invocation
/create-code-app    # Bootstrap a new app
/add-dataverse      # Add a Dataverse connector
/add-sharepoint     # Add SharePoint
/deploy             # Build + deploy
```

No tests. No CI workflow specific to this plugin.

## Deployment

Each user code app is deployed via `npx power-apps push`. The plugin itself is distributed via the marketplace (`/plugin install code-apps-preview@power-platform-skills` — **note the `-preview` suffix in the marketplace name**).

## Testing strategy

| Layer | Tooling | Location |
|-------|---------|----------|
| Unit | (none — no scripts) | — |
| Eval | (none today) | — |
| Manual | Plug into a real Power Platform environment | After each deploy, validate connector behavior at runtime |

## Plugin naming gotcha

- Marketplace name: **`code-apps-preview`**
- Source directory: **`plugins/code-apps`**
- Plugin manifest `name` field: **`code-apps-preview`**

When editing the marketplace or per-plugin manifest, preserve this mapping. CI (`validate-plugin-names.yml`) enforces consistency.

## Open risks / debt

- **No script tests because there are no scripts** — but the SKILL.md files themselves are not unit-testable. Eval coverage would be valuable.
- **No CI workflow gates this plugin** beyond marketplace name validation. PRs that touch `add-*` skills are reviewed manually.
- **Plugin is in `preview`** per the marketplace name. Treat semver loosely until removed.
- **The `shared/` directory at the plugin root is a different pattern** than `power-pages` uses (`scripts/lib/`). Don't confuse the two; the convention is plugin-local.
