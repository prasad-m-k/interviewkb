# LP02 - Ownership

> Leaders are owners. They think long term and don't sacrifice long-term value for short-term results. They act on behalf of the entire company, beyond just their own team. They never say "that's not my job."

---

## STAR Format

**Situation:** During a platform transition at OpenText (formerly Micro Focus), an internal tooling system I had built during the HP years was still being used by support teams across three regions. When the migration team announced they were deprecating the environment, no one had formally assessed the downstream impact on those teams.

**Task:** I was not formally assigned to the migration project. My role had shifted to architecture. But I had built the original tooling and knew exactly where the hidden dependencies lived.

**Action:** I raised the issue directly with the migration lead and offered to do the dependency audit myself. Over two weeks, outside my primary deliverables, I catalogued every integration point, documented the data flows, and produced a migration impact report. I then worked with the affected support teams to prioritize what needed to be rebuilt versus what could be retired. I also drafted the replacement runbook structure so the new team would not be starting from scratch.

**Result:** Three critical support workflows that would have gone dark during the cutover were identified and preserved. The migration lead credited the audit with preventing what would have been a multi-region support outage. I had no formal stake in the outcome — I just knew the system and the cost of getting it wrong.

---

## SOAR Format

**Situation:** A legacy tooling environment I originally built at HP was still active inside OpenText after the acquisition. A migration project was in flight, but no one had traced the operational dependencies those tools still carried.

**Obstacle:** I was not on the migration team, had no formal mandate to intervene, and my current responsibilities were already full. Raising the issue meant taking on additional work with no visibility credit.

**Action:** I initiated the dependency audit myself, documented all downstream integration points, and produced a structured impact report. I coordinated directly with three regional support teams to triage what had to survive the migration and what could be safely sunset.

**Result:** Three workflows that would have failed silently during cutover were rescued. The migration completed on schedule, with no support disruption. The runbook documentation I produced was carried forward into the new environment.

---

## STAR Format (Story 2 - Zero-Drift API Framework)

**Situation:** A colleague introduced breaking changes to several REST API contracts at PNC while using AI coding tools. The failures surfaced in downstream systems, not in the CI pipeline. The tests had stayed green because assertions were quietly updated to match the new behavior rather than guard the original contract. The breakage was not my code, not my change, not my area.

**Task:** The downstream teams were blocked. Someone had to trace the root cause, fix the broken contracts, and prevent the same failure class from recurring. I had the context to do it and stepped in without being assigned.

**Action:** I traced the drift to the structural separation of Postman and REST Assured as independent test surfaces. I fixed the immediate contract violations and then designed the Zero-Drift framework to address the root cause: a four-part governance model anchored on the live OpenAPI spec as the single source of truth. I published the full framework publicly so the fix extended beyond our team and could benefit any engineering team facing the same problem.

**Result:** The broken contracts were restored. The governance model was documented and published. The downstream teams resumed normal operation. By treating the incident as an ownership problem rather than a blame problem, the fix produced a durable structural change instead of a one-time patch.

---

## SOAR Format (Story 2 - Zero-Drift API Framework)

**Situation:** Vibe-coded API changes from a colleague broke contracts that downstream teams depended on. The CI pipeline did not catch it. The person who broke it had moved on. The downstream teams were left holding the failure.

**Obstacle:** It was easy to walk past this. It was not my code, not my sprint, and diagnosing silent drift across two separate test silos required time I had to take from other work.

**Action:** I owned the problem end to end: diagnosed the structural cause, fixed the broken contracts, designed the Zero-Drift governance framework to prevent recurrence, and published it so other teams would not have to rediscover the same lesson.

**Result:** The immediate breakage was repaired. The structural fix was documented and made public. The Zero-Drift framework is now available to any team dealing with API contract drift — a broader outcome than a simple incident fix.

---

*Tags: #amazon-lp #ownership #opentext #micro-focus #migration #tooling #zero-drift #api-contracts #pnc #vibecoding*
