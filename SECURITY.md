# Security Policy

## Reporting

Report security issues privately at [mttrly@mttrly.com](mailto:mttrly@mttrly.com).

Please include:

- affected endpoint or MCP client
- observed behavior and expected behavior
- timestamps, request IDs, or MCP error codes when available
- whether the issue happened during discovery, OAuth login, token exchange, or a tool call

Do not include live secrets, API keys, OAuth tokens, or private server credentials in reports.

## Scope

In scope:

- hosted remote MCP endpoint at `https://api.mttrly.com/mcp`
- OAuth discovery and token flow served from `https://app.mttrly.com`
- public MCP metadata files: `server.json`, `glama.json`, and README instructions

Out of scope:

- social engineering
- denial-of-service testing without written permission
- vulnerabilities in third-party MCP clients
- issues caused by user-managed server configuration outside mttrly control

We aim to acknowledge credible reports within 72 hours.
