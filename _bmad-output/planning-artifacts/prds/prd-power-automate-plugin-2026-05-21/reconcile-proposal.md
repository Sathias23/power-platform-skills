---
title: Power Automate plugin — Proposal ↔ PRD reconciliation
created: 2026-05-21
input: plugins/power-automate/PROPOSAL.md
output_prd: prd.md
output_addendum: addendum.md
---

# Reconciliation pass — PROPOSAL.md vs PRD + Addendum

Purpose: surface the gaps between what the user supplied in `PROPOSAL.md` and what the PRD/addendum now say. Each item is tagged **deliberate** (matches a decision-log entry) or **accidental** (no trace in the decision log). The author of the PRD can sign off on each one.

The PROPOSAL was a *design proposal* — it mixed capability intent with implementation choices. The PRD/addendum chose to narrow scope (v1 = edit-existing only) and re-pillar around a competitive wedge that the PROPOSAL did not articulate. Most reframes flow from that decision.

---

## 1. Items from PROPOSAL that landed in PRD

| PROPOSAL item | PRD location |
|---|---|
| 6th plugin in `power-platform-skills` marketplace | §1 Context (implicit), §7.5 (plugin loads via `claude --plugin-dir`) |
| Web API `/api/data/v9.2/workflows` as primary authoring surface | §5 F1.4, F2.2, F2.3, F3.1; §6 N1 |
| `pac auth` for auth, reuse its token | §5 F1.2, F1.3, F1.5; §6 N2 |
| Clone-first authoring strategy ("avoids invalid operationMetadataId GUIDs, wrong apiId paths, malformed connection references") | §2 pillar 1 (verbatim quote preserved) |
| Planner + builder agent split (analogous to canvas-app-planner / canvas-screen-builder) | §5 F2.4 (planner sub-agent), F2.5 (builder sub-agent); §6 N6 (sub-agent architecture pillar) |
| `/configure-power-automate` skill — pac auth, env pick, Web API verification | §5 F1 (all sub-items) |
| `/cloud-flow` unified skill | §5 F2 (EDIT mode only) |
| `deploy-flow` internal skill — PATCH workflow via Web API | §5 F3 |
| `add-connection-reference` skill — halt and instruct user to create connection in maker portal | §5 F3.3 (halt-only); §6 N5 |
| Schema validation against `workflowdefinition.json#` before deploy | §5 F2.6; §6 N3 |
| `pac` CLI dependency, detect + prompt-to-install | §5 F1.1; §9 (pac ≥ 2.4.1) |
| No `.mcp.json` (no MCP server to register) | Implicit in §9 (no MCP dependency listed) |
| Microsoft-sanctioned surfaces only / rejection of `api.flow.microsoft.com` | §2 pillar 2; §6 N1 |
| Schema-churn / parser tolerance for unknown properties | §6 N4 (lifted from PROPOSAL "open question 2" + extended) |

## 2. Items from PROPOSAL that landed in ADDENDUM

| PROPOSAL item | Addendum location |
|---|---|
| Full Options Considered table (5 surfaces: api.flow, mgmt connector, Web API, pac solution, first-party MCP) | A1 — preserved verbatim with one tweak (pac solution marked "post-v1") |
| `clientdata` JSON shape (the code block with properties/definition/connectionReferences) | A2 — preserved verbatim |
| Planner/builder agent-architecture rationale | A3 |
| Full proposed plugin tree (agents/, skills/, references/, scripts/) | A4 — annotated with (v1) / (v2) tags |
| References list (Microsoft Learn links) | A8 — preserved and expanded |

## 3. Items from PROPOSAL that were DROPPED entirely

### 3a. Deliberate drops (decision-log backed)

| Dropped item | Decision-log entry |
|---|---|
| **Create-from-scratch flow authoring** (PROPOSAL skill `/cloud-flow` CREATE mode: "pick template family → planner → builder → deploy") | Decision: "v1 scope cut. Configure + Edit existing + Deploy (in-place PATCH)." and "Create-from-scratch is explicitly NOT v1." |
| **Cross-environment deploy via `pac solution pack` + `ImportSolution`** | Decision: "v1 explicit exclusions: cross-env solution-pack/import (stretch)" |
| **Seeded template flow JSONs per trigger family** (`references/templates/`, `TemplateCatalog.md`) | Decision: "Multi-trigger-family template catalog" excluded from v1 |
| **`add-connection-reference` as an active wizard** that polls Web API until the connection becomes visible | Decision: "connection-reference creation wizard (only if a v1 edit needs one)" — PRD F3.3 reduces it to halt + instruct |
| **`pac solution clone` for source-control export** as a day-1 capability | Decision flowed from v1 cut — addendum A1 explicitly defers `pac solution` to v2 |

### 3b. Accidental / quiet drops (no decision-log trace)

These are items the PROPOSAL stated as concrete design intent that the PRD/addendum do not mention at all. They may have been silently rolled into "implementation detail" but they deserve a sign-off:

| Dropped item | Why it matters |
|---|---|
| **`definition-validator.{js,mjs}` JSON Schema validator implementation detail** | Addendum A4 lists the file as "(v1)" but PRD §5/§6 only require validation as a behaviour (F2.6, N3), not the named module. Probably fine; flag in case the architecture phase needs it. |
| **`AGENTS.md`, `CLAUDE.md` symlink, `README.md`** as deliverables | Listed in PROPOSAL plugin tree; addendum A4 carries them forward but PRD §7 success criteria only require `plugin.json`. May lead to the v1 ship skipping plugin documentation. |
| **`ConnectorPatterns.md`** scope ("Dataverse, Outlook, Teams, HTTP, SharePoint") | Addendum A4 reduces this to "v1 — minimal: Outlook, Teams, Dataverse, HTTP" — SharePoint silently dropped from the v1 list. Likely deliberate scope-trim but not in the decision log. |
| **`ExpressionLanguage.md`** content scope ("`@triggerOutputs()`, `@body()`, `@items()`, conditions, functions") | Addendum A4 marks it v1 but neither doc captures the content boundary. Possibly fine; the reference will get authored later. |
| **"Discover existing flows in the environment to use as templates"** (flow-planner responsibility for CREATE) | Function dropped along with CREATE mode — consistent with v1 cut, but PROPOSAL's flow-planner also did this work *for EDIT* (resolving connector apiId/operationId). PRD F2.4 only describes mapping edits; it does not preserve the connector resolution duty. Worth checking. |
| **"Resolves required connectors and their `apiId` / `operationId` values"** (flow-planner responsibility) | Not present in PRD F2.4 or addendum A3. This was load-bearing for accurate edits (PROPOSAL open question 2 specifically called out "unknown operationId" as the gap a describe_api lookup would fill). The PRD acknowledges the gap in N3 NOTE ("unknown operationId won't be caught until Web API rejects") but the planner duty itself was dropped. Likely accidental. |
| **"Maps `connectionReferences` the user already has vs. ones they need to create"** (flow-planner responsibility) | Dropped from PRD F2.4. F3.3 handles the *missing* case at deploy time, but the planner's pre-flight mapping is gone. Possibly accidental — that mapping is what enables the halt-and-instruct UX to be friendly rather than late. |
| **"`add-connection-reference` polls via Web API until visible, returns the `connectionReferenceLogicalName` for the builder to wire in"** | The polling mechanism — useful even as v2 design intent — is absent from PRD and addendum. Decision log only says "wizard out of v1" but the *mechanism design* the PROPOSAL outlined is lost. Likely accidental for v2 design context. |
| **`pac solution clone` produces a YAML layout (PAC CLI ≥ 2.4.1) with a `workflows/` directory** as the "fits source control cleanly" framing | The repo-aligned pillar (PRD §2 pillar 3) mentions "diff-able JSON in your repo" but loses the `pac solution` YAML angle — which the PROPOSAL framed as the canonical source-control story. The PRD pivot to "in-place PATCH only" effectively means v1 has *no* repo artifact at all unless the user manually saves the diff. This is a meaningful posture shift not called out in the decision log. |

## 4. Items that got SUBTLY REFRAMED

| PROPOSAL framing | PRD/addendum framing | Comment |
|---|---|---|
| "Lets an AI coding agent **author, edit, and deploy** Power Automate cloud flows (automated, instant, and scheduled)" | "lets a developer using an AI coding agent **clone, edit, and deploy** an existing Power Automate cloud flow" | "Author" → "clone". The trigger-family list (automated/instant/scheduled) is dropped. v1 cut explains "author" → "edit", but "clone" in the PRD now means "load existing via Web API" — *not* the PROPOSAL's "`pac solution clone`" sense. Same word, different meaning. |
| "Clone-first" = always clone a template (existing flow or seeded template) before mutating | "Clone-existing, not generate-from-scratch" (PRD §2 pillar 1) | Slightly different — PROPOSAL's clone-first is a *technique* for both create and edit; PRD's clone-existing is a *competitive wedge* against Flow Studio MCP. Same vocabulary, different load. |
| "Mirrors how the existing Copilot Studio-style skills in this repo work" | "Mirrors the canvas-apps pattern (`canvas-app-planner` + `canvas-screen-builder`)" (Addendum A3) | The PROPOSAL's "Copilot Studio-style" reference is replaced by canvas-apps. This is likely a correction (the repo has no Copilot Studio skills; canvas-apps is the actual analog), but it is an unacknowledged factual edit. |
| "Same architectural conventions as the existing plugins (canvas-apps, code-apps, mcp-apps, model-apps, power-pages)" | "Honor the conventions documented in `power-pages/PLUGIN_DEVELOPMENT_GUIDE.md`" + "5 UX & reliability pillars" (PRD §6 N6) | The PROPOSAL pointed at all 5 plugins as exemplars; the PRD anchors solely on power-pages' development guide. This is a real narrowing — canvas-apps' MCP-server pattern, code-apps' connector-add pattern, etc. are no longer reference points. |
| Open question 2: "A 'describe API' lookup like the Canvas Authoring MCP server provides would help" | PRD N3 [NOTE FOR PM]: "schema validation catches structural errors only — unknown `operationId` won't be caught until Web API rejects the PATCH. Acceptable for v1; revisit if we hit recurring runtime failures." | Reframed from "this would be nice to have" → "acceptable failure mode for v1." The PROPOSAL's implicit ask for a describe-API capability is gone; PRD treats deploy-time rejection as the validation layer. |
| Open question 3: "`pac` CLI availability — canvas-apps requires .NET 10 SDK for its MCP server. We'd require `pac` CLI" | PRD F1.1: "Detect whether `pac` CLI is installed. If missing, prompt the user with the install command and exit cleanly." | Operationalized cleanly — no real loss, just a reframe from "thing to figure out" to "concrete requirement." |
| "Edits the cloned `clientdata` JSON — swaps trigger, replaces / inserts actions, rewires expressions, updates `connectionReferences`" | PRD F2.5: "swap action input values, change expressions, rename action display names, modify `runAfter`, add/remove a single action under an existing scope" | PRD scope is *narrower* than PROPOSAL's edit list — PROPOSAL implied trigger swaps and connectionReference updates were table stakes; PRD F2.5 omits trigger swaps entirely and F3.3 actively forbids new connection-reference creation. Worth a sign-off — is "edit the trigger" really out of v1? |

## 5. Qualitative / intent content the FR/NFR structure may have flattened

Several pieces of the PROPOSAL's *posture* (the why-this-feels-right register) survive only as fragments in the PRD. The PRD compensates with a new positioning section (§2) that the PROPOSAL did not have, but specific qualitative phrasings were lost:

- **"No first-class local source format the way Canvas (pa-yaml) or Pages do"** — the PROPOSAL opens by acknowledging this as the central design constraint. PRD never states it. The reader has to infer why JSON-in-Dataverse is the only path. The constraint is load-bearing context that the FR/NFR structure cannot express.

- **"Language-agnostic, supported, low-friction"** (PROPOSAL's three-word justification for Web API) — collapsed to PRD N1 ("Microsoft-sanctioned surfaces only"). The PROPOSAL's vibe was "this is the *easy* path"; the PRD's vibe is "this is the *correct* path." Different feel.

- **"Avoids building a parallel MSAL flow and keeps the user's existing environment selection"** — PROPOSAL's rationale for reusing `pac auth` tokens. PRD F1.5 and N2 require the behaviour but lose the *why*. Implementer reading only the PRD might invent a parallel auth pattern not knowing this was a conscious avoidance.

- **"This mirrors how the existing Copilot Studio-style skills in this repo work"** — even though the reference was wrong (corrected to canvas-apps), the *intent* — "this isn't a new pattern, it's the same pattern we already use" — is a positioning statement that helps onboard contributors. PRD N6 partially recovers this by pointing at power-pages' guide.

- **PROPOSAL framing as "draft / exploration. Not a working plugin yet. This document captures the research and the proposed shape so the actual `agents/`, `skills/`, and `references/` can be built against an agreed plan."** — The PROPOSAL is explicitly an artefact for *agreement*. The PRD is an artefact for *commitment*. That shift of register is appropriate but invisible to anyone reading the PRD alone; they will not know which decisions are still "I would like sign-off on this" vs "this is locked."

- **Open questions as a first-class section.** PROPOSAL had three; PRD §10 carries forward five (different ones, mostly). PROPOSAL's "open question 1" (seed template flows in `references/templates/` vs. always-clone-from-environment) is fully gone — collapsed into the v1 scope cut. The *tension* it captured (shipping behaviour vs. depending on the user's environment) is a design question that will resurface for v2 and is no longer being tracked.

- **The "wedge" framing itself** (PRD §2) is *new* — the PROPOSAL never used the word "wedge," "positioning," or named competitors. The PRD added a whole competitive frame (Flow Studio MCP, Microsoft Copilot in Power Automate). This is upgrade rather than loss, but worth noting: the user's PROPOSAL did not ask for this framing, and if they have a different read of the competitive landscape, the PRD's pillars may not match their mental model.

- **PROPOSAL tone — "low-key, technical, here-are-the-options-we-surveyed."** PRD tone — "strategic, audience-targeted, defensible." Both are legitimate; just be aware the register shift happened without explicit decision-log acknowledgement.

---

## Recommended sign-offs (the short list)

If you only validate four things, validate these:

1. **Was the loss of `pac solution` as v1 source-control posture deliberate?** PROPOSAL implied a repo-aligned story from day 1; PRD v1 ships with no repo artifact at all (in-place PATCH only). Decision log covers the scope cut but not the loss of "diff-able JSON in your repo" as a v1 capability.
2. **Are trigger-swap edits really out of v1?** PRD F2.5 lists "swap action input values, change expressions… add/remove a single action" but does not include trigger swaps. PROPOSAL treated trigger editing as table stakes.
3. **Is the planner's connector-resolution / connection-reference-mapping duty preserved anywhere?** PROPOSAL gave the planner three jobs (template discovery, connector resolution, conn-ref mapping). PRD F2.4 keeps only "map intent to diff." The other two duties are gone with no decision-log entry.
4. **Does the v1 ship include `AGENTS.md` and `README.md`?** Addendum A4 carries them forward; PRD §7 success criteria do not require them. Could be a quiet drop.
