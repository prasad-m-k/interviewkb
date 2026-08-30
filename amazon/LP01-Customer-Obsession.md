# LP01 - Customer Obsession

> Leaders start with the customer and work backwards. They work vigorously to earn and maintain customer trust. Although leaders pay attention to competitors, they obsess over customers.

---

## STAR Format

**Situation:** At PNC, a downstream application team was repeatedly hitting a paginated alert endpoint that returned inconsistent results. The root cause was that our REST API lacked formal pagination support — it either returned everything or nothing, depending on undocumented caller behavior. Business teams were making decisions based on stale or incomplete alert data.

**Task:** I was responsible for the alert/announcement endpoints in the Spring Boot service. I needed to redesign the pagination layer without breaking existing integrations that expected the old flat response structure.

**Action:** I introduced nullable `Integer` parameters (`page`, `size`) on the endpoint so callers could opt into pagination without any forced migration. I built a `PagedResponse` wrapper class that preserved backward compatibility for un-paginated callers. I also wrote a JPQL correlated subquery to accurately compute active alert counts at query time, eliminating the stale-count bug that had been affecting dashboard consumers for months. I coordinated with two downstream teams to validate the new contract before deployment.

**Result:** The paginated endpoint went live with zero downstream incidents. Active alert counts matched real-time state, and the dashboard teams confirmed data accuracy improved noticeably. The design became the reference pattern for two other endpoints in the same service.

---

## SOAR Format

**Situation:** Business users at PNC were making risk decisions based on alert data from an API that silently dropped records when under load. The pagination design was missing entirely, and callers had no way to request subsets of data reliably.

**Obstacle:** Any redesign risked breaking existing integrations. The callers ranged from internal dashboards to downstream batch jobs, and there was no formal API contract documented anywhere.

**Action:** I designed a backward-compatible pagination model using nullable query parameters, built a typed `PagedResponse` wrapper, and introduced a correlated JPQL subquery to resolve the count accuracy issue at the database layer. I ran joint validation sessions with affected teams before the release.

**Result:** The API now handles high-volume pagination without data loss. Alert accuracy for dashboard consumers improved to near real-time. The pattern was adopted across the service, and the downstream teams formally removed their workaround polling logic.

---

*Tags: #amazon-lp #customer-obsession #spring-boot #pnc #api-design*
