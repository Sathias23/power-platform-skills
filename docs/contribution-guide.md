# Contribution Guide

> See `CONTRIBUTING.md` at the repo root for the Microsoft CLA / legal preamble. This guide focuses on the technical contribution workflow.

## Before you start

- Read this repo's top-level `AGENTS.md` for global conventions.
- Read the plugin-specific `plugins/<plugin>/AGENTS.md` for plugin-local conventions.
- If you're touching `power-pages` specifically, also read `plugins/power-pages/PLUGIN_DEVELOPMENT_GUIDE.md` for the five UX & reliability pillars (loading experience, action verification, sub-agent architecture, deterministic execution, transparency & approval).

## What the existing instruction docs say (highlights)

These are non-negotiable rules called out in the repo's `AGENTS.md` / `CLAUDE.md` files. If you can't follow one, surface the question in the PR — don't silently break it.

### Skill / SKILL.md authoring

- **`allowed-tools` is a comma-separated string.** Not a YAML list. Not a JSON array. Validators reject both.
- **No `hooks:` in skill frontmatter.** Hooks register centrally in `plugins/<plugin>/hooks/hooks.json`.
- **Phase structure is 5–8 phases:** Prerequisites → Discover/Gather → Plan/Review → Implement → **Verify** (mandatory standalone) → Deploy/Summarize. Don't merge or skip phases without updating cross-references.
- **`TaskCreate` all phase tasks upfront** at Phase 1. One task per phase. Each needs `subject` / `activeForm` / `description`.
- **`AskUserQuestion` three times:** after discovery, after plan, before deploy. Work autonomously between checkpoints.
- **Shell-agnostic docs:** No PowerShell cmdlets in `bash` code blocks. Use `<placeholder>` angle-bracket style for variables. Describe filesystem/JSON intent in prose; let the agent use Read/Write/Edit/Glob.
- **`${CLAUDE_PLUGIN_ROOT}/...` paths** in SKILL.md — never hardcode `plugins/<name>/...`.
- **Skill tracking pointer:** Final phase records usage via `> Reference: ${CLAUDE_PLUGIN_ROOT}/references/skill-tracking-reference.md`. New skill? Add it to the mapping table in that file.

### Code (Node.js scripts)

- **Test runner is `node --test` only.** No Jest, Vitest, Mocha, or any other framework. Tests live under `plugins/<plugin>/scripts/tests/`, one `*.test.js` per source file.
- **Every new or modified script ships with tests in the same PR.** Validator changes are not exempt.
- **Default to no comments.** Explain WHY only when non-obvious (hidden constraint, workaround for a specific PAC CLI bug, surprising behavior). Don't explain WHAT well-named code already says.
- **Don't reference the current task** in comments ("added for the X flow") — that rots when the task name is forgotten.
- **Hook scripts run on every Skill tool use.** Gate any debug logging behind `process.env.DEBUG`. Only true errors write to `stderr` unconditionally.
- **DRY:** Always grep `plugins/<plugin>/scripts/lib/` (or `plugins/<plugin>/shared/` for code-apps) before writing a new helper. Then check `references/`. Then check `shared/skills/`.

### Power Platform / Dataverse specifics

- **Use the shared `getAuthToken`** helper in `plugins/power-pages/scripts/lib/validation-helpers.js`. Never shell out to `az` directly for tokens.
- **`az login --allow-no-subscriptions` only on `az login` itself.** Other `az` subcommands reject it (exit 2).
- **Refresh tokens every ~20 records / 3–4 tables / ~60 seconds** during bulk Dataverse operations.
- **No inline PowerShell `Invoke-RestMethod`** for Dataverse calls. Use Node.js scripts with `getAuthToken` + `makeRequest`.
- **Dataverse-backed validation is opt-in.** Default test runs and CI must not require Dataverse connectivity. Gate behind flags like `--validate-dataverse-relationships`.
- **Power Pages only supports SPA frameworks** (React, Vue, Angular, Astro). Server-rendered (Next.js / Nuxt / Remix / SvelteKit) is not supported — never suggest or accept these.

### Cross-plugin shared skills

- Workflow content lives ONCE in `shared/skills/<skill>/<workflow>.md`.
- `SKILL.template.md` carries the frontmatter template with `{{PLUGIN_NAME}}` placeholder.
- Per-plugin wrapper at `plugins/<plugin>/skills/<skill>/SKILL.md` is **thin** — frontmatter + one-line reference. Never duplicate workflow content into the wrapper.
- Editing the shared workflow? Update every per-plugin wrapper in the same PR.

### Marketplace metadata

- Editing the marketplace? Both `.claude-plugin/marketplace.json` AND the affected `plugins/<name>/.claude-plugin/plugin.json` must be updated.
- Marketplace name and per-plugin name can differ (e.g., marketplace name `code-apps-preview` → source dir `plugins/code-apps`). Preserve this mapping.

## Common review pitfalls (caught by PR review repeatedly)

1. **Phase cross-references break silently.** When you renumber/reorder phases, grep the entire skill directory AND `references/` for old phase numbers.
2. **Validators must match the exact stated constraint.** Rule "no exports at all" → block `module.exports` AND bare `exports.foo = …`. Rule "try/catch required" → assert BOTH `try` AND `catch`. Test boundary cases.
3. **Hook scripts run on every Skill tool use.** Unconditional `process.stderr.write` becomes user-visible noise. Gate behind `process.env.DEBUG`.
4. **Template placeholders inside `<script>` blocks need care.** `render-template.js` injects values as-is (no encoding). Safe in HTML text contexts, risky inside JavaScript. Don't declare JS variables with `"__PLACEHOLDER__"` in script blocks; use the DOM or `JSON.stringify`.
5. **Contradictions within a single skill.** If section A says "always use raw fetch" and a framework table later recommends Axios without qualification, reviewers will flag it. Re-read the whole SKILL.md end-to-end before submitting.

## PR checklist (paste this into your PR description)

```markdown
- [ ] Touched a script → added/updated `node:test` coverage in `plugins/<plugin>/scripts/tests/`
- [ ] Touched a SKILL.md → grepped for stale phase numbers and contradictory guidance
- [ ] Touched a shared workflow under `shared/skills/<name>/` → updated every per-plugin wrapper
- [ ] Touched the marketplace → updated BOTH `.claude-plugin/marketplace.json` AND `plugins/<name>/.claude-plugin/plugin.json`
- [ ] No PowerShell cmdlets in `bash` code blocks
- [ ] No JSON-array or YAML-list `allowed-tools`
- [ ] All new skills carry the version-check line (power-pages) — confirmed via `node scripts/ensure-skill-version-check.js --check`
- [ ] No live-Dataverse / Azure dependency in default test runs
```

## CI feedback

Four GitHub Actions workflows guard merges to `main`:

- `ensure-skill-version-check.yml` — Power Pages SKILL.md version-check line presence
- `validate-plugin-names.yml` — Marketplace ↔ plugin.json name consistency
- `power-pages-script-tests.yml` — `node --test` for power-pages
- `github-repo-stats.yml` — Stats only, non-blocking

If a workflow fails, the failure message tells you exactly which file or rule is at fault. Fix the root cause; don't disable the check.

## Reporting issues / asking questions

- Bugs / feature requests → GitHub Issues via the `.github/ISSUE_TEMPLATE/` templates.
- Each plugin also has a `/report-issue` skill (`shared/skills/report-issue/` is the source-of-truth workflow).
- For security disclosures: see `SECURITY.md` at the repo root.
