---
uid: f34dfbc0-721c-4727-92eb-da0e8742654d
---

# Model Context Protocol (MCP)

**Topic:** [[agents/topics/tool-use]]
**Related:** [[agents/concepts/tool-calling]], [[agents/concepts/guardrails-and-safety]]

---

## What it is

MCP is an open protocol (introduced by Anthropic, since adopted broadly) that standardizes how LLM applications connect to external tools, data sources, and prompts. Before MCP, every application that wanted to expose, say, a GitHub integration or a filesystem to an LLM had to write custom, one-off integration code. MCP defines a common client-server interface so a tool/data integration can be built **once** and used by any MCP-compatible application.

The analogy usually taught: MCP is to LLM tool integrations roughly what USB-C or a language server protocol (LSP) is to hardware/editor integrations — a standard interface that decouples the number of integrations needed from *N applications × M tools* down to *N + M*.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                                HOST                                │
│    (the AI application — Claude Desktop, an IDE, a custom app)     │
│                                                                    │
│    ┌────────────────┐   ┌────────────────┐   ┌────────────────┐    │
│    │ MCP Client 1   │   │ MCP Client 2   │   │ MCP Client 3   │    │
│    └────────────────┘   └────────────────┘   └────────────────┘    │
└─────────────┼────────────────────┼────────────────────┼────────────┘
              │ 1:1 connection     │                    │
              ▼                    ▼                    ▼
     ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
     │ MCP Server A    │  │ MCP Server B    │  │ MCP Server C    │
     │ (filesystem)    │  │ (GitHub)        │  │ (database)      │
     └─────────────────┘  └─────────────────┘  └─────────────────┘
          exposes:             exposes:             exposes:
           - Tools              - Tools              - Tools
         - Resources          - Resources          - Resources
          - Prompts            - Prompts            - Prompts
```

- **Host** — the LLM application the user interacts with.
- **Client** — lives inside the host; maintains a 1:1 stateful connection to a single MCP server.
- **Server** — a lightweight process (local or remote) that exposes a specific integration's capabilities over the protocol.

---

## What a server exposes

| Primitive | What it is | Analogous to |
|---|---|---|
| **Tools** | Executable functions the model can call (matches [[agents/concepts/tool-calling]] mechanics) | A function/API endpoint |
| **Resources** | Read-only data the host can attach to context (files, query results, docs) | A GET endpoint / file |
| **Prompts** | Reusable, parameterized prompt templates the server provides | A saved query / macro |

Tools are typically model-invoked (the LLM decides to call them, as in [[agents/concepts/tool-calling]]); resources and prompts are typically application- or user-invoked (attached deliberately, not chosen autonomously by the model).

---

## Why it matters for agent design

| Without MCP | With MCP |
|---|---|
| Every app writes custom integration code per tool | Tool/data providers ship one server; any compliant host can use it |
| Tool availability tightly coupled to one application | Tools are portable across Claude Desktop, IDEs, custom agents, etc. |
| No standard for exposing resources vs. actions | Clear separation: tools (act) vs. resources (read) vs. prompts (template) |
| Permissioning is ad hoc per integration | Consistent connection-level and call-level permission model |

MCP doesn't change the *fundamentals* of tool calling (see [[agents/concepts/tool-calling]]) — it standardizes the transport, discovery, and packaging layer around it.

---

## Anticipated Questions

1. "Is MCP a new way for the model to 'think'?" — No. MCP is plumbing — it standardizes how tools/data get *exposed and discovered*, not how the model reasons about using them. The model-facing mechanics are the same structured tool call described in [[agents/concepts/tool-calling]].
2. "Why not just use REST APIs directly?" — You can, but each API has its own auth, schema conventions, and pagination quirks that an integration author has to hand-adapt for LLM use every time. MCP standardizes the adaptation once per tool provider instead of once per (application, tool) pair.
3. "Where do permissions live in MCP?" — At both the connection level (does this host trust this server at all) and the call level (does this specific tool call require user approval before executing) — this composes with the general guardrails discussed in [[agents/concepts/guardrails-and-safety]], MCP doesn't replace that layer.
4. "Do MCP servers run remotely or locally?" — Either. A local server might expose the filesystem or a local database over stdio; a remote server might expose a SaaS product's API over HTTP. The host doesn't need to know which — the protocol is transport-agnostic.

---

## Sources
- [[agents/concepts/tool-calling]]
- [[agents/concepts/guardrails-and-safety]]
- [[agents/topics/tool-use]]
