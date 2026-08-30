# Agent Fundamentals

**Related concepts:** [[agents/concepts/what-is-an-agent]], [[agents/concepts/agent-loop]], [[agents/concepts/reasoning-and-planning]]

## The Core Idea

An AI agent is an LLM given the ability to act (via tools) and the autonomy to decide, in a loop, what action to take next based on what it observes — until a goal is met. This is the single unifying idea behind everything else in this knowledge base; see [[agents/concepts/what-is-an-agent]] for the full definition and the workflow-vs-agent distinction.

```
   ┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
   │ THINK         │───►│ ACT           │───►│ EXECUTE       │───►│ OBSERVE       │
   └───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
   reason           emit tool        orchestrator     result fed
   over goal        call or          runs the tool    back into
   + history        final answer                      context

   (OBSERVE loops back to THINK with the new result)
```
Full detail: [[agents/concepts/agent-loop]].

## The Spectrum from LLM Call to Agent

| | Single LLM call | Workflow | Agent |
|---|---|---|---|
| Who decides what happens next | N/A (one call) | Developer's code | The model, at runtime |
| Predictability | High | High | Lower |
| Flexibility | None | Fixed | High |
| Best for | Simple, single-step tasks | Known, repeatable processes | Open-ended, exploratory tasks |

See [[agents/concepts/what-is-an-agent]] for the full spectrum diagram and the "when to use an agent" decision table.

## The Building Blocks

| Block | What it does | Deep dive |
|---|---|---|
| Reasoning | Decides the next step, decomposes goals | [[agents/concepts/reasoning-and-planning]] |
| Tool use | Lets the agent act on the world | [[agents/concepts/tool-calling]], [[agents/topics/tool-use]] |
| Memory | Retains state within and across sessions | [[agents/concepts/memory-architectures]], [[agents/topics/memory-and-context]] |
| Guardrails | Bounds what the agent is allowed to do | [[agents/concepts/guardrails-and-safety]] |
| Evaluation | Measures whether the agent works | [[agents/concepts/agent-evaluation]] |

## Under the Hood: What the "L" in LLM Actually Is

Everything above treats the model as a given. For students without an ML background, it's worth spending a session on what's actually happening inside the "Think" step before going further — the rest of the KB (hallucination, context limits, RAG, memory) makes a lot more sense once these aren't black boxes:

| Foundation | What it explains | Page |
|---|---|---|
| Neural networks | What the model fundamentally is and how it learns | [[agents/foundations/neural-networks]] |
| Transformers & attention | How the model actually processes a sequence of tokens | [[agents/foundations/transformers-and-attention]] |
| NLP fundamentals (tokenization) | Why "context window" and cost are measured in tokens, not words | [[agents/foundations/nlp-fundamentals]] |
| Embeddings & similarity | How "semantic search" and "similar meaning" are computed | [[agents/foundations/embeddings-and-similarity]] |
| Vector databases | How similarity search stays fast at scale — the engine behind RAG and agent memory | [[agents/foundations/vector-databases]] |

## Origin and Terminology Note

The term "agent" predates LLMs — it comes from classical AI (an entity that perceives its environment and acts upon it, going back to Russell & Norvig's *perceive-reason-act* framing). What's new with LLM-based agents is that the *reasoning* step is now a general-purpose language model rather than hand-coded rules or a narrow trained policy, which is what makes them applicable to open-ended, previously unprogrammable tasks.

## Anticipated Questions

- "Is 'AI agent' a precise technical term or a marketing term?" — Both, depending on who's using it. Technically precise usage centers on the workflow-vs-agent distinction ([[agents/concepts/what-is-an-agent]]); marketing usage often applies "agent" to any LLM product regardless of whether the model actually controls the flow. Teach the precise version first.
- "Do agents need to be autonomous end-to-end, with zero human involvement?" — No — see the tiered human-in-the-loop discussion in [[agents/concepts/guardrails-and-safety]]. Most production agents are autonomous within bounds, not unsupervised.
- "What's the minimum viable agent?" — An LLM, one tool, and a loop with a stopping condition. Everything else (memory, multi-agent, planning) is an addition layered on when the task demands it.

## Sources
- [[agents/concepts/what-is-an-agent]]
- [[agents/concepts/agent-loop]]
- [[agents/foundations/neural-networks]]
- [[agents/foundations/transformers-and-attention]]
