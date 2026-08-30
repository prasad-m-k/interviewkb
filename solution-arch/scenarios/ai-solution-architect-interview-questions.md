# AI Solution Architect — Scenario Interview Questions

**Related:** [[solution-arch/topics/ai-solution-architecture]], [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/openai-platform-architecture]], [[solution-arch/scenarios/interview-questions]]

> Complements [[solution-arch/scenarios/interview-questions]] (classic SA fundamentals — read that one first if system design interviewing is new to you). These 28 questions are specific to the AI/Agentic/OpenAI layer. Answer framework for all of them: state the trade-off, not just a tool name.

---

## Role & Skills

### Q1: What's the difference between an AI Solution Architect and an ML Engineer?
**A:** An ML Engineer builds and trains models — feature pipelines, model architecture, training infra. An AI Solution Architect designs how an LLM/agent fits into a larger enterprise system — integration, security, cost, governance, resilience — usually *consuming* models via API rather than training them. Full comparison: [[solution-arch/topics/ai-solution-architecture]].

### Q2: A stakeholder asks you to "just add AI" to an existing product. How do you respond?
**A:** Push back to a concrete problem statement first: what decision or task is currently slow/inconsistent/expensive that an LLM would measurably improve? Then run the decision framework — prompting vs RAG vs fine-tuning vs agentic (see [[solution-arch/topics/ai-solution-architecture]]) — rather than defaulting to the most complex/impressive-sounding solution. "Just add AI" without a target problem produces a Level 0 chatbot wrapper that fails silently in production.

### Q3: How do you decide if a task needs an agent vs a simpler LLM call?
**A:** If the task can be answered with a single well-structured prompt (possibly with retrieval), don't build an agent. Reach for agentic architecture only when the task requires an unpredictable number of steps or tool calls determined at runtime based on intermediate results. See the complexity spectrum in [[solution-arch/patterns/agentic-workflow-patterns]].

---

## Agentic AI Architecture

### Q4: Walk me through the architecture of a tool-using agent.
**A:** The ReAct-style loop: perceive (build context) → reason (LLM call decides next action) → act (execute tool call with validation) → observe (append result) → repeat until a final answer or a hard stop (max iterations/budget/timeout). Full diagram: [[solution-arch/topics/agentic-ai-architecture]].

### Q5: How do you prevent an agent from looping forever or blowing through cost?
**A:** Hard caps on max iterations, max tool calls, max tokens, and wall-clock timeout per request — the agentic equivalent of a missing circuit breaker. Without these, a confused agent is an unbounded cost and latency risk, not just a quality risk.

### Q6: When would you use a multi-agent system instead of one agent?
**A:** When sub-tasks need genuinely different tools/context that would bloat a single agent's context window, when safety requires structural separation (a reviewer agent that can never take write actions), or when independent sub-tasks benefit from parallel execution. Full checklist: [[solution-arch/patterns/multi-agent-orchestration]]. Don't reach for multi-agent just because it sounds more sophisticated — it multiplies token cost and introduces new coordination failure modes.

### Q7: What's the difference between a "workflow" and an "agent"?
**A:** A workflow is a fixed, developer-defined sequence of LLM calls and tool invocations. An agent is a system where the model's own output dynamically decides the sequence — the model is the router, not the developer's code. Most production systems are a hybrid: a deterministic outer workflow with agentic sub-steps where variability genuinely requires it.

### Q8: How would you design memory for a long-running agent conversation?
**A:** Layer it: short-term/working memory is the active context window, managed via sliding window or summarization as it grows; long-term memory persists facts/past interactions in a vector store, retrieved like RAG at session start; episodic memory logs past agent actions/outcomes to avoid repeating failed strategies. Detail: [[solution-arch/topics/agentic-ai-architecture]].

### Q9: An agent needs to call a "delete_account" tool. How do you architect that safely?
**A:** Human-in-the-loop gate is mandatory for irreversible, high-cost actions — never auto-execute. Pause the agent's state, surface the proposed action with reasoning to a human, resume on approval. The execution itself must be idempotent in case of retries. Full pattern: [[solution-arch/patterns/human-in-the-loop]].

### Q10: How do you evaluate whether an agent is working correctly, given non-deterministic output?
**A:** Trajectory evaluation (did it take the right sequence of actions, not just reach a plausible final answer) plus outcome evaluation against a golden set, gated in a CI-like pipeline before any prompt/model change ships. See [[solution-arch/concepts/llm-observability-and-evals]].

---

## OpenAI Platform

### Q11: When would you use the Assistants API vs building your own orchestration with Chat Completions?
**A:** Assistants API when you want a managed, stateful agent runtime and don't need custom orchestration control — fastest to stand up. Chat Completions when you need full control over the tool-call loop, context management, or must remain provider-portable. Full trade-off table: [[solution-arch/topics/openai-platform-architecture]].

### Q12: How does OpenAI function calling actually work end to end?
**A:** You register tools with JSON Schema; the model, given the conversation, emits a tool_call (name + JSON string arguments) instead of a final answer when it decides a tool is needed; your code parses/validates the arguments, executes the real function, and appends the result back into the conversation before calling the model again. Full detail: [[solution-arch/concepts/function-calling-and-tool-use]].

### Q13: Your team wants the model to always return valid JSON matching a specific schema — how?
**A:** Use Structured Outputs (`strict` JSON Schema mode), which constrains the output at the token-sampling level, not just via a prompted request. Eliminates the "usually valid JSON but occasionally malformed" class of bug that pure prompting leaves.

### Q14: When is fine-tuning the right answer, and when is it a trap?
**A:** Right for consistent output format/style/behavior, or for shrinking to a cheaper smaller model at equivalent task quality. Wrong for "teach the model new facts" — fine-tuning doesn't reliably inject new knowledge and the model can still hallucinate around fine-tuned facts; use RAG for that instead. Decision framework: [[solution-arch/topics/ai-solution-architecture]].

### Q15: Would you recommend OpenAI direct or Azure OpenAI for a regulated financial services client?
**A:** Azure OpenAI, in most cases — inherits the existing Azure AD identity, network isolation (Private Link/VNet), and compliance certifications (SOC 2, HIPAA, FedRAMP) the enterprise already has, plus regional deployment control for data residency. OpenAI direct wins only when access to the newest models before Azure catches up is the dominant requirement. Full comparison: [[solution-arch/topics/openai-platform-architecture]].

### Q16: How do you handle OpenAI's rate limits at enterprise scale?
**A:** Client-side request queuing with exponential backoff + jitter on 429s, model routing to spread load, and — at real scale — negotiating a provisioned-throughput tier rather than relying on shared-pool limits. Same resilience thinking as [[solution-arch/concepts/rate-limiting]], applied to your system as the client of an external rate-limited dependency.

---

## RAG & LLM Application Architecture

### Q17: Design a RAG system that respects existing document permissions.
**A:** Capture the source system's ACLs as metadata on every indexed chunk at ingestion; filter the retrieval query itself by the asking user's current group membership (not a post-hoc check after retrieval, and not trusted to the LLM to enforce). Full walkthrough: [[solution-arch/scenarios/design-enterprise-rag-system]].

### Q18: Your RAG system gives confidently wrong answers on rare edge-case questions. What's your architecture response?
**A:** First check retrieval quality (is the right document even being retrieved — a retrieval failure, not a generation failure, is the more common root cause); add output guardrails requiring citations and allowing "I don't know" as a valid answer when confidence/relevance score is low; add the failing case to the eval golden set so future changes are checked against it.

### Q19: How do you manage a growing conversation history without hitting the context window limit?
**A:** Sliding window (keep last N turns verbatim), periodic summarization of older turns, or retrieval-based memory (embed and retrieve only relevant past turns instead of keeping full history) — same mechanics as RAG, applied to conversation history. See [[solution-arch/topics/llm-application-architecture]].

### Q20: What's the LLM-application equivalent of a CI/CD pipeline?
**A:** An eval-gated deploy pipeline: every prompt/model/retrieval-config change runs against a fixed golden set before shipping, canaries to a small % of live traffic, monitors production proxy metrics (thumbs-down rate, escalation rate), then full rollout or rollback — directly analogous to [[solution-arch/patterns/blue-green-canary]] applied to a prompt artifact instead of a code artifact.

### Q21: How would you reduce a $1M/year LLM API bill without hurting quality?
**A:** In order: model routing (send simple tasks to a cheap/small model), prompt caching (structure prompts with static content first), context window discipline (summarize history, truncate retrieved docs to what's needed), response/semantic caching for repeated queries. Only after those, consider quality-affecting levers (smaller fine-tuned model, reduced retrieval depth) validated against the eval suite. Full lever list: [[solution-arch/topics/cost-architecture-finops]].

---

## MCP & Tool Integration

### Q22: What problem does the Model Context Protocol (MCP) solve that function calling alone doesn't?
**A:** Function calling defines the model-facing contract for a single request. MCP standardizes the integration layer underneath — how an application discovers and connects to a growing ecosystem of tool servers, avoiding an N×M bespoke-integration problem as both the number of AI applications and the number of tools grows. Full detail: [[solution-arch/concepts/model-context-protocol-mcp]].

### Q23: What new security boundary does MCP introduce?
**A:** An MCP server is often a separate process, potentially third-party — it becomes a new trust boundary. Apply least-privilege scoping per server, and treat data returned from an MCP server's "resources" as untrusted content subject to the same prompt-injection defense as any other retrieved content.

---

## Guardrails, Governance & Security

### Q24: Why can't you defend against prompt injection the same way you defend against SQL injection?
**A:** SQL injection exploits a syntactic boundary that escaping/parameterization closes completely. Prompt injection exploits a semantic boundary — the model has no hard separation between "instructions" and "data," both are just tokens. There's no complete syntactic fix; defense is layered and probabilistic (classifiers, least-privilege tool scoping, human gates on high-risk actions), not a single deterministic sanitization step. Full detail: [[solution-arch/concepts/ai-guardrails-and-safety]].

### Q25: A regulator asks why your AI system denied a customer's application. What do you show them?
**A:** A full, immutable trace: the exact prompt sent, the retrieved context, the model version used, the output, and any tool calls made — reconstructable for the specific point in time the decision was made (requires versioned prompts and versioned retrieval index snapshots, not just "current" state). This requirement should shape the architecture from day one, not be retrofitted. See [[solution-arch/topics/ai-governance-responsible-ai]].

### Q26: How do you prevent an agent's tool call from executing twice if it retries after a timeout?
**A:** Every side-effecting tool call carries an idempotency key (derived from conversation/turn ID); the downstream service dedupes on that key. Same principle as [[solution-arch/concepts/idempotency]] in traditional distributed systems, now triggered by a non-deterministic caller instead of a flaky network client.

### Q27: How would you classify an AI use case's risk tier, and why does it matter architecturally?
**A:** Under a framework like the EU AI Act's risk tiers (unacceptable/high/limited/minimal), the tier determines what governance infrastructure is legally required — logging, human oversight, documented accuracy testing — not just best-practice advice. Classify the risk tier FIRST, before designing the system, because it determines mandatory architecture, not optional hardening. Full detail: [[solution-arch/topics/ai-governance-responsible-ai]].

---

## Cost, Enterprise Architecture & Trade-offs

### Q28: Frame the build-vs-buy decision for an internal AI agent platform vs adopting a vendor's Assistants API.
**A:** Weigh control over orchestration logic, vendor lock-in/exit cost, team skill, time-to-market, and whether agent orchestration is a genuine differentiator for the business or undifferentiated plumbing. Vendor-managed (buy) usually wins for time-to-market on a first use case; custom orchestration (build) usually wins once you need fine-grained control over guardrails, multi-provider routing, or have scale that makes vendor per-call pricing costly. Full framework: [[solution-arch/topics/enterprise-architecture-frameworks]].

---

## Sources
- [[solution-arch/scenarios/interview-questions]]
- [[solution-arch/topics/ai-solution-architecture]]
