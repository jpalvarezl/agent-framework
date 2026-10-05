---
status: proposed
date: 2026-10-01
deciders: eavanvalkenburg, westey-m
---

# Adopt the MCP v2 negotiating client behind Python MCP tools

## Context and Problem Statement

The Python `MCPTool` implementation owns an MCP transport, constructs a low-level `ClientSession`, and always calls
`initialize()`. That lifecycle supports handshake-era protocol versions through 2025-11-25, but it cannot connect to
a 2026-07-28 server, where `server/discover` replaces the initialization handshake.

The MCP Python SDK v2 provides a high-level `Client(mode="auto")` that probes `server/discover` and falls back to
`initialize` when the probe does not provide positive evidence of a compatible modern peer. Agent Framework needs to
adopt that negotiation without replacing its public `MCPTool`, `MCPStdioTool`, or `MCPStreamableHTTPTool` classes, or
regressing async context management, reconnect behavior, caller-owned sessions, and request-scoped authentication
headers.

## Decision Drivers

- One Agent Framework code path must support 2026-07-28 and 2025-era MCP peers.
- Protocol negotiation should remain owned by the MCP SDK rather than being independently reimplemented.
- Existing public tool classes, `async with` ergonomics, and tool/prompt discovery behavior must remain compatible.
- Framework-created resources must still be closed by the task that opened them, including cancellation and failed
  connection attempts.
- Caller-supplied sessions and HTTP clients must remain caller-owned.
- Dynamic HTTP header identity must continue to bind discovery and later requests to the same authenticated identity.
- Stdio and Streamable HTTP must share the same protocol lifecycle.

## Considered Options

- Enter the MCP SDK v2 `Client(mode="auto")` behind the existing Agent Framework tool classes.
- Reimplement `server/discover` and legacy fallback directly around `ClientSession`.
- Add separate modern and legacy Agent Framework tool classes or a public protocol-mode switch.

## Decision Outcome

Chosen option: "Enter the MCP SDK v2 `Client(mode="auto")` behind the existing Agent Framework tool classes",
because it uses the SDK's supported negotiation path while preserving the Agent Framework API and transport-specific
behavior.

For framework-created connections:

- `MCPStdioTool` and `MCPStreamableHTTPTool` continue to create their existing transport context managers.
- `MCPTool` passes that transport to `mcp.Client(mode="auto")` and enters the client on its existing `AsyncExitStack`.
- The SDK probes `server/discover`. A compatible modern peer is adopted without sending `initialize`; a probe that
  does not establish a compatible modern peer falls back to the initialization handshake on the same code path.
  `METHOD_NOT_FOUND` is the representative legacy response covered by Agent Framework's contract test, not the SDK's
  only fallback condition.
- Agent Framework retains a private reference to the high-level client and exposes its underlying `ClientSession`
  through the existing `session` attribute for compatibility with current integrations.
- Negotiated protocol version and server capabilities are read from the client's era-neutral properties rather than
  captured only from an initialize result.
- Tools configured for the existing server-initiated sampling callback use `mode="legacy"`. The MCP migration guide
  requires legacy mode for that back-channel behavior because 2026-07-28 refuses server-initiated sampling on every
  transport. Auto mode remains the default when no legacy-only callback behavior is requested.

Agent Framework retains its lifecycle owner task and locks. They protect framework state, preserve AnyIO task
ownership during teardown, serialize identity-changing reconnects, and roll back partially loaded discovery state.
They no longer implement protocol negotiation. A reset or reconnect closes the whole SDK client and transport, then
constructs a new auto-negotiating client. `ping` is not used as a universal liveness preflight because it does not
exist in 2026-07-28; operation failures drive reconnect, while any retained legacy ping behavior is gated by the
negotiated protocol.

Caller-supplied `ClientSession` remains a compatibility path:

- Agent Framework does not enter, close, or replace the session.
- An already negotiated session is reused through its `protocol_version` and `server_capabilities` properties.
- An unnegotiated session retains the existing compatibility behavior and uses `initialize()`. A caller that supplies
  a modern low-level session negotiates it with `discover()` before passing it to Agent Framework. This avoids
  duplicating the SDK's broader auto-negotiation policy, which is implemented only by the high-level `Client`.

Dynamic and static HTTP headers remain below the negotiating client at the Streamable HTTP transport boundary. The
effective header set continues to define connection identity; changing it closes and rebuilds the whole SDK client
before discovery or tool calls proceed. This preserves the trust and approval boundaries documented in
[ADR 0043](0043-python-mcp-runtime-context.md).

The MCP Streamable HTTP implementation uses `httpx2` types, as required by MCP SDK v2. `httpx` and `httpx2` may
coexist elsewhere in the repository; this decision does not require unrelated HTTP features to migrate.

### Consequences

- Good, because modern and legacy servers use one public Agent Framework API and one SDK-supported negotiation path.
- Good, because stdio, Streamable HTTP, reconnect, cancellation, and header identity remain Agent Framework concerns
  without duplicating protocol-version selection.
- Good, because later work can use the high-level client for per-request metadata, subscriptions, MRTR, and caching.
- Neutral, because Agent Framework keeps both a private high-level client and the public low-level `session` view.
- Neutral, because caller-supplied unnegotiated sessions remain handshake-era unless the caller negotiates them first.
- Bad, because tests that mocked `ClientSession` construction must move toward client/transport contract tests.

## Validation

Contract tests cover both branches through the same Agent Framework tool:

- A modern-only Streamable HTTP peer accepts `server/discover`, rejects `initialize`, serves `tools/list`, and records
  that the negotiated protocol is 2026-07-28.
- A 2025-era peer rejects `server/discover`, accepts `initialize`, and serves the same tool operations.
- Equivalent stdio coverage verifies that negotiation is transport-independent.
- Existing lifecycle, reconnect, cancellation, caller-ownership, and dynamic-header identity tests remain green.

Follow-on tests cover per-request logging metadata, `subscriptions/listen` with legacy notification fallback, prompts,
skills, MRTR, and caching. Tasks remain deferred until the Python MCP SDK exposes the 2026 Tasks extension runtime.

## Pros and Cons of the Options

### Enter the MCP SDK v2 client behind existing tool classes

- Good, because the SDK owns current and future negotiation rules.
- Good, because the existing transport subclasses and public API remain intact.
- Good, because it matches the .NET direction of delegating connection creation and negotiation to its MCP SDK.
- Bad, because the existing connection code and mocks must be reshaped around a higher-level lifecycle owner.

### Reimplement negotiation around ClientSession

- Good, because it minimizes the first code diff and preserves direct session construction.
- Bad, because Agent Framework would duplicate the SDK's discover, fallback, adoption, and future-version behavior.
- Bad, because later high-level SDK features would still require a second migration.

### Add era-specific tool classes or a public mode switch

- Good, because callers could force a known protocol era.
- Bad, because callers should not need to know a server's era before connecting.
- Bad, because it duplicates public classes, tests, documentation, and lifecycle behavior.
- Bad, because a mode switch can accidentally disable fallback and fragment compatibility.

## More Information

- [Python MCP 2026-07-28 umbrella issue](https://github.com/microsoft/agent-framework/issues/8245)
- [MCP Python SDK v1-to-v2 migration guide](https://py.sdk.modelcontextprotocol.io/migration/#clients)
- [.NET MCP Tasks migration](https://github.com/microsoft/agent-framework/pull/7774)
- The .NET declarative MCP handler delegates connection creation to `McpClient.CreateAsync`; its protocol stub rejects
  `server/discover` with `METHOD_NOT_FOUND` before accepting `initialize`, demonstrating SDK-owned legacy fallback.
