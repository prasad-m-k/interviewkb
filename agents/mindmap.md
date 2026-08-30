# Agentic AI — Mindmap

## Foundations (Underlying ML Concepts)

### Neural Networks
- Weighted sum → non-linear activation, stacked in layers
- Learns via forward pass → loss → backprop → gradient update
- LLM = a neural network trained on next-token prediction
- [[agents/foundations/neural-networks]]

### Transformers and Attention
- Tokens → embedding + positional encoding → self-attention → feed-forward
- Self-attention: every token weighs relevance of every other token
- Quadratic cost in sequence length — why context isn't free
- [[agents/foundations/transformers-and-attention]]

### NLP Fundamentals
- Tokenization (BPE/WordPiece) — the actual billing/context unit
- Language modeling objective: predict the next token
- [[agents/foundations/nlp-fundamentals]]

### Embeddings and Similarity
- Text → vector; similar meaning → nearby vectors
- Cosine similarity: angle between vectors
- Powers RAG, agent memory, semantic tool selection
- [[agents/foundations/embeddings-and-similarity]]

### Vector Databases
- ANN search: HNSW (layered graph), IVF (clustering), LSH, ScaNN
- Recall vs. latency vs. memory tradeoff — no free lunch
- Pinecone, Weaviate, Milvus, Qdrant, pgvector, Chroma, FAISS
- [[agents/foundations/vector-databases]]

## Fundamentals

### What Is an Agent
- Model decides control flow at runtime, not code
- Test: swap the model for a dumber one — does behavior still hold?
- Spectrum: single call → workflow → agent
- [[agents/concepts/what-is-an-agent]]

### The Agent Loop
- Think → Act → Execute → Observe, repeated
- Orchestrator executes, model never touches the world directly
- Stopping conditions: max iterations, cost budget, timeout, loop detection
- [[agents/concepts/agent-loop]]

### Reasoning and Planning
- CoT: internal reasoning only, no environment contact
- ReAct: Thought → Action → Observation, grounded in real observations
- Tree of Thought: explore/evaluate multiple branches, expensive
- Plan-and-execute: full plan upfront, re-plan on failure
- [[agents/concepts/reasoning-and-planning]]

## Tool Use

### Tool Calling
- Schema (JSON Schema) → structured model call → orchestrator executes → result fed back
- Parallel vs. sequential calls
- Narrow, single-purpose tools beat generic ones
- [[agents/concepts/tool-calling]]

### MCP (Model Context Protocol)
- Host → Client (1:1) → Server
- Server exposes: Tools, Resources, Prompts
- Solves the N×M integration problem
- [[agents/concepts/mcp-protocol]]

## Memory and Context

### Memory Architectures
- Short-term: live context window
- Long-term: semantic / episodic / procedural
- Storage: buffer, KV store, vector DB
- [[agents/concepts/memory-architectures]]

### Context Engineering
- Naive accumulation fails: cost, latency, lost-in-the-middle
- Compaction, selective retrieval, tool-output trimming, sub-agent isolation
- [[agents/concepts/context-engineering]]

### Agentic RAG
- Model decides whether/how many times to retrieve
- Multi-hop retrieval, query reformulation
- [[agents/concepts/agentic-rag]]

## Architectures

### Single-Agent Patterns
- ReAct — [[agents/patterns/react-pattern]]
- Plan-and-execute — [[agents/patterns/plan-and-execute]]
- Reflection (generate → critique → revise) — [[agents/patterns/reflection-pattern]]
- Agentic RAG — [[agents/patterns/agentic-rag-pattern]]

### Multi-Agent Systems
- Sequential, supervisor-worker, parallel fan-out, evaluator-optimizer, peer-to-peer
- Supervisor-worker most common in production — [[agents/patterns/supervisor-worker-pattern]]
- Coordination tax: token cost, handoff design, error compounding
- [[agents/concepts/multi-agent-orchestration]]

## Safety and Quality

### Guardrails
- Execution-boundary enforcement, not just prompted instructions
- Tool permissioning, human-in-the-loop approval gates, sandboxing
- Prompt injection: data vs. instructions must be separated
- [[agents/concepts/guardrails-and-safety]]

### Agent Evaluation
- Outcome, trajectory, efficiency, safety — four levels
- LLM-as-judge, human eval, regression suites, production tracing
- [[agents/concepts/agent-evaluation]]

## Applied System Design
- Customer support agent — [[agents/scenarios/customer-support-agent-design]]
- Coding agent — [[agents/scenarios/coding-agent-design]]
- Research agent — [[agents/scenarios/research-agent-design]]
- Debugging playbook — [[agents/scenarios/agent-debugging-playbook]]
