---
uid: f50b0186-a04d-460b-be5b-22989ec7851e
---

# Function Calling & Tool Use

**Topic:** [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/openai-platform-architecture]]
**Related:** [[solution-arch/concepts/model-context-protocol-mcp]], [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/concepts/idempotency]]

## What it is

Function/tool calling is the mechanism by which an LLM requests that *your* code execute a specific function with specific arguments, based on the conversation so far — the foundation that turns a text generator into an agent that can act on the world (query a database, call an API, run code, send an email). The model never executes anything itself; it emits a structured request, and your application layer is responsible for executing it safely and returning the result.

## How it works

```
1. You register tools with typed schemas (JSON Schema) describing
   name, description, and parameters — the model uses the
   description text to decide WHEN and HOW to call each tool, so
   description quality directly drives correct tool selection.

2. Model receives user input + tool definitions. If it determines
   a tool is needed, it emits a tool call request (name + JSON
   arguments) INSTEAD OF a final answer.

3. Your application code:
   a. Parses the arguments (they arrive as a JSON string — always
      validate against the schema; never trust blindly)
   b. Checks permission/allow-list — is this tool call allowed in
      this context, for this user?
   c. Executes the actual function/API call
   d. Returns the result back into the conversation, tagged to the
      specific tool-call ID

4. Model receives the tool result appended to context and either
   calls another tool or produces a final answer.

┌────────────────────────────────────────────────────────────┐
│  User: "What's the status of order 4471, and refund it if    │
│         it's delayed more than 3 days?"                       │
│                                                                 │
│  LLM  → tool_call: get_order_status(order_id="4471")           │
│  App  → executes, returns: {status: "delayed", days: 5}         │
│  LLM  → tool_call: issue_refund(order_id="4471", reason=...)     │
│  App  → validates: is refund tool call within policy limits?     │
│         executes (idempotently — see idempotency key below)      │
│         returns: {refund_id: "R-991", status: "processed"}        │
│  LLM  → final answer to user: "Order 4471 was delayed 5 days,      │
│          I've processed a refund (R-991)."                          │
└────────────────────────────────────────────────────────────┘
```

## Complexity

Not algorithmic — the architectural cost is (a) tokens spent on tool definitions in every request (scales with number of registered tools — too many tools degrades selection accuracy and burns context budget) and (b) latency: each tool call is a full round-trip (execute + another model call), so multi-tool-call chains compound latency linearly unless parallelized.

## When to use

```
Use function calling when:
  ✅ The task requires CURRENT or SYSTEM-SPECIFIC data the model
     can't know from training (order status, account balance,
     live inventory)
  ✅ The task requires taking an ACTION (send email, update record,
     book something), not just generating text
  ✅ Deterministic computation is needed (exact math, date logic) —
     models are unreliable at precise calculation; delegate to a
     tool instead of trusting generated numbers

Avoid over-registering tools when:
  ❌ Many overlapping tools with similar descriptions confuse
     selection — the model may call the wrong one. Keep tool sets
     scoped and descriptions unambiguous; consider a routing layer
     (see [[solution-arch/patterns/agentic-workflow-patterns]]) that
     narrows the active tool set per conversation stage instead of
     exposing all tools all the time
```

## Common interview angles

```
Q: "The model called a tool with malformed or nonsensical
    arguments — what's your architecture response?"
A: Never assume valid input. Validate arguments against the JSON
   Schema before execution; on failure, return a structured error
   back into the conversation (not a crash) so the model can retry
   with corrected arguments — same defensive posture as validating
   any external/untrusted input at a system boundary.

Q: "How do you prevent a side-effecting tool (e.g. 'process
    refund') from executing twice if the agent retries after a
    timeout?"
A: Every side-effecting tool call must be idempotent — pass an
   idempotency key (e.g. derived from conversation/turn ID) that
   the downstream service uses to dedupe repeated calls. This is
   the exact same problem and solution as
   [[solution-arch/concepts/idempotency]] in traditional distributed
   systems, now triggered by a non-deterministic caller (the model)
   instead of a flaky network client.

Q: "How do you scope permissions so a compromised or confused
    agent can't do unbounded damage?"
A: Principle of least privilege applied per tool: a 'read order
   status' tool should use a read-only credential; a 'refund' tool
   should be capped at a monetary limit and/or require human
   approval above a threshold (see human-in-the-loop pattern). Never
   give an agent a single, broad, all-powerful API credential.

Q: "You have 40 tools registered — selection accuracy dropped.
    What do you do?"
A: Group tools by domain and use a router/orchestrator pattern to
   narrow the active tool set contextually (e.g. only expose
   billing tools once the conversation is classified as a billing
   issue), rather than exposing all 40 to every call. Also audit
   tool descriptions for overlap/ambiguity — this is usually a
   larger cause of misselection than raw tool count.
```

## Examples

```
get_weather(location: string) -> {temp, conditions}
search_knowledge_base(query: string, top_k: int) -> [documents]
create_support_ticket(summary: string, priority: enum) -> {ticket_id}
issue_refund(order_id: string, amount: number, idempotency_key: string)
  -> {refund_id, status}
```

## Sources
- [[solution-arch/topics/openai-platform-architecture]]
- [[solution-arch/sources/building-effective-agents-anthropic]]
