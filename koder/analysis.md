# Model Context Protocol (MCP) – Integration Analysis for Codex CLI

Revision 2 – 12 June 2025 (after reviewing docs in `koder/docs/mcp/*`)

This analysis supersedes the initial draft and incorporates the additional
technical detail and sample code provided in the *Perplexity* and *Grok*
research notes located in `koder/docs/mcp/`.

-------------------------------------------------------------------------------

## 0  TL;DR

• Integration remains **feasible**, but the true scope is larger than a simple
  “provider” plug-in because MCP adds *context aggregation, resource browsing,
  and tool execution* – none of which exist in current Codex CLI.

• Effort is therefore revised from ~4 PD to **10–15 PD (80–120 h)** depending on
  depth of UX & security hardening.

• The recommended strategy is a *layered extension* that keeps upstream diffs
  small: implement a stand-alone **`mcp-client` utility module** that handles
  JSON-RPC and transport logic via `@modelcontextprotocol/sdk`, then expose
  high-level hooks to the rest of Codex CLI through the existing Ink/React TUI
  components.

-------------------------------------------------------------------------------

## 1  Recap of MCP Architecture (from docs)

| Layer          | Key points Relevant to Codex CLI |
|----------------|----------------------------------|
| **Host**       | Codex acts as an *MCP Host*: it orchestrates client
|                | connections, aggregates resources/prompts/tools, and
|                | mediates user consent. |
| **Client**     | 1:1 connection to each MCP server. Implemented with
|                | `@modelcontextprotocol/sdk` (`Client` + chosen Transport).
| **Server**     | External, may be stdio-launched or remote HTTP. Codex only
|                | needs to *connect* to them, not implement them. |
| **Transport**  | Stdio (local exec), Streamable HTTP, or WebSocket. The SDK
|                | abstracts these. Codex must allow dynamic selection. |

MCP uses **JSON-RPC 2.0** for all messages and defines typed methods for:

• Listing resources/prompts/tools  • Reading a resource  • Calling a tool
• Subscribing to updates            • Capability negotiation & diagnostics

Security requirement: *explicit user authorization* for every tool call.

-------------------------------------------------------------------------------

## 2  Gap Analysis – Codex CLI vs MCP Requirements

| Capability                              | Present in Codex CLI | Required for MCP | Gap |
|-----------------------------------------|----------------------|------------------|-----|
| Basic outbound HTTP / fetch             | ✔ (openai client)    | ✔                | – |
| JSON-RPC framing                        | ✖                    | ✔                | **High** |
| Persistent multi-server connections     | ✖                    | ✔                | **Med** |
| Resource browser UI                     | ✖                    | ✔                | **Med** |
| Tool execution with user consent        | Partial (edit apply) | ✔ (generic)      | **Med** |
| Credential storage (per server)         | ✖ (only OpenAI key)  | ✔                | **Low** |
| Transport selection (stdio / HTTP)      | ✖                    | ✔                | **Low** |
| Type-safe param validation (zod)        | ✖                    | ✔                | **Low** |
| Testing (mock JSON-RPC channels)        | ✖                    | ✔                | **Med** |

Conclusion: **additional subsystems** (JSON-RPC client manager, resource/tree
viewer, consent prompts) must be added on top of the simple provider shim.

-------------------------------------------------------------------------------

## 3  Revised Effort Estimate (80–120 h)

| Work-package | Deliverables | Complexity | Hours |
|--------------|-------------|------------|-------|
| Familiarise with MCP spec & SDK | Up-to-date spec review, PoC client connection | Low | 8 |
| Transport & client manager | `mcp/client-manager.ts` – connect, reconnect, pool | Med | 12 |
| JSON-RPC wrapper utilities | error taxonomy, logging, cancellation | Med | 6 |
| Credential & server config | update settings schema, ENV vars, UI prompt | Low | 6 |
| Resource browser UI        | Ink tree-view component + actions | Med | 16 |
| Tool consent UX            | Modal dialog using existing approval overlay | Med | 12 |
| API surface for chat agent | helper that injects MCP context/resources into prompts | Med | 12 |
| Unit / integration tests   | Mock transports, Vitest suites | Med | 12 |
| Docs & CHANGELOG           | docs/mcp.md, help overlay updates | Low | 8 |
| Buffer & code review       | – | – | 8 |
| **Total** |  |  | **~100 h ≈ 12 PD** |

Notes: A “Minimum Viable” integration (single server, no resource tree-view)
could ship in ~6 PD, but full spec compliance matches Perplexity guide’s
80-120 h estimate.

-------------------------------------------------------------------------------

## 4  Integration Strategy – Layered Extension

1. **Standalone MCP sub-package**  (`codex-cli/src/mcp/`)
   • Houses all SDK-dependent code.  • No imports from the rest of Codex except
   for narrow interfaces (events, config).  ➜ This keeps upstream merges safe.

2. **Client Manager**
   • API: `connect(serverConfig) → Promise<ClientHandle>`
   • Emits events: `resourceAdded`, `resourceUpdated`, `toolRegistered` …
   • Internally uses SDK `Client` + chosen Transport (stdio / HTTP).

3. **Context Aggregator**
   • Collects resources/prompts from all connected servers.
   • Exposes query functions used by chat agent for prompt construction.

4. **UI integration** (Ink components)
   • New overlay `McpResourceOverlay` listing resources & allowing preview.
   • Reuse existing Approval overlay for *“Allow tool `<name>` to run?”* flow.

5. **Chat pipeline hooks**
   • When LLM response contains a JSON block requesting a tool invocation,
     route call through MCP client manager instead of local tools.

6. **Config & CLI flags**
   • `codex --mcp-server <uri>` or configuration file array.
   • `CODEX_MCP_TOKEN_*` env vars resolved by server name.

-------------------------------------------------------------------------------

## 5  Maintainability & Upstream Compatibility

• **Zero intrusive patches** to existing provider system: MCP lives in its own
  directory and surfaces *events* rather than touching OpenAI request flow.

• **Optional dependency** – lazy-`require` the SDK; if absent, features are
  hidden.  This avoids bloating the CLI for users uninterested in MCP.

• **Version pinning** – lock SDK major + minor to avoid silent breaking
  changes; expose `codex doctor` check.

-------------------------------------------------------------------------------

## 6  Risks & Mitigations (updated)

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| SDK breaking changes | Medium | Build / runtime failures | semver pin, CI integration |
| UX overload (new overlays) | Medium | User confusion | staged rollout + docs |
| Tool misuse (security) | Low-Med | Data loss | mandatory consent modal |
| Performance (many resources) | Low | CLI lag | incremental loading & virtual list |

-------------------------------------------------------------------------------

## 7  Recommendation

Adopt the **layered extension** plan summarised above. Begin with a *connection
and tool-call-only* MVP (single server, no resource tree). Incrementally add
resource/ prompt browsing once the foundation is stable.  This phased approach
delivers value early while keeping complexity under control and aligns with
the docs’ emphasis on user consent and standard JSON-RPC tooling.

-------------------------------------------------------------------------------

## Appendix – Sample MVP Code Snippet

```ts
// codex-cli/src/mcp/connect.ts
import { Client, StdioClientTransport } from "@modelcontextprotocol/sdk";

export async function connectLocalServer(cmd: string, args: string[] = []) {
  const client = new Client({ name: "codex-cli", version: "0.1.0" });
  const transport = new StdioClientTransport({ command: cmd, args });
  await client.connect(transport);
  return client;
}
```

This thin wrapper isolates SDK code and can be iteratively expanded.
