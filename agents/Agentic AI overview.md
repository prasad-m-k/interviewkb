# Agentic AI Knowledge Base — Overview

## What This Wiki Covers
Instructor-prep material for training candidates/students on AI Agents and Agentic AI: core definitions, the mechanics that make an agent work (tool use, memory, reasoning), the standard design patterns (ReAct, plan-and-execute, multi-agent orchestration), and applied system design for common agent use cases. Written to be taught from directly — every concept page ends with anticipated student questions.

---

## Mental Model for Teaching This Material

### The One Idea Everything Else Hangs Off
An agent is a system where the **LLM decides the control flow at runtime**, not a system where code decides the control flow and an LLM fills in one step. Every other concept in this wiki — the loop, tool use, memory, multi-agent — is a refinement or extension of this one idea. Teach [[agents/concepts/what-is-an-agent]] first, and revisit it whenever a new pattern is introduced ("is this agentic, or is this a workflow with an LLM step?").

### The Four Layers a Good Course Should Build, In Order
0. **The foundations** *(skip this layer for students who already have ML background)* — what a neural network is, how a transformer processes tokens, what an embedding and a vector database actually do. ([[agents/foundations/neural-networks]], [[agents/foundations/transformers-and-attention]], [[agents/foundations/embeddings-and-similarity]], [[agents/foundations/vector-databases]])
1. **The loop** — think/act/execute/observe, and why it needs bounds. ([[agents/concepts/agent-loop]])
2. **The extensions** — tools, memory, reasoning strategies, layered onto the loop one at a time. ([[agents/topics/tool-use]], [[agents/topics/memory-and-context]], [[agents/concepts/reasoning-and-planning]])
3. **The composition** — multiple agents, evaluation, safety, applied to real system designs. ([[agents/topics/multi-agent-systems]], [[agents/concepts/agent-evaluation]], [[agents/concepts/guardrails-and-safety]])

Teaching multi-agent design before students are solid on the single-agent loop is the most common sequencing mistake — nearly every multi-agent coordination problem is a single-agent problem (context bloat, ungrounded output, unclear stopping conditions) that got distributed across more actors, not eliminated. The inverse mistake also happens: don't spend a full session on backpropagation math for a room that's ready to start building — layer 0 is diagnostic, not mandatory. A quick poll ("who can explain what a vector embedding is?") tells you whether to spend 20 minutes there or skip straight to the loop.

---

## Key Themes Across the Wiki

### 1. Every Design Choice Trades Predictability for Flexibility
Workflow → single agent → multi-agent is a ladder of increasing flexibility and decreasing predictability, at increasing cost. The recurring teaching point: justify each step up the ladder against the task's actual requirements ([[agents/concepts/what-is-an-agent]], [[agents/topics/agent-architectures]]) — sophistication is not automatically better.

### 2. The Model Never Touches the World Directly
Every safety property in this wiki traces back to one architectural fact: the model emits a *request*; an orchestrator executes it. This separation is what makes permissioning, sandboxing, and approval gates possible at all — see [[agents/concepts/tool-calling]] and [[agents/concepts/guardrails-and-safety]].

### 3. Context Is a Finite, Actively Managed Budget
LLMs are stateless between calls; "memory" and "long context" are both, mechanically, just information re-inserted into a prompt. As sessions get longer or systems get more agents, naive accumulation breaks — cost, latency, and lost-in-the-middle attention degradation. See [[agents/concepts/context-engineering]].

### 4. Outcome-Only Evaluation Is Blind to How
An agent that reaches the right answer via an unsafe or lucky path is indistinguishable from a well-behaved one under outcome-only measurement. Trajectory, efficiency, and safety metrics matter alongside task success. See [[agents/concepts/agent-evaluation]].

### 5. Patterns Compose
ReAct, plan-and-execute, reflection, agentic RAG, and supervisor-worker are not mutually exclusive — a supervisor's individual workers are often ReAct agents; a plan-and-execute agent's steps can include a reflection pass. See how [[agents/scenarios/research-agent-design]] and [[agents/scenarios/coding-agent-design]] combine multiple patterns in one system.

---

## Top 5 Foundations (for students without ML background)
1. [[agents/foundations/neural-networks]] — a neural network is weighted sums + non-linear activations, trained by gradient descent
2. [[agents/foundations/transformers-and-attention]] — self-attention lets every token weigh every other token; cost grows quadratically with length
3. [[agents/foundations/nlp-fundamentals]] — tokens (not words) are the actual unit of context and cost
4. [[agents/foundations/embeddings-and-similarity]] — similar meaning → nearby vectors; cosine similarity measures "how nearby"
5. [[agents/foundations/vector-databases]] — ANN indexes make similarity search fast at scale; no index is free on every axis (recall/latency/memory)

---

## Top 10 Concepts to Teach Cold
1. [[agents/concepts/what-is-an-agent]] — the workflow-vs-agent distinction
2. [[agents/concepts/agent-loop]] — the core loop and stopping conditions
3. [[agents/concepts/tool-calling]] — how the model actually acts on the world
4. [[agents/concepts/reasoning-and-planning]] — CoT, ReAct, plan-then-execute
5. [[agents/concepts/memory-architectures]] — short-term vs. long-term, the three memory types
6. [[agents/concepts/context-engineering]] — why naive context accumulation fails
7. [[agents/concepts/agentic-rag]] — retrieval as a tool the model chooses to use
8. [[agents/concepts/multi-agent-orchestration]] — topologies and the coordination tax
9. [[agents/concepts/guardrails-and-safety]] — execution-boundary enforcement, prompt injection
10. [[agents/concepts/agent-evaluation]] — trajectory evaluation, not just outcome

---

## Top System Design Exercises
1. Customer support agent (tiered autonomy, approval gates) → [[agents/scenarios/customer-support-agent-design]]
2. Coding agent (context at scale, verification discipline) → [[agents/scenarios/coding-agent-design]]
3. Research agent (multi-agent + agentic RAG composition) → [[agents/scenarios/research-agent-design]]
4. Live debugging drill (given a broken trajectory, diagnose it) → [[agents/scenarios/agent-debugging-playbook]]

---

## Teaching Strategy
- **Ground every abstraction in a diagram.** Students retain the loop, the topology, the architecture far better as an ASCII/box diagram than as prose — every page in this wiki leads with one for this reason.
- **Use the "swap the model" test to teach workflow vs. agent.** It's concrete and gives students a repeatable heuristic rather than a fuzzy definition.
- **Anchor multi-agent topics in cost.** Students consistently underestimate that N agents can cost more in total tokens than one — make them compute it once.
- **Gate the foundations layer on a quick diagnostic, not a blanket assumption.** A room mixing ML engineers and non-technical PMs needs layer 0 taught very differently (or skipped) for each — see [[agents/foundations/neural-networks]] onward.
- **Run the debugging playbook live.** Give students a broken trajectory transcript and have them classify the failure using [[agents/scenarios/agent-debugging-playbook]] before showing the answer — this is where the concepts actually stick.
- **Connect back to RAG they may already know.** Most students with ML background already know [[ml/concepts/rag]]; use it as the on-ramp to [[agents/concepts/agentic-rag]] rather than teaching agentic RAG cold.
