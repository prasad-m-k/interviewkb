# LP07 - Insist on the Highest Standards

> Leaders have relentlessly high standards — many people may think these standards are unreasonably high. Leaders are continually raising the bar and driving their teams to deliver high-quality products, services, and processes. Leaders ensure that defects do not get sent down the line and that problems are fixed so they stay fixed.

---

## STAR Format

**Situation:** While developing the Agent Security Policy (ASP) paper — a research contribution on trust boundaries in agentic AI systems — an early draft framed the problem primarily as a prompt injection defense issue. The framing was technically accurate but conceptually shallow. It would have passed review as a competent survey paper, but it did not advance the field.

**Task:** I was both the author and the quality bar. No external reviewer was going to push back hard enough before submission. If I shipped the shallow version, it would represent my name in the public record.

**Action:** I reframed the paper around a structural gap: that agentic AI systems lack a trust boundary layer analogous to what network security has in firewalls and what OS security has in privilege separation. I developed the ASP construct as a first-class architectural element, not a prompt engineering technique. I then rebuilt the argument from that foundation — rewriting the problem statement, the threat model, and the proposed framework. The target venue shifted to USENIX PEPR, which requires a more rigorous security systems argument than a general AI conference.

**Result:** The reframed paper is substantively stronger and defensible against a security systems audience. It introduces a construct that does not yet exist in the literature, which is the standard I was holding myself to. The submission timeline extended, but the quality of the contribution justified it.

---

## SOAR Format

**Situation:** The first draft of the ASP research paper solved a real problem but framed it at the surface level. It would have been publishable but forgettable — a contribution that would not be cited or built upon.

**Obstacle:** There was no external pressure to go deeper. The first draft was technically correct. Raising the bar meant adding weeks of rework with no guaranteed payoff.

**Action:** I identified that the paper's real contribution was structural, not tactical. I rebuilt the argument around the missing trust boundary layer concept, redeveloped the threat model from that foundation, and retargeted to a venue that would stress-test the security systems claim more rigorously.

**Result:** The paper now makes a claim that is falsifiable, novel, and structurally significant. The quality gap between draft one and the current version is the difference between a competent review paper and an original contribution.

---

## STAR Format (Story 2 - Zero-Drift API Framework)

**Situation:** A colleague on the team was using AI-assisted coding tools and broke several REST API contracts in the process. The build stayed green. Tests still passed. But downstream consumers started hitting failures that no alarm had flagged. The root cause was structural: Postman collections and REST Assured tests were maintained as two independent silos, so they drifted apart silently. Anyone could rewrite a failing assertion to match the new behavior, CI would go green, and the regression traveled to production dressed as a passing test.

**Task:** I fixed the immediate breakage. But fixing one incident was not enough. The same failure mode would repeat as long as the underlying governance structure was intact. I needed to address the root cause, not just the symptom.

**Action:** I designed the Zero-Drift API framework: a four-part governance model anchored on the principle that the running Spring Boot application's live OpenAPI spec must be the single source of truth for both Postman and REST Assured. Part 1 eliminated the silo at the developer loop by unifying both test surfaces against the live spec. Part 2 addressed large-team governance through API versioning and CODEOWNERS to prevent silent contract redefinition. Part 3 targeted AI-assisted development by using the OpenAPI spec as a semantic anchor that grounded LLM code generation, eliminating hallucinated payload keys. Part 4 enforced contract integrity against autonomous AI agents using pre-commit hooks, `.ai-rules.json` constraint files, and a Coder/Auditor multi-agent pattern. I published the complete framework as a four-part series on dev.to.

**Result:** The framework went from incident response to a published engineering standard. The series was indexed publicly and covers the full spectrum from solo developer to autonomous AI agent pipelines. The core principle throughout: a build should be green because the contract is intact, not because someone updated the assertion.

---

## SOAR Format (Story 2 - Zero-Drift API Framework)

**Situation:** Vibe-coded changes from an AI-assisted developer broke multiple API contracts at PNC. The builds stayed green because the tests had been silently updated to match the regression rather than catch it. Downstream teams absorbed the failures before anyone traced the source.

**Obstacle:** The surface fix was straightforward. The structural problem was not. Postman and REST Assured had always lived in separate silos, and that architecture made silent drift the default outcome. Solving it properly meant designing a governance model that worked at the individual developer level, at the large-team level, and specifically against AI-generated code — three distinct failure surfaces.

**Action:** I built the Zero-Drift framework to address all three layers: spec-anchored test unification for developers, contract governance for teams, OpenAPI as a semantic anchor for AI-assisted coding, and deterministic pipeline enforcement for autonomous agents. I then published the full four-part framework publicly on dev.to to make it available to any team facing the same structural problem.

**Result:** The broken APIs were fixed. More importantly, the governance model that lets them break silently was replaced with a deterministic contract integrity system. The published framework reached a broader engineering audience, establishing the Zero-Drift approach as a named, citable methodology.

---

*Tags: #amazon-lp #highest-standards #asp-paper #agentic-ai #research #usenix #security #zero-drift #api-contracts #vibecoding #pnc*
