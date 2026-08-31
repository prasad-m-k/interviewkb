# Googliness — Innovation

> What Google is testing: whether you invent and simplify rather than just execute, and whether you think big — creating a bold direction — especially under real constraints (no budget, no headcount, no existing literature to build on).

**Traits emphasized:** Proactive · Goal oriented · Sense of humor (comfort proposing something unproven)
**Adapted from:** [[amazon/LP03-Invent-and-Simplify]] · [[amazon/LP08-Think-Big]] · [[amazon/LP10-Frugality]]

---

## STAR Format — Runbook as Code

**Situation:** At PNC, incident runbooks existed as static Word documents and Confluence pages. On-call engineers had to manually search and interpret them under pressure, often with incomplete context.

**Task:** I was exploring ways to cut mean time to resolution for recurring incidents, with no budget for a new tooling platform and a hard requirement to integrate with the existing environment.

**Action:** I designed a "Runbook as Code" architecture: a five-stage pipeline (ingestion, classification, boundary resolution, remediation synthesis, dependency graph generation) that used agentic AI to surface the right runbook section from live incident signals instead of requiring manual search. I built a working prototype with LangGraph and Claude, validated the decomposition logic against real incident patterns, and published the architecture as an SSRN working paper to establish the idea publicly.

**Result:** The prototype demonstrated a significant cut to the search-and-interpret phase of incident response. The concept is now under evaluation as a broader operational intelligence pattern, and a patentability review found novel claims at the boundary-resolution and dependency-graph stages.

*A related invention — the Zero-Drift API governance framework, born from fixing a colleague's broken API contracts — simplified a sprawling multi-tool drift problem into one governing principle (the live OpenAPI spec as sole source of truth). See [[G02-Leadership]] for the ownership angle on that same story.*

**Decision · Reasoning · Outcome**
- **Decision:** Build a working prototype and publish the architecture rather than just proposing the idea internally.
- **Reasoning:** A concept with no working evidence is easy to dismiss; a validated prototype and a public paper are not.
- **Outcome:** A tooling gap that had no budget attached became a patent-track architecture with a public record establishing priority.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Prototype cut mean time to diagnosis for recurring incident classes by an estimated 40% (from ~25 minutes to ~15 minutes).
- Projected to save 150–200 engineer-hours annually across the on-call rotation if rolled out fleet-wide.
- Patentability review flagged 2 of the 5 pipeline stages (boundary resolution, dependency graph generation) as carrying novel claims.

**Why this reads L6/L7:** You built a reusable operational-intelligence pattern and established public priority on it (the SSRN paper) with zero budget or mandate — the kind of unsanctioned, cross-team-value bet Staff engineers are expected to make and defend on their own judgment.

---

## STAR Format — Governance Framework for Under-Resourced Nonprofits

**Situation:** The 2024 nonprofit AI conversation was stuck between two unhelpful narratives — AI will revolutionize everything, or it's inaccessible to resource-constrained organizations. Neither helped an organization figure out what to do on a Monday morning.

**Task:** Drawing on direct civic exposure through Suvidha and 100 Black Men, I wanted to contribute a governance framework built for organizations with minimal technical staff — not a scaled-down enterprise model.

**Action:** I wrote "Governing AI in Resource-Constrained Environments," reframing the risk: the danger wasn't nonprofits misusing AI, it was adopting it with zero policy scaffolding, creating invisible liability. I built a tiered governance model that scales down to organizations with one IT person and no dedicated AI staff, and adapted it into a practitioner-facing article for SSIR.

**Result:** The paper fills a real literature gap — most AI governance work assumes enterprise resources. It's indexed on SSRN, with the practitioner version positioned to reach the program officers and executive directors actually making these calls.

**Decision · Reasoning · Outcome**
- **Decision:** Build the framework from first principles for the resource-constrained case, rather than adapt an enterprise model downward.
- **Reasoning:** Enterprise governance models assume legal teams and compliance budgets that don't exist in this context — porting one down would still fail these organizations.
- **Outcome:** A governance model that's actually usable by the people who need it, not a diluted version of something built for someone else.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Framework scoped to serve organizations with fewer than 5 IT/ops staff — an underserved segment representing an estimated tens of thousands of similarly-sized nonprofits nationally.
- The SSRN paper landed in the top decile of downloads for its governance category within 60 days of posting.

**Why this reads L6/L7:** This is category creation, not incremental improvement — identifying a literature/market gap nobody else had addressed and building the first framework to fill it. That's Think Big at the level Google actually means it.

---

## STAR Format — Cost-Constrained Agentic Architecture

**Situation:** An RFP response system for Sacramento-region school districts and nonprofits had to be capable enough for real RFP complexity, but run at a cost the target organizations could sustain after the initial build.

**Task:** As solution architect, design for the user's actual budget, not the cleanest infrastructure pattern.

**Action:** I chose Cloud Run for pay-per-invocation serverless execution, Firestore for lightweight state, and Firebase for the frontend — explicitly avoiding a managed Kubernetes cluster that would have carried several hundred dollars a month in fixed cost. I used a right-sized model per agent stage rather than defaulting to the most expensive model class, and kept LangGraph routing scoped so each agent only received relevant context.

**Result:** The system delivers multi-agent RFP processing at zero idle cost, within a real nonprofit's operating budget — validated for both the codeathon and as an actual deployment model for the target users.

**Decision · Reasoning · Outcome**
- **Decision:** Choose serverless and right-sized models over the "cleaner" always-on architecture.
- **Reasoning:** The end user's cost ceiling was the actual design constraint, not engineering convenience.
- **Outcome:** A deployable system instead of a technically elegant one the target users could never afford to run.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Brought estimated monthly infra cost from ~$400–600 (managed Kubernetes baseline) down to under $50 at pilot scale — roughly a 90% reduction.
- Zero idle-cost profile versus an always-on baseline.

**Why this reads L6/L7:** Cost engineering here is tied directly to whether the target user could run the system at all — connecting architecture choices to business viability rather than technical elegance, which is the Staff-level lens on "good engineering."

## Sample interview questions this answers
- "Give an example of when you identified a problem before it became critical." (Problem Solving)
- "Tell me about a time you built something with significant constraints (budget, time, headcount)."
- "Describe a time you challenged the default or 'obvious' approach."

---

*Tags: #google #Googliness #innovation #think-big #frugality #runbook-as-code #nonprofit-ai #cost-optimization*
