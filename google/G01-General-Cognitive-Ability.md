# General Cognitive Ability

> What Google is testing: how you break down a complex, ambiguous problem and think out loud — not whether you land on the "right" answer immediately. Interviewers watch the reasoning trail as much as the conclusion.

**Traits emphasized:** Inquisitive · Goal oriented · Honest (about risk)
**Adapted from:** [[amazon/LP04-Are-Right-A-Lot]]

---

## STAR Format — Eligibility-First RFP Architecture

**Situation:** While scoping an RFP response generation system for Sacramento-region school districts and nonprofits, the team's default architecture used a single general-purpose LLM call to evaluate eligibility and generate the response narrative in one pass, on the assumption that combining steps would reduce latency.

**Task:** As solution architect, I had to determine whether that assumption held — specifically, whether folding eligibility logic into the same pass as content generation would produce responses that read as complete but silently failed compliance.

**Action:** I broke the problem down before arguing a position. I traced what a single-pass model actually does when eligibility is just a prompt instruction rather than a hard gate: it optimizes for a fluent, complete-looking answer, not for a verified compliance check. I built a concrete failure-mode analysis using real district RFP criteria to test the hypothesis rather than debate it abstractly, then presented the reasoning chain — not just the recommendation — so the team could evaluate the logic itself. I proposed a five-agent pipeline where eligibility became a first-class computational gate with binary output, separated from generation.

**Result:** When tested against sample RFPs, the eligibility gate rejected three inputs that the single-pass design had let through with confident, well-written, but disqualifying responses. The architecture distinction — reasoning made visible and testable rather than asserted — became the core technical differentiator in the patent disclosure.

**Decision · Reasoning · Outcome**
- **Decision:** Separate eligibility assessment from generation as an independent, gated step.
- **Reasoning:** A single model call cannot be trusted to enforce a hard constraint it was only told about in a prompt; the risk had to be tested, not assumed away for the sake of latency.
- **Outcome:** Three false positives caught pre-launch that the faster design would have missed — the slower path was the correct one.

**Quantified impact** *(illustrative estimate — replace with real pilot numbers before using live)*
- 3 of ~40 sample RFPs tested (~7.5%) were caught as non-compliant that the single-pass design missed.
- Each caught case averts an estimated 15–20 hours of wasted proposal-writing effort per district, plus the compliance/reputational exposure of a disqualified submission.
- Across the ~50-organization pilot cohort, projected to prevent 2–3 disqualified submissions per grant cycle.

**Why this reads L6/L7:** The contribution wasn't the catch — it was the architectural pattern (verification as a separate, gated agent, not an embedded instruction) that shaped how every subsequent agent in the pipeline was designed. That's a pattern decision other engineers on the team reused, not a one-off fix.

## Sample interview questions this answers
- "Describe a complex problem you solved at work." (Problem Solving)
- "Tell me about a time you had to make a decision with incomplete information." (Problem Solving)
- "Walk me through how you'd approach [ambiguous system design problem]."

---

*Tags: #google #Googliness #general-cognitive-ability #problem-solving #rfp-agent #judgment*
