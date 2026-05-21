# mttrly MCP Server

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
[![smithery badge](https://smithery.ai/badge/dvmaslennikov/mttrly)](https://smithery.ai/servers/dvmaslennikov/mttrly)

Remote MCP access to mttrly, an AI SRE agent for server health, diagnostics, playbooks, approvals, and audit history.

## Scope

mttrly MCP is a remote Streamable HTTP connector. Every tool call reaches the mttrly Central API first; Central then talks to connected agents over the existing secure agent channel. A standalone agent does not expose a local MCP surface.

Variant A is intentionally remote-first: public catalogs point at the hosted endpoint, OAuth handles user authorization, and no npm package is required for normal client setup.

## Connect

### Claude.ai and Claude Desktop

Use the connector UI:

1. Open Settings -> Connectors.
2. Choose Add custom connector.
3. Enter `https://api.mttrly.com/mcp`.
4. Complete the OAuth sign-in flow.

Do not put the remote URL into `claude_desktop_config.json`; that file is for local stdio MCP servers.

### Claude Code

```bash
claude mcp add --transport http mttrly https://api.mttrly.com/mcp
```

### OpenAI Codex

Add this to `~/.codex/config.toml`:

```toml
[mcp_servers.mttrly]
url = "https://api.mttrly.com/mcp"
```

Then complete OAuth login:

```bash
codex mcp login mttrly
```

### Cursor

Add `https://api.mttrly.com/mcp` as a remote MCP server URL. Cursor should discover the protected resource metadata from the server challenge and start OAuth.

## Auth

- Remote transport uses OAuth 2.1 with PKCE S256 against `https://app.mttrly.com`.
- The protected resource metadata URL is `https://api.mttrly.com/.well-known/oauth-protected-resource/mcp`.
- Browser-origin MCP clients must be explicitly allowlisted with `MCP_ALLOWED_ORIGINS`.
- Requests without an `Origin` header receive the normal 401 OAuth challenge.
- Requests with an untrusted `Origin` header are rejected with 403 by design.

## Tools

40 tools are available: 21 read tools and 19 execute tools. Always call `mttrly_get_capabilities` first to discover the current plan and any restrictions.

### Read tools (21)

| Tool ID | Title | Plan | Description |
| --- | --- | --- | --- |
| `mttrly_get_capabilities` | Get Plan Capabilities | `watchdog` | Get current plan, available tools, restricted tools, and plan limits. Call this first before using any other tool. |
| `mttrly_list_servers` | List Servers | `watchdog` | List all connected servers with online/offline status, server IDs, names, last seen, and agent version. Use mttrly_get_server_status for per-server CPU, RAM, disk, and active alert counts. |
| `mttrly_get_server_status` | Get Server Status | `watchdog` | Get detailed server health including CPU, RAM, disk usage, and active alert count. |
| `mttrly_get_alerts` | Get Server Alerts | `watchdog` | Get alerts for a server with severity and status filtering. Plan-based retention limits apply. |
| `mttrly_quick_triage` | Quick Server Triage | `watchdog` | Get a shallow read-only diagnostic summary from safe Watchdog data: status, active alerts, and resource percentages. |
| `mttrly_run_diagnostic` | Run Server Diagnostic | `deployment_bro` | Run a non-destructive diagnostic on a server. Describe the problem and the system will investigate. 30-second cooldown per server. Deployment Bro+ only — Watchdog has read-only inspect tools (status, alerts, list_playbooks). |
| `mttrly_list_playbooks` | List Playbooks | `watchdog` | List available remediation playbooks. Returns id, name, description, category, requires_approval flag, and optional parameters array. Filter by server_id or category. |
| `mttrly_get_service_reality` | Get Service Reality | `deployment_bro` | Get the latest cached or live-refreshed service reality snapshot for a server, including discovery findings, services, runtimes, and current health context. |
| `mttrly_refresh_reality` | Refresh Reality | `deployment_bro` | Run a live service reality refresh for a server and wait for the fresh snapshot, with a hard per-server cooldown window. |
| `mttrly_get_server_timeline` | Get Server Timeline | `deployment_bro` | Get a unified recent timeline for a server that merges incidents, deploys, and audit events into one ordered stream. |
| `mttrly_run_deploy_verifier` | Run Deploy Verifier | `deployment_bro` | Run the generic deploy verifier for a server. Returns healthcheck outcome, active incident context, and latest deploy status. |
| `mttrly_read_file` | Read File | `deployment_bro` | Read allowlisted logs, Docker logs, Docker images, or managed env files from a server with shared redaction. |
| `mttrly_read_privileged_file` | Read Privileged File | `deployment_bro` | Read an extended-allowlist privileged file, such as sudoers overrides or root authorized_keys, with shared redaction. |
| `mttrly_search_files` | Search Files | `deployment_bro` | Search allowlisted files on a server for a literal query string, with optional glob filtering and shared redaction. |
| `mttrly_diff_file` | Diff Files | `deployment_bro` | Diff two allowlisted files on a server and return a unified diff with shared redaction. |
| `mttrly_compare_source_to_dist` | Compare Source To Dist | `deployment_bro` | Compare a source tree to dist output on a server through the canonical read-only compare capability. |
| `mttrly_get_install_health` | Get Install Health | `deployment_bro` | Read installer-owned health state, drift, and repair planning for a server through the canonical recovery backend path. |
| `mttrly_search_knowledge` | Search Knowledge | `deployment_bro` | Search saved postmortems, recipes, and notes using full-text search. Optionally filter to a server. |
| `mttrly_get_morning_brief` | Get Morning Brief | `deployment_bro` | Get the current proactive brief for the user, including newly detected anomalies and recent knowledge artifacts. |
| `mttrly_get_investigation_bypass` | Get Investigation Bypass | `deployment_bro` | Read the current bypass state for an investigation. Use between exec calls to verify the bypass is still active without consuming an action slot. |
| `mttrly_acknowledge_proactive_event` | Acknowledge Proactive Event | `deployment_bro` | Mark a proactive event as accepted or as a false positive on behalf of the user. Drives the false-positive budget that mutes noisy detectors. |

### Execute tools (19)

| Tool ID | Title | Plan | Description |
| --- | --- | --- | --- |
| `mttrly_run_playbook` | Run Playbook | `deployment_bro` | Execute a playbook on a server. Read-only playbooks run immediately; approval-required playbooks return pending_approval. |
| `mttrly_execute_command` | Execute Shell Command | `deployment_bro` | Request execution of an arbitrary shell command on a server. Read-only commands and active investigation bypasses can execute immediately; otherwise returns pending_approval. |
| `mttrly_get_pending_actions` | Get Pending Actions | `deployment_bro` | List actions awaiting approval, created by mttrly_run_playbook or mttrly_execute_command. Returns action_id, server, type, description, and expiry. Actions expire after 30 minutes. Use mttrly_approve_action to approve or mttrly_cancel_action to reject. |
| `mttrly_get_action_result` | Get Action Result | `deployment_bro` | Retrieve the current or terminal result for an action created by mttrly_run_playbook or mttrly_execute_command. |
| `mttrly_approve_action` | Approve or Reject Action | `deployment_bro` | Approve or reject a pending action. Only call after explicit user confirmation; blocks until execution completes. |
| `mttrly_cancel_action` | Cancel Action | `deployment_bro` | Cancel a pending action by rejecting it. Only call after explicit user confirmation. |
| `mttrly_get_audit_log` | Get Audit Log | `deployment_bro` | Get the chronological audit trail for a server. Includes command executions, playbook runs, alert triggers, and agent fixes with source attribution (mcp/telegram/dashboard/api) and outcome. |
| `mttrly_write_file` | Write File | `deployment_bro` | Write base64-encoded content to a managed server file. Always returns a pending approval action. |
| `mttrly_cleanup_file` | Cleanup File | `deployment_bro` | Delete an allowlisted managed server file. Always returns a pending approval action. |
| `mttrly_execute_script` | Execute Script | `deployment_bro` | Execute an allowlisted managed script or an inline batch script on a server through the canonical script capability. Active investigation bypasses can execute immediately; otherwise returns pending_approval. |
| `mttrly_bootstrap_recover` | Bootstrap Recover | `deployment_bro` | Generate the canonical bootstrap recovery one-liner for a server and install token. |
| `mttrly_repair_install` | Repair Install | `deployment_bro` | Run canonical install recovery strategies such as issue-driven repair, full install-state convergence, wrapper reprovision, or agent uninstall. Always returns a pending approval action. |
| `mttrly_regenerate_install_token` | Regenerate Install Token | `watchdog` | Generate a new reinstall command for an existing offline server. Requires approval, rotates agent credentials, preserves server history, and does not clean the host. |
| `mttrly_decommission_server` | Delete Offline Server | `watchdog` | Remove an offline/dead server from the Central registry only. Requires approval and does not clean files, users, or systemd on the host. |
| `mttrly_update_agent` | Update Agent | `deployment_bro` | Update the mttrly agent on a server through the canonical core update capability. Always returns a pending approval action. |
| `mttrly_start_investigation` | Start Investigation | `deployment_bro` | Start or resume a server investigation bound to the current MCP session. Requires a short-lived MCP session. |
| `mttrly_generate_retrospective` | Generate Retrospective | `deployment_bro` | Generate a deterministic retrospective from the current server timeline, verifier state, and stored knowledge. Persists a postmortem entry. |
| `mttrly_acquire_investigation_bypass` | Acquire Investigation Bypass | `deployment_bro` | Acquire a session-wide approval bypass on an open investigation. Requires 2FA. Subsequent execute_command/execute_script on the investigation server skip per-command approval until expiry, max actions, or revoke. |
| `mttrly_revoke_investigation_bypass` | Revoke Investigation Bypass | `deployment_bro` | Immediately revoke an active investigation bypass. Subsequent exec returns to per-command approval. Idempotent: revoking an already-revoked bypass returns the current state instead of erroring. |

## Errors

| Code | Description | Suggested action |
| --- | --- | --- |
| `ACTION_EXPIRED` | Pending action exceeded the 30-minute TTL and was automatically cancelled. | Re-run the original command or playbook to create a new pending action. |
| `ACTION_NOT_FOUND` | Pending action was not found. | List pending actions again and use a fresh action_id. |
| `AUTH_FAILED` | Authentication failed or the provided credentials are invalid. | Refresh authentication and retry with a valid API key or OAuth token. |
| `CONNECTION_ERROR` | The MCP server could not reach the backend API. | Retry the request after connectivity is restored. |
| `CONTRACT_VERSION_UNSUPPORTED` | The client requested an unsupported MCP contract major version. | Upgrade the MCP client or use a currently supported contract major. |
| `DIAGNOSTIC_COOLDOWN` | The per-server diagnostic cooldown has not expired yet. | Wait for the cooldown period to expire before retrying. |
| `DOCKER_NOT_AVAILABLE` | Docker is not installed on the target server, so Docker playbooks are unavailable. | Skip Docker playbooks or install Docker on the server first. |
| `FORBIDDEN` | The requested MCP resource belongs to a different user or plan context. | Re-fetch the latest server, playbook, or action IDs for the current account. |
| `INTERNAL_ERROR` | The backend hit an unexpected internal error. | Retry once. If it keeps failing, inspect backend logs or contact support. |
| `NOT_FOUND` | The requested MCP resource was not found. | Refresh the relevant list endpoint and retry with a valid identifier. |
| `PLAN_REQUIRED` | The requested tool requires a higher subscription plan. | Inform the user about the plan limitation and show the upgrade URL. |
| `POLICY_DENIED` | The request was rejected by a known Central-side safety policy before execution. | Choose a permitted target path or operation and retry. |
| `PLAYBOOK_NOT_AVAILABLE` | The selected playbook is not currently available on the target server. | Refresh the playbook list for that server and pick one of the advertised playbooks. |
| `PLAYBOOK_NOT_FOUND` | The specified playbook ID does not exist. | Call mttrly_list_playbooks and choose a valid playbook_id. |
| `RATE_LIMITED` | The request exceeded the current MCP rate limit window. | Wait for the rate limit window to reset and retry. |
| `SERVER_NOT_FOUND` | The specified server ID does not exist or does not belong to the current user. | Call mttrly_list_servers to see the available server IDs. |
| `SERVER_OFFLINE` | The target server agent is not currently connected. | Check whether the mttrly agent is running on the server. |
| `SERVER_ONLINE` | The target server is online, so registry-only delete is blocked. | Use Delete to uninstall the agent first, or wait until the server is offline before deleting the registry record. |
| `TIMEOUT` | The request exceeded the configured timeout. | Retry the request, or narrow the scope if the operation is expensive. |
| `UNKNOWN_ERROR` | An unknown error occurred in the MCP server. | Retry the request and inspect the raw error details if it persists. |
| `VALIDATION_ERROR` | The request failed schema or parameter validation. | Check the request parameters against the advertised tool schema. |
| `APPROVAL_NOTIFICATION_FAILED` | A pending action was created but the approval prompt could not be delivered to the user; the action has been failed-closed. | Inspect the action_id in the error details to confirm the action is no longer pending, then retry the original request to create a fresh action. |
| `INVESTIGATION_CLOSED` | The investigation is already closed and cannot acquire a bypass. | Open a fresh investigation with mttrly_start_investigation before requesting bypass. |
| `BYPASS_ALREADY_ACTIVE` | This investigation already has an active, non-revoked bypass. | Use the current bypass; or call mttrly_revoke_investigation_bypass and re-acquire if you need different limits. |
| `BYPASS_ACQUIRE_FAILED` | Could not acquire investigation bypass — investigation state changed between check and update. | Re-read the investigation status and retry; if the investigation closed, start a new one. |

## Plans

### Limits

| Plan | Servers | Log lines | Log retention (days) | Scheduled checks |
| --- | --- | --- | --- | --- |
| `watchdog` | 1 | 50 | 7 | 0 |
| `deployment_bro` | 3 | 500 | 30 | 5 |
| `deployment_crew` | 9 | 500 | 30 | Unlimited |

### Access matrix

| Tool | Watchdog | Deployment Bro | Deployment Crew |
| --- | --- | --- | --- |
| `mttrly_get_capabilities` | ✓ | ✓ | ✓ |
| `mttrly_list_servers` | ✓ | ✓ | ✓ |
| `mttrly_get_server_status` | ✓ | ✓ | ✓ |
| `mttrly_get_alerts` | ✓ | ✓ | ✓ |
| `mttrly_quick_triage` | ✓ | ✓ | ✓ |
| `mttrly_run_diagnostic` | — | ✓ | ✓ |
| `mttrly_list_playbooks` | ✓ | ✓ | ✓ |
| `mttrly_run_playbook` | — | ✓ | ✓ |
| `mttrly_execute_command` | — | ✓ | ✓ |
| `mttrly_get_pending_actions` | — | ✓ | ✓ |
| `mttrly_get_action_result` | — | ✓ | ✓ |
| `mttrly_approve_action` | — | ✓ | ✓ |
| `mttrly_cancel_action` | — | ✓ | ✓ |
| `mttrly_get_audit_log` | — | ✓ | ✓ |
| `mttrly_get_service_reality` | — | ✓ | ✓ |
| `mttrly_refresh_reality` | — | ✓ | ✓ |
| `mttrly_get_server_timeline` | — | ✓ | ✓ |
| `mttrly_run_deploy_verifier` | — | ✓ | ✓ |
| `mttrly_read_file` | — | ✓ | ✓ |
| `mttrly_read_privileged_file` | — | ✓ | ✓ |
| `mttrly_search_files` | — | ✓ | ✓ |
| `mttrly_diff_file` | — | ✓ | ✓ |
| `mttrly_compare_source_to_dist` | — | ✓ | ✓ |
| `mttrly_get_install_health` | — | ✓ | ✓ |
| `mttrly_write_file` | — | ✓ | ✓ |
| `mttrly_cleanup_file` | — | ✓ | ✓ |
| `mttrly_execute_script` | — | ✓ | ✓ |
| `mttrly_bootstrap_recover` | — | ✓ | ✓ |
| `mttrly_repair_install` | — | ✓ | ✓ |
| `mttrly_regenerate_install_token` | ✓ | ✓ | ✓ |
| `mttrly_decommission_server` | ✓ | ✓ | ✓ |
| `mttrly_update_agent` | — | ✓ | ✓ |
| `mttrly_start_investigation` | — | ✓ | ✓ |
| `mttrly_generate_retrospective` | — | ✓ | ✓ |
| `mttrly_search_knowledge` | — | ✓ | ✓ |
| `mttrly_get_morning_brief` | — | ✓ | ✓ |
| `mttrly_get_investigation_bypass` | — | ✓ | ✓ |
| `mttrly_acquire_investigation_bypass` | — | ✓ | ✓ |
| `mttrly_revoke_investigation_bypass` | — | ✓ | ✓ |
| `mttrly_acknowledge_proactive_event` | — | ✓ | ✓ |

## Support

- Support email: [mttrly@mttrly.com](mailto:mttrly@mttrly.com)
- Issues: [github.com/mttrly-hq/mttrly-mcp/issues](https://github.com/mttrly-hq/mttrly-mcp/issues)
- Marketing page: [mttrly.com/mcp](https://mttrly.com/mcp)
- Docs: [mttrly.com/docs#mcp](https://mttrly.com/docs#mcp)
