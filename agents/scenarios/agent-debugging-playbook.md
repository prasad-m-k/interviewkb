# Scenario: Agent Debugging Playbook

**Category:** Production Debugging
**Difficulty:** Medium
**Related:** [[agents/concepts/agent-loop]], [[agents/concepts/agent-evaluation]], [[agents/concepts/context-engineering]], [[agents/concepts/guardrails-and-safety]]

---

## The Scenario

Your agent isn't behaving: it's looping without progress, calling the wrong tools, running up cost, or silently producing wrong answers with high confidence. This is a systematic checklist for diagnosing *why*, using the full trajectory (thoughts, actions, observations), not just the final output — see [[agents/concepts/agent-evaluation]] on why trajectory-level tracing is required.

---

## Step 1: Classify the symptom

| Symptom | Likely category |
|---|---|
| Same action repeated, no new information gained | Infinite/repeating loop |
| Tool called with malformed or nonsensical arguments | Tool-calling failure |
| Correct-sounding but factually wrong final answer | Hallucination / ungrounded synthesis |
| Agent forgets something stated earlier in a long session | Context/memory failure |
| Cost or step count far exceeds expectation for the task | Runaway loop / inefficient trajectory |
| Agent takes an action it shouldn't have been allowed to | Guardrail failure |

---

## Step 2: Infinite / repeating loop

```python
# Detect: hash the last N actions; flag if the same action repeats
# without a changing observation
recent_actions = [step.action for step in history[-5:]]
if len(set(recent_actions)) == 1:
    flag_stuck_loop()
```

Common causes:
- A failing tool call whose error message doesn't give the model enough information to try something different — fix the error message to be actionable, not just "Error: failed."
- No max-iteration ceiling — see the stopping-condition table in [[agents/concepts/agent-loop]].
- The model re-derives the same (wrong) plan every time because the context doesn't clearly show that the previous attempt already failed — make failures explicit and salient in the observation, not buried.

---

## Step 3: Tool-calling failures

| Cause | Fix |
|---|---|
| Ambiguous tool description → wrong tool picked | Sharpen description; narrow tool scope — see [[agents/topics/tool-use]] |
| Malformed arguments (wrong type, missing required field) | Validate against schema before executing; return a specific validation error as the observation, not a generic failure |
| Model hallucinates a tool that doesn't exist | Harness must reject unknown tool names explicitly and inform the model of the actual available tool set |
| Tool succeeds but returns unusable/bloated output | Trim and structure tool output — see [[agents/concepts/context-engineering]] |

---

## Step 4: Hallucination / ungrounded synthesis

```
Check: does the final answer contain any claim NOT traceable
to a specific Observation in the trajectory?

If yes → the model is answering from parametric memory instead
of the retrieved/observed evidence.
```

Fixes: strengthen the system prompt's instruction to answer only from observations, add a groundedness check step (compare claims in the draft answer against the trajectory before finalizing — a reflection-style pass, see [[agents/patterns/reflection-pattern]]), and consider requiring structured, source-attributed claims as in [[agents/scenarios/research-agent-design]].

---

## Step 5: Context / memory failures

| Symptom | Diagnosis |
|---|---|
| Agent contradicts something stated 20 turns ago | Likely truncated or summarized out of context — check what's actually still in the live context window |
| Agent re-asks for information already provided | Same as above, or the information was never written to memory in the first place |
| Agent's behavior degrades as the session gets long | Classic context-overflow symptom — see the failure-mode table in [[agents/concepts/context-engineering]] |

Debug by inspecting exactly what was in context at the failing step — not what the full session history contains, but what actually survived compaction/truncation into that specific model call.

---

## Step 6: Runaway cost / inefficient trajectory

Compare against a reference trajectory or expected step count for this task class (see trajectory evaluation in [[agents/concepts/agent-evaluation]]).

```
Actual: 40 tool calls, $2.10
Reference: ~8 tool calls, ~$0.40 for this task class

→ investigate: redundant searches? re-deriving already-known
  information? no early-exit when the answer was already found?
```

Common cause: no relevance/sufficiency check after retrieval or search, so the agent keeps gathering more than it needs — see the relevance-check step in [[agents/patterns/agentic-rag-pattern]].

---

## Step 7: Guardrail failures

If the agent took an action it should not have been able to take at all, this is a **stop-and-fix-the-boundary** issue, not a prompting issue:

1. Was the tool even permissioned for this context? (should have been denied at the execution layer, not just discouraged in the prompt)
2. Was an approval gate bypassed or missing for a high-risk action? (see the tiered approval model in [[agents/concepts/guardrails-and-safety]])
3. Log and treat as an incident — a guardrail failure that happened once can happen again on a different input.

---

## Debug Checklist Summary

1. Pull the full trajectory, not just the final output.
2. Classify the symptom against the table in Step 1.
3. For loops: check for repeated identical actions and missing max-iteration limits.
4. For tool errors: check schema validation and error-message quality.
5. For hallucination: check every claim in the final answer against actual observations.
6. For context issues: inspect what was *actually* in context at the failing step.
7. For cost blowups: compare trajectory length against a reference for that task class.
8. For guardrail failures: fix the execution-boundary control, not just the prompt — then treat it as an incident.

---

## Sources
- [[agents/concepts/agent-loop]]
- [[agents/concepts/agent-evaluation]]
- [[agents/concepts/context-engineering]]
- [[agents/concepts/guardrails-and-safety]]
