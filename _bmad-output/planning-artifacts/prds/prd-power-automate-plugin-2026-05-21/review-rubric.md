# PRD Quality Review — Power Automate plugin

## Overall verdict

This is a tight, internally-calibrated PRD that knows what it is: a v1 pilot scoped to one developer (Brad), with a sharply-named wedge ("clone-existing + Microsoft-sanctioned surfaces") and an honest deferred-scope list. The thesis holds together, the deploy-safety NFRs (N3–N5, N9) are product-specific rather than boilerplate, and the addendum carries the receipts. The main weakness is **done-ness clarity on F2.5 edit operations** — terms like "where the rest of the flow remains compatible" and "single action under an existing scope" need testable boundaries before an engineer can confidently scope the work. Secondarily, success criterion #1 ("under 5 minutes") and #4 (the "self-review passes" gate) are too loosely specified to function as a real ship gate.

## Decision-readiness — strong

The PRD makes its decisions and labels them. §2 names the wedge in a callout block and defends it with two pillars rather than hedging. Trade-offs are stated with what was given up: "v1 only edits existing flows" (gives up create-from-scratch), "in-place PATCH only" (gives up cross-env deploy), "halts when a needed connection ref is missing" (gives up auto-creation). The decision log shows real tensions resolved with explicit choices ("Repo-aligned pillar — DROPPED," "Trigger swaps — IN v1"), and the surviving `[NOTE FOR PM]` callouts in N3 and Open Question #4 sit on genuinely deferred decisions rather than safe checkpoints.

A decision-maker reading this can act: the v1 boundary is unambiguous (clone + edit + in-place PATCH, against a single pilot user). Counter-metrics in §7 ("Time-to-first-deploy is fast but edits regularly corrupt flows → we're shipping a footgun") show the PM actually thought about how the work could fail rather than hedging it as universally beneficial.

### Findings

- **low** Open Question #3 (state preservation) reads more like an acceptance test than an open question (§10) — *Fix:* either commit it as a verify step under F3.2 / §7, or label it explicitly as "verification task" not a strategic open.

## Substance over theater — strong

Almost no furniture. The persona section (§3) is one paragraph, four bullets, and a "Out of scope for v1" callout — exactly the weight an internal-team PRD warrants, and it actually drives decisions downstream (CLI fluency assumption underpins F1.1's "exit cleanly if pac missing"). No invented persona names with photos and quotes.

NFRs are product-specific, not boilerplate: N1 names a specific endpoint to *not* call (`api.flow.microsoft.com`); N4 describes a specific failure mode ("schema version mismatch" on silent Power Platform release-wave changes); N5 calls out connection-reference rebinding as a footgun. These are earned by addendum A7's research, not pasted in.

The positioning section (§2) names two named competitors (Microsoft Copilot, Flow Studio MCP) and says what each does and where the gap is. That's differentiation that did the work, not differentiation theater.

### Findings

- **low** N6 references "the 5 pillars" by linking to `power-pages/PLUGIN_DEVELOPMENT_GUIDE.md` without restating them. The link is load-bearing for §7 success #4 — *Fix:* either inline a 1-line summary of each pillar or accept that §7 #4 is unverifiable without opening the external doc (see Done-ness finding below).

## Strategic coherence — strong

The PRD has a thesis and the features serve it. Thesis: "Microsoft-sanctioned, in-place edit of an existing flow from a coding agent is an unfilled niche; Data3 ships flow changes regularly and pays a recurring tax on the maker portal." Every v1 feature traces back: F1 (Configure) and F3.3 (halt-don't-create connection refs) defend "sanctioned surfaces only"; F2.2 (discover existing flows) and the deferred create-from-scratch defend "clone-existing"; N4 (schema-churn resilience) defends "survives API churn better than competitors."

MVP scope kind is clearly **problem-solving** (named friction → narrow capability cut), and the scope logic matches: §8 cuts everything that doesn't prove the wedge. The success metrics in §7 are tied to the thesis (ship real edits without corruption, against multiple environments) rather than activity metrics. Counter-metrics are present and well-chosen.

### Findings

None.

## Done-ness clarity — thin

This is the weakest dimension and where downstream story creation will hurt most.

**F2.5 is fuzzy at the boundary that matters.** "swapping the trigger type entirely (manual → scheduled, etc.) **where the rest of the flow remains compatible**" — what's the test for "compatible"? "add/remove a single action **under an existing scope**" — what counts as a scope, and what happens if the user asks for two actions? "[ASSUMPTION] more complex restructures — nested scopes, switch/case, parallel branches — are stretch. The builder **flags them and asks the user to break the change down**" — what triggers the flag? A static AST check, a runtime schema failure, or builder discretion? An engineer cannot write a story from this without inventing the boundary.

**F2.6 says "the specific schema violation"** which is testable; good. But F2.3 ("parse properties.definition") doesn't say what happens when `clientdata` contains a field the parser doesn't recognize — N4 hints at "tolerate unknown top-level properties (preserve, don't strip)" but the FR doesn't link to it.

**§7 success criteria mix the testable with the squishy.** #2 is excellent ("at least three real flow edits ... on a real Data3 customer engagement"). #3 is excellent ("at least three distinct Data3 customer environments"). But:
- #1: "under 5 minutes" but §4 says "under 2 minutes." Which is the bar? And measured how — wall clock, agent turns?
- #4: "a self-review against power-pages/PLUGIN_DEVELOPMENT_GUIDE.md passes" — no rubric, no defined pass criteria. A self-review where Brad says "looks good" trivially passes.

**UX bounds are adjectives.** "engaging loading," "Report success or surface the exact HTTP error" (F1.4) is good; "Reports success with a link to the maker-portal view for sanity check" (§4 step 6) is good. But "presents a diff showing the proposed expression change" doesn't specify diff format (and Open Question #2 admits this is undecided).

### Findings

- **high** F2.5 edit-operation boundaries are not testable (§5 F2.5) — terms like "where the rest of the flow remains compatible," "a single action under an existing scope," and "flags them and asks the user to break the change down" leave engineering to invent the bar. *Fix:* enumerate the compatibility test for trigger swaps (e.g., "trigger output schema is unchanged or only additive"), define "scope" by reference to the Workflow Definition Language scope action types, and specify the flag mechanism (static structural check vs. schema-validation failure vs. builder LLM judgment).
- **high** §7 success #4 is unverifiable as written ("a self-review against PLUGIN_DEVELOPMENT_GUIDE.md passes") — *Fix:* either turn it into a checklist inlined in §7 (one bullet per pillar with a binary criterion) or remove it and rely on #1–#3 + #5 as the ship gate.
- **medium** Time-target inconsistency between §4 ("under 2 minutes") and §7 #1 ("under 5 minutes"), with no measurement protocol — *Fix:* pick one number, name what's measured (e.g., "wall-clock from `/cloud-flow` invocation to successful re-fetch verification, on a flow with ≤20 actions"), and drop the other.
- **medium** F2.3 parse behaviour does not reference N4's unknown-property preservation rule — *Fix:* add an FR-level statement (or cross-reference) that parsing preserves unknown properties round-trip, so the diff doesn't accidentally strip Microsoft-added fields.
- **low** §4 step 3 says the planner "presents a diff" but Open Question #2 admits the diff format is undecided — *Fix:* note inline in §4 that diff format follows the resolution of Open Question #2, so a reader doesn't mistake the UJ for a committed UX spec.

## Scope honesty — strong

§8 ("Out of scope for v1") does real work: eight named exclusions, each with one-line justification, each tied to a v2 trigger or stretch path. Create-from-scratch, cross-env deploy, connection-ref creation wizard, repo-aligned source-control — all the things a reader might silently assume are covered are explicitly excluded.

`[ASSUMPTION]` tags appear at exactly the right places: §3 (audience characteristics), F1.5 (config file location convention), F2.5 (complex restructures as stretch). The decision log confirms the §1 pain-point assumption was upgraded to fact after user confirmation — that's the assumption-to-fact lifecycle working correctly.

`[NOTE FOR PM]` callouts sit on real tensions: N3 (schema validation only catches structural errors, not unknown operationIds); Open Question #4 (test fixture strategy). Open-items density (5 Open Questions, 3 inline `[ASSUMPTION]`, 2 `[NOTE FOR PM]`) is appropriate for an internal v1 pilot.

### Findings

- **low** F1.5 `[ASSUMPTION]` ("location follows the convention used by the existing code-apps and power-pages plugins") is verifiable by reading the other plugins — leaving it as an assumption is a small drag on F1 implementation. *Fix:* spend 10 minutes confirming the convention pre-architecture and either pin it ("`.power-platform/session.json` per the code-apps pattern") or note explicitly that resolution moves to the architecture phase.

## Downstream usability — adequate

This PRD will feed architecture and epics. Cross-references mostly resolve: §4 references §5; §6 N6 references §7 success #4; §8 references §2's "natural v2 evolution" framing. IDs are contiguous and unique (F1.1–F1.5, F2.1–F2.7, F3.1–F3.5, N1–N9).

No glossary section exists, but the terminology stays consistent across the document: `clientdata`, `connectionReferences`, `category = 5`, `pac auth`, `Workflow Definition` schema all appear with consistent capitalization and meaning. Domain nouns are used identically.

The UJ-1 section is good (named, persona-anchored to §3's "Data3 internal developer," walks through Configure → Edit → Deploy). One UJ is appropriate for a single-operator capability spec — see Shape fit.

**Where it falls short:** §6 N6 references "the 5 pillars" by external link only. If an architect or story-writer source-extracts §6, they cannot evaluate N6 without opening another file. Similarly, §9 dependencies say "`scripts/lib/validation-helpers.js` pattern from power-pages" without naming the actual functions or signatures the plugin will reuse — Open Question #1 acknowledges this is unresolved, which is fine, but a downstream extractor has to chase the source.

### Findings

- **medium** N6 and §9 both depend on external references (`power-pages/PLUGIN_DEVELOPMENT_GUIDE.md`, `scripts/lib/validation-helpers.js`) that aren't summarized inline (§6, §9) — *Fix:* inline a one-bullet-per-item summary of the 5 pillars in N6, and a one-line description of what `getAuthToken` / `makeRequest` do (signature + purpose) in §9. Keeps the PRD self-contained for source-extraction.
- **low** No glossary section. The PRD doesn't drift, but a glossary would still help downstream skills extract `clientdata`, `connectionReferences`, `category = 5`, `workflow row` as canonical terms. *Fix:* add a tiny §11 Glossary with 6–8 terms.

## Shape fit — strong

This is a **capability spec for a single-operator internal tool**, and the PRD wears that shape correctly:

- One UJ (UJ-1), not five — appropriate for "edit-an-existing-flow" being the only v1 capability.
- One persona paragraph, not a persona portfolio — appropriate for v1 = Brad.
- Success metrics are operational (deploy round-trip works against 3 envs; pilot ships 3 real edits) rather than user-facing engagement metrics — correct for an internal tool.
- NFRs read like reliability constraints for a deploy tool (N3 schema validation, N5 connection-reference safety, N9 no-auto-rollback), not like a consumer product's NFR placeholder set.

The PRD is also honest about its brownfield-adjacent nature: §1 explicitly distinguishes the new plugin's role from the existing `power-pages/add-cloud-flow` skill, and the addendum A4 documents that the file layout follows the canvas-apps planner/builder pattern. Existing-code references appear accurate (the conventions in N6, N7, N8 match the other plugins).

If anything, the PRD is *under-formalized* in one place — the success criteria for "the self-review pillar pass" (§7 #4) leans on external doc, see Done-ness finding — but that's an internal-tool-appropriate level of informality at the boundary of acceptable.

### Findings

None.

## Mechanical notes

- **Glossary drift:** none observed. `clientdata`, `connectionReferences`, `category = 5`, `pac auth list`, `Workflow Definition` all stable.
- **ID continuity:** F1.1–F1.5, F2.1–F2.7, F3.1–F3.5, N1–N9 are contiguous and unique. UJ-1 is the only UJ and is referenced once (in §7 #1). No broken cross-references.
- **Assumptions Index roundtrip:** no dedicated Assumptions Index section — 3 inline `[ASSUMPTION]` tags exist (§3, F1.5, F2.5) but are not indexed. For an internal v1 PRD this is acceptable; for the architecture handoff it would be nicer to have a §11 listing them.
- **UJ persona linkage:** UJ-1 implicitly anchors to §3's "Data3 internal developer" via the "Developer runs ..." phrasing but doesn't name the persona by exact label. Light, fine for a single-persona PRD.
- **Required sections:** Context, Positioning, Users, UJ, FRs, NFRs, Success, Out-of-scope, Dependencies, Open Questions all present. No Glossary, no Assumptions Index, no formal Acceptance section (acceptance lives implicitly in F2.6, F3.4, F3.5).
