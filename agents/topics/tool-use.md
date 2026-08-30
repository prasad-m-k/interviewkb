# Tool Use

**Related concepts:** [[agents/concepts/tool-calling]], [[agents/concepts/mcp-protocol]], [[agents/concepts/guardrails-and-safety]]

## Overview

Tool use is what separates an agent from a text generator: the ability to act on and query the world beyond the model's own weights. This page surveys the mechanics, standardization, and design considerations; [[agents/concepts/tool-calling]] has the full step-by-step mechanics.

## The Tool-Calling Loop

```
Orchestrator exposes       Model emits a         Orchestrator          Result fed
tool schemas to      ──►   structured call ──►   executes the    ──►   back as an
the model                  (name + args)         actual function       observation
```
Full detail: [[agents/concepts/tool-calling]].

## Categories of Tools

| Category | Examples | Notes |
|---|---|---|
| **Information retrieval** | Web search, document search, database query | Often the highest-value, lowest-risk tool class — see [[agents/concepts/agentic-rag]] |
| **Computation** | Calculator, code execution sandbox | Offloads exact arithmetic/logic the model is unreliable at doing "in its head" |
| **Communication** | Send email, post a message, create a ticket | Usually needs guardrails — see [[agents/concepts/guardrails-and-safety]] |
| **State mutation** | Write to a database, delete a file, update a record | Highest risk class; strongest case for approval gates |
| **Sub-agent delegation** | Spawn another agent for a sub-task | Bridges into multi-agent design — see [[agents/topics/multi-agent-systems]] |

## Standardizing Tool Access: MCP

Before a common protocol, every application re-implemented tool integrations from scratch. [[agents/concepts/mcp-protocol]] standardizes the client-server interface so a tool/data source is built once and usable by any compliant host.

```
Host application ──1:1──► MCP Client ──connection──► MCP Server ──► exposes
                                                                    Tools /
                                                                    Resources /
                                                                    Prompts
```

## Designing Good Tools (checklist)

1. **Narrow scope** — one tool, one job. Prefer `refund_order(order_id)` over a generic `execute_sql(query)`.
2. **Clear description** — the model selects tools by matching its understanding of the task to the tool's description text.
3. **Typed, constrained parameters** — enums and typed fields over free-text arguments wherever the valid values are known.
4. **Structured, concise results** — return what's needed for the next decision, not a raw data dump (see [[agents/concepts/context-engineering]]).
5. **Errors as observations, not crashes** — a failed call should return a message the model can act on, not break the loop.
6. **Right-sized permissions** — scope credentials to the minimum the tool needs; never rely on the model "choosing" not to misuse excess access.

**At scale:** once an agent has dozens or hundreds of tools, listing every schema in every prompt stops being viable — a common fix is embedding tool descriptions and retrieving only the most relevant ones for the current task, the same semantic-search mechanism behind RAG. See [[agents/foundations/embeddings-and-similarity]] and [[agents/foundations/vector-databases]].

## Anticipated Questions

- "How does the model 'know' which tool to use?" — It matches the task at hand against each available tool's name and description in the schema it was given — which is exactly why description quality directly drives tool-selection accuracy.
- "What if two tools could plausibly handle the same request?" — This is a design smell — overlapping tools increase the chance of the model picking the wrong one. Prefer consolidating into one tool with parameters, or making the descriptions sharply distinct.
- "Is MCP required to build tool-using agents?" — No. Tool calling works with any framework's native function-calling support. MCP adds portability and standardization across applications, not a new capability.

## Sources
- [[agents/concepts/tool-calling]]
- [[agents/concepts/mcp-protocol]]
- [[agents/concepts/guardrails-and-safety]]
- [[agents/foundations/embeddings-and-similarity]]
