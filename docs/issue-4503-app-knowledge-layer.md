# App Knowledge Layer for AI Assistants (#4503)

## Problem

AI models connected to Buzz have no reliable, current knowledge about Buzz itself — its UI, settings, providers, agent behavior, tools, MCP integration, experimental features, release changes, or known limitations. Models hallucinate settings, confuse Chat vs Agent behavior, or describe features from other versions.

Buzz already has the plumbing: agent MCP server support (`buzz-agent/src/mcp.rs`) and the `buzz-dev-mcp` reference implementation using `rmcp`. What's missing is a **knowledge server** that any connected model can query.

## Design

### New crate: `buzz-kb-mcp`

An MCP server that ships versioned documentation as embedded data and exposes it through MCP tools. It follows the same `rmcp` + `#[tool]` pattern as `buzz-dev-mcp`.

### Knowledge format

Docs stored as structured Markdown files under `crates/buzz-kb-mcp/knowledge/`, embedded at compile time via `include_str!()`. Each file has YAML frontmatter with version metadata:

```markdown
---
title: OpenAI-compatible Provider
category: providers
version: 0.5.4
tags: [openai, api, configuration]
---
# OpenAI-compatible Provider
...
```

Categories: `providers`, `settings`, `tools`, `agent-behavior`, `mcp`, `experimental`, `release-notes`, `known-issues`, `capabilities`.

### MCP Tools

| Tool | Description |
|------|-------------|
| `search_buzz_docs` | Full-text search across all knowledge docs. Returns ranked results with excerpts. |
| `get_buzz_setting` | Get documentation for a specific setting key. |
| `get_buzz_provider` | Get documentation for a specific provider. |
| `get_buzz_release_notes` | Get release notes for the installed version (or a specific version). |
| `search_known_issues` | Search known issues and limitations. |
| `inspect_capabilities` | List supported features, providers, and tools for the installed version. |
| `inspect_config` | Return non-sensitive current config (provider list, enabled features, version). |

### Resolution behavior

When a user asks about Buzz, the agent should:
1. Search the knowledge source before answering
2. Distinguish between: available features, experimental features, planned features, older-version features
3. If docs don't contain an answer, say so instead of guessing

### Architecture fit

```
buzz-kb-mcp  (new crate — stdio MCP server)
  ├─ Uses rmcp (workspace dep, same version as buzz-dev-mcp)
  ├─ Embeds knowledge/ directory at compile time
  ├─ Exposes search/query tools
  └─ Wired into agent's MCP server discovery (like buzz-dev-mcp)

buzz-agent/src/mcp.rs  (existing — MCP client)
  └─ Already spawns MCP servers as child processes via TokioChildProcess
```

### Version awareness

The server reads its own crate version (`CARGO_PKG_VERSION`) and matches it against doc frontmatter. When a doc's `version` field differs from the running version, it annotates the response (e.g., "Note: documented for v0.5.3, you are running v0.5.4").

## Implementation Plan

### Phase 1 — Core crate and knowledge content

1. Create `crates/buzz-kb-mcp/` with Cargo.toml, lib.rs, main.rs
2. Add `buzz-kb-mcp` to workspace `Cargo.toml` members
3. Create initial knowledge docs for:
   - Provider catalog (OpenAI, Anthropic, OpenRouter, Bedrock, local models)
   - Key settings (BUZZ_RELAY_URL, BUZZ_PRIVATE_KEY, provider env vars)
   - Agent behavior (Chat vs Agent, tools, Memory/engrams)
   - Release notes for current version
4. Implement MCP server with search + query tools
5. `cargo check` + `rustfmt` + `clippy` on the new crate

### Phase 2 — Wire into agent discovery

1. Wire `buzz-kb-mcp` into the agent's MCP server startup (like dev-mcp)
2. Test end-to-end: agent can query its own knowledge

### Phase 3 — Desktop integration (optional/future)

1. UI toggle for enabling/disabling the knowledge server
2. Refresh knowledge from remote source

## Open Questions

- Should knowledge be refreshable from a remote URL (e.g., GitHub raw docs), or purely embedded at build time? Embedded is simpler and version-correct; refreshable is more up-to-date.
- Should this be Buzz-specific, or designed as a reusable convention other apps could adopt (as the issue author suggests)? Starting Buzz-specific, with an eye toward extraction later.
