# LP03 - Invent and Simplify

> Leaders expect and require innovation and invention from their teams and always find ways to simplify. They are externally aware, look for new ideas from everywhere, and are not limited by "not invented here." As we do new things, we accept that we may be misunderstood for long periods of time.

---

## STAR Format

**Situation:** At PNC, incident runbooks existed as static Word documents and Confluence pages. Engineers on call had to manually search, read, and interpret them during live incidents — often under pressure, often with incomplete context. The knowledge was there but the delivery mechanism was slow and brittle.

**Task:** I was exploring ways to reduce mean time to resolution for recurring incident patterns. There was no budget for a new tooling platform, and any solution had to integrate with the existing environment.

**Action:** I designed a "Runbook as Code" architecture where runbooks are decomposed into structured, machine-readable stages using a five-step pipeline: ingestion, classification, boundary resolution, remediation synthesis, and dependency graph generation. The pipeline used agentic AI to surface the right runbook section based on live incident signals rather than requiring engineers to search manually. I built a working prototype using LangGraph and Claude, validated the decomposition logic against real incident patterns from the environment, and published the conceptual framework as an SSRN working paper to establish prior thinking.

**Result:** The prototype demonstrated that structured runbook decomposition could reduce the search-and-interpret phase of incident response by a significant margin. The paper documented the architecture and was indexed for the public research record. The concept is now being evaluated as a broader operational intelligence pattern.

---

## SOAR Format

**Situation:** Incident runbooks at PNC were static documents. During live incidents, engineers spent significant time just locating and reading the right section — time that compounded with every escalation.

**Obstacle:** There was no formal program to modernize runbooks, no dedicated tooling budget, and the problem did not surface cleanly in any team's backlog. It was a cost that everyone absorbed but no one tracked.

**Action:** I designed a five-stage agentic decomposition pipeline that converts prose runbooks into structured, queryable intelligence. I built a working prototype using LangGraph, published the architecture as an SSRN working paper, and assessed patentability against existing IBM and commercial prior art to understand where the novel claims sat.

**Result:** The prototype validated the core design. The working paper established the framework in the public record. The architecture was assessed as having patentable claims at the boundary resolution and dependency graph stages.

---

## STAR Format (Story 2 - Zero-Drift API Framework)

**Situation:** After a colleague's AI-assisted code silently broke multiple API contracts at PNC, I fixed the immediate breakages and then stepped back to look at the underlying structure. The real problem was that there was no single source of truth: Postman collections and REST Assured tests existed as independent artifacts that could drift apart without any system catching it.

**Task:** I needed to design a governance model that was not just a policy, but a structural change to how the test infrastructure was organized. The model had to work at every scale: one developer, fifty developers, and autonomous AI agents.

**Action:** I invented the Zero-Drift API framework, a four-layer architecture where the live Spring Boot OpenAPI spec becomes the canonical reference that all test surfaces are derived from. The framework simplified a sprawling multi-silo problem into a single governing principle: one spec, multiple consumers, zero manual synchronization. Each layer of the framework added governance without adding overhead: spec-anchored test generation, CODEOWNERS for contract changes, semantic grounding for AI tools, and `.ai-rules.json` constraint files for autonomous agents. I published the full framework as a four-part series.

**Result:** A structural problem that had existed across the industry for years now has a named, publishable framework. The simplification came from the insight that Postman and REST Assured were not two tools — they were two views of the same API, and treating them that way removed the entire class of silent drift failures.

---

## SOAR Format (Story 2 - Zero-Drift API Framework)

**Situation:** API contract drift was a known problem. Every team had some version of it. But there was no framework that addressed it coherently across the individual developer loop, large team governance, AI-assisted coding, and autonomous agent pipelines simultaneously.

**Obstacle:** Previous approaches treated Postman and REST Assured as separate concerns. Fixing one without the other just shifted where the drift lived. A complete solution required rethinking the architecture, not patching the workflow.

**Action:** I invented a single governing principle: the running application's live OpenAPI spec is the source of truth, and all test surfaces are consumers of it. I built the Zero-Drift framework around that principle and designed each of the four layers to enforce it at a different scale point in the delivery pipeline.

**Result:** The framework made a complex multi-surface governance problem into one solvable principle. Published as a public series, it now gives any Spring Boot team a concrete, buildable approach to eliminating API contract drift from the individual developer all the way to autonomous AI agents.

---

*Tags: #amazon-lp #invent-simplify #runbook-as-code #agentic-ai #langgraph #ssrn #pnc #zero-drift #api-governance #openapi*
