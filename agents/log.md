# Agentic AI Knowledge Base — Log

## [2026-07-07] ingest | Initial Agentic AI knowledge base build
- Created: index, log, Agentic AI overview, mindmap
- Created topics: agent-fundamentals, agent-architectures, tool-use, memory-and-context, multi-agent-systems
- Created concepts: what-is-an-agent, agent-loop, reasoning-and-planning, tool-calling, memory-architectures, context-engineering, mcp-protocol, agentic-rag, guardrails-and-safety, agent-evaluation, multi-agent-orchestration
- Created patterns: react-pattern, plan-and-execute, reflection-pattern, supervisor-worker-pattern, agentic-rag-pattern
- Created scenarios: customer-support-agent-design, coding-agent-design, research-agent-design, agent-debugging-playbook
- Created flashcards: agentic-ai-top20
- Notes: Instructor-prep build for a training on AI Agents/Agentic AI. Every concept and pattern page uses an ASCII diagram and ends with an "Anticipated Questions" section aimed at what students actually ask. Cross-links into the existing ml/ knowledge base where relevant (ml/concepts/rag, ml/concepts/embeddings, ml/topics/model-evaluation) rather than duplicating that material — agentic-rag builds directly on the existing rag and rag-pattern pages.

## [2026-07-07] update | ASCII diagram alignment fixes across the whole KB
- Rebuilt every misaligned diagram (box widths, branch connectors, arrow columns) using a small verified-alignment method rather than hand-eyeballed spacing
- Fixed in: agent-loop, what-is-an-agent, agent-evaluation, agentic-rag, context-engineering, guardrails-and-safety, mcp-protocol, memory-architectures, multi-agent-orchestration, reasoning-and-planning, tool-calling, agent-fundamentals, tool-use, agent-architectures, supervisor-worker-pattern, coding-agent-design, customer-support-agent-design, research-agent-design
- Also fixed 4 instances where a `[[wikilink]]` had been accidentally split across two lines inside a code fence (rendered as broken literal text) — moved the reference into surrounding prose instead
- Notes: root cause was hand-typed ASCII where box borders and connector columns drifted; fix verified programmatically (box top/bottom widths and interior wall columns checked to match) rather than re-eyeballed.

## [2026-07-08] ingest | Foundations section: neural networks, transformers, embeddings, vector databases, NLP
- Created foundations/: neural-networks (what a NN is, how it learns, why LLMs are NNs), transformers-and-attention (token embedding → self-attention → FFN pipeline, why context isn't free), embeddings-and-similarity (vector representations, cosine similarity), vector-databases (ANN algorithms — HNSW/IVF/LSH/ScaNN, index tradeoffs, popular systems, how agents use them for RAG/memory/multi-agent shared memory), nlp-fundamentals (tokenization, language modeling objective)
- Updated: index.md (new Foundations section), mindmap.md (new Foundations section), memory-architectures, agentic-rag, context-engineering, what-is-an-agent, agent-loop, multi-agent-orchestration, agent-fundamentals, memory-and-context, tool-use, multi-agent-systems, agent-architectures (added links out to the new foundations pages wherever they previously just name-dropped these concepts)
- Notes: Requested by the user to help students without an ML background understand what's happening underneath the agent loop. Each foundations page is explicitly framed "for agent builders" — light on math, heavy on why the mechanism matters for agent design (hallucination, context cost, RAG, memory) — and links out to the deeper ml/concepts/ pages (embeddings, attention-mechanism, transformers, llm-fundamentals) rather than duplicating them. All new ASCII diagrams built and verified with the same alignment method as the diagram-fix pass above.
