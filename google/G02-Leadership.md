# Leadership

> What Google is testing: taking initiative and showing ownership when things go wrong — leading without a title, developing others, and delivering on commitments under pressure. Google's leadership bar is about emergent leadership: stepping up because the situation needs it, not because a role requires it.

**Traits emphasized:** Proactive · Goal oriented · Empathetic
**Adapted from:** [[amazon/LP02-Ownership]] · [[amazon/LP06-Hire-and-Develop-the-Best]] · [[amazon/LP12-Deliver-Results]]

---

## STAR Format — Ownership of a Breakage That Wasn't Mine

**Situation:** A colleague introduced breaking changes to several REST API contracts at PNC while using AI coding tools. The failures surfaced downstream, not in CI, because the assertions had been quietly updated to match the new (broken) behavior instead of guarding the original contract. The breakage was not my code, my change, or my area of the codebase.

**Task:** Downstream teams were blocked. Someone had to trace the root cause, fix the contracts, and prevent the failure class from recurring — I had the context to do it and no one had been assigned.

**Action:** I traced the drift to a structural gap: Postman and REST Assured existed as independent, unsynchronized test surfaces. I fixed the immediate contract violations, then designed the Zero-Drift framework — a four-part governance model anchored on the live OpenAPI spec as the single source of truth — and published it publicly so the fix would outlast the incident.

**Result:** The broken contracts were restored, downstream teams resumed normal operation, and the governance model is now a citable framework any Spring Boot team can adopt.

**Decision · Reasoning · Outcome**
- **Decision:** Step in and own a problem outside my formal scope.
- **Reasoning:** Treating it as a blame problem would have fixed the symptom; someone needed to treat it as a structural ownership problem to fix the cause.
- **Outcome:** A one-time incident became a durable, published governance model instead of a repeat failure.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- 5 contract violations fixed immediately; the framework was subsequently adopted across ~40 endpoints in the service.
- Contract-drift incidents dropped from an estimated 2/month to near zero within one quarter of adoption (~90%+ reduction).
- Each prevented incident saves an estimated 10–15 engineer-hours of downstream triage that previously had no owner.

**Why this reads L6/L7:** The move from "I fixed a bug" to "I designed and published a governance framework three teams adopted" is the exact Staff-level jump — from individual fix to reusable system that changes how other teams operate, without you having authority over them.

---

## STAR Format — Developing Two Junior Engineers

**Situation:** At HPE, an influx of engineers with strong support instincts but no exposure to system-level architecture thinking were solving tickets without diagnosing root-cause patterns across products.

**Task:** As a senior engineer, I was informally asked to help two of them build the architectural reasoning layer they were missing.

**Action:** I designed a six-week structured pairing program, narrating my diagnostic reasoning out loud on real cases rather than just handing over the fix. We built a shared "why behind the fix" document capturing the reasoning behind each case.

**Result:** Both engineers became independent within two quarters; one became a technical lead. The shared document outlived the pairing and became a team-wide knowledge base.

**Decision · Reasoning · Outcome**
- **Decision:** Invest six weeks of unrecognized personal overhead in structured mentorship.
- **Reasoning:** Fixing their tickets for them would have solved this week's queue; teaching the reasoning process would compound.
- **Outcome:** Reduced escalation volume to senior staff for two years after the pairing ended.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Escalations from the two mentees to senior engineers dropped an estimated 60% within two quarters.
- Average time-to-root-cause fell from roughly 4 hours to 1.5 hours per case.
- One mentee was promoted to technical lead within 12–18 months; the shared reasoning document was still in active use by the wider team years later.

**Why this reads L6/L7:** The impact is measured in someone else's throughput, not yours — a multiplier signal. It's also durable: the artifact (the shared "why behind the fix" doc) kept generating value long after you stopped actively coaching.

---

## STAR Format — Delivering an EB1A Petition Solo

**Situation:** Building an EB1A extraordinary ability petition as an independent applicant — no immigration counsel, no university affiliation, while working full-time as a Solution Architect — required sustained execution across multiple evidence categories over roughly twelve months.

**Task:** The judging and peer-review criterion had to be built from zero: no existing academic profile, no shortcuts.

**Action:** I executed across every workstream in parallel — four SSRN working papers, conference speaking submissions, hackathon judging engagements, civic contribution documentation, and careful IP separation from employer work — treating the petition like a delivery program with a fixed evidentiary bar rather than a series of one-off tasks.

**Result:** Within twelve months: four SSRN-indexed papers, one SSIR-targeted article, an agentic system at patent-disclosure stage, a published book, and an active public writing record — a defensible portfolio built without institutional backing.

**Decision · Reasoning · Outcome**
- **Decision:** Run every workstream concurrently instead of sequentially.
- **Reasoning:** No single workstream alone would clear the evidentiary bar, and sequential execution wouldn't fit inside a year of nights and weekends around a full-time job.
- **Outcome:** A multi-criteria portfolio delivered on schedule, documented to evidentiary standard from day one.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- 4 SSRN papers, 1 SSIR-targeted article, 1 patent-disclosure-stage system, and 1 published book delivered in ~12 months solo, versus a typical 18–24 month timeline for a comparable petition with institutional/counsel support.

**Why this reads L6/L7:** This is cross-workstream program management against a fixed, externally-defined bar, executed without a team, a budget, or the ability to delegate — the same operating mode a Staff engineer needs when driving a cross-org initiative with no direct reports.

## Sample interview questions this answers
- "Tell me about a time you led a team through a difficult project." (Leadership & Teamwork)
- "Describe a situation where you had to motivate a struggling team member." (Leadership & Teamwork)
- "Give an example of when you delegated effectively." (Leadership & Teamwork)
- "Tell me about a time you took ownership of something that wasn't your responsibility."

---

*Tags: #google #Googliness #leadership #ownership #mentorship #execution #zero-drift #eb1a*
