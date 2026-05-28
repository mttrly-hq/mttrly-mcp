# Changelog

All notable changes to the mttrly remote MCP server are documented here.

This repository is the public, docs-only home for the mttrly MCP. The server itself runs at https://api.mttrly.com/mcp and is not installable from this repo — see the README for client setup.

## [0.1.2] - 2026-05-21

### Added
- Initial public publish of the mttrly remote MCP server under Variant A (remote-only, OAuth 2.1 + PKCE, no npm package).
- Documentation, license, security policy, and machine-readable metadata for catalog ingestion:
  - `README.md` — connect instructions for Claude.ai / Claude Desktop, Claude Code, Codex CLI, and Cursor.
  - `LICENSE` — MIT.
  - `SECURITY.md` — vulnerability disclosure policy (mttrly@mttrly.com).
  - `server.json` — MCP Registry server metadata (`com.mttrly/mcp`, schema 2025-12-11, `remotes` → `https://api.mttrly.com/mcp`).
  - `glama.json` — Glama server metadata.

### Catalog presence
- **MCP Registry** — `com.mttrly/mcp` v0.1.2 published with DNS-based domain authentication for `mttrly.com`.
- **Glama** — listed as both a Connector (auto-ingested from MCP Registry) and a Server (https://glama.ai/mcp/servers/mttrly-hq/mttrly-mcp).
- **Smithery** — listed at https://smithery.ai/servers/dvmaslennikov/mttrly with 40 tools scanned via OAuth.

### Capabilities
- 40 tools total: 21 read-only inspection tools and 19 execute tools, gated by plan.
- 25 documented error codes returned through the MCP error channel.
- Streamable HTTP transport at `https://api.mttrly.com/mcp` with RFC 9728 OAuth protected-resource metadata.
