# Agentic AI Architecture

**Related:** [[solution-arch/topics/ai-solution-architecture]], [[solution-arch/topics/openai-platform-architecture]], [[solution-arch/topics/llm-application-architecture]]
**Concepts:** [[solution-arch/concepts/function-calling-and-tool-use]], [[solution-arch/concepts/model-context-protocol-mcp]], [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/concepts/llm-observability-and-evals]]
**Patterns:** [[solution-arch/patterns/agentic-workflow-patterns]], [[solution-arch/patterns/multi-agent-orchestration]], [[solution-arch/patterns/human-in-the-loop]]
**Scenarios:** [[solution-arch/scenarios/design-agentic-customer-support-system]]

---

## What "Agentic" Means (Precisely)

An **agent** is an LLM wrapped in a loop where the model's *own output* decides what happens next — which tool to call, whether to keep going, when to stop — rather than a human or a fixed script making that decision. This is the load-bearing distinction interviewers probe for:

```
NOT agentic (a workflow):                 AGENTIC:
  Fixed sequence, developer-defined:        Model decides the sequence:

  Step 1: call retrieval API                 while not done:
  Step 2: call LLM with retrieved docs          response = llm(context, tools)
  Step 3: return answer                        if response.wants_tool_call:
                                                    result = execute(tool)
  → Deterministic, easy to test,                    context += result
    but can't adapt to novel situations          else:
                                                    done = True
                                                    return response
                                               → Non-deterministic control flow;
                                                 the model is the router.
```

Anthropic's engineering distinction (widely cited in interviews): **workflows** are systems where LLM calls and tools are orchestrated through predefined code paths; **agents** are systems where the LLM dynamically directs its own process and tool usage. Most production "agentic" systems are actually a *mix* — a deterministic outer workflow with agentic sub-steps. Recommend starting with the simplest composition (a single well-prompted call) and adding agentic complexity only when task variability demands it. Over-engineering an agent for a task a fixed workflow handles fine is a common architecture smell.

---

## The Core Agent Loop

```
┌─────────────────────────────────────────────────────────────┐
│                      SINGLE-AGENT LOOP                       │
└─────────────────────────────────────────────────────────────┘

   User Input
       │
       ▼
 ┌──────────────┐
 │  Perceive     │  Gather context: conversation history,
 │  (build       │  retrieved documents, tool results so far,
 │  context)     │  system prompt with available tools
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │   Reason      │  LLM call: "given this context, what's
 │  (LLM call)   │  the next action?" → tool call, or final answer
 └──────┬───────┘
        │
        ▼
   ┌────────────┐        NO
   │ Tool call? ├──────────────────┐
   └─────┬──────┘                  │
        YES                        ▼
        │                    Return final answer
        ▼                    to user; loop ends
 ┌──────────────┐
 │     Act       │  Execute tool (API call, DB query,
 │  (execute     │  code execution, search) — with
 │   tool)       │  guardrails/validation on inputs
 └──────┬───────┘
        │
        ▼
 ┌──────────────┐
 │   Observe     │  Append tool result to context
 │  (append      │
 │   result)     │
 └──────┬───────┘
        │
        └──────────► loop back to Reason
                     (bounded by max_iterations / budget)
```

This loop is the basis of **ReAct** (Reason + Act), the foundational agent pattern almost every framework (LangGraph, OpenAI's Agents SDK, Claude's tool use, AWS Bedrock Agents) implements under different names.

**Critical architecture requirement interviewers look for:** a hard stop condition. Every agent loop needs a max-iteration cap, a token/cost budget, and a timeout — otherwise a confused agent loops indefinitely and burns cost. This is the agentic equivalent of a missing circuit breaker.

---

## Agent Design Patterns (Building Blocks)

These compose — a production system is usually 2-3 of these combined, not one in isolation.

```
1. AUGMENTED LLM (the atomic unit)
   LLM + retrieval + tools + memory, all behind one call.
   Every pattern below is built from this atomic unit.

        ┌─────────────────────────────┐
        │         LLM call            │
   in ─▶│  ┌────────┐  ┌───────────┐  │▶ out
        │  │Retrieval│  │  Tools    │  │
        │  └────────┘  └───────────┘  │
        │       Memory (session)      │
        └─────────────────────────────┘

2. PROMPT CHAINING (deterministic workflow, not agentic)
   Task decomposed into fixed sequential LLM calls; each step's
   output feeds the next. Add a gate (code-based check) between
   steps to fail fast.

   in ─▶ [LLM 1] ─▶ (gate?) ─▶ [LLM 2] ─▶ (gate?) ─▶ [LLM 3] ─▶ out

3. ROUTING
   Classify the input, then dispatch to a specialized prompt/model.
   Good when inputs fall into distinct categories that are better
   handled separately (e.g. billing question vs technical question).

                    ┌──▶ [Specialist prompt A] (e.g. refunds)
   in ─▶ [Router] ──┼──▶ [Specialist prompt B] (e.g. tech support)
                    └──▶ [Specialist prompt C] (e.g. general Q&A)

4. PARALLELIZATION
   Sectioning: split a task into independent subtasks run in
   parallel, then aggregate. Voting: run the same task N times
   and take a majority/consensus result (raises reliability for
   high-stakes classification).

   in ─▶ ┌─▶ [LLM: subtask 1] ─┐
         ├─▶ [LLM: subtask 2] ─┼─▶ [Aggregate] ─▶ out
         └─▶ [LLM: subtask 3] ─┘

5. ORCHESTRATOR-WORKER
   A central orchestrator LLM dynamically breaks a task into
   subtasks (not fixed in advance) and delegates to worker LLMs,
   then synthesizes results. Differs from routing/parallelization
   because subtasks aren't known ahead of time.

   in ─▶ [Orchestrator LLM] ──dynamically spawns──▶ [Worker 1]
                              │                     [Worker 2]
                              │                     [Worker N]
                              ◀──── results ────────┘
                              │
                              ▼
                        [Synthesizer]  ─▶ out

6. EVALUATOR-OPTIMIZER
   One LLM generates a response; a second LLM (or the same one in
   a separate call) critiques it against explicit criteria; loop
   until it passes or a max-iteration cap is hit. Effective when
   evaluation criteria are clear and iterative refinement measurably
   improves quality (e.g. code generation, translation).

   in ─▶ [Generator] ─▶ [Evaluator] ──fail──▶ back to Generator
                              │                (with feedback)
                             pass
                              ▼
                             out

7. AUTONOMOUS AGENT (full agentic loop)
   The pattern described in "The Core Agent Loop" above — used
   when the number of steps can't be predicted and the model must
   make open-ended decisions. Requires the most guardrails.
```

See [[solution-arch/patterns/agentic-workflow-patterns]] for template code and when-to-use decision criteria per pattern.

---

## Multi-Agent Systems

When a single agent's context grows too large, or distinct specialized capabilities are needed, split into multiple cooperating agents. See [[solution-arch/patterns/multi-agent-orchestration]] for full pattern detail; summary here:

```
SUPERVISOR (hierarchical) PATTERN — most common in production

                    ┌─────────────────┐
                    │   Supervisor     │  Decides which sub-agent
                    │   Agent          │  handles the next step;
                    └────────┬────────┘  owns the overall goal
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   ┌─────────────┐   ┌─────────────┐    ┌─────────────┐
   │ Research     │   │ Coding       │    │ Review       │
   │ Agent        │   │ Agent        │    │ Agent        │
   │ (tools:      │   │ (tools:      │    │ (tools: none;│
   │  web search) │   │  code exec)  │    │  critiques)  │
   └─────────────┘   └─────────────┘    └─────────────┘

Each sub-agent has its OWN context window, own tool access,
own system prompt — this is what lets multi-agent systems handle
tasks too large for one context window, at the cost of coordination
overhead and higher total token spend.
```

**When multi-agent is worth the complexity (interview framing):**
```
Use multi-agent when:
  ✅ Task naturally decomposes into independent parallel work
     (e.g. research across many sources simultaneously)
  ✅ Sub-tasks need genuinely different tools/context that would
     bloat a single agent's context window
  ✅ You need a separation of concerns for safety (e.g. a "reviewer"
     agent that never gets write access, only critiques)

Avoid multi-agent when:
  ❌ A single well-structured prompt handles the task
  ❌ Sub-agents need to share a lot of state — coordination cost
     exceeds the benefit (same trap as premature microservices
     decomposition, see [[solution-arch/patterns/microservices-decomposition]])
  ❌ You can't yet evaluate a single agent reliably — adding more
     agents compounds an unmeasured failure rate
```

---

## Memory Architecture

```
Short-term (working) memory
  → The current context window: conversation turns, recent tool
    results. Bounded by model context length; must be actively
    managed (truncation, summarization) as conversations grow.

Long-term memory
  → Persisted across sessions. Usually implemented as:
    - Vector store of past interactions (semantic recall)
      see [[solution-arch/concepts/vector-databases]]
    - Structured key-value facts extracted from conversation
      ("user prefers email over SMS")
  → Retrieved and injected into context at the start of a new
    session, same mechanics as RAG.

Episodic memory
  → A log of past agent *actions and outcomes* (not just chat),
    used to avoid repeating failed strategies or to few-shot the
    agent with its own successful past trajectories.
```

**Context window management is itself an architecture problem**, not just a model limitation — see [[solution-arch/concepts/prompt-engineering-and-context-design]] for compaction, summarization, and sliding-window strategies used once a long-running agent session approaches its context limit.

---

## Guardrails and Failure Containment for Agents

Agents that can *take actions* (not just generate text) need containment the same way a distributed system needs bulkheads. This is the single most under-designed area in candidate answers.

```
┌────────────────────────────────────────────────────────────┐
│              AGENT GUARDRAIL STACK                          │
├────────────────────────────────────────────────────────────┤
│ Input guardrail    → sanitize/classify user input before    │
│                      it reaches the agent (prompt injection, │
│                      jailbreak detection)                    │
├────────────────────────────────────────────────────────────┤
│ Tool allow-list     → agent can only call explicitly         │
│                      registered tools with typed schemas;    │
│                      never allow arbitrary code execution    │
│                      without a sandbox                       │
├────────────────────────────────────────────────────────────┤
│ Permission scoping  → each tool call runs with the MINIMUM   │
│                      privilege needed (a "refund" tool should│
│                      not have delete-account privileges)     │
├────────────────────────────────────────────────────────────┤
│ Idempotency on      → side-effecting tools (charge, email,   │
│ side-effecting      │  refund) must be idempotent — an agent │
│ tools               │  retry must not double-execute         │
│                      │  (see [[solution-arch/concepts/idempotency]])│
├────────────────────────────────────────────────────────────┤
│ Human-in-the-loop   → high-risk actions (irreversible,       │
│ checkpoint          │  high monetary value, destructive)      │
│                      │  require explicit human approval before│
│                      │  execution — see                       │
│                      │  [[solution-arch/patterns/human-in-the-loop]]│
├────────────────────────────────────────────────────────────┤
│ Output guardrail    → validate/filter the final response     │
│                      (PII leakage, policy violations) before  │
│                      it reaches the user                      │
├────────────────────────────────────────────────────────────┤
│ Budget & iteration  → max tokens, max tool calls, max wall-   │
│ cap                  clock time per request — prevents runaway│
│                      loops and cost blowouts                  │
└────────────────────────────────────────────────────────────┘
```

Full detail: [[solution-arch/concepts/ai-guardrails-and-safety]].

---

## Evaluation: How You Know the Agent Actually Works

Traditional unit tests don't work on non-deterministic output. Architecture must budget for:

```
1. Trajectory evaluation
   Did the agent take the RIGHT SEQUENCE of tool calls, not just
   arrive at a plausible-looking final answer? (e.g., did it check
   inventory before confirming an order?)

2. Outcome evaluation
   Compare final output against a labeled golden set — exact match
   for structured tasks, LLM-as-judge for open-ended ones.

3. Regression suite gate
   Every prompt/model/tool change runs against a fixed eval set
   before deploy — this is the CI/CD equivalent for agentic systems.

4. Production monitoring
   Sample live traffic, flag low-confidence or anomalous
   trajectories for human review; feed back into the eval set.
```

Full detail: [[solution-arch/concepts/llm-observability-and-evals]].

---

## Sources
- [[solution-arch/sources/building-effective-agents-anthropic]]
- [[ml/concepts/reinforcement-learning]]
- [[ml/scenarios/llm-service-design]]
