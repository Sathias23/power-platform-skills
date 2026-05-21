# Prose Polish — Power Automate Plugin PRD (v1, pre-lock)

**Target:** `prd.md` (173 lines, 10 sections)
**Reviewer mode:** Clinical copy-edit, internal-team register preserved
**Reader type:** humans (internal Data3 dev team)
**Scope:** Final pass before lock — looking for ambiguous claims, weasel words, fillers, unhelpful passive voice, sentences that say two things and lose both, contradictions or echoes between sections.

## Overall verdict

The prose is in good shape. It has the direct, specific voice you asked us to preserve: short imperatives in the requirements, named owners, em-dash asides that carry the argument, no marketing register. Most of what follows is small. Two items are worth resolving before lock:

1. **One stray author marker** at line 172 (`[NOTE FOR PM]`) — pre-lock artifact, should be resolved or removed.
2. **One UK spelling** at line 85 (`recognise`) in an otherwise US-spelling document.

The rest are micro-edits. Nothing structural, nothing that changes meaning.

---

## Findings (three-column)

| Original Text | Revised Text | Changes |
|---|---|---|
| **§10 Q4, line 172** — `Ship a sanitized fixture in `scripts/tests/fixtures/`? [NOTE FOR PM]` | `Ship a sanitized fixture in `scripts/tests/fixtures/`?` | Strip the unresolved `[NOTE FOR PM]` author marker. This is the final pre-lock pass — either resolve the note or remove it. No other section carries similar markers, so it stands out as forgotten scaffolding. |
| **§5 F2.3, line 85** — `The parser preserves any unknown top-level properties intact per N4 — never strips fields it does not recognise.` | `The parser preserves unknown top-level properties per N4 — never strips fields it does not recognize.` | Two fixes: (a) `preserves … intact` is redundant — `preserves` already implies intact; drop `any` and `intact`. (b) `recognise` is the only UK spelling in the document (rest uses `authoring`, `optimizing`, `organize`); switch to `recognize` for consistency. |
| **§6 N3, line 113** — `Schema validation catches structural errors only — semantic errors (unknown `operationId`, invalid expression syntax) are not caught by JSON Schema.` | `Schema validation catches structural errors only — JSON Schema does not catch semantic errors (unknown `operationId`, invalid expression syntax).` | Passive `are not caught by JSON Schema` flipped to active. Same subject (`JSON Schema`) carries the negation; reads cleaner and matches the active register of the surrounding bullets. |
| **§6 N4, line 114** — `When a change Microsoft makes breaks our schema validator, fail with a clear "schema version mismatch" message rather than corrupting the user's flow.` | `When a Microsoft change breaks our schema validator, fail with a clear "schema version mismatch" message rather than corrupting the user's flow.` | `a change Microsoft makes breaks` chains three noun-ish tokens before the verb and reads stilted. `a Microsoft change` is cleaner with no loss of meaning — Microsoft is the only party that ships these schema changes. |
| **§6 N6.2, line 118** — `Verify uses a different code path than implementation — it does not just trust that the implementation succeeded.` | `Verify uses a different code path than implementation — it does not trust that the implementation succeeded.` | Drop the filler `just`. Removing it strengthens the rule rather than softening it ("does not trust" > "does not just trust"). |
| **§7 #2 third sub-bullet, line 134** — `A **failure log** is maintained in the pilot engagement covering:` | `Brad maintains a **failure log** in the pilot engagement covering:` | Passive with the responsible party named in the parent bullet (`Brad (v1 pilot) has shipped…`). Naming the owner enforces accountability and matches the active voice the rest of §7 uses (`Brad uses it once and reverts…`). |
| **§7 counter-metric #2, line 141** — `more than half the edits Brad attempts during the pilot engagement get abandoned mid-session and finished in the portal instead` | `Brad abandons more than half the edits mid-session and finishes them in the portal` | Two passives (`get abandoned`, `finished`) flipped to active with Brad as subject — consistent with the active phrasing in counter-metric #1 ("edits regularly corrupt flows") and with the §7 #2 fix above. Also drops `during the pilot engagement` as it's redundant with the counter-metric framing. |
| **§2, line 39** — `Flow Studio MCP could add a `clone_existing_flow` capability and a "use Web API" flag in a week and close the capability gap.` | `Flow Studio MCP could add a `clone_existing_flow` capability and a "use Web API" flag in a week and close the gap.` | `capability … capability gap` echoes within one sentence. Drop the second `capability` — the antecedent is unambiguous. |
| **§2, line 41** — `Hand-off with `power-pages/add-cloud-flow` supports end-to-end Power Pages + flow workflows that a standalone MCP server cannot easily match.` | `Hand-off with `power-pages/add-cloud-flow` supports end-to-end Power Pages + flow workflows that a standalone MCP server cannot match.` | `cannot easily match` is a hedge. Either it can match or it can't — and the structural defense argument requires the stronger claim. Drop `easily`. |
| **§9 Dep #4 (helper), line 161** vs **§10 Q1, line 169** — `re-implements the minimal subset locally` (§9) vs `does power-automate re-implement the minimum subset?` (§10) | Pick one: `minimal subset` in both places (recommended — `minimal` is the more idiomatic engineering term here). | Terminology drift between two sections that reference the same concept. `minimum subset` reads as a quantity floor; `minimal subset` reads as "the smallest viable set" — which is what's meant in both spots. |
| **§9 Dep #5 (catalog), line 162** — `Unknown-connector edits halt rather than guess.` | `Unknown-connector edits halt rather than guessing.` | Parallel verb form. `halt` is the verb of the subject (`edits`); the comparison needs the participle form to read as a parallel action ("the edits halt; they do not guess" → `halt rather than guessing`). Alternative: rewrite as `On unknown connectors, the plugin halts rather than guesses` — but the current phrasing only needs the `-ing` fix. |

---

## Items considered and deliberately left alone

These appeared in the scan but on reflection are intentional voice or acceptable for the internal register. Listed so the reader can confirm the call:

- **§1 line 18** — `That single fact … is the constraint that shapes every design decision in this PRD.` "Every design decision" is a mild overclaim, but the rhetorical weight is load-bearing for §2. Keep.
- **§2 line 22** — `the current cost is a recurring drag on delivery`. `recurring drag` is vague but the surrounding sentence already enumerates the specific pain (rewire actions, swap triggers, etc.). Keep.
- **§3 line 48** — `[ASSUMPTION] characteristics:`. Fragment-as-label; matches the `[ASSUMPTION]` markers used elsewhere (§5 F1.5). Consistent. Keep.
- **§5 F3.1, line 101** — `In-place edit only in v1; cross-environment deploy via `pac solution pack` + `ImportSolution` is out of scope.` Echoes §8 line 148. Acceptable because F3.1 is the inline scope guard for that requirement and §8 is the canonical out-of-scope list. Two-place statement is intentional belt-and-braces. Keep.
- **§4 line 67** — `Total elapsed: under 5 minutes wall-clock for a simple change (per success criterion §7.1).` Cross-references §7 #1 in the other direction. Both cross-refs are intentional. Keep.
- **§6 N6.1, line 117** — `Drives the agent spinner.` Fragment for punch. Internal-team register. Keep.
- **§6 N9, line 124** — `Never auto-rollback. Report what landed, what didn't, let the user decide remediation.` Imperative-fragment cadence; works. Keep.
- **§7 counter-metric #1, line 140** — `regularly` is a weasel word, but it is immediately defined by `measured as: more than 1 in 5 edits …`. The definition disarms the weasel. Keep.

---

## Cross-section contradiction / echo check

No contradictions found between sections after careful read-through. The two echoes that exist are both intentional belt-and-braces:
- §5 F3.1 ↔ §8: cross-environment deploy declared out of scope in both places.
- §4 ↔ §7 #1: 5-minute wall-clock budget referenced in both directions.

The one near-contradiction worth flagging for the reader (not for prose change — content question):
- **§5 F2.5 "Single means one action per `/cloud-flow` invocation"** vs **§7 #2 sub-bullet "add or remove an action plus modify an adjacent action in the same `/cloud-flow` session"**. These are interpretable as compatible (add/remove counts as one; modifying an existing action is not an add/remove, so it doesn't trip the single-action limit). But a reader could trip on this. **Not a prose fix** — flagging only in case the PM wants to add a parenthetical to F2.5 like `("single" applies to add/remove; modifying an existing action is not bounded by this rule)`.

---

## Lock readiness

After applying the 10 micro-fixes above (plus removing the `[NOTE FOR PM]` marker and resolving the `minimum`/`minimal` consistency), the document is prose-ready for v1 lock. The voice, structure, and argumentative flow do not need further intervention.
