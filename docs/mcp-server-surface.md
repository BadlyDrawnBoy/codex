# MCP Server Surface Design

## Overview
The MCP server surface exposes configured tools to Codex clients through a uniform discovery, invocation, and approval pipeline. It aligns the new MCP registry and invocation behavior with existing `codex` and `codex-reply` tools while providing stronger sandbox defaults, session-aware environments, and auditable approval flows.

## Goals
- Present a consistent tool discovery contract for clients and orchestrators.
- Forward tool calls with explicit approval and sandbox rules to reduce accidental or malicious executions.
- Support per-session environment shaping (CWD/env) derived from `config.toml` while allowing overrides when explicitly allowed.
- Provide reliable observability (logging/streaming) and cancellation handling to keep UX predictable.
- Ship behind a feature gate with a migration path from the current `codex`/`codex-reply` tool experience.

## Non-goals
- Redesigning tool semantics themselves (input/output schemas remain defined by each tool).
- Implementing new sandbox providers; the design reuses existing sandbox abstraction.

## Tool Discovery
- On startup, the MCP server reads `config.toml` MCP entries and materializes an internal registry of tools.
- Discovery surface exposes each tool with:
  - `name` (unique across session), `display_name`, and `description`.
  - Capability metadata: `requires_approval`, `sandbox_profile`, `default_timeout_ms`, `allows_streaming`.
  - Input schema (verbatim from MCP config) and declared side-effect flags (filesystem/network hints when present).
- Discovery payloads must include per-session defaults (CWD, env) to allow clients to display execution context before approval.
- Registry updates should be hot-reload friendly: changes to `config.toml` trigger a refresh that re-computes tool visibility and filters without requiring a new process when supported by the host.

## Tool Call Forwarding
- Invocation path: client request → approval gate → sandboxed executor → result mapper.
- Calls include the effective CWD/env snapshot resolved for the session.
- Executor enforces sandbox profile (seatbelt/default) and timeouts; it streams stdout/stderr when allowed.
- Cancellation propagates to the running process via the sandbox driver (e.g., SIGTERM) and surfaces a cancellation status to the client.
- Error mapping:
  - Sandbox violations → `permission_denied` with actionable message.
  - Timeout → `deadline_exceeded` and includes partial logs if available.
  - Spawn failures → `internal` with stderr excerpt.

## Approval Flow
- Default: tools require user approval per invocation unless `requires_approval = false` in config.
- Approval prompts must show:
  - Tool name and description.
  - Effective CWD and key env vars (non-sensitive subset) that will be passed.
  - Declared side-effect hints and sandbox profile.
  - Requested arguments (with redaction for secrets marked in schema when provided).
- Approvals are logged with timestamp, session ID, tool name, arguments hash, and outcome (approved/denied/expired).
- Sessions can opt into trust-once or trust-for-session modes per tool; trust must never cross sessions.

## Sandbox and Environment Inheritance
- Each MCP entry may specify `sandbox_profile` (`default`, `seatbelt`, or `none` for fully trusted internal tools) with `default` used when unspecified.
- CWD resolution order: session override → tool-specific CWD from config → workspace default.
- Env resolution order: session overrides → tool-level env from config → process baseline env. Sensitive tokens are marked and redacted in logs/prompts.
- When sandboxing is enabled, network is disabled by default unless `allow_network = true` on the MCP entry.

## Auth and Token Handling
- MCP config may reference secrets via `${VAR}` placeholders; resolution occurs at startup and on hot reload from the process environment or secret store.
- Tokens are never echoed to logs or approval prompts; only the presence of a token is indicated.
- Rotations: reload picks up new token values; running sessions see refreshed values on subsequent calls.
- When auth is required for a tool call, failures are mapped to `unauthenticated` with hints (e.g., token missing/expired) without leaking raw errors.

## Timeouts and Cancellation
- Each tool supports `timeout_ms` in config; default applies when omitted (e.g., 120s).
- Client-provided timeouts choose the smaller of client timeout and tool default ceiling.
- Cancellation requests cancel the sandboxed process and emit a terminal `cancelled` event; partial logs may be flushed before termination.
- Long-running tools should regularly flush output to support responsive streaming and cancellation acknowledgement.

## Logging and Streaming Requirements
- Structured logs record: session, tool, args hash, CWD, sandbox profile, timeout, approval decision, start/stop timestamps, exit status, and byte counts for stdout/stderr.
- Streaming: stdout/stderr are multiplexed to the client when `allows_streaming` or when tool marks streaming-compatible. Back-pressure from clients must throttle process pipes to avoid unbounded buffering.
- Logs must avoid embedding secrets; redaction uses schema hints and known env key patterns (e.g., `*_TOKEN`).
- Client-facing errors should include a correlation ID to link to server logs.

## Mapping `config.toml` MCP Entries to Tools
- For each `[mcp.<tool_name>]` entry:
  - Load `command`, `args`, `cwd`, `env`, `sandbox_profile`, `allow_network`, `requires_approval`, `timeout_ms`, `streaming`, and `description`.
  - Apply allow/deny filters:
    - Global allow/deny lists restrict which tool names become visible; deny wins when both match.
    - Path filters can restrict CWDs or arguments (e.g., allow only under workspace root); violations surface as discovery omissions, not runtime failures, when possible.
  - Per-session shaping:
    - Session state may provide `cwd_override` and `env_overrides`; merged per the inheritance rules above and persisted only for that session.
    - Session metadata (user, workspace) is attached to registry entries so approvals/logging include the correct context.
  - Exposed tool payload to clients includes the resolved defaults and flags whether session overrides are active.
- Tools with identical names across entries are rejected at load with a clear error.

## Security Expectations
- Least privilege: default sandbox with network off; explicit opt-in required for network or unsandboxed execution.
- Approval prompts are mandatory for tools marked as high-risk (filesystem writes, network) even if config disables approval; the server enforces this classification.
- Errors must map to friendly client messages without leaking file paths or internal stack traces; detailed diagnostics live in logs keyed by correlation ID.
- Deny-listed tools or invalid configurations must not appear in discovery responses.

## UX Expectations
- Discovery responses clearly indicate required approvals, sandbox status, default CWD, and streaming availability.
- Approval dialogs emphasize potential side effects and context (CWD/env) before execution.
- Streaming output should show incremental logs with an explicit completion or cancellation marker.
- Timeouts and cancellations surface concise status text plus a link or ID for deeper logs.

## Migration and Flagging Plan
- Feature gate: guarded by `enable_mcp_server_surface` (env var or config flag). Off by default for one release; on by default after stabilization.
- Compatibility:
  - Existing `codex` and `codex-reply` tools continue to function; when the gate is on, they are re-exposed through the MCP surface with identical names/args but the new approval+sandbox defaults.
  - Legacy workflows can disable the gate to retain current behavior while migrating.
- Rollout steps:
  1. Ship gated implementation with dual discovery paths (legacy + MCP surface) and ensure parity tests.
  2. Add opt-in flag to config and document in `config.toml` schema.
  3. Collect feedback and harden logging/approval UX.
  4. Flip default to on; retain opt-out for one deprecation window.
- Backwards compatibility is maintained by keeping legacy tool IDs stable and mapping errors to existing client codes where possible.
