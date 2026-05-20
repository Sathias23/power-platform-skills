# Architecture — `mcp-apps`

> Plugin v1.0.0 — pure-markdown plugin that generates self-contained HTML widgets using the MCP Apps protocol.

## Executive summary

`mcp-apps` is the leanest plugin in the repo. **No `.mcp.json`. No Node.js scripts. No agents. No hooks. No tests.** It is a single skill (`/generate-mcp-app-ui`) that takes a tool description plus JSON sample output and produces a single self-contained HTML file using the [MCP Apps protocol](https://modelcontextprotocol.io/extensions/apps/overview). The resulting widget is hostable in any MCP Apps client (Claude, ChatGPT, VS Code, Microsoft 365 Copilot).

## Technology stack

| Category | Technology | Purpose |
|----------|-----------|---------|
| Authoring | Markdown SKILL.md + YAML frontmatter | Skill workflow |
| Downstream output | Plain HTML (CDN-loaded dependencies) | The generated widget — no build step |
| Downstream libraries | `@modelcontextprotocol/ext-apps` (CDN), Fluent UI (CDN) | Widget runtime |
| Downstream themes | Light + dark, auto-detected | Theme tokens from MCP Apps protocol |

## Architecture pattern

**Single-skill generator.** `/generate-mcp-app-ui` takes natural-language description + JSON sample, produces an HTML file. After generation, the user iterates by describing changes in chat ("make it more colorful", "add a chart", "switch to cards"). Each iteration regenerates the HTML.

## Skills inventory

| Skill | Purpose |
|-------|---------|
| `/generate-mcp-app-ui` | Generate a single self-contained MCP App HTML widget |
| `/report-issue` | Bug report (shared cross-plugin skill) |

## Source tree

```text
plugins/mcp-apps/
├── .claude-plugin/plugin.json
├── README.md                          # NO AGENTS.md / CLAUDE.md — README is the only doc
├── references/
│   ├── mcp-apps-reference.md          # MCP Apps API, Fluent UI components, CDN patterns
│   └── design-guidelines.md           # Visual defaults, theme tokens
├── samples/
│   ├── flight-status-widget.html      # Read-only sample
│   └── weather-refresh-widget.html    # Interactive sample (callServerTool)
└── skills/
    ├── generate-mcp-app-ui/SKILL.md
    └── report-issue/SKILL.md          # Thin wrapper of shared/skills/report-issue/
```

## Key concepts

### MCP Apps protocol

A protocol for embedding interactive widgets inside MCP tool responses. The host (Claude, ChatGPT, etc.) loads the HTML and provides the widget with a runtime API for theming, tool callbacks, and host-provided data.

### Self-contained HTML

The output is **a single HTML file**:
- All CSS inline
- All JS inline
- Dependencies loaded from public CDNs (`@modelcontextprotocol/ext-apps`, Fluent UI)
- No build step, no bundler, no package manager
- Light + dark theme auto-detected via MCP Apps theme tokens
- Polished defaults via Fluent UI components

### Iteration loop

After the initial generation, the user refines by describing changes in chat. The skill regenerates the HTML in-place each iteration.

## Data architecture

No persistent data owned by the plugin. The generated widget receives its data from the host either:
1. **Statically** — provided once when the widget is loaded (e.g., flight-status sample), OR
2. **Dynamically** — via `callServerTool` callbacks back to the original MCP tool (e.g., weather-refresh sample).

## API design

The plugin neither exposes nor consumes any service-side API. The output HTML uses two protocol surfaces:

| Surface | Provider | Purpose |
|---------|----------|---------|
| MCP Apps protocol (`window.openai`, etc.) | MCP Apps host | Theme, tool callbacks, data |
| Fluent UI CDN | `cdn.jsdelivr.net` / `unpkg` | UI components, icons |

## Component overview

The widget is what the plugin generates — not what it ships. There are no reusable components inside the plugin source. The two `samples/` HTML files are intentional reference patterns:

| Sample | Pattern demonstrated |
|--------|----------------------|
| `flight-status-widget.html` | Static data widget — receives JSON once at load |
| `weather-refresh-widget.html` | Interactive widget — uses `callServerTool` for live data refreshes |

## Development workflow

```bash
# Local development
claude --plugin-dir /path/to/plugins/mcp-apps

# Skill invocation
/generate-mcp-app-ui Show travel attractions on an interactive map

Here's my tool's test output:
{"attractions":[...]}
```

The skill produces an HTML file. Open it in a browser or load it in an MCP Apps host to verify behavior.

## Deployment

There is no deployment surface for the *plugin's output* — the host loads the generated HTML directly when the MCP tool response references it. The plugin itself is distributed via the marketplace (`/plugin install mcp-apps@power-platform-skills`).

## Testing strategy

| Layer | Tooling | Location |
|-------|---------|----------|
| Unit | (none — no scripts) | — |
| Eval | Eval framework | `evals/mcp-apps/generate-mcp-app-ui/evals.json` — **53 eval test cases** + type-mismatch stress tests |
| Eval runbook | Markdown | `evals/mcp-apps/generate-mcp-app-ui/eval-runbook.md` |
| Manual | Open the HTML in a browser, then in an MCP Apps host | Validated via the samples |

The eval suite is the most extensive in the repo for a single skill — 53 widget types covering maps, charts, dashboards, tables, cards, and stress tests for type-mismatch handling.

## Open risks / debt

- **No script tests because there are no scripts.** If file-system or generation utilities are added later, follow the `power-pages` convention.
- **The two samples must stay in sync with the protocol.** When the MCP Apps protocol evolves (new theme tokens, new tool callback shapes), update both samples in the same PR.
- **README is the only authoring guidance** — no separate AGENTS.md / CLAUDE.md exists for this plugin. If conventions diverge from the repo-level `AGENTS.md`, add a plugin-level one.
