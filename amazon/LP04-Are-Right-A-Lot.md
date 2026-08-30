# LP04 - Are Right, A Lot

> Leaders are right a lot. They have strong judgment and good instincts. They seek diverse perspectives and work to disconfirm their beliefs.

---

## STAR Format

**Situation:** During the development of an RFP response generation system for Sacramento-region school districts and nonprofits, the initial architecture discussion centered on using a single general-purpose LLM call to evaluate eligibility and generate content in one pass. The assumption was that combining the steps would reduce latency.

**Task:** I was the solution architect. My read was that combining eligibility assessment and content generation in a single pass would produce responses that looked complete but failed on compliance criteria — because the model had no hard gate forcing it to verify eligibility before generating narrative.

**Action:** I pushed back on the combined-pass design and proposed a five-agent pipeline where eligibility reasoning was a first-class computational gate, not an embedded prompt instruction. I documented the failure mode explicitly: without a dedicated eligibility agent, the system would generate well-written but non-compliant responses that a reviewer would catch too late. I presented the architecture using a concrete failure scenario from a real district RFP to make the risk tangible rather than theoretical.

**Result:** The team aligned on the eligibility-first design. When we tested against sample RFPs, the gate rejected three inputs that a single-pass model had passed through with confident-sounding but technically disqualifying responses. The architecture distinction became the core differentiator in the patent disclosure and the codeathon pitch.

---

## SOAR Format

**Situation:** An RFP response generation system was being scoped with a combined eligibility-and-generation step to minimize latency. The technical risk was that the model would generate polished responses for RFPs the applicant was not eligible for.

**Obstacle:** The counterargument from the team was that modern LLMs could handle nuanced eligibility logic inline if the prompt was well-structured. Disagreeing meant slowing down a design that already had momentum.

**Action:** I produced a failure-mode analysis using actual district RFP criteria. I showed that even a well-prompted single-pass model passed ineligible inputs at a rate that would create legal and reputational exposure. I proposed the eligibility gate as a separate agent with explicit rejection logic and binary output before any generation step ran.

**Result:** The gate design caught three false positives during testing that the single-pass model had missed. The eligibility-first architecture became the differentiating claim in both the patent disclosure and the competitive positioning for the codeathon.

---

*Tags: #amazon-lp #are-right-a-lot #solution-architecture #rfp-agent #gcp #vertex-ai #judgment*
