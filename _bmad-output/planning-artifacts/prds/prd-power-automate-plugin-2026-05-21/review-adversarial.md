---
title: Adversarial Review — Power Automate plugin PRD
created: 2026-05-21
reviewer: adversarial-review (cynical mode)
scope: prd.md + addendum.md + .decision-log.md
---

# Adversarial Review — Power Automate plugin PRD

Verdict up front: **Not ready to lock as v1 without three small but load-bearing scope tightening edits.** The PRD is internally consistent and the wedge is plausible — but several success criteria, NFRs, and out-of-scope deferrals are written in a way that lets v1 ship without actually being useful. The most damaging finding isn't a missing requirement; it's a feedback loop between F3.3, §8 ("conn-ref creation deferred"), and success #2 ("Brad ships three real edits") that could plausibly stall v1 on contact with a real customer flow.

Findings are tagged CRITICAL / HIGH / MEDIUM / LOW. Each finding quotes the source. I've ordered them roughly by impact on whether v1 ships *and is useful*, not by section number.

---

## CRITICAL

### C1. The F3.3 ↔ §8 ↔ success #2 trap: v1 may be functionally non-shippable on the first real edit

The PRD's most consequential design choice is buried across three separate sections that, when read together, describe a failure mode the document never acknowledges.

- **F3.3 (PRD §5):** *"Preserve the flow's existing `connectionReferences`. If the edit added an action whose required connection reference is missing, halt and surface a 'you need to create connection reference X in the maker portal' message with a link. Do not attempt to auto-create connection references in v1."*
- **§8 (Out of scope):** *"Connection-reference creation wizard. v1 halts when a needed connection ref is missing; v2 may guide the user."*
- **Success #2 (PRD §7):** *"Brad (v1 pilot) has shipped at least three real flow edits through it on a real Data3 customer engagement."*

What this means in practice: **any edit that adds an action requiring a new connector** (which is one of the most common edit shapes a customer pays Data3 to make — "add a Teams notification," "log this to a new SharePoint list," "POST to this new HTTP endpoint") will halt the plugin and send the user back to the maker portal. The plugin then is no longer the authoring surface; it's a JSON validator wrapped around a 60-second portal trip.

The PRD does not quantify what fraction of real Data3 customer edits add a new connector vs. modify existing actions. Without that number, the v1 useful-edit surface is unbounded between "every meaningful edit" and "almost no real edits." Brad's "three real flow edits" success bar can be met by cherry-picking three same-connector edits — which would satisfy the criterion but not validate the wedge.

**Severity rationale:** This is the single failure mode that could make v1 ship-but-die. The PRD silently optimizes for the easy path (edit-existing-action) and defers the hard path (add-action-with-new-connector) — but the user has no way of knowing in advance which path a given customer ask falls into.

**Fix to consider before locking:** Either (a) add a bounded conn-ref creation path to v1 (even a one-shot "create+bind via Web API, fail loud") or (b) rewrite success #2 to explicitly carve out which edit shapes count: *"at least three real flow edits, of which at least one added an action requiring a new connector and resolved end-to-end without leaving the terminal."* The current criterion is gradeable but not falsifiable.

---

### C2. Success criterion #2 is self-graded by the PRD author against an unbounded edit population

PRD §7 success #2: *"Brad (v1 pilot) has shipped at least three real flow edits through it on a real Data3 customer engagement. Other team adoption is post-v1."*

A single-pilot success criterion with no independent attestation, no edit-shape diversity requirement, and no failure-mode log is not a v1 done bar — it's a journal entry. Specific holes:

- **Cherry-picking risk.** Brad picks three edits that the plugin's v1 scope happens to handle (modify-existing-action, no new conn-refs, no complex restructure). The criterion clears. The wedge remains unproven.
- **No fail-log requirement.** If Brad tries 15 edits and 3 succeed, the criterion still clears. The "12 reverted to maker portal" data — which is the actual signal — has no surface in §7.
- **No durability requirement.** "Has shipped" is past tense and one-shot. There's no "and the flow is still running unchanged 7 days later" — but the counter-metric "edits regularly corrupt flows" is open-ended.
- **No customer feedback loop.** A Data3 customer never sees or judges the plugin; the customer only sees the resulting flow. Success #2 cannot tell you whether the plugin shipped a flow that *looks* right but behaves subtly wrong (wrong runAfter, dropped optional property, broken connection-reference binding that survives schema validation but fails at runtime).

The counter-metrics in §7 are good as concepts ("Developers use it once and revert" / "edits regularly corrupt flows") but they have no measurement story attached. There's no telemetry plan, no log format, no "we'll review at week 4." They are aspirational guardrails, not gates.

**Severity rationale:** This is how internal tools quietly fail. The bar is met, the plugin is "shipped," and three months later nobody is using it but no one ever wrote down why.

**Fix to consider:** Tighten success #2 to require (a) edit-shape diversity (at least one each of: action-input change, expression rewrite, action-add with existing conn-ref, trigger swap), (b) a failure log of attempted-but-aborted edits with reason codes, and (c) a 7-day-post-deploy "flow still healthy" check.

---

## HIGH

### H1. The wedge is correctly described but undefended against the obvious competitive move

PRD §2 frames the wedge as: *"Edit an existing production flow from your coding agent, in-place, using only Microsoft-sanctioned surfaces (Dataverse Web API + `pac solution`)."*

This is a real wedge **today**. It is not a defensible wedge **next quarter**. The PRD does not address what happens when Flow Studio MCP — which already does "create, edit, deploy live flows" against `api.flow.microsoft.com` — adds:

1. A `clone_existing_flow` tool. (Trivial; they almost certainly have the read already.)
2. A "use Dataverse Web API instead of api.flow.microsoft.com" config flag. (Pure plumbing.)

Both are about a week of work for a team that already has the auth, schema, and deploy plumbing. Once they ship those, this plugin's wedge collapses to "open-source-and-in-the-Microsoft-marketplace" — which is real differentiation for some buyers but is a *distribution* wedge, not a *capability* wedge.

The "Microsoft-sanctioned surfaces" claim is also doing more rhetorical work than its evidence supports. PRD §2 says: *"`api.flow.microsoft.com` is unsupported by Microsoft's own docs; Web API + `pac solution` are the documented programmatic paths. This survives API churn better than competitors."*

Reality check: Microsoft "unsupported" surfaces routinely outlive supported ones in practice; the `api.flow.microsoft.com` endpoints have been "unsupported" for years and the Power Platform community uses them constantly. The schema-churn argument is plausible but **A7 (addendum)** notes that even the documented `clientdata` shape *"is undocumented-stable but evolves silently between release waves."* So Microsoft-sanctioned doesn't actually buy schema stability — it buys API-endpoint stability, which is a weaker claim than the PRD makes it sound.

**Severity rationale:** The PRD is being sold on a wedge that's real-for-now but undefended over the 12-month horizon. For an internal tool that's fine if v1 ships fast; it's a problem if the team plans v2/v3 around this same framing.

**Fix to consider:** Reframe §2 honestly. The wedge is "we ship today, on the surfaces Brad's team trusts, with open-source code Data3 can audit and modify." That's a *Data3 internal* wedge — completely valid given the v1 audience — but it's not a *competitive* wedge. The "Microsoft-sanctioned surfaces" framing originated from research, not from the proposal, and it's the framing most likely to mislead a v2 planning discussion. Pull the load-bearing language back.

---

### H2. The two-plugin split (AUTHOR here, REGISTER in power-pages) will confuse users — including Brad

Decision log: *"`power-pages/add-cloud-flow` REGISTERS existing flows into a site (metadata + client wiring); it does NOT author them. The new plugin owns AUTHORING. Two-plugin split mirrors canvas-apps (author) ↔ power-pages (integrate)."*

The canvas-apps ↔ power-pages analogy is doing a lot of work and isn't quite right. The canvas-apps case has a strong physical boundary: a canvas app is a deployable artifact that exists independently of any Power Pages site. The flow case has a much weaker boundary: a "Power Automate cloud flow used by a Power Pages site" *is the same Dataverse row* whether you're authoring it or registering it.

User-facing failure modes the PRD doesn't address:

- A user in the middle of a Power Pages engagement says "add a cloud flow that emails me when someone submits this form." Which skill triggers? `power-pages/add-cloud-flow` (which assumes the flow exists) or `power-automate/cloud-flow` (which won't create from scratch in v1)? Neither. The user has to manually go create the flow in the portal — the exact pain v1 is supposed to remove for power-pages users.
- A user edits a flow with `/cloud-flow`, then needs to re-register it because the trigger URL changed. Two skills, two invocations, no shared state. The PRD's F1.5 mentions a session config but doesn't address cross-plugin state.
- The two skill *names* overlap semantically: `/cloud-flow` (this plugin) and `add-cloud-flow` (power-pages). A user typing "cloud flow" in slash-command autocomplete will see both. The PRD does not address disambiguation.

The PRD also makes no mention of whether the new plugin's `deploy-flow` skill should signal `power-pages/add-cloud-flow` to re-sync its metadata if the flow's trigger characteristics changed (URL, schema, etc.). If a Power Pages dev edits a flow used by their site and the page-side registration silently stops matching, the v1 plugin has just shipped a worse outcome than the maker portal.

**Severity rationale:** This isn't fatal to v1 (Brad's pilot is unlikely to span both plugins on the same engagement), but it's a known issue the PRD acknowledges without mitigating. v2/v3 will inherit a confusing surface area.

**Fix to consider:** Add a §5 functional requirement: *"`/cloud-flow` MUST detect when the edited flow is referenced by a Power Pages site's metadata (via the existing `add-cloud-flow` registration artifacts in the user's workspace) and warn if the edit changes the flow's trigger contract."* Or, less invasive: a §10 open question explicitly asking the architecture phase to resolve the boundary.

---

### H3. N3 admits its own insufficiency; the "acceptable for v1" hand-wave needs a number

PRD §6 N3: *"Every PATCH is preceded by a JSON Schema validation pass against `workflowdefinition.json#`. Deploys never silently submit invalid JSON. [NOTE FOR PM] schema validation catches structural errors only — unknown `operationId` for a connector won't be caught until the Web API rejects the PATCH. Acceptable for v1; revisit if we hit recurring runtime failures."*

The "[NOTE FOR PM] ... Acceptable for v1" is the PRD writing a check it cannot cash. The thing schema validation **doesn't** catch is exactly the thing the planner sub-agent (F2.4) is most likely to get wrong — connector `operationId` / `apiId` mappings on a connector the plugin hasn't seen before. This is the recurring failure mode the wedge specifically claims to solve ("avoids the recurring failure mode of generating invalid `operationMetadataId` GUIDs, wrong `apiId` paths" — PRD §2 pillar 1).

So the PRD says:
- §2 pillar 1: We avoid invalid operationMetadataId / apiId by cloning.
- F2.4: The planner *resolves new connector references the edit will need*.
- F2.5: The builder can *add an action under an existing scope* (which requires producing valid `operationId` / `apiId` for that action's connector).
- N3: We validate against the schema — but the schema *doesn't catch invalid `operationId`*.
- N3 [NOTE FOR PM]: Acceptable for v1.

These statements are not mutually contradictory but they're in deep tension. The clone-existing wedge protects you when you're modifying actions that *already exist in the flow*. The moment F2.5 lets you *add a new action* (even under an existing scope), you're back in generate-from-scratch territory for that action's connector binding — and N3 admits the validator won't catch it.

A "revisit if we hit recurring runtime failures" gate has no trigger. How many is "recurring"? Who counts? Where's the log?

**Severity rationale:** This is the technical analog of C1 — the PRD has identified a real risk and waved at it. The wedge claim relies on "we don't fabricate connector metadata" but the v1 scope explicitly allows fabricating connector metadata for added actions.

**Fix to consider:** Either (a) restrict F2.5 to forbid adding actions in v1 entirely (only modify existing), which is a real scope cut but defensible, or (b) require the planner sub-agent to verify any newly-introduced `apiId`/`operationId` against a known-good catalog (the `references/ConnectorPatterns.md` mentioned in A4) before the builder runs, and document the v1 catalog coverage explicitly in success criteria.

---

### H4. N4 (schema-churn resilience) has no test-fixture story and probably can't be verified

PRD §6 N4: *"The `clientdata` shape is documented but evolves silently across Power Platform release waves. The plugin's parser must tolerate unknown top-level properties (preserve, don't strip). When a change Microsoft makes breaks our schema validator, fail with a clear 'schema version mismatch' message rather than corrupting the user's flow."*

This is two requirements jammed together:

1. **Preserve-unknown-properties on parse/round-trip.** This is testable in isolation: feed the parser a `clientdata` with an unknown property, modify a known property, serialize, check unknown property is still there. Easy to write, easy to verify. Open question #4 ("Test fixture strategy. ... Ship a sanitized fixture in `scripts/tests/fixtures/`? [NOTE FOR PM]") — *which has a [NOTE FOR PM] tag* — would resolve this. But the open question hasn't been resolved, so N4 as written has no test plan.

2. **Fail with a clear 'schema version mismatch' message when Microsoft changes things.** This is **fundamentally unverifiable in v1** because the only way to test it is to wait for Microsoft to change the schema. The plugin can't simulate a future schema break without inventing a fake one. So this requirement is effectively "be careful, I guess."

Combined with open question #4, N4 is a requirement that is **partially testable and partially aspirational**, presented as if it were a single uniform NFR. The "preserve, don't strip" half is real; the "schema version mismatch" half is wishful.

**Severity rationale:** Not blocking for v1 ship, but N4 will likely be the source of the first "but it passed all the tests" incident.

**Fix to consider:** Split N4 into N4a (parser preserves unknown properties — testable, must ship with fixtures) and N4b (deploy fails loud on schema-validator mismatch — best-effort, no v1 verification possible). Resolve open question #4 before locking the PRD.

---

### H5. The "no run-history awareness" out-of-scope (§8) hides a v1 verification gap

PRD §8 defers: *"Run-history / debugging surface. Reading run failures, suggesting fixes from runtime errors."*

PRD F3.4: *"Verify success by re-fetching the workflow and confirming the `clientdata` round-trips."*

These together mean: v1's definition of "deploy succeeded" is "the JSON we PATCHed is what came back." It does *not* mean "the flow can still run." A flow can pass `clientdata` round-trip and then fail on its very next trigger because the edit introduced a runtime issue (broken expression that's syntactically valid, action input that references a property the action doesn't expose, runAfter referencing an action that was renamed). The plugin will report success. The user will then discover at the customer site that the flow is broken.

Success criterion #1 requires UJ-1 end-to-end "in under 5 minutes." UJ-1's step 6 is *"Plugin verifies by re-fetching the workflow and confirming the change landed."* That is a round-trip check, not a runtime check.

Combined with success #2's "shipped three real flow edits" (where "shipped" is undefined — does it mean "PATCHed successfully" or "the flow ran successfully on its next trigger"?), v1 can ship by every PRD criterion while shipping broken flows to customers.

**Severity rationale:** The counter-metric "edits regularly corrupt flows → we're shipping a footgun" *exists in §7*, so the team has thought about this — but there's no v1 mechanism to detect corruption short of customer complaint.

**Fix to consider:** Add to F3.4: *"After successful PATCH, if the flow has a manual trigger or test capability, offer to invoke a test run and report the outcome. For automatic/scheduled-trigger flows, surface a 'monitor next run within 24h' reminder."* Or accept the gap and add an explicit "v1 cannot verify runtime correctness; user is responsible for post-deploy monitoring" warning to the §4 UJ-1 happy path.

---

## MEDIUM

### M1. The §2 framing originated from research, not from Brad — and the PRD does not flag this

Per the review context: the "clone-existing + Microsoft-sanctioned surfaces" framing came out of research the agent did, not from Brad's original proposal. Brad confirmed it. The decision log records the research finding but the PRD presents the wedge as if it were Brad's load-bearing original claim.

This matters because (a) Brad has not had the friction of defending the framing himself against the obvious alternatives (Flow Studio MCP, maker-portal Copilot), so he hasn't pressure-tested it, and (b) when the wedge bumps into reality during the pilot, the response will be "the research said this would work" rather than "I had a reason and it didn't pan out."

The decision log line *"Defensible angle for this plugin: clone-existing + ALM-first + Microsoft-sanctioned surfaces ... Must be sharply named in the PRD's 'why this exists.'"* is **prescriptive** ("must be sharply named") rather than reporting a confirmed hypothesis. The PRD then dutifully sharply names it. The framing has been promoted from research finding to PRD load-bearing claim without an explicit "Brad endorsed and tested this rationale" gate.

**Severity rationale:** Common failure mode in agent-assisted PRDs — research generates a plausible-sounding wedge, the PRD adopts it, and the human author signs off because it sounds smart. Six weeks later the wedge doesn't hold and nobody is sure who actually believed it.

**Fix to consider:** Add a §2 footnote: *"This positioning was developed during research; the v1 pilot is in part a test of whether it holds."* It costs nothing and sets honest expectations.

---

### M2. Open question #1 (helper sharing) is wrongly deferred to architecture

PRD §10.1: *"Helper sharing across plugins. Should `getAuthToken` / `makeRequest` move to a repo-level `shared/scripts/` location, or does power-automate re-implement the minimum subset?"*

This is filed as "non-blocking, surface for architecture phase." It is **functionally blocking** for N6 ("Repo conventions — the 5 pillars") because pillar 4 is "deterministic Node.js scripts for all Web API calls." If the plugin re-implements its own subset, it forks the convention; if it imports cross-plugin, it creates a coupling the repo's current structure may not support cleanly (CLAUDE.md: *"Each plugin has shared utilities (e.g., `scripts/lib/`)"* — note **each plugin has** its own, not cross-plugin shared).

Architecture phase will be forced to make this decision under time pressure. PRD should commit one way now — either "v1 re-implements, with TODO to extract in v2" or "v1 blocks on a `shared/scripts/` PR being merged first."

**Severity rationale:** Low-stakes but the kind of deferral that becomes a 3-day architecture-phase rabbit hole.

---

### M3. F1.5 (session config) [ASSUMPTION] is invisible to the user and could collide silently

PRD F1.5: *"Save the resolved environment URL, environment ID, and auth profile to a session config file the other skills consume. [ASSUMPTION] location follows the convention used by the existing `code-apps` and `power-pages` plugins."*

If the assumption is wrong (or the conventions differ between code-apps and power-pages), `/cloud-flow` could read stale state from a previous `/configure-power-platform`-equivalent session and silently target the wrong environment. There is no functional requirement that `/cloud-flow` re-confirm the target environment from session config before any destructive operation.

**Severity rationale:** Real footgun. A pilot user running against a customer environment yesterday and a dev environment today is exactly the v1 scenario.

**Fix to consider:** Add F2.0: *"Before any flow discovery or edit, /cloud-flow MUST display the active environment (name + URL) and require an explicit y/n to proceed against it."*

---

### M4. "Under 5 minutes" / "under 2 minutes" in §4 and §7 are vibes, not measurements

PRD §4: *"Total elapsed: under 2 minutes for a simple change."*
PRD §7.1: *"complete UJ-1 (the journey in §4) end-to-end against a real production flow in under 5 minutes."*

There's no instrumentation requirement, no stopwatch protocol, no "wall-clock or LLM time?" definition. With Claude Code's per-turn latency variability, 5 minutes is plausible-or-not depending entirely on prompt length and model load. This is a softness, not a critical issue, but it's the kind of criterion that's met-or-not-met based on who's running it and when.

**Severity rationale:** Soft. Worth tightening if you want the metric to mean anything.

---

### M5. Counter-metrics in §7 are unfalsifiable

PRD §7: *"Counter-metrics (signals we're optimizing the wrong thing): Time-to-first-deploy is fast but edits regularly corrupt flows → we're shipping a footgun. Developers use it once and revert to the maker portal → the wedge isn't real for our team."*

Both are good signals. Neither has a measurement plan. "Regularly corrupt" — how often? "Use it once and revert" — over what window?

A counter-metric without a threshold is decorative. It will be brought up post-hoc to rationalize whatever the outcome was.

---

### M6. Dependency on `pac` CLI version is under-specified vs. v1 actual use

PRD §9: *"pac CLI ≥ 2.4.1 (for `pac solution` YAML format if/when ALM lands; v1 only needs `pac auth list` / `pac auth create`)."*

v1 doesn't use `pac solution` so the 2.4.1 floor is over-tight. But more importantly — the PRD says v1 only needs `pac auth list` / `pac auth create`, which means v1's actual dependency is "any pac CLI version that supports auth." That's basically every released version, including very old ones. Locking to 2.4.1 may cause friction for developers on older customer-installed `pac` versions for no v1 benefit.

**Fix:** Either lower the v1 floor to the actual minimum (`pac auth` has been there forever — what's the real minimum?) or be explicit that 2.4.1 is forward-compat insurance for v2, not a v1 hard requirement.

---

## LOW

### L1. The F2.5 "trigger swap" addition is more dangerous than the PRD treats it

Decision log: *"Trigger swaps — IN v1. F2.5 now lists both trigger edits (incl. swapping trigger type where the rest of the flow stays compatible) and action edits."*

PRD F2.5 first bullet: *"swapping the trigger type entirely (manual → scheduled, etc.) where the rest of the flow remains compatible."*

"Where the rest of the flow remains compatible" is doing all the work in that sentence. Trigger swaps change the trigger output schema, which downstream actions reference via expressions (`triggerBody()`, `triggerOutputs()`, `@triggerBody()?.field`). A manual→scheduled swap will silently break every action that referenced trigger-specific properties. The planner sub-agent (F2.4) is expected to catch this — but the PRD doesn't say *how*, and there's no NFR requiring trigger-output reference validation before approving a trigger swap.

**Severity rationale:** Low because trigger swaps are rare. Medium-if-attempted because they're the most likely path to a "deploy succeeded, flow is silently broken" failure mode.

**Fix:** Add to F2.4: *"For trigger swaps specifically, the planner must enumerate every expression in the flow that references trigger outputs and surface them in the diff plan as 'these references may break.'"*

---

### L2. N7 (shell-agnostic) and the existence of `pac auth create` interactive flow are in tension

N7 forbids PowerShell and requires bash-fenced commands for `pac`. But `pac auth create` is an *interactive* flow on first run (browser auth pop). The PRD doesn't address what happens when the plugin tries to drive an interactive subprocess from a Claude Code session. F1.2 says "guide the user through `pac auth create`" which probably means "print the command, wait for the user to run it themselves" — but that should be explicit.

---

### L3. "ASSUMPTION" tags are inconsistent across the PRD

Some assumptions are flagged ([ASSUMPTION] in §3, F1.5, F2.5) and some equally-load-bearing ones are not (e.g., the entire §2 wedge framing). Either flag all assumptions consistently or drop the tag convention. Inconsistent use makes the document feel more locked-down than it is.

---

### L4. Addendum A6 evidence is "Mixed Reddit / Microsoft Q&A / blog" — fine for v1, but don't promote it later

A6's pain-points list is community-sourced anecdotes, fine for an internal v1. If this PRD becomes the basis for a marketplace pitch (v2 ambition), the evidence needs upgrading. Flag for the v1→v2 transition.

---

## What I'd ask Brad before locking

1. **C1/H3:** What fraction of your recent customer flow edits added an action requiring a new connector? If it's >25%, F3.3's halt-and-instruct is unacceptable and v1 needs a bounded conn-ref creation path.
2. **C2:** What does "shipped" mean in success #2? PATCHed successfully, or running successfully against real triggers? These are different bars.
3. **H1:** If Flow Studio MCP ships clone-from-existing in their next release (which is a week of work for them), does this plugin still ship? What's the actual reason to keep going?
4. **H2:** When a Power Pages dev says "add a cloud flow that emails me when this form is submitted," what skill — `power-pages/add-cloud-flow`, `/cloud-flow`, or neither — should fire in v1?
5. **M1:** Did *you* believe the "clone-existing + Microsoft-sanctioned" framing before the research surfaced it, or are you adopting it because the research made it sound plausible?

---

## Summary

The PRD is well-organized and the wedge is plausible-for-now. It is *not* yet a lockable v1 spec because:

- The connection-reference halt (F3.3) + the conn-ref deferral (§8) + the single-pilot success bar (§7 #2) form a loop that lets v1 ship without ever actually being useful to a customer engagement (C1, C2).
- The wedge framing is more rhetorically loaded than the underlying evidence supports, and was research-generated rather than founder-generated (H1, M1).
- N3 and N4 contain admissions and aspirational requirements presented as committed NFRs (H3, H4).
- Runtime verification is absent from v1 even though the counter-metric "edits corrupt flows" exists in §7 (H5).

None of these block a `v1-rev-2` of the PRD that ships in a few hours of editing. They do block a `v1.0` lock today.
