---
tags:
  - index
  - agents
  - agentic-ai
  - ai-agents
  - llm
  - interview-prep
  - training
---

# Agentic AI Knowledge Base — Index
Last updated: 2026-07-08

## Overview
- [[agents/Agentic AI overview]] — High-level synthesis, mental model, and teaching strategy

## Foundations
*Supporting ML concepts for students without an ML background — start here if "neural network," "attention," "embedding," or "vector database" aren't already solid.*
- [[agents/foundations/neural-networks]] — What a neural network is, how it learns, why LLMs are neural networks
- [[agents/foundations/transformers-and-attention]] — Token embedding → self-attention → feed-forward pipeline
- [[agents/foundations/nlp-fundamentals]] — Tokenization, the language modeling objective, why context is measured in tokens
- [[agents/foundations/embeddings-and-similarity]] — Vector representations of meaning, cosine similarity
- [[agents/foundations/vector-databases]] — ANN search (HNSW/IVF), index tradeoffs, popular systems, how agents use them

## Topics
- [[agents/topics/agent-fundamentals]] — Core definition, the agent loop, building blocks
- [[agents/topics/agent-architectures]] — Single-agent and multi-agent pattern map, how to choose
- [[agents/topics/tool-use]] — Function calling, tool design, MCP
- [[agents/topics/memory-and-context]] — Memory hierarchy vs. context budget management
- [[agents/topics/multi-agent-systems]] — Topologies, coordination challenges, when to use

## Concepts
- [[agents/concepts/what-is-an-agent]] — Agent vs. workflow vs. chatbot; the defining test
- [[agents/concepts/agent-loop]] — Think-act-execute-observe loop, stopping conditions
- [[agents/concepts/reasoning-and-planning]] — CoT, ReAct, Tree of Thought, explicit planning
- [[agents/concepts/tool-calling]] — Function-calling mechanics, schemas, parallel calls
- [[agents/concepts/memory-architectures]] — Short-term/long-term, semantic/episodic/procedural
- [[agents/concepts/context-engineering]] — Compaction, retrieval, sub-agent isolation
- [[agents/concepts/mcp-protocol]] — Model Context Protocol: host/client/server, tools/resources/prompts
- [[agents/concepts/agentic-rag]] — Model-directed retrieval, multi-hop, query reformulation
- [[agents/concepts/guardrails-and-safety]] — Permissioning, approval gates, prompt injection defense
- [[agents/concepts/agent-evaluation]] — Trajectory eval, LLM-as-judge, regression suites
- [[agents/concepts/multi-agent-orchestration]] — Topologies, coordination tax, when to split

## Patterns
- [[agents/patterns/react-pattern]] — Thought → Action → Observation, skeleton + worked example
- [[agents/patterns/plan-and-execute]] — Upfront plan + execution + re-planning on failure
- [[agents/patterns/reflection-pattern]] — Generate → critique → revise
- [[agents/patterns/supervisor-worker-pattern]] — Orchestrator routes to specialized workers
- [[agents/patterns/agentic-rag-pattern]] — Multi-hop retrieval loop, query reformulation skeleton

## Scenarios
- [[agents/scenarios/customer-support-agent-design]] — Tiered autonomy, refund gating, escalation
- [[agents/scenarios/coding-agent-design]] — Context management at scale, verification discipline
- [[agents/scenarios/research-agent-design]] — Supervisor-worker + agentic RAG, groundedness
- [[agents/scenarios/agent-debugging-playbook]] — Systematic checklist for loops, hallucination, cost blowups

## Flashcards
- [[agents/flashcards/agentic-ai-top20]] — 20 Q&A covering fundamentals through evaluation

## Mindmap
- [[agents/mindmap]] — Visual overview (obsidian-mind-map compatible)
