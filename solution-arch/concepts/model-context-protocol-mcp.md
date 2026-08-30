---
uid: c5f001db-233f-4629-9155-42ac54b2bd2f
---

# Model Context Protocol (MCP)

**Topic:** [[solution-arch/topics/agentic-ai-architecture]]
**Related:** [[solution-arch/concepts/function-calling-and-tool-use]], [[solution-arch/concepts/api-gateway]], [[solution-arch/patterns/ai-gateway-pattern]]

## What it is

MCP (Model Context Protocol) is an open standard for connecting AI applications to external tools, data sources, and systems through a single, uniform protocol — instead of every application writing bespoke, one-off integration code for every tool and every model provider needing its own tool-calling adaptation for every service. It's frequently described by the analogy: **MCP is to AI tool integration what USB-C is to device connectors, or what LSP (Language Server Protocol) is to code editor/language integrations** — a common interface that decouples the two sides so they can evolve independently.

## How it works

```
WITHOUT MCP (N×M integration problem):

  App A ──custom──▶ Tool 1        App A ──custom──▶ Tool 1
  App A ──custom──▶ Tool 2        App B ──custom──▶ Tool 1
  App B ──custom──▶ Tool 1        App B ──custom──▶ Tool 2
  App B ──custom──▶ Tool 2        App C ──custom──▶ Tool 1
  App C ──custom──▶ Tool 1        App C ──custom──▶ Tool 2
  App C ──custom──▶ Tool 2        (N apps × M tools = N×M
                                   bespoke integrations to build
                                   and maintain)

WITH MCP (each side integrates ONCE):

  App A ──┐                           ┌──▶ Tool 1 (MCP server)
  App B ──┼──▶ MCP (standard protocol)┼──▶ Tool 2 (MCP server)
  App C ──┘                           └──▶ Tool 3 (MCP server)

  Each application integrates the MCP CLIENT once.
  Each tool/system exposes an MCP SERVER once.
  N + M integrations total, not N × M.
```

```
┌────────────────────────────────────────────────────────────┐
│                      MCP ARCHITECTURE                         │
├────────────────────────────────────────────────────────────┤
│  Host (the AI application — e.g. an IDE, a chat client,        │
│  an agent runtime)                                              │
│         │                                                       │
│         ▼                                                       │
│  MCP Client (lives inside the host, speaks MCP protocol)         │
│         │  JSON-RPC over stdio (local) or HTTP/SSE (remote)      │
│         ▼                                                       │
│  MCP Server (exposes one system's capabilities):                 │
│    - Tools      → callable functions (like function calling,      │
│                   but standardized discovery/invocation)            │
│    - Resources  → readable data (files, DB records, API           │
│                   responses) the model can pull into context        │
│    - Prompts    → reusable prompt templates the server can offer    │
│         │                                                          │
│         ▼                                                          │
│  Underlying system (a database, a SaaS API, the local             │
│  filesystem, a Git repo, ...)                                       │
└────────────────────────────────────────────────────────────┘
```

**Key architectural distinction from plain function calling:** function/tool calling (see [[solution-arch/concepts/function-calling-and-tool-use]]) is the *model-facing contract* — how the model requests an action. MCP is the *integration-layer standard* underneath that — how your application discovers, connects to, and invokes a growing ecosystem of external tool servers without writing custom glue code for each one. An agent framework typically uses MCP to populate its available tool set, which then gets exposed to the model via the provider's native function-calling format.

## Complexity

Not algorithmic. The architectural cost/benefit is integration maintenance: MCP trades a small protocol-overhead cost (an extra JSON-RPC hop, a discovery handshake) for a large reduction in the number of bespoke integrations a growing agent platform must maintain as it adds tools and switches or adds model providers.

## When to use

```
Use MCP when:
  ✅ You expect to integrate MANY tools/data sources over time —
     the N×M problem gets worse as either N (apps/agents) or M
     (tools) grows, and MCP's payoff grows with it
  ✅ You want to be able to swap the underlying model/agent
     framework without rewriting every tool integration
  ✅ You're building a platform others will extend (third parties
     writing their own MCP servers to plug into your agent)

A direct, custom function-calling integration is still fine when:
  ✅ You have one or two tools, tightly coupled to one specific
     application, with no expectation of reuse or of a third party
     ever needing to integrate — the protocol overhead isn't
     justified for a closed, small integration surface
```

## Common interview angles

```
Q: "Why not just use function calling directly instead of adding
    a whole extra protocol layer?"
A: Function calling defines HOW a model requests a tool call in a
   single request/response. It says nothing about how your
   application DISCOVERS what tools exist, connects to a growing
   ecosystem of them, or shares a tool integration across multiple
   different agent applications/frameworks. MCP standardizes that
   integration layer so tool builders and AI application builders
   can work independently — same value proposition as why we don't
   hand-write a bespoke database driver per application when JDBC/
   ODBC already standardizes that boundary.

Q: "What security concerns does MCP introduce that a single
    in-process tool call doesn't?"
A: An MCP server is often a separate process/service, potentially
   third-party — it becomes a new trust boundary. Applies the same
   guardrail thinking as [[solution-arch/concepts/ai-guardrails-and-safety]]:
   least-privilege scoping per MCP server, validating what an MCP
   server claims its tools do vs what they actually do, and treating
   data returned from an MCP server's "resources" as untrusted
   content subject to prompt-injection defense, same as any other
   retrieved data.

Q: "How does MCP fit into a multi-agent architecture?"
A: Each specialized agent in a supervisor/worker topology (see
   [[solution-arch/patterns/multi-agent-orchestration]]) can connect
   to only the MCP servers relevant to its role — a research agent
   connects to a web-search MCP server, a coding agent connects to
   a filesystem/git MCP server — enforcing tool-access separation
   at the integration layer, not just via prompt instruction.
```

## Examples

```
An MCP server exposing a company's internal ticketing system:
  Tools: create_ticket, update_ticket_status, search_tickets
  Resources: ticket-{id} (readable ticket detail as context)

Any MCP-compatible agent host (an IDE assistant, a support-desk
agent, an internal chatbot) can connect to this ONE server and gain
all three capabilities, without the ticketing team building a
separate integration per consuming application.
```

## Sources
- [[solution-arch/concepts/function-calling-and-tool-use]]
- [[solution-arch/concepts/api-gateway]]
