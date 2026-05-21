---
title: Structural review — Power Automate plugin PRD addendum
created: 2026-05-21
reviewer: bmad-editorial-review-structure
target: addendum.md
scope: structural improvements within the addendum only (no moving material to/from PRD body)
---

# Structural review — addendum.md

## Document Summary

- **Purpose:** Hold material that earned its place from PROPOSAL.md and external research but does not belong in the PRD body; serve as a direct reference for downstream architecture, epics, and design documents.
- **Audience:** Architects and epic-writers (downstream consumers of this PRD set).
- **Reader type:** humans (with LLM-reader sympathies — these readers will scan, cross-reference, and quote)
- **Structure model:** Reference/Database with Strategic/Pyramid framing per section. Random-access expected (an epic-writer will jump to A4; an architect will jump to A3, A7).
- **Current length:** ~1,100 words across 8 sections (A1–A8) + frontmatter.
- **Overall verdict:** Structurally sound. The document is tight, well-formatted, and serves its downstream audience. Recommendations below are refinements, not surgery. No section needs cutting.

---

## Section-by-section diagnosis

| # | Section | Format | Verdict |
|---|---------|--------|---------|
| A1 | Authoring surface: options considered and rejected | Table | Strong. Minor reorder suggested (chosen-first). |
| A2 | `clientdata` shape | Code block + 1-sentence framing | Tight. Keep as-is. |
| A3 | Agent architecture (planner + builder) | Prose + bullets | Good. Convention-quote at end is useful for architects. |
| A4 | Proposed plugin shape (post-v1 target) | File tree with inline v1/v2 tags | Excellent for epic-writers. Keep. |
| A5 | Competitive landscape | Bullets + thesis sentence | Bury-the-lede issue. Lead with the thesis. |
| A6 | Pain points evidence | Bold-lead bullets | Strong, consistent pattern. Keep. |
| A7 | NFR footguns (the long form) | Bold-lead bullets | Misplaced — design constraint, not evidence. Heading subtitle is filler. |
| A8 | References | Link list | Standard. Keep. |

---

## Recommendations (prioritized)

### 1. REORDER — Move A7 (NFR footguns) before A5 (Competitive landscape)

**Rationale:** A7 is design-constraint material that architects will consume alongside A3 (agent architecture) and A4 (plugin shape). A5 and A6 are research/justification material — a different reader-type concern. Grouping decisions+design (A1–A4, A7) before research (A5–A6) before references (A8) tightens the random-access flow and reduces cross-section hopping.

**Proposed new order:** A1 → A2 → A3 → A4 → A7 → A5 → A6 → A8.

**Renumbering:** Rename to keep numeric integrity:
- A5 (new) = current A7 (NFR footguns)
- A6 (new) = current A5 (Competitive landscape)
- A7 (new) = current A6 (Pain points evidence)
- A8 unchanged.

**Impact:** No words cut. Significant flow improvement for architects (the heaviest downstream consumer).

**Caveat:** If the PRD body or other downstream docs already link to "A5" / "A6" / "A7" by anchor, renumbering breaks those links. Audit incoming links first; if any exist, prefer renaming-but-not-renumbering (e.g., keep A7 as-is but move A5/A6 to the end as "A7-research" / "A8-research" — uglier but link-safe).

### 2. RENAME — A7 heading "NFR footguns (the long form)" → "NFR footguns (design constraints)"

**Rationale:** "(the long form)" suggests this is verbose backup material; in fact it's the substantive constraint list that architects need. "(design constraints)" signals function, not size. Also clarifies why it sits next to A3/A4 after the reorder.

**Impact:** ~3 words. Heading clarity.

### 3. SIMPLIFY — A5 lead-with-thesis (pyramid)

**Rationale:** The closing sentence "Nobody combines AI agent + clone-existing-flow + Web API authoring + Microsoft-sanctioned surfaces. That's the defensible niche." is the thesis. A skimming epic-writer should hit it first.

**Before:**
> Captured from external research run 2026-05-21. Justifies the wedge framing in PRD §2.
>
> - **Microsoft Copilot in Power Automate.** ...
> - **Flow Studio MCP** ...
> - **GitHub Copilot (generic).** ...
> - **PAC CLI.** ...
>
> Nobody combines AI agent + clone-existing-flow + Web API authoring + Microsoft-sanctioned surfaces. That's the defensible niche.

**After:**
> **Thesis:** Nobody combines AI agent + clone-existing-flow + Web API authoring + Microsoft-sanctioned surfaces. That is the defensible niche. Evidence (captured 2026-05-21; justifies PRD §2 wedge framing):
>
> - **Microsoft Copilot in Power Automate.** ...
> - **Flow Studio MCP** ...
> - **GitHub Copilot (generic).** ...
> - **PAC CLI.** ...

**Impact:** Same word count, lead-with-conclusion structure for random-access readers.

### 4. REORDER (within A1) — Chosen rows first, rejected rows second

**Rationale:** A1 is a decision table. Pyramid principle says the decision comes first. Currently rows 1–2 are rejected, row 3–4 are chosen, row 5 is "not available." Reorder to: chosen (rows 3, 4) → rejected (rows 1, 2) → unavailable (row 5). Renumber the `#` column accordingly.

**Alternative:** Add a "Status" column header rename from `Verdict` → `Decision`, and sort by decision-class. Either works.

**Impact:** Same word count. Reader sees what was picked before reading what was discarded — useful when an architect lands on the section cold.

### 5. SIMPLIFY (optional) — Strip A4 framing sentence

**Rationale:** "For reference. v1 ships a subset; the full target is preserved here so the team knows the trajectory." duplicates what the section heading "(post-v1 target)" already signals. Could cut to: "v1 ships a subset; full target shown for trajectory."

**Before:** "For reference. v1 ships a subset; the full target is preserved here so the team knows the trajectory." (20 words)

**After:** "v1 ships a subset; full target shown for trajectory." (9 words)

**Impact:** ~11 words. Minor.

### 6. PRESERVE — A3 convention-quote, A4 file tree, A2 code block

**Rationale:** These are exactly the artifacts downstream consumers will quote or copy. Resist any temptation to compress them. The A3 quote of `_bmad-output/project-context.md` is load-bearing for architects deciding sub-agent topology.

**Impact:** 0 words.

### 7. QUESTION — A6 anecdotal item ("Connector documentation friction")

**Rationale:** The final bullet ends with "(anecdotal)." Every other bullet has a concrete artefact (URL, quote, ticket ID). The anecdotal item dilutes the evidentiary weight of the section. Either: (a) cut it, (b) find a concrete source, or (c) move it to a clearly-labeled "Unsourced signals" sub-bullet so the strong evidence stays clean.

**Impact:** ~25 words if cut. Author decision.

---

## What is NOT recommended

- **No section should be cut.** Every A-section is referenced by either architecture, epics, or design downstream.
- **No section should be merged.** A3 (agent architecture) and A4 (plugin shape) are tempting to merge, but A3 is conceptual (planner/builder responsibilities) and A4 is structural (file layout); they serve different reader queries.
- **No section should be split.** A7 is dense but it's a single concern (design constraints); splitting it would scatter related footguns.
- **Tables vs prose:** Current choices are correct. A1 is genuinely tabular (option × verdict × rationale). A5–A7 are bullets because each item is heterogeneous. No conversions needed.

---

## Summary

- **Total recommendations:** 7 (1 reorder of sections, 1 rename, 2 simplify, 1 in-section reorder, 1 preserve, 1 question).
- **Estimated word reduction:** ~15–40 words (negligible). This document is already lean.
- **Primary structural win:** Recommendation #1 (reorder A7 next to A3/A4) materially improves the architect's reading path.
- **Meets length target:** No target specified; document is already at appropriate density for its purpose.
- **Comprehension trade-offs:** None. All recommendations preserve content; most preserve word count.
- **Overall:** Document is structurally sound. Recommended changes are refinements that sharpen an already-effective reference doc.
