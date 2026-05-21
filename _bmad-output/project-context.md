---
project_name: 'power-platform-skills'
user_name: 'Brad'
date: '2026-05-20'
sections_completed: ['technology_stack', 'authoring_language', 'plugin_framework', 'dry_shared', 'hooks', 'dataverse', 'testing', 'code_quality', 'workflow', 'dont_miss']
status: 'complete'
rule_count: 100
optimized_for_llm: true
---

# Project Context for AI Agents

_This file contains critical rules and patterns that AI agents must follow when implementing code in this project. Focus on unobvious details that agents might otherwise miss._

---

## Technology Stack & Versions

**Repository type:** Plugin marketplace for Claude Code & GitHub Copilot CLI. Not an application — every plugin is consumed externally via `/plugin marketplace add microsoft/power-platform-skills`.

**Marketplace manifest:** `.claude-plugin/marketplace.json` is the source of truth. Active plugins:
- `power-pages` — Power Pages code sites (SPA frameworks only)
- `model-apps` — Power Apps generative pages for model-driven apps
- `mcp-apps` — MCP App widget generator
- `canvas-apps` — Canvas Apps via Canvas Authoring MCP
- `code-apps-preview` — Power Apps code apps (source dir: `plugins/code-apps`)
- `power-automate` — proposal only, not in marketplace yet

**Plugin runtime:**
- Markdown SKILL.md workflows with YAML frontmatter
- Node.js for scripts; tests via `node --test` (built-in runner) — no Jest/Vitest/Mocha
- No bundlers, no TypeScript build at the repo root

**External CLIs invoked by skills:**
- `pac` (Power Platform CLI) — installed by `scripts/install.js`
- `az` (Azure CLI) — auth/token for Dataverse API
- `dotnet` — **.NET 10 SDK required** for `canvas-apps` (Canvas Authoring MCP server)
- `node`

**MCP servers used by plugins:**
- `canvas-authoring` (canvas-apps) — `mcp__canvas-authoring__*`
- `playwright` (model-apps, power-pages) — `mcp__plugin_<plugin>_playwright__*`
- `microsoft-learn` (power-pages) — docs search/fetch

**Downstream target stacks (sites/apps the plugins build):**
- SPA frameworks: **React, Vue, Angular, Astro only**
- **Server-rendered frameworks (Next.js, Nuxt, Remix, SvelteKit) are NOT supported** — never suggest, scaffold, or accept them in Power Pages

**Repo-level paths to know:**
- `plugins/<name>/` — per-plugin code; each has its own `AGENTS.md` / `CLAUDE.md`
- `shared/skills/<name>/` — cross-plugin shared skill workflows + `SKILL.template.md`
- `scripts/install.js`, `scripts/ensure-skill-version-check.js` — top-level marketplace tooling
- `evals/` — per-plugin evaluation suites (currently `mcp-apps`, `model-apps`, `power-pages`)

## Critical Implementation Rules

### SKILL.md & YAML Frontmatter Rules

**Frontmatter shape (required keys):**

```yaml
---
name: <skill-name>          # kebab-case, matches directory name
description: >-              # when to use; SHOULD list explicit USE WHEN / DO NOT USE WHEN cues
  ...
user-invocable: true|false
argument-hint: <optional>
allowed-tools: Read, Write, Edit, Bash, ...   # COMMA-SEPARATED STRING ONLY
model: opus | sonnet         # opus for complex skills, sonnet for lightweight
---
```

- **`allowed-tools` must be a comma-separated string.** Never JSON array (`["Read", "Write"]`) and never YAML list (`- Read`). Validators will reject these.
- **Do not add `hooks:` to skill frontmatter.** Hooks are registered centrally per plugin (e.g., `plugins/power-pages/hooks/hooks.json`).
- **Plugin version-check line is mandatory** on power-pages skills, immediately after the closing `---`, before the `#` title:

  ```markdown
  > **Plugin check**: Run `node "${CLAUDE_PLUGIN_ROOT}/scripts/check-version.js"` — if it outputs a message, show it to the user before proceeding.
  ```

**Cross-plugin shared skills:**

- Workflow lives in `shared/skills/<name>/<workflow>.md` (single source of truth)
- `shared/skills/<name>/SKILL.template.md` carries the frontmatter template and supports `{{PLUGIN_NAME}}` substitution
- Per-plugin wrapper at `plugins/<plugin>/skills/<name>/SKILL.md` is **thin** — frontmatter + one-line reference to the shared workflow. Never duplicate workflow content into the wrapper.

**Shell-agnostic authoring rule** (applies to all SKILL.md, agent, reference docs):

- Use ` ```bash ` (or plain ` ``` `) fences for cross-platform commands only: `pac`, `az`, `dotnet`, `node`.
- **No PowerShell cmdlets** in code blocks: `Get-ChildItem`, `Test-Path`, `New-Item`, `Get-Content`, `Remove-Item`, `ConvertFrom-Json`, `Invoke-RestMethod`, etc.
- **No PowerShell-only variable syntax** in shell commands: `$var = command`, `$env:...`. Prefer `<placeholder>` angle-bracket style (e.g., `<envUrl>`).
- Repo-runtime placeholders like `**Initial request:** $ARGUMENTS` are allowed in prose/templates — they are not shell commands.
- For filesystem/JSON ops, describe intent in prose and let the agent use `Glob`/`Read`/`Write`/`Edit` — don't prescribe shell commands.

**Path resolution rule:**

- Reference shared docs and scripts via `${CLAUDE_PLUGIN_ROOT}/...` — never hardcode `plugins/<name>/...` from inside a SKILL.md.
- Cross-plugin references (shared skills referencing back) use `${CLAUDE_PLUGIN_ROOT}/../../shared/skills/<name>/...`.

**Node.js script rules:**

- Tests use Node's built-in runner: `node --test plugins/<plugin>/scripts/tests/`. **Don't introduce Jest, Vitest, or Mocha.**
- Every new or modified script needs `node:test` coverage in `scripts/tests/` (validators are not exempt).
- Gate debug logging behind `process.env.DEBUG`. Only true errors go to `stderr` unconditionally — hooks fire on every Skill use and noisy stderr creates user-visible noise.

### Skill Workflow & Plugin Framework Rules

**Phase-wise workflow (mandatory structure):**

- Every skill = 5–8 phases: Prerequisites → Discover/Gather → Plan/Review → Implement → **Verify** (mandatory standalone phase) → Deploy/Summarize.
- Never skip, reorder, or merge phases without updating all cross-references in `references/` docs.
- After any phase renumber/reorder: grep the whole skill directory + referenced files for old phase numbers. Phase cross-references break silently.

**Task tracking (per skill):**

- Create all tasks **upfront at Phase 1 start** using `TaskCreate` — one task per phase.
- Each task needs `subject` (imperative), `activeForm` (present continuous — drives the spinner), `description`.
- Mark `in_progress` when entering a phase, `completed` when leaving it.
- Include a progress-tracking table at the end of the SKILL.md.

**User confirmation gates:**
Pause with `AskUserQuestion` at all of: after gathering requirements, after presenting a plan, after implementation, before deployment. Never auto-deploy.

**Deployment prompt pattern:**
Skills that modify site artifacts end by asking "Ready to deploy?" and invoke `/deploy-site` on yes. Don't short-circuit by calling deploy automatically.

**Agent spawning:**

- Process **sequentially**, not in parallel.
- Wait for each agent to complete, then present its output for user approval before continuing.

**Skill tracking (final phase):**
Every skill records usage via the pointer pattern:

```markdown
> Reference: ${CLAUDE_PLUGIN_ROOT}/references/skill-tracking-reference.md
```

When adding a new skill, also add its entry to the skill-name mapping table in `references/skill-tracking-reference.md`.

**Git commit cadence (skills that produce artifacts):**
Commit after every significant milestone — each page/component, design foundation pass, phase completion. Don't bundle everything into one final commit.

### DRY / Shared Module Rules

**Power Pages shared modules (do not duplicate these):**

- `plugins/power-pages/scripts/lib/validation-helpers.js` — boilerplate, path finders, `getAuthToken`, `makeRequest`, constants
- `plugins/power-pages/scripts/lib/powerpages-config.js` — loading/parsing `.powerpages-site` table-permission and site-setting YAML; keep it focused on parsing only (validation/business rules go in separate validator modules)
- `plugins/power-pages/scripts/generate-uuid.js` — UUID generation; never copy into skill-specific directories
- `plugins/power-pages/scripts/lib/powerpages-hook-utils.js` — hook lifecycle helpers
- `plugins/power-pages/references/` — shared reference docs; link via `${CLAUDE_PLUGIN_ROOT}/references/...`

Before writing a new helper, check `scripts/lib/` and `references/` for an existing one.

**Templates:**
Use `__PLACEHOLDER__` tokens (e.g., `__SITE_NAME__`) in template files. Replaced during scaffolding. The `gitignore` file is stored without the dot prefix and renamed to `.gitignore` on scaffold.

### Hooks (Power Pages)

- Hooks register **centrally** in `plugins/power-pages/hooks/hooks.json` using `PostToolUse` with `matcher: "Skill"`. Validation runs when a tracked skill completes.
- Hook scripts fire on **every** Skill tool use. Keep them fast. Gate any debug output behind `process.env.DEBUG`. Only true errors go to `stderr`.

### Power Platform / Dataverse Integration Rules

**Auth & token handling:**

- Use the shared `getAuthToken` helper from `scripts/lib/validation-helpers.js`. Never shell out to `az` directly for tokens.
- **`az login --allow-no-subscriptions` is valid only for `az login`.** Other subcommands (`az account get-access-token`, `az account show`) reject it with exit 2.
- Recommend plain `az login` first; only fall back to `--allow-no-subscriptions` if the user has no Azure subscription (the token still works for Dataverse).
- Refresh tokens every ~20 records / 3–4 tables / ~60 seconds during bulk operations.

**Dataverse API calls:**

- Use deterministic Node.js scripts under the skill's `scripts/` directory.
- Import `getAuthToken` and `makeRequest` from `scripts/lib/validation-helpers.js`.
- **Never** use inline PowerShell `Invoke-RestMethod` — scripts are reliable, testable, cross-platform.

**Dataverse-backed validation:**
Must stay opt-in for local runs. Do not require live Dataverse connectivity in CI or default test runs. Gate behind explicit flags such as `--validate-dataverse-relationships`.

**Graceful failure (bulk ops):**
Track per-call results, never auto-rollback, report failures clearly, continue with remaining items. Surface a summary at the end.

### Testing Rules

**Runner:**

- Use Node's built-in test runner only: `node --test plugins/<plugin>/scripts/tests/`.
- **Do not introduce** Jest, Vitest, Mocha, Ava, or any other test framework — keep the zero-dependency rule.

**File layout:**

- One `*.test.js` per script/module under test (mirror the source file name: `validation-helpers.js` → `tests/validation-helpers.test.js`).
- Tests live under each plugin's `scripts/tests/` directory — not co-located with sources.

**Coverage scope:**

- **Every** new or modified script ships with `node:test` coverage in the same PR. **Validator changes are not an exception.**
- Test boundary cases when implementing a validator: if the rule is "no exports at all", verify both `module.exports` AND `exports` are blocked. If the rule is "try/catch required", verify the validator catches code with only `try` and code with only `catch`.

**Dataverse / live-service tests:**

- Default test runs must pass with **no Dataverse / Azure connectivity**.
- Live-service checks are opt-in: gate behind explicit flags like `--validate-dataverse-relationships`. CI must not require them.

**Eval suites:**

- Per-plugin evals live under `evals/<plugin>/` (currently `mcp-apps`, `model-apps`, `power-pages`).
- Treat evals as long-running, separate from `node --test` unit coverage. Don't fold eval scenarios into the unit test directory.

**What NOT to test:**

- Don't write tests that exercise the Markdown SKILL.md content directly. SKILL.md is consumed by an LLM; behavioural correctness is covered by evals, not unit tests.
- Don't snapshot generated YAML/JSON output if the only assertion is "produces a file" — assert on the schema/shape that actually matters.

### Code Quality & Style Rules

**File organization (within a plugin):**

- Skills: `plugins/<plugin>/skills/<skill-name>/SKILL.md` (kebab-case directory and `name:`)
- Agents: `plugins/<plugin>/agents/<agent-name>.md`
- Shared scripts: `plugins/<plugin>/scripts/` and `plugins/<plugin>/scripts/lib/`
- Script tests: `plugins/<plugin>/scripts/tests/`
- Shared references: `plugins/<plugin>/references/`
- Templates: stored next to the consuming skill, with `__PLACEHOLDER__` tokens

**Naming conventions:**

- Skill directory + `name:` frontmatter: **kebab-case**, identical (`add-data-source`, `setup-auth`).
- Node.js helper files: **kebab-case** (`validation-helpers.js`, `powerpages-config.js`).
- Reference docs: **kebab-case** (`skill-tracking-reference.md`).
- Templates: prefixed `__SOMETHING__` placeholders, e.g., `__SITE_NAME__`.

**Comments & docs (Node.js scripts):**

- Default to no comments. Explain WHY only when non-obvious (hidden constraint, workaround for a specific Power Platform / PAC CLI bug, surprising behaviour).
- Don't explain WHAT well-named code already says. Don't reference the current task or callers ("added for the X flow") — that rots.
- Don't write planning/decision/analysis docs unless the user asks. Work from conversation context.

**Markdown style (SKILL.md, agent, reference docs):**

- Keep AGENTS.md and CLAUDE.md concise — detailed docs belong in `PLUGIN_DEVELOPMENT_GUIDE.md` or individual SKILL.md / agent files.
- No emojis in files unless the user explicitly asks. (Frontmatter and skill content occasionally use ⚠️ / ✅ for must-read callouts — match the existing style of the file you're editing rather than adding new ones.)
- Use fenced code blocks with explicit language hints (` ```bash `, ` ```yaml `, ` ```markdown `) so syntax highlighting works in the marketplace UIs.

**Guidance consistency (within a single skill):**

- If one section says "always use raw fetch", a framework-specific table later in the same file must not recommend a different HTTP client without qualification. Reviewers will flag contradictions — re-read the whole file before submitting.

**Marketplace metadata:**

- Each plugin has both `.claude-plugin/plugin.json` (per-plugin) and an entry in the root `.claude-plugin/marketplace.json`. **Both must be updated** when adding, renaming, or removing a plugin.
- Marketplace `name` and per-plugin `name` can differ (e.g., marketplace name `code-apps-preview` → source dir `plugins/code-apps`); preserve the existing mapping.

### Development Workflow Rules

**Local plugin development:**

- Test a plugin in isolation by launching the agent with its directory:

  ```bash
  claude --plugin-dir /path/to/power-platform-skills/plugins/<plugin-name>
  ```

- There is **no root-level build, lint, or test command**. All tooling lives inside each plugin.

**Marketplace install path (what users actually run):**

- `/plugin marketplace add microsoft/power-platform-skills` then `/plugin install <plugin>@power-platform-skills`.
- Or `scripts/install.js` (Node) — auto-installs `pac` CLI, registers the marketplace, enables auto-update.

**Branches & commits:**

- Default branch: `main`. PRs target `main`.
- Use conventional, terse commit messages — match the style visible in `git log`. No mandated prefix scheme. Subject under ~72 chars.
- Commit after every significant milestone inside a skill workflow (per-page, per-component, per-phase). Don't bundle a whole feature into one final commit.

**Pull-request hygiene:**

- Touch a script → ship `node:test` coverage in the same PR.
- Touch a SKILL.md → re-grep the plugin for stale phase numbers, contradicting guidance, and stale `${CLAUDE_PLUGIN_ROOT}/...` paths.
- Touch a shared workflow under `shared/skills/<name>/` → update every per-plugin wrapper at `plugins/*/skills/<name>/SKILL.md` in the same PR (frontmatter alignment, `{{PLUGIN_NAME}}` substitution).
- Touch the marketplace → update both `.claude-plugin/marketplace.json` and the affected `plugins/*/.claude-plugin/plugin.json`.

**Plugin version-check pipeline:**

- `scripts/ensure-skill-version-check.js` enforces that every Power Pages SKILL.md carries the version-check pointer line. Run it locally before pushing skill-shaped changes.

**Deployment (for skills that produce site artifacts):**

- Always end with "Ready to deploy?" — call `/deploy-site` only on user confirmation.
- Never auto-rollback on partial failure during deploy. Report each item's status; let the user decide remediation.

**Activated downstream sites:**

- `pac` CLI is the authoritative deployment surface for code-apps, model-apps, and Power Pages sites.
- The `power-platform-skills` repo itself is not deployed — releases happen by merging to `main`, which the marketplace auto-update mechanism consumes.

### Critical Don't-Miss Rules (Anti-Patterns & Gotchas)

**Common review pitfalls (these have triggered repeat PR feedback — check before submitting):**

1. **Phase cross-references break silently.** Renaming/reordering phases in a SKILL.md? Grep the entire skill directory AND `references/` for the old phase number. Update the Key Decision Points section and any cross-skill links.

2. **Validators must match the exact stated constraint.**
   - Rule "no exports at all" → block all `module.exports` and bare `exports.foo = ...`, not just an allowlist of names.
   - Rule "try/catch required" → assert BOTH `try` AND `catch` exist. Test the case with only one.
   - Re-read the constraint verbatim from the SKILL.md before writing the validator. Test boundary cases.

3. **Hook scripts run on every Skill tool use.** Unconditional `process.stderr.write` becomes noise the user sees on every skill invocation. Gate debug output behind `process.env.DEBUG`. Reserve unconditional stderr for real errors.

4. **Template placeholders inside `<script>` blocks need special care.** `render-template.js` injects values as-is (no encoding). Safe in HTML text contexts, risky inside JavaScript. **Don't declare JS variables with `"__PLACEHOLDER__"` in script blocks** — read from the DOM or use `JSON.stringify` for JS contexts.

5. **Contradictions within a single skill.** If section A says "always use raw fetch" and a later framework table recommends Axios without qualification, reviewers will flag it. Re-read the whole file end-to-end before submitting.

**Anti-patterns (never do these):**

- **Duplicating logic** that already exists in `scripts/lib/validation-helpers.js`, `scripts/lib/powerpages-config.js`, `scripts/generate-uuid.js`, or `references/`. Always grep first.
- **Inline PowerShell** for Dataverse / Azure API calls. Use Node.js scripts importing `getAuthToken` + `makeRequest`.
- **PowerShell cmdlets** in `bash` code blocks across SKILL.md / agent / reference docs (`Get-ChildItem`, `Invoke-RestMethod`, `Test-Path`, `$env:...`, etc.).
- **Suggesting Next.js / Nuxt / Remix / SvelteKit** for Power Pages — only SPA frameworks (React, Vue, Angular, Astro) are supported.
- **`allowed-tools` as YAML list or JSON array.** Always comma-separated string.
- **Hooks defined in skill frontmatter.** Hooks register centrally in `plugins/<plugin>/hooks/hooks.json`.
- **`az ... --allow-no-subscriptions`** on any subcommand other than `az login`. It causes exit 2 with an unrecognized-argument error.
- **Auto-deploying or auto-rolling-back.** Always pause for user confirmation; report partial failures, never silently revert.
- **Parallel agent spawns** for skill sub-tasks. Sequential only; wait + present output for approval between agents.
- **Adding Jest / Vitest / Mocha** or any other test framework. `node --test` only.
- **Skipping `node:test` coverage** when changing or adding a script (validators are not exempt).
- **Requiring live Dataverse / Azure connectivity** in default test runs or CI — must be opt-in via explicit flag.
- **Hardcoded `plugins/<name>/...` paths** inside a SKILL.md. Use `${CLAUDE_PLUGIN_ROOT}/...`.

**Security:**

- Treat any Dataverse / Azure token returned by `getAuthToken` as sensitive. Never log it. Never serialize it into telemetry or generated artifacts.
- Generated YAML / JSON config (table permissions, site settings) ships into customer environments. Don't include sample tokens, environment-specific GUIDs hard-coded, or commented-out credentials.

**Performance / API hygiene:**

- Bulk Dataverse operations: refresh the Azure token roughly every 20 records / 3–4 tables / 60 seconds — whichever comes first. Long-running scripts will otherwise hit 401s mid-run.
- Don't fire Dataverse calls in parallel without a small concurrency cap — Power Platform throttles aggressively.

---

## Usage Guidelines

**For AI Agents:**

- Read this file before implementing any code in the `power-platform-skills` repo.
- Follow ALL rules exactly as documented. When in doubt, prefer the more restrictive option.
- For plugin-specific guidance, also read the plugin's own `AGENTS.md` / `CLAUDE.md` (e.g., `plugins/power-pages/AGENTS.md`).
- Update this file when new patterns emerge or existing rules go stale.

**For Humans:**

- Keep this file lean and focused on agent-relevant rules — detailed how-tos belong in `PLUGIN_DEVELOPMENT_GUIDE.md` or individual SKILL.md files.
- Update when the technology stack, marketplace shape, or shared modules change.
- Review periodically and prune rules that become obvious or no longer apply.

Last Updated: 2026-05-20
