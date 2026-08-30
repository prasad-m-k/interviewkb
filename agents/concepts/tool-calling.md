---
uid: 11d8007e-70be-4bf6-9eac-8d600e33ce7a
---

# Tool Calling (Function Calling)

**Topic:** [[agents/topics/tool-use]]
**Related:** [[agents/concepts/agent-loop]], [[agents/concepts/mcp-protocol]], [[agents/concepts/guardrails-and-safety]]

---

## What it is

Tool calling (a.k.a. function calling) is the mechanism by which an LLM requests that a specific, developer-defined function be executed with specific arguments, so it can act on or query the world beyond its own weights. The model does not execute anything itself — it emits a structured request; the **orchestrator** (application code, harness, or MCP client) executes it and returns the result.

This is the single mechanism that turns a text generator into something that can search the web, run code, query a database, or send an email.

---

## How it works

```
   ORCHESTRATOR                                            LLM
   (your code)
        │                                                   │
        │  1. tool schemas (JSON Schema)                    │
        ├───────────────────────────────────────────────────►
        │                                                   │
        │  2. structured tool call                          │
        │     {"name": "get_weather",                       │
        │      "arguments": {"city": "Austin"}}             │
        ◄───────────────────────────────────────────────────┤
        │                                                   │
        │  3. execute the function                          │
        │                                                   │
        │  4. result: {"temp": 94, "cond": "sunny"}         │
        │                                                   │
        │  5. result fed back as an observation             │
        ├───────────────────────────────────────────────────►
                                                         (continues
                                                          the loop)
```

### Step 1: Define a tool schema
Every tool is described to the model as a name, a description, and a parameter schema (typically JSON Schema):

```json
{
  "name": "get_weather",
  "description": "Get current weather conditions for a city",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {"type": "string", "description": "City name"},
      "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["city"]
  }
}
```

### Step 2: Model emits a structured call
The model, trained to recognize when a tool is needed, outputs a structured object (not free text) matching the schema, rather than trying to answer from its own knowledge.

### Step 3: Orchestrator executes
The application code — never the model — actually invokes the function, hits the API, or runs the query. This boundary is the basis of every safety and permission control in agent design.

### Step 4: Result fed back
The tool's output is serialized (usually to JSON or plain text) and appended to the conversation as an "observation," and the loop continues — see [[agents/concepts/agent-loop]].

---

## Parallel vs. sequential tool calls

- **Sequential:** each tool call depends on the result of the previous one (e.g., look up a user ID, then use it to fetch their orders). Must run one at a time.
- **Parallel:** independent tool calls the model can issue in a single turn (e.g., check weather in three different cities). Modern tool-calling APIs support emitting multiple tool calls in one model turn, executed concurrently by the orchestrator — meaningfully reduces latency for fan-out tasks.

```
Sequential (dependent):         Parallel (independent):
  lookup_user(id)               get_weather("NYC")     ┐
       │                        get_weather("LA")      ├─ run concurrently
       ▼                        get_weather("Austin")  ┘
  get_orders(user_id)                              │
                                                   ▼
                                     all results returned together
```

---

## Schema design — what makes tools usable by a model

| Principle | Why it matters |
|---|---|
| Clear, specific descriptions | The model selects tools by matching intent to description text — vague descriptions cause wrong tool selection |
| Narrow, single-purpose tools | A `search_orders` tool beats an overloaded `db_query(sql)` tool — narrower surface = fewer ways to misuse it |
| Strong typing / enums over free text | Constrains the model to valid arguments, reduces malformed calls |
| Return structured, concise results | Bloated tool output wastes context and buries the signal — see [[agents/concepts/context-engineering]] |
| Idempotent where possible | Safe to retry after a transient failure without double side-effects |

---

## Error handling

Tool calls fail: bad arguments, API timeouts, permission denials, empty results. A well-designed harness returns the **error as an observation**, not as a crash — the model can then decide to retry, try a different tool, or ask for help.

```
Action: get_weather(city="Atlantis")
Observation: Error: city not found. Did you mean a real city name?
Thought: That city doesn't exist; I should ask the user to clarify.
```

Silently swallowing errors, or crashing the whole loop on one bad call, are the two most common production mistakes.

---

## Anticipated Questions

1. "Does the model actually run the code?" — No, never directly. The model only emits a structured request; execution is entirely the orchestrator's responsibility. This separation is what makes permissioning and sandboxing possible — see [[agents/concepts/guardrails-and-safety]].
2. "What happens if the model calls a tool with the wrong arguments?" — Depends on validation. Good harnesses validate against the JSON Schema before executing and return a validation error as an observation so the model can self-correct, rather than executing malformed input.
3. "How is this different from a plugin system?" — Conceptually similar; tool calling is the *model-facing* half (schema + structured output). [[agents/concepts/mcp-protocol]] standardizes the *transport and discovery* half so tools aren't reinvented per application.
4. "Can a model call a tool that doesn't exist / hallucinate a tool?" — It can attempt to (a hallucinated tool name or made-up arguments); the harness must validate the call against the actual registered tool set and reject unknown calls with a clear error observation.

---

## Sources
- [[agents/concepts/agent-loop]]
- [[agents/concepts/mcp-protocol]]
- [[agents/concepts/guardrails-and-safety]]
