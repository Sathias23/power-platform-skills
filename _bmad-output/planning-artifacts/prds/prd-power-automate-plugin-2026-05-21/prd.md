---
title: Power Automate plugin — PRD
status: final
created: 2026-05-21
updated: 2026-05-21
audience: internal-data3
version_target: v1
---

# Power Automate plugin — PRD

> v1 target: an internal Data3 plugin that lets a developer using an AI coding agent (Claude Code, GitHub Copilot CLI) **clone, edit, and deploy** an existing Power Automate cloud flow without leaving the terminal. Create-from-scratch is explicitly out of v1.

## 1. Context

The `power-platform-skills` marketplace already ships 5 plugins covering Canvas Apps, Code Apps, Model Apps, MCP Apps, and Power Pages. Power Automate is the gap.

Power Automate has no first-class local source format the way Canvas Apps has `.pa.yaml` or Power Pages has its site-as-code layout. Flow definitions live as a JSON string (`clientdata`) inside a Dataverse row. That single fact — no local-file authoring surface — is the constraint that shapes every design decision in this PRD.

## 2. Why this plugin exists

Today, when a Data3 developer needs to change a production cloud flow, the only blessed authoring surface is the Power Automate maker portal — a web designer with no diff, no version control, no edit-in-context-of-your-codebase story. Data3 developers regularly need to edit production flows — rewire actions, swap triggers, change expressions, update connector parameters — and the current cost is a recurring drag on delivery.

The existing `power-pages/add-cloud-flow` skill **registers** flows into a Power Pages site (metadata + client wiring). It does **not** author them. That gap — code-first authoring of the flow definition itself — is what this plugin fills.

Two adjacent tools cover overlapping ground but not the wedge:

- **Microsoft Copilot in Power Automate** — natural-language flow authoring **inside the maker portal**. Web-only, English-only, subset of connectors, no run-history awareness, no code-first surface.
- **Flow Studio MCP** — third-party MCP server with one-click Claude Code install. Creates and edits live flows **from scratch via natural language**.

Neither addresses what this plugin owns:

> **Edit an existing production flow from your coding agent, in-place, using only Microsoft-sanctioned surfaces (Dataverse Web API + `pac solution`).**

The defense rests on two pillars:
1. **Clone-existing, not generate-from-scratch.** Avoids the recurring failure mode of generating invalid `operationMetadataId` GUIDs, wrong `apiId` paths, or malformed connection references from a blank slate.
2. **Microsoft-sanctioned surfaces only.** `api.flow.microsoft.com` is unsupported by Microsoft's own docs; Web API + `pac solution` are the documented programmatic paths. This survives API churn better than competitors.

**Wedge fragility.** This is today's wedge, not a permanent moat. Flow Studio MCP could add a `clone_existing_flow` capability and a "use Web API" flag in a week and close the capability gap. The durable defenses are structural rather than capability-based:
- **Marketplace position.** This plugin lives in Microsoft's sponsored `power-platform-skills` open-source marketplace alongside 5 sibling plugins.
- **Sibling-plugin integration.** Hand-off with `power-pages/add-cloud-flow` supports end-to-end Power Pages + flow workflows that a standalone MCP server cannot easily match.
- **ALM-first trajectory.** A repo-aligned source-control story (flow definitions as files on disk via `pac solution clone`, the posture canvas-apps takes with `.pa.yaml`) is the natural v2 evolution and compounds advantage over time. v1 does not deliver it (see §8).

v1 ships on the wedge; the durable position is the v2 trajectory.

## 3. Users & personas

**v1 audience: Data3 internal developers building solutions on the Power Platform.** [ASSUMPTION] characteristics:
- Comfortable with `pac` CLI and Azure CLI (`az`).
- Already use Claude Code or similar AI coding agents day-to-day.
- Work primarily in Dataverse-backed environments (solution-aware, not personal/default).
- Need to ship changes to production flows on a regular cadence — not one-off experiments.

Out of scope for v1: low-code makers without CLI fluency, customer-facing scenarios via the marketplace, citizen developers in personal Power Apps environments.

## 4. User journey (v1 happy path)

UJ-1: **"Change the recipient of the 'New Order Notification' flow."**

1. Developer runs `/configure-power-automate`. The plugin checks `pac auth list`, confirms an active profile, lists available environments, lets the developer pick the target (production solution environment), and verifies Web API access by listing workflows.
2. Developer runs `/cloud-flow` and describes the change: "in the New Order Notification flow, change the To: field of the Send Email action to use the AccountManager column instead of OrderOwner."
3. The plugin's planner agent loads the flow by name, parses the `clientdata` JSON, identifies the Send Email action, and presents a diff showing the proposed expression change.
4. Developer approves. The builder agent applies the change, validates against the Workflow Definition schema, and shows the validated diff.
5. Developer confirms deploy. Plugin PATCHes the workflow row via Web API. Flow updates in-place; state is preserved.
6. Plugin verifies by re-fetching the workflow and confirming the change landed. Reports success with a link to the maker-portal view for sanity check.

Total elapsed: under 5 minutes wall-clock for a simple change (per success criterion §7.1). No browser context-switch.

## 5. Functional requirements

Grouped by feature. IDs are stable and globally unique within this PRD.

### F1 — Environment configuration (`/configure-power-automate`)

- **F1.1** — Detect whether `pac` CLI is installed. If missing, prompt the user with the install command and exit cleanly.
- **F1.2** — Detect an active `pac auth` profile via `pac auth list`. If none, guide the user through `pac auth create` for the target environment.
- **F1.3** — List available environments the authenticated user can reach; let the user select one. Persist the choice for the session.
- **F1.4** — Verify Web API access by issuing a `GET /api/data/v9.2/workflows?$top=1&$filter=category eq 5` against the chosen environment's Dataverse URL. Report success or surface the exact HTTP error.
- **F1.5** — Save the resolved environment URL, environment ID, and auth profile to a session config file the other skills consume. [ASSUMPTION] location follows the convention used by the existing `code-apps` and `power-pages` plugins.

### F2 — Edit an existing cloud flow (`/cloud-flow`, EDIT mode)

- **F2.1** — Accept the user's natural-language edit intent (free text).
- **F2.2** — Discover existing cloud flows in the configured environment (`GET /workflows?$filter=category eq 5`). Let the user select the target flow by name or display name. Allow `--by-id` for scripted invocation.
- **F2.3** — Load the selected workflow's `clientdata` JSON. Parse `properties.definition` (the Logic Apps Workflow Definition) and `properties.connectionReferences`. The parser preserves unknown top-level properties per N4 — never strips fields it does not recognize.
- **F2.4** — Spawn a planner sub-agent that:
  - Resolves any new connector metadata the edit will need (`apiId` / `operationId` / required parameters for the connector / action being added or modified). The planner consults the bundled `references/ConnectorPatterns.md` reference (a curated catalog of connector metadata for the v1 connector set — see §9). When the connector or operation is not in the bundled catalog, the planner halts and asks the user rather than guessing. This pre-flight is the structural defense for §2 pillar 1 — the plugin never fabricates connector metadata at deploy time.
  - Maps the flow's existing `connectionReferences` against what the edit will require — pre-flight identification of any missing connection reference, surfaced now rather than at deploy time.
  - Maps the user's intent to a concrete diff plan: which triggers / actions / expressions / connection-references will change, and which will not.
  - Presents the plan for explicit user approval before any modification.
- **F2.5** — Spawn a builder sub-agent that applies the approved plan to the in-memory `clientdata`. The exact v1 scope:
  - **Trigger parameter edits.** Change trigger inputs and parameters of the existing trigger type: recurrence schedule for scheduled flows; table / filter / condition for Dataverse triggers; headers / body shape for HTTP triggers; parameters of a manual trigger. Always supported.
  - **Trigger type swap.** Replace the trigger entirely (e.g., manual → scheduled, scheduled → Dataverse) only when the **trigger-swap compatibility test** passes: the new trigger's output schema (the `outputs.body` shape made available to downstream actions) is a superset of the old trigger's output schema. When it is not, halt and tell the user which action(s) would break.
  - **Action edits inside an existing scope.** Within an existing scope-typed container (top-level `actions`, or any node whose `type` is one of `Scope`, `If`, `Foreach`, `Until`, `Switch`) the builder may: swap action input values; change expressions; rename action display names; modify `runAfter`; add a single action; remove a single action. "Single" means one action per `/cloud-flow` invocation, not one per edit session.
  - **Restructures explicitly out of v1.** Adding or removing scope-typed containers themselves; adding or removing `Switch.cases` or `Switch.default`; splitting an action into parallel branches; nesting an existing action inside a new scope. The builder halts with a named restriction message ("cannot add new Scope action in v1; remove the surrounding scope manually in the maker portal first, or break this change down") rather than attempting the restructure.
- **F2.6** — Validate the modified definition against the published Logic Apps `workflowdefinition.json#` schema before deploy. On failure, show the specific schema violation and offer to retry or abort.
- **F2.7** — Show the user the final diff (before vs after, JSON-level) and require explicit deploy confirmation.

### F3 — Deploy an edited flow (`deploy-flow`, internal skill called by `/cloud-flow`)

- **F3.1** — Issue `PATCH /api/data/v9.2/workflows(<id>)` with the new `clientdata` payload. In-place edit only in v1; cross-environment deploy via `pac solution pack` + `ImportSolution` is out of scope.
- **F3.2** — Preserve the flow's existing state (on/off). Never silently toggle a running flow off as a side effect of an edit.
- **F3.3** — Preserve the flow's existing `connectionReferences`. If the edit added an action whose required connection reference is missing, halt and surface a "you need to create connection reference X in the maker portal" message with a link. Do **not** attempt to auto-create connection references in v1.
- **F3.4** — Verify success by re-fetching the workflow and confirming the `clientdata` round-trips. Report failure clearly with the HTTP response body if non-2xx.
- **F3.5** — On any failure during deploy, never silently leave the workflow in an intermediate state. Either the PATCH succeeds and the diff lands, or the original `clientdata` is unchanged.

## 6. Non-functional requirements

Cross-cutting; not feature-specific.

- **N1 — Microsoft-sanctioned surfaces only.** No calls to `api.flow.microsoft.com`. All authoring goes through Dataverse Web API; all ALM via `pac solution`.
- **N2 — Auth & token hygiene.** Use the existing repo helper pattern (`scripts/lib/validation-helpers.js` style — `getAuthToken`, `makeRequest`). Refresh tokens on long-running operations; never log tokens; never serialize them into generated artifacts.
- **N3 — Schema validation before deploy.** Every PATCH is preceded by a JSON Schema validation pass against `workflowdefinition.json#`. Deploys never silently submit invalid JSON. Schema validation catches structural errors only — JSON Schema does not catch semantic errors (unknown `operationId`, invalid expression syntax). F2.4 planner partially mitigates this via the bundled `ConnectorPatterns.md` catalog for the v1 connector set (Outlook, Teams, Dataverse, HTTP); unknown connectors halt at plan time rather than at deploy. Residual risk: semantic errors inside known connectors slip through to Web API rejection. Acceptable for v1; revisit if rejections become frequent in the pilot failure log.
- **N4 — Schema-churn resilience.** The `clientdata` shape is documented but evolves silently across Power Platform release waves. The plugin's parser must tolerate unknown top-level properties (preserve, don't strip). When a change Microsoft makes breaks our schema validator, fail with a clear "schema version mismatch" message rather than corrupting the user's flow.
- **N5 — Connection-reference safety.** Never invalidate or rebind an existing connection reference during an edit. If a new action needs a new connection reference, halt and instruct the user.
- **N6 — Repo conventions (the 5 pillars).** Honor the five pillars documented in `power-pages/PLUGIN_DEVELOPMENT_GUIDE.md`. Inlined for testability:
  1. **Engaging loading experience.** Every skill creates its task list upfront via `TaskCreate` at phase 1; tasks transition to `in_progress` / `completed` as phases enter and leave. Drives the agent spinner.
  2. **Action verification.** Every skill has a dedicated standalone Verify phase. Verify uses a different code path than implementation — it does not trust that the implementation succeeded.
  3. **Sub-agent architecture.** Complex multi-step authoring delegates to specialist agents (here: flow-planner, flow-builder). Agents run sequentially per repo convention, not in parallel.
  4. **Deterministic execution.** All Web API calls, schema validation, and JSON manipulation happen in Node.js scripts (built-in modules only). No agent-generated Web API calls.
  5. **Transparency & user approval.** Three-point approval: after discovery / target-flow selection, after plan, before deploy. Autonomous between checkpoints.
- **N7 — Shell-agnostic authoring.** No PowerShell cmdlets in code blocks; no `Invoke-RestMethod`; no `$env:` syntax. Node.js scripts for filesystem and HTTP; `bash`-fenced commands only for cross-platform CLIs (`pac`, `az`, `node`).
- **N8 — No new runtime dependencies.** Node's built-in modules only for scripts. Tests via `node --test`. No Jest/Vitest/Mocha. Matches the rest of the repo.
- **N9 — Safe failure on partial deploy.** Never auto-rollback. Report what landed, what didn't, let the user decide remediation.

## 7. Success criteria for v1

A v1 ship is "done" when **all** of the following hold:

1. A Data3 developer can complete UJ-1 (the journey in §4) end-to-end against a real production flow in under 5 minutes of wall-clock time, measured from `/configure-power-automate` invocation to deploy-verified.
2. Brad (v1 pilot) has shipped at least **three real flow edits** through the plugin on a live Data3 customer engagement, meeting **all** of the following:
   - Edits span at least three distinct shapes: at least one trigger-parameter or trigger-swap edit (F2.5 trigger edits); at least one multi-action change (add or remove an action plus modify an adjacent action in the same `/cloud-flow` session); at least one expression change in an existing action.
   - No edit corrupts the target flow (corruption defined as: deploy reports success per F3.4 but the flow's next real trigger fires and fails for a reason traceable to the edit).
   - Brad maintains a **failure log** in the pilot engagement covering: edits that hit F3.3 (missing connection reference, kicked back to maker portal), edits that hit F2.5's restructure-out-of-v1 halt, and any deploy-time Web API rejections. The log is the input to v2 scope.
3. The Configure → Edit → Deploy round-trip works against at least three distinct Data3 customer environments.
4. The plugin honors all five UX/reliability pillars defined in §6 N6. Verify each binary criterion in N6 holds before locking v1.
5. The plugin's `.claude-plugin/plugin.json`, `AGENTS.md`, `CLAUDE.md` symlink, and `README.md` exist and follow the conventions of the other 5 plugins in this repo. The plugin loads cleanly via `claude --plugin-dir plugins/power-automate`. (Not yet added to `marketplace.json` — that's the v2 gate.)

**Counter-metrics** (signals we're optimizing the wrong thing):
- *Time-to-first-deploy is fast but edits regularly corrupt flows* — measured as: more than 1 in 5 edits in the failure log are corruption-class (as defined in success #2) → we're shipping a footgun.
- *Brad uses it once and reverts to the maker portal* — measured as: Brad abandons more than half the edits mid-session and finishes them in the portal → the wedge isn't real for our team.

## 8. Out of scope for v1

Explicitly deferred. None of these block v1 shipping; all should land in the v2 / future scope discussion when v1 is proven.

- **Create-from-scratch.** Includes cloning seeded templates per trigger family.
- **Cross-environment deploy.** `pac solution pack` + `ImportSolution` for promoting a flow from dev → prod. v1 is in-place PATCH only.
- **Connection-reference creation wizard.** v1 halts when a needed connection ref is missing; v2 may guide the user.
- **Run-history / debugging surface.** Reading run failures, suggesting fixes from runtime errors.
- **Multi-trigger-family template catalog.** The seeded `references/templates/` directory mentioned in the proposal.
- **Marketplace listing.** v1 is internal-only; marketplace registration in `marketplace.json` is gated on v1 proof.
- **Repo-aligned source-control story.** See §2 (ALM-first trajectory bullet). v1 ships in-place PATCH with no repo artifact; v2 brings `pac solution clone`.
- **Complex definition restructures.** Nested scopes, switch/case, parallel branches — flag for user, don't attempt.

## 9. Dependencies

- **`pac` CLI** ≥ 2.4.1 (for `pac solution` YAML format if/when ALM lands; v1 only needs `pac auth list` / `pac auth create`).
- **Node.js** runtime — built-in modules only, no npm install.
- **Dataverse Web API** v9.2 on the target environment.
- **`scripts/lib/validation-helpers.js`** pattern from `power-pages` (`plugins/power-pages/scripts/lib/validation-helpers.js`) — exposes `getAuthToken`, `makeRequest`, and common path / boilerplate helpers used across all Dataverse-touching skills in the repo. The new plugin either reuses these directly (cross-plugin import) or re-implements the minimal subset locally; decision deferred to architecture phase (see §10 open question 1).
- **Bundled connector reference catalog** — `references/ConnectorPatterns.md` ships with the plugin, covering the v1 minimum connector set (Outlook, Teams, Dataverse, HTTP). Used by the planner per F2.4 to resolve `apiId` / `operationId` without fabrication. Unknown-connector edits halt rather than guess.
- **Active Power Platform environment** with `category = 5` (cloud flow) workflows present. The plugin does not create flows; it edits existing ones.

## 10. Open questions

Non-blocking. Surface for the architecture / epics phases.

1. **Helper sharing across plugins.** Should `getAuthToken` / `makeRequest` move to a repo-level `shared/scripts/` location, or does power-automate re-implement the minimal subset? Affects this plugin and any future plugin needing Dataverse Web API access.
2. **Diff presentation UX.** Side-by-side JSON diff or a semantic diff ("changed To: field from X to Y")? Semantic is better for the user but harder to author. v1 starts with raw JSON diff and iterates if confusing.
3. **State preservation on edit.** Microsoft documents that programmatically created flows land in state 0 (off). Confirm that an existing on-flow stays on after a PATCH — needs an explicit test in v1 verify.
4. **Test fixture strategy.** No live-Dataverse-required tests in CI, but local development needs a fixture flow. Ship a sanitized fixture in `scripts/tests/fixtures/`?
5. **Plugin naming on disk vs in marketplace.** Source dir is `plugins/power-automate/`; marketplace name (when registered) — `power-automate` or `power-automate-preview` matching the `code-apps-preview` convention? Deferred until v1 proves out.
