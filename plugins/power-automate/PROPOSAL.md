# Power Automate plugin — design proposal

Status: draft / exploration. Not a working plugin yet. This document captures the research and the proposed shape so the actual `agents/`, `skills/`, and `references/` can be built against an agreed plan.

## Goal

Add a 6th plugin to the `power-platform-skills` marketplace that lets an AI coding agent author, edit, and deploy **Power Automate cloud flows** (automated, instant, and scheduled), with the same architectural conventions as the existing plugins (canvas-apps, code-apps, mcp-apps, model-apps, power-pages).

## Authoring surface — decision

Cloud flows have no first-class local source format the way Canvas (pa-yaml) or Pages do. We surveyed the realistic options and chose a hybrid.

### Options considered

| # | Surface | Verdict |
|---|---------|---------|
| 1 | `api.flow.microsoft.com` | **Rejected.** Microsoft explicitly documents this as unsupported: "subject to change, breaking changes could occur." |
| 2 | Power Automate Management / Admin connectors (`flowmanagement`, `microsoftflowforadmins`) | **Rejected as primary.** Designed to be invoked from inside a flow (or the COE Starter Kit). Admin-shaped (list/disable/delete), not authoring-shaped (set a definition). May still be useful as a fallback for environment-wide admin operations. |
| 3 | **Dataverse Web API `/api/data/v9.2/workflows`** | **Chosen as primary authoring surface.** Microsoft's officially recommended programmatic path. Cloud flows are `workflow` rows with `category = 5`; the definition lives in a `clientdata` JSON string. |
| 4 | **`pac solution` (clone / unpack / pack / sync)** | **Chosen as packaging + deployment surface.** Standard ALM tooling. `pac solution clone` produces a YAML layout (PAC CLI ≥ 2.4.1) with a `workflows/` directory that fits source control cleanly. |
| 5 | First-party MCP server for flow authoring | **Not available.** Logic Apps Standard has a "create an MCP server *from* a workflow" feature, and Agent 365 has an MCP Management server, but neither is an authoring surface for Power Automate cloud flows. |

### Chosen approach: Web API for authoring, `pac solution` for ALM

- **Read / create / update / delete flows** at dev time via the Dataverse Web API (`POST/PATCH/DELETE` on `workflows`). Language-agnostic, supported, low-friction.
- **Export, version-control, and deploy** flows via `pac solution clone` + `pac solution pack` + `ImportSolution`.
- **Auth:** shell out to `pac auth` and reuse its token for Web API calls. Avoids building a parallel MSAL flow and keeps the user's existing environment selection.

### Flow definition format

The `clientdata` field is a JSON string with two top-level properties:

```json
{
  "properties": {
    "connectionReferences": { "<logicalName>": { "api": { "name": "..." }, "connection": { ... } } },
    "definition": {
      "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
      "contentVersion": "1.0.0.0",
      "parameters": { "$connections": {...}, "$authentication": {...} },
      "triggers": { ... },
      "actions": { ... }
    }
  },
  "schemaVersion": "1.0.0.0"
}
```

The `definition` block is the **Azure Logic Apps Workflow Definition Language** — well-documented, schema-validatable, and the actual source-of-truth shape Power Automate uses internally.

## Authoring strategy: clone-first

New flows are not generated from scratch. The skill always:

1. Picks an existing template flow from the environment (or a seeded template per trigger family — manual, automated/Dataverse, scheduled, HTTP, etc.) and **clones** it via `pac solution clone` or by reading the workflow row.
2. Edits the cloned `clientdata` JSON — swaps trigger, replaces / inserts actions, rewires expressions, updates `connectionReferences`.
3. Pushes the modified definition back via Web API `PATCH` (for in-place edits) or via `pac solution pack` + `ImportSolution` (for deployment).

This mirrors how the existing Copilot Studio-style skills in this repo work, and avoids the recurring failure mode of generating invalid `operationMetadataId` GUIDs, wrong `apiId` paths, or malformed connection references from a blank slate.

## Proposed plugin shape

Mirrors `canvas-apps` conventions (planner + builder, references-heavy, lean scripts).

```
plugins/power-automate/
├── .claude-plugin/
│   └── plugin.json                       # name: power-automate, keywords, version
├── AGENTS.md                             # plugin-specific guidance (architecture, skills/agents tables)
├── CLAUDE.md                             # symlink → AGENTS.md
├── README.md                             # user-facing marketing + skill reference
├── agents/
│   ├── flow-planner.md                   # Picks template, plans triggers/actions/connections, writes plan.md + connection map
│   └── flow-builder.md                   # Clones template clientdata, edits definition JSON, validates schema
├── skills/
│   ├── cloud-flow/SKILL.md               # Unified CREATE/EDIT user-invocable skill; orchestrates planner + builder + deploy
│   ├── configure-power-automate/SKILL.md # pac auth, environment pick, Web API access verification
│   ├── add-connection-reference/SKILL.md # Internal; user creates connection in maker portal, skill verifies via Web API
│   └── deploy-flow/SKILL.md              # PATCH workflow via Web API, or pack + ImportSolution
├── references/
│   ├── WorkflowDefinitionLanguage.md     # $schema, triggers, actions, expressions, runAfter, scopes
│   ├── ConnectorPatterns.md              # Dataverse, Outlook, Teams, HTTP, SharePoint: apiId / operationId / common params
│   ├── ExpressionLanguage.md             # @triggerOutputs(), @body(), @items(), conditions, functions
│   ├── ClientDataSchema.md               # Shape of workflow.clientdata; connectionReferences contract
│   ├── TemplateCatalog.md                # Seeded template flows per trigger family used for clone-first
│   └── SolutionPacking.md                # pac solution clone/pack workflow for source control + deployment
└── scripts/
    └── lib/
        ├── pac-auth.{js,mjs}             # Shell out to pac, read token, env URI
        ├── webapi-client.{js,mjs}        # GET/POST/PATCH/DELETE workflows; OData helpers
        └── definition-validator.{js,mjs} # JSON schema validation against workflowdefinition.json#
```

No `.mcp.json` — no MCP server to register.

## Agent responsibilities (sketch)

- **flow-planner** (analogous to `canvas-app-planner`)
  - Discovers existing flows in the environment to use as templates.
  - Resolves required connectors and their `apiId` / `operationId` values.
  - Maps `connectionReferences` the user already has vs. ones they need to create.
  - Writes a `flow-plan.md` listing trigger family, chosen template, action sequence, and expressions.
- **flow-builder** (analogous to `canvas-screen-builder`)
  - Loads the chosen template's `clientdata`.
  - Applies the plan: swap trigger inputs, insert/remove actions, rewire `runAfter`, substitute expressions.
  - Validates against the workflow definition schema before handing off to deploy.

## Skills (sketch)

- `/cloud-flow` — unified create / edit. CREATE: pick template family → planner → builder → deploy. EDIT: load existing flow by ID or name → planner produces diff → builder applies → deploy.
- `/configure-power-automate` — `pac auth create`, environment selection, verify Web API access by listing workflows.
- `add-connection-reference` (internal, not user-invocable) — guides the user to create a needed connection in the maker portal, polls via Web API until visible, returns the `connectionReferenceLogicalName` for the builder to wire in.
- `deploy-flow` (internal) — either `PATCH /workflows(<id>)` with new `clientdata` (in-place), or `pac solution pack` + `ImportSolution` (cross-environment).

## Open questions

None blocking for the initial scaffold. Things to revisit once the skeleton is in place:

1. **Template catalog seeding** — do we ship a set of starter template flow JSONs in `references/templates/`, or always clone from whatever exists in the user's environment? Probably both: ship a minimal seed set, prefer environment templates when available.
2. **Schema validation depth** — JSON Schema validation against `workflowdefinition.json#` catches structural errors but not semantic ones (e.g., unknown `operationId` for a connector). A "describe API" lookup like the Canvas Authoring MCP server provides would help; without it, we rely on Web API rejection at deploy time.
3. **`pac` CLI availability** — canvas-apps requires .NET 10 SDK for its MCP server. We'd require `pac` CLI (Microsoft.PowerApps.CLI). The `configure-power-automate` skill should detect and prompt to install if missing.

## References

- [Work with cloud flows using code (Dataverse Web API)](https://learn.microsoft.com/power-automate/manage-flows-with-code) — authoritative; explicitly deprecates `api.flow.microsoft.com`.
- [pac solution (unpack/pack/clone/sync)](https://learn.microsoft.com/power-platform/developer/cli/reference/solution) — YAML source-control format from PAC CLI 2.4.1+.
- [Azure Logic Apps Workflow Definition Language](https://learn.microsoft.com/azure/logic-apps/logic-apps-workflow-definition-language) — schema for the `definition` block inside `clientdata`.
- [Power Automate Management connector](https://learn.microsoft.com/connectors/flowmanagement/) — admin fallback, not primary.
- [Create a cloud flow in a solution](https://learn.microsoft.com/power-automate/create-flow-solution) — solution-aware flow model that `pac solution clone` operates on.
