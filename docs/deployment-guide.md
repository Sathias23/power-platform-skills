# Deployment Guide

> How `power-platform-skills` itself ships, and how downstream artifacts produced by the plugins are deployed.

## ⚠️ Local fork — NOT publishing to the Microsoft marketplace

This working copy is an **experimental fork for internal use only**. We do NOT publish from this clone:

- The upstream `microsoft/power-platform-skills` marketplace is owned by Microsoft. Any "deploy" or "release" described below refers to the upstream project — it does not apply here.
- Changes made locally in this fork stay local. We are not pushing to `microsoft/power-platform-skills`, not opening a marketplace PR, and not running `scripts/install.js` against the upstream repo URL.
- Install / auto-update mechanisms (`/plugin marketplace add microsoft/power-platform-skills`, `curl … | node`) pull from upstream `main`, not from this fork. If you need to test plugin changes in this fork, use **local plugin development** instead:
  ```bash
  claude --plugin-dir /path/to/this-fork/plugins/<plugin-name>
  ```
- Treat the upstream marketplace info below as **reference documentation** describing how the official project ships — not as instructions for what to do with this clone.

If at some point we DO want to contribute changes upstream, that flow is: fork → branch → PR against `microsoft/power-platform-skills` `main` — never direct-push from this working copy.

## Deploying the marketplace (upstream — reference only)

The `power-platform-skills` repo is **not a deployed application**. Releases happen by merging to `main`:

1. PR is merged into `main`.
2. The marketplace auto-update mechanism (enabled via `scripts/install.js`) pulls the new commit on the next session for any user who has installed plugins.
3. For Power Pages specifically, the per-skill version-check line (`scripts/check-version.js`) compares the local plugin version against `origin/main` and prompts the user if a newer version is available.

There is no build artifact, no container, no package registry publish. The repository IS the distribution.

## Version bumping a plugin

When meaningfully changing a plugin's contracts, behavior, or skill list:

1. Update `plugins/<plugin>/.claude-plugin/plugin.json` — bump the `version` field.
2. If the change affects the marketplace listing (description, keywords, name), also update `.claude-plugin/marketplace.json`.
3. Commit, PR, merge to `main`. The marketplace auto-update mechanism distributes the new version.

Current versions (2026-05-20):

| Plugin | Version | Marketplace name |
|--------|---------|------------------|
| power-pages | 1.3.0 | `power-pages` |
| model-apps | 1.0.6 | `model-apps` |
| mcp-apps | 1.0.0 | `mcp-apps` |
| canvas-apps | 2.1.0 | `canvas-apps` |
| code-apps | 1.0.0 | `code-apps-preview` |

> Note the rename: source directory `plugins/code-apps/` is published as `code-apps-preview` in the marketplace. Preserve this mapping when editing manifests.

## CI gates that must pass before merge

Configured in `.github/workflows/`:

| Workflow | What it enforces |
|----------|------------------|
| `ensure-skill-version-check.yml` | Every `power-pages` SKILL.md has the version-check line immediately after the YAML frontmatter |
| `validate-plugin-names.yml` | Plugin names in marketplace.json are consistent with each plugin's own plugin.json |
| `power-pages-script-tests.yml` | `node --test plugins/power-pages/scripts/tests/` passes (zero-dependency, Dataverse-free) |
| `github-repo-stats.yml` | Repo telemetry / stats collection |

## Downstream deployment surfaces (artifacts the plugins produce)

Each plugin produces a different runtime artifact; here is the authoritative deployment surface for each.

| Plugin | What it produces | Deployment surface | Owner of deploy command |
|--------|-------------------|---------------------|--------------------------|
| `power-pages` | Static SPA bundle (React / Vue / Angular / Astro) | `pac pages upload-code-site` to a provisioned Power Pages site | `/deploy-site` skill |
| `model-apps` | React 17 + Fluent UI v9 single-file `.tsx` generative page | `pac model genpage` deploy to a model-driven app | `/genpage` skill |
| `code-apps` (`code-apps-preview`) | React + Vite + TS sandboxed app with Power Platform connectors | `npx power-apps push` (Power Apps NPX CLI) | `/deploy` skill |
| `canvas-apps` | `.pa.yaml` files for Power Apps Canvas Apps | `compile_canvas` + `sync_canvas` MCP tools against a live coauthoring session | `canvas-app` skill (via Canvas Authoring MCP) |
| `mcp-apps` | Self-contained HTML widget files | No deploy step — host loads HTML directly from the MCP App protocol surface | N/A (artifact handed back to host) |

### Common deployment principles enforced by skill workflows

- **Always pause before deploying.** Skills that modify customer environments end with "Ready to deploy?" via `AskUserQuestion` and call `/deploy-site` (or equivalent) only on user confirmation.
- **Never auto-rollback on partial failure.** Track per-call results, report each item's status, let the user decide remediation.
- **Token refresh during bulk Dataverse operations** — refresh the Azure CLI / Power Platform token roughly every 20 records / 3–4 tables / 60 seconds.
- **Don't fire parallel Dataverse calls without a concurrency cap.** Power Platform throttles aggressively.

## Power Platform authentication

| Surface | Auth mechanism |
|---------|----------------|
| Dataverse Web API (power-pages, code-apps Dataverse) | `az login` → `getAuthToken` helper in `plugins/power-pages/scripts/lib/validation-helpers.js` mints a Dataverse-scoped AAD token |
| Power Platform admin API (activation, cache) | Same `getAuthToken` helper, different scope |
| `pac` CLI (all plugins) | `pac auth create` / `pac auth select` — plugin-agnostic, user-managed |
| Canvas Authoring MCP (canvas-apps) | Inherits the browser-side coauthoring session — no separate auth, but the Studio tab MUST remain open |
| Power Apps NPX CLI (code-apps) | `npx power-apps` reuses the `pac` CLI token |

### Azure CLI flag gotcha

`az login --allow-no-subscriptions` is valid **only on `az login` itself**. Other subcommands (`az account get-access-token`, `az account show`, etc.) reject the flag and exit 2.

If the user has no Azure subscription, suggest plain `az login` first, and only fall back to `--allow-no-subscriptions` on the login step itself.

## Hook-based validation lifecycle (power-pages)

`plugins/power-pages/hooks/hooks.json` registers a `PostToolUse` hook on the `Skill` tool. When a tracked Power Pages skill completes, the dispatcher (`run-skill-posttool-validation.js`) maps the skill name to a validator script (via `scripts/lib/powerpages-hook-utils.js`) and runs it.

Validator output:

- `approve()` from `validation-helpers.js` — passes, skill exit is clean.
- `block(reason)` — surfaces an error to the user before the skill exits.

Hook scripts run on **every** Skill tool use. Keep them fast and gate any debug output behind `process.env.DEBUG` — only true errors should write unconditionally to stderr.

## Security & sensitive data

- Treat any Dataverse / Azure token returned by `getAuthToken` as sensitive. Never log it. Never serialize it into telemetry or generated artifacts.
- Generated YAML / JSON config (table permissions, site settings, web roles) ships into customer environments. Never include sample tokens, hardcoded environment-specific GUIDs, or commented-out credentials.
- Azure Key Vault integration (`plugins/power-pages/scripts/{create-azure-keyvault,list-azure-keyvaults,store-keyvault-secret}.js`) is the supported path for storing site secrets.
