---
title: Power Automate plugin — PRD addendum
created: 2026-05-21
updated: 2026-05-21
---

# Addendum — Power Automate plugin

Reference material drawn from the user's PROPOSAL.md and external research. Kept out of the PRD body to keep it lean; downstream documents (architecture, epics, design) consume this directly.

## A1 — Authoring surface: options considered and rejected

| # | Surface | Verdict | Rationale |
|---|---------|---------|-----------|
| 1 | **Dataverse Web API `/api/data/v9.2/workflows`** | **Chosen as primary authoring surface** | Microsoft's officially recommended programmatic path. Cloud flows are `workflow` rows with `category = 5`; the definition lives in a `clientdata` JSON string. |
| 2 | **`pac solution` (clone / unpack / pack / sync)** | **Chosen as packaging + deployment surface (post-v1)** | Standard ALM tooling. `pac solution clone` produces a YAML layout (PAC CLI ≥ 2.4.1) with a `workflows/` directory that fits source control. v1 uses in-place PATCH; multi-env deploy via `pac solution` is v2. |
| 3 | `api.flow.microsoft.com` | Rejected | Microsoft explicitly documents this as unsupported: "subject to change, breaking changes could occur." |
| 4 | Power Automate Management / Admin connectors (`flowmanagement`, `microsoftflowforadmins`) | Rejected as primary | Designed to be invoked from inside a flow (or the COE Starter Kit). Admin-shaped (list/disable/delete), not authoring-shaped (set a definition). Retain as a fallback for environment-wide admin operations. |
| 5 | First-party MCP server for flow authoring | Not available | Logic Apps Standard has a "create an MCP server *from* a workflow" feature, and Agent 365 has an MCP Management server, but neither is an authoring surface for Power Automate cloud flows. |

## A2 — `clientdata` shape

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

The `definition` block uses the Azure Logic Apps Workflow Definition Language: well-documented, schema-validatable, and the source-of-truth shape Power Automate uses internally.

## A3 — Agent architecture (planner + builder)

Mirrors the canvas-apps pattern (`canvas-app-planner` + `canvas-screen-builder`). User-visible behaviour appears in PRD §4–§5; the agent split is the implementation mechanism.

- **flow-planner** — Discovers the target flow, parses `clientdata`, maps the user's edit intent to a concrete plan of definition mutations (which actions, which expressions, which connection references). Produces a plan document the user approves before any modification.
- **flow-builder** — Loads the approved plan, applies mutations to a working copy of `clientdata`, runs schema validation, and hands the validated result to the deploy skill.

Sub-agents run sequentially per repo convention (no parallel spawns for skill sub-tasks). This is the same pattern documented in `_bmad-output/project-context.md` rule "Process sequentially, not in parallel; wait for each agent to complete, then present its output for user approval before continuing."

## A4 — Proposed plugin shape (post-v1 target)

v1 ships a subset; full target shown for trajectory.

```
plugins/power-automate/
├── .claude-plugin/
│   └── plugin.json
├── AGENTS.md
├── CLAUDE.md  (symlink → AGENTS.md)
├── README.md
├── agents/
│   ├── flow-planner.md       (v1)
│   └── flow-builder.md       (v1)
├── skills/
│   ├── cloud-flow/SKILL.md             (v1, EDIT only)
│   ├── configure-power-automate/SKILL.md (v1)
│   ├── add-connection-reference/SKILL.md (v2)
│   └── deploy-flow/SKILL.md            (v1, in-place PATCH only)
├── references/
│   ├── WorkflowDefinitionLanguage.md   (v1)
│   ├── ConnectorPatterns.md            (v1 — minimal: Outlook, Teams, Dataverse, HTTP)
│   ├── ExpressionLanguage.md           (v1)
│   ├── ClientDataSchema.md             (v1)
│   ├── TemplateCatalog.md              (v2 — clone-from-template)
│   └── SolutionPacking.md              (v2 — multi-env deploy)
└── scripts/
    └── lib/
        ├── pac-auth.{js,mjs}           (v1)
        ├── webapi-client.{js,mjs}      (v1)
        └── definition-validator.{js,mjs} (v1)
```

No `.mcp.json` — no MCP server to register.

## A5 — Competitive landscape (research findings)

Captured from external research run 2026-05-21. Justifies the wedge framing in PRD §2.

**Thesis:** no surveyed tool combines AI agent, clone-existing-flow, Web API authoring, and Microsoft-sanctioned surfaces. That's the defensible niche.

Adjacent tooling surveyed:

- **Microsoft Copilot in Power Automate.** Web designer chat; creates and edits flows. Constraints: English-only, subset of connectors, no run-history awareness, no live data validation, can't edit non-Open-API flows. Lives in the maker portal — no code-first surface.
- **Flow Studio MCP** (`mcp.flowstudio.app`). Third-party MCP server with one-click VS Code install for GitHub Copilot AND Claude Code. Creates, edits, deploys, debugs live flows (`update_live_flow`, `set_live_flow_state`, `add_live_flow_to_solution`). Direct competitor on greenfield create.
- **GitHub Copilot (generic).** IDE chat; answers questions, generates expression snippets — does not create flows.
- **PAC CLI.** Terminal; mechanical pack/unpack/export/import, no AI.

## A6 — Pain points evidence

Mixed Reddit / Microsoft Q&A / blog evidence, captured 2026-05-21:

- **Expression authoring friction** — "expression builder text field too small"; hand-written syntax with no autocomplete; multi-choice fields especially painful.
- **No real version control / diff** — official idea ticket exists (90335ee0-96f3-4f57-95f0-8f8c54a1d31c on `ideas.powerautomate.com`); Git integration only recently in preview.
- **Connection-reference fragility on environment promotion** — "flows using a connection reference without an associated connection will get inactivated during import"; best-practice guidance says strip them from deployed solutions.
- **No code-first editing surface** — the fact that "flows are stored as JSON on the backend" is treated as a workaround, not a feature.
- **Trigger-condition authoring** — "have to be coded manually with no expression builders."
- **Debugging trigger→action wiring** — Copilot explicitly "doesn't have access to your flow's run history or execution context."
- **Connector documentation friction** *(anecdotal — no single concrete source, but a recurring theme across surveyed posts)* — having to trial-and-error connector parameter shapes.

## A7 — NFR footguns (design constraints)

Background for PRD §6 (N3, N4, N5).

- **Schema churn.** v1→v2 desktop schema migration deprecated Oct 2024; March 2026 wave deprecates legacy controls/connectors. The cloud-flow `clientdata` definition shape is stable in practice but undocumented, and evolves silently between release waves.
- **Connection references break on import** when not pre-bound; needs deployment-settings JSON or post-import remap (relevant to v2 cross-env deploy).
- **State 0 default.** Newly *created* flows land in state 0 (off). v1 only edits existing flows, but verify-after-edit must confirm state was preserved.
- **Personal MSA accounts deprecated July 2025.** Auth path assumes work/school accounts only.
- **Non-Open-API (legacy) connectors.** Can't be touched by Copilot; likely same constraint applies to any agent editing `clientdata`. Flag clearly when encountered; don't attempt the edit.
- **Repack fragility.** "One misplaced character … Power Automate will refuse to import." Strict JSON validation pre-deploy is non-negotiable (PRD N3).
- **Run-history blind spot.** Authoring agent has no view of runtime failures. v1 explicitly out of scope; flagged for v2.

## A8 — References

- [Work with cloud flows using code (Dataverse Web API)](https://learn.microsoft.com/en-us/power-automate/manage-flows-with-code) — authoritative; explicitly deprecates `api.flow.microsoft.com`.
- [Process (Workflow) table reference](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/reference/entities/workflow) — required props (`category`, `name`, `type`, `primaryentity`, `clientdata`); state-0 default.
- [pac solution (unpack/pack/clone/sync)](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/solution) — YAML source-control format from PAC CLI 2.4.1+.
- [Azure Logic Apps Workflow Definition Language](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-workflow-definition-language) — schema for the `definition` block.
- [Power Automate Management connector](https://learn.microsoft.com/en-us/connectors/flowmanagement/) — admin fallback, not primary.
- [Create a cloud flow in a solution](https://learn.microsoft.com/en-us/power-automate/create-flow-solution) — solution-aware flow model.
- [Copilot in cloud flows FAQ](https://learn.microsoft.com/en-us/power-automate/faq-copilot-cloud-flows) — competitor capability boundaries.
- [Flow Studio MCP](https://mcp.flowstudio.app/) — direct AI-agent competitor.
- [Pre-populate connection references for automated deployments](https://learn.microsoft.com/en-us/power-platform/alm/conn-ref-env-variables-build-tools) — conn-ref import behaviour.
- [Important changes coming in Power Platform](https://learn.microsoft.com/en-us/power-platform/important-changes-coming) — schema-churn timeline.
- [Power Automate Standards: Connection References (Matthew Devaney)](https://www.matthewdevaney.com/power-automate-coding-standards-for-cloud-flows/power-automate-standards-connection-references/) — community best-practice corpus.
- [Editing Power Automate Export Packages (Edvaldo Guimaraes)](https://edvaldoguimaraes.com.br/2025/10/03/editing-power-automate-export-packages/) — community evidence of the unpack/edit/repack workflow.
- [Understanding the Power Automate Definition (DEV.to)](https://dev.to/wyattdave/understanding-the-power-automate-definition-42po) — `clientdata` shape walkthrough.
