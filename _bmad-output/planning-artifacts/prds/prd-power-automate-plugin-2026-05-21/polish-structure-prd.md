# PRD Structural Polish Review — Power Automate Plugin v1

**Reviewed file:** `prd.md` (3,088 words, 10 numbered sections, 175 lines)
**Review scope:** structure only — cuts, reorganizations, section consolidation, heading tightening. No scope changes, no new/removed requirements, no challenge to ideas (content is sacrosanct).
**Reader register:** internal Data3 team, downstream consumers = architect + epic-writer.
**Structure model applied:** Strategic/Context (Pyramid) — PRD conventions.

---

## Document Summary

- **Purpose:** Lock v1 scope for an internal Data3 Power Automate plugin (clone/edit/deploy existing flows) so an architect can design and an epic-writer can break down work.
- **Audience:** Data3 internal team — architect (next consumer), epic-writer, v1 pilot (Brad).
- **Reader type:** humans (technical practitioners — preserve scannability and reasoning chains; cut warmth/marketing).
- **Structure model:** Strategic/Context (Pyramid).
- **Current length:** 3,088 words across 10 sections.
- **Overall verdict:** Structurally sound. The document is well-organized for its purpose; recommendations below are tightening, not surgery. Estimated net reduction if all accepted: ~200–280 words (7–9%). No section should be cut entirely.

---

## Structural Map (word counts approximate)

| # | Section | ~Words | Role | Health |
|---|---------|--------|------|--------|
| — | Frontmatter + TL;DR blockquote | 60 | Headline | Good |
| 1 | Context | 240 | Background | Slightly long; some overlap with §2 |
| 2 | Why this plugin exists (positioning) | 410 | Strategic justification | Dense but earned; one redundancy with §1 |
| 3 | Users & personas | 90 | Audience | Tight |
| 4 | User journey (UJ-1 happy path) | 230 | Concrete grounding | Good |
| 5 | Functional requirements (F1–F3) | 870 | Core spec | Healthy; one nested paragraph buries a key constraint |
| 6 | Non-functional requirements (N1–N9) | 530 | Cross-cutting | Healthy; one item has body-of-text rationale that belongs in an open question or appendix |
| 7 | Success criteria | 350 | Definition of Done | Good; one nested checklist duplicates N6 |
| 8 | Out of scope | 130 | Scope fence | Tight; one item duplicates §2 |
| 9 | Dependencies | 160 | External surfaces | Good |
| 10 | Open questions | 200 | Deferred decisions | Good |

---

## Recommendations (priority order)

### 1. MERGE — §1 Context tail into §2 positioning lead
**Where:** §1 paragraphs 3 and 4 (the `add-cloud-flow` gap callout and the "Data3 developers regularly need to edit…" pain restatement) overlap with §2's framing.
**Change:** Cut §1 paragraph 4 entirely (the developer-pain restatement is already implied by §1 paragraphs 1–2 and re-stated in §4 user journey). Move §1 paragraph 3 (the `add-cloud-flow` distinction) to become the second paragraph of §2 — it's a positioning point, not a context point.
**Rationale:** §1 currently does the job of §2 in its back half; the positioning narrative reads cleaner when "what the gap is" lives next to "why we own the gap."
**Impact:** ~80 words saved; tightens the lead-in to the strategic section.
**Comprehension note:** No loss — the developer-pain framing survives in §1 paragraph 1 and §4.

**Before (current §1 ¶3–4):**
> The existing `power-pages/add-cloud-flow` skill **registers** flows into a Power Pages site (metadata + client wiring). It does **not** author them. That gap — code-first authoring of the flow definition itself — is what this plugin fills.
>
> Data3 developers regularly need to edit existing production flows — rewire actions, swap triggers, change expressions, update connector parameters — and the current cost of doing this through the maker portal (no diff, no version control, browser context-switch, error-prone expression authoring) is a recurring drag on delivery.

**After:** §1 ends after paragraph 2. The `add-cloud-flow` distinction moves into §2 right after the "Two adjacent tools already exist" list (extends the competitive surface), framed as: "Note also: this plugin's `power-pages/add-cloud-flow` sibling registers flows into a Power Pages site — it does not author them. That gap — code-first authoring of the flow definition itself — is what this plugin fills."

---

### 2. SIMPLIFY — §2 "Wedge fragility" paragraph
**Where:** §2 final paragraph (the bolded "Wedge fragility" block, ~170 words).
**Change:** Keep the paragraph but break it. The sentence "The durable defenses are not the two pillars above…" runs three clauses (a), (b), (c) inline. Convert to a 3-bullet list. Cut the closing sentence "v1 ships on the wedge; the durable position is the v2 trajectory" — it restates the bullet list's implication.
**Rationale:** This is the most strategically important paragraph in the PRD (it's the answer to "what stops Flow Studio MCP from eating our lunch?"). Burying it as wall-of-prose costs the architect cycles parsing it. A list makes the three structural defenses scannable.
**Impact:** ~25 words saved; major readability gain on a load-bearing paragraph.

**Before:**
> The durable defenses are not the two pillars above — those are competitive at most for one product cycle — but the structural position: (a) this plugin lives in Microsoft's sponsored `power-platform-skills` open-source marketplace alongside 5 sibling plugins, (b) it integrates with the existing `power-pages/add-cloud-flow` registration skill to support end-to-end Power Pages + flow workflows that a standalone MCP server cannot easily match, (c) ALM-first posture (v2 `pac solution` integration) compounds advantage over time. v1 ships on the wedge; the durable position is the v2 trajectory.

**After:**
> The durable defenses are not the two pillars above — those are competitive at most for one product cycle — but the structural position:
> - **Marketplace home.** Lives in Microsoft's sponsored `power-platform-skills` open-source marketplace alongside 5 sibling plugins.
> - **Sibling integration.** Composes with the existing `power-pages/add-cloud-flow` registration skill for end-to-end Power Pages + flow workflows that a standalone MCP server cannot easily match.
> - **ALM-first trajectory.** v2 `pac solution` integration compounds advantage over time.

---

### 3. CUT — Duplicate scope-fence in §8 "Repo-aligned source-control story"
**Where:** §8 bullet "Repo-aligned source-control story" (~50 words).
**Change:** Replace the bullet with a one-liner. The first half ("Flow definitions as on-disk files…the canvas-apps `.pa.yaml` posture is the natural v2 evolution via `pac solution clone`") is restated verbatim from §2 paragraph 4. Keep only "Repo-aligned source-control story (on-disk flow definitions). See §2 — explicitly v2 via `pac solution clone`."
**Rationale:** True duplication — same idea, same length, two locations. §2 carries the strategic framing; §8 only needs the scope fence.
**Impact:** ~35 words saved.
**Comprehension note:** None — §2 still carries the full framing for any reader who lands there.

---

### 4. MERGE — Success criterion #4 checklist with §6 N6 list
**Where:** §7 success criterion #4 (the binary checklist) restates the 5 pillars from §6 N6.
**Change:** Two options, pick one:
- **Option A (preferred — terser):** Replace #4's body with "The plugin honors all five UX/reliability pillars (§6 N6). Each N6 sub-item is the binary acceptance check." Drop the checkbox list.
- **Option B (if architect prefers an in-place tickable list):** Keep #4's checklist but cut §6 N6's numbered list and replace with "See §7 #4 for the binary acceptance list."

Whichever direction, only one location should carry the list verbiage.
**Rationale:** The 5 pillars currently appear in three places — `power-pages/PLUGIN_DEVELOPMENT_GUIDE.md` (referenced), §6 N6 (inlined for testability), and §7 #4 (checkbox-ified). Two of those is enough.
**Impact:** ~60 words saved (Option A) or ~80 words saved (Option B).
**Comprehension note:** Option A keeps the reasoning in §6 (which is the natural home for non-functional requirements) and the acceptance gate in §7. Cleaner separation of "what is required" vs "how we know it's done."

---

### 5. SIMPLIFY — §6 N3 inline mitigation discussion
**Where:** §6 N3 ("Schema validation before deploy") — the paragraph runs ~140 words and mixes the requirement with a multi-sentence mitigation discussion ("Schema validation catches structural errors only — semantic errors…F2.4 planner partially mitigates this via the bundled `ConnectorPatterns.md`…Residual risk: semantic errors…Acceptable for v1; revisit if rejections become frequent…").
**Change:** Keep the first sentence as N3. Move the mitigation/residual-risk paragraph to a new open question #6 in §10, or to a one-sentence "Known limitation:" addendum. The acceptance language ("Acceptable for v1; revisit if rejections become frequent in the pilot failure log") is an open-question phrasing, not an NFR phrasing.
**Rationale:** Non-functional requirements are crisper when each item is "this is the rule." Embedded decision-justification dilutes the contract. The failure-log linkage already exists in §7 #2's "failure log" reference — this is where revisit triggers belong.
**Impact:** ~40 words saved in §6; ~30 added to §10 — net ~10 words. The real win is heading clarity, not raw word count.

---

### 6. RENAME — Section heading polish (low-effort, high-readability)
**Where:** Several headings carry parenthetical narration that belongs in body text.
**Change:**
- §2 "Why this plugin exists (positioning)" → "Positioning" (the "why this plugin exists" framing is the section's lead sentence anyway)
- §4 "User journey (v1 happy path)" → "User journey" (the v1 qualifier is implicit — the whole doc is v1-scoped)
- §F2 "Edit an existing cloud flow (`/cloud-flow`, EDIT mode)" → "Edit an existing cloud flow (`/cloud-flow`)" — "EDIT mode" is meaningful only if there's another mode; v1 has only edit, so the qualifier is noise. (If the architect adds a CREATE mode later, restore the qualifier then.)
- §7 "Success criteria for v1" → "Success criteria" (same reasoning as §4)
**Rationale:** PRD scope is established by the frontmatter (`version_target: v1`) and the title. Re-asserting "v1" in three subheadings adds nothing.
**Impact:** ~12 words; pure heading polish.
**Comprehension note:** None.

---

### 7. REORDER — §2 sub-paragraphs (minor)
**Where:** §2 first three structural paragraphs: "Two adjacent tools…" → "Neither addresses the wedge…" → numbered defense pillars → v2 source-control sentence → "Wedge fragility" paragraph.
**Change:** Move the standalone "A repo-aligned source-control story…is the natural v2 evolution…" sentence out of §2 entirely; it's a §8 (Out of scope) point that breaks the §2 reasoning chain. §8 already carries this with a `pac solution clone` callout. If recommendation #3 lands, the §2 sentence becomes the duplicate-source, and §8's becomes the canonical home.
**Rationale:** §2 should answer "why this plugin exists" without context-switching to "and here's a v2 idea." That interruption forces the reader to hold the v2 sidebar in working memory while parsing the wedge-fragility analysis that follows.
**Impact:** ~45 words moved (or saved entirely if #3 is accepted alongside).
**Comprehension note:** None — the v2 source-control posture lives in §8 where it belongs.

---

## Items explicitly PRESERVED (could look cuttable but earn their keep)

- **§4 User journey UJ-1.** Concrete, named ("New Order Notification flow") and short. This is the single most useful artifact in the PRD for the architect and the epic-writer — it grounds every abstract F-requirement in §5. Do not condense.
- **§5 F2.5 detailed scope breakdown** (trigger parameter edits / trigger swap / action edits / restructures explicitly out of v1). Verbose but every clause is load-bearing for the builder agent's halt conditions. Keep verbatim.
- **§7 Counter-metrics block.** Looks like editorial commentary; is actually the falsification criteria for the success definition. Reads as Amazon-PRFAQ-style anti-goals. Keep.
- **§5 F2.4 bullet on pre-flight connector resolution** — long but earns its length because it ties F2.4 (planner behavior) to §2 pillar 1 (the strategic defense). Preserve the explicit cross-reference; that's exactly the kind of traceability the architect needs.
- **§10 Open questions.** All five are crisp, non-blocking, and named to the right downstream phase. Don't trim.

---

## Items NOT recommending (considered and rejected)

- **Reordering §5 / §6.** Considered moving Non-functional ahead of Functional (some PRD conventions do this). Rejected — the F-requirements here directly motivate several N-requirements (e.g., F2.4's planner motivates N3's mitigation discussion). Forward references would proliferate.
- **Cutting §3 Users & personas.** Considered — only 90 words and arguably already covered by the frontmatter `audience: internal-data3`. Rejected — the explicit "out of scope" half (low-code makers, citizen developers) is doing real scope-fencing work that the frontmatter cannot.
- **Splitting §2 into "Adjacent tools" + "Defensive posture" subsections.** Considered — would aid scannability. Rejected — adds heading weight to a 410-word section that already reads as one coherent argument. Recommendation #2 (bulletize the wedge-fragility close) achieves most of the scannability gain without sub-heading overhead.
- **Adding a TOC.** Rejected — 10 numbered sections at one heading level deep is already a TOC. Markdown render in any tool shows the outline.

---

## Summary

- **Total recommendations:** 7 (4 structural + 1 simplification + 1 rename batch + 1 reorder)
- **Estimated reduction:** ~200–280 words (7–9% of original)
- **Meets length target:** No length target was specified; reductions are opportunistic, not forced.
- **Comprehension trade-offs:** None — every recommendation preserves the load-bearing content. The architect and epic-writer get a tighter, less-duplicated document with the strategic argument (§2) made scannable.
- **Lock-readiness:** With #1, #2, #3, #4 (Option A), and #6 applied, the document is ready to lock. #5 and #7 are nice-to-have polish.

**Recommended apply order if doing a single polish pass:**
1. #6 (renames — 30-second mechanical edit)
2. #3 (§8 duplication cut — one bullet)
3. #2 (§2 wedge-fragility bulletize — high-leverage clarity win)
4. #1 (§1→§2 merge — biggest restructure)
5. #4 (§6 N6 vs §7 #4 dedup — pick Option A)
6. #5 (§6 N3 simplify — optional)
7. #7 (§2 v2-sentence move — folded into #1 and #3 if both land)
