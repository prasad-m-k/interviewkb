# LP10 - Frugality

> Accomplish more with less. Constraints breed resourcefulness, self-sufficiency, and invention. There are no extra points for growing headcount, budget, or fixed expense.

---

## STAR Format

**Situation:** The RFP response generation system for Sacramento-region school districts and nonprofits was built on GCP, but the target users were organizations with minimal IT budgets. The architecture had to work at a cost point those organizations could sustain — not just in compute, but in maintenance overhead.

**Task:** As the solution architect, I needed to design a system that was capable enough to handle real RFP complexity while running at a cost that a small nonprofit could afford after the initial build.

**Action:** I designed the pipeline using Cloud Run for serverless execution (pay-per-invocation, no idle cost), Firestore for lightweight state persistence, and Firebase for the frontend — avoiding the overhead of a managed Kubernetes cluster that would have cost several hundred dollars per month minimum. I used Claude Sonnet via Vertex AI rather than defaulting to the most expensive model class, tuned for the specific reasoning tasks in each agent stage. The LangGraph orchestration kept token usage targeted by routing only the relevant context to each agent rather than passing the full document to every stage.

**Result:** The architecture delivered multi-agent RFP processing at a cost profile well within a nonprofit's operating budget. The serverless design meant zero idle cost when the system was not actively processing. The design was validated as suitable for the codeathon and as a real deployment model for the target users.

---

## SOAR Format

**Situation:** Building an agentic AI system capable of handling complex procurement documents sounds inherently expensive. The real users of this system, small school districts and nonprofits, had no budget for infrastructure that cost more than a modest SaaS subscription.

**Obstacle:** Modern agentic architectures default toward over-engineered infrastructure. The path of least resistance was a full Kubernetes deployment with always-on compute — technically cleaner to operate but financially inaccessible to the target users.

**Action:** I made every infrastructure choice through the lens of the end user's cost constraint. Serverless execution, lightweight state persistence, targeted context routing, and right-sized model selection all came from the same principle: do not spend what the user cannot afford to sustain.

**Result:** The system processes complex RFP eligibility and response generation at a cost profile that maps to the target users' actual budgets. The frugality in the architecture is not a limitation — it is a feature that makes the system deployable in the context it was designed for.

---

*Tags: #amazon-lp #frugality #gcp #cloud-run #vertex-ai #nonprofit #rfp-agent #cost-optimization*
