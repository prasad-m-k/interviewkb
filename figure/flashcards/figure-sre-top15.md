# Figure AI — Staff SRE Top 15 Flashcards

**Company:** [[figure/index]]
**Format:** Front (Q) / Back (A)

---

## Company and Role Context

**Q1. What makes SRE at Figure AI fundamentally different from SRE at a FAANG web company?**
A: Three things distinguish it. First, "production" is a physical environment — a manufacturing floor where a bad deploy can harm humans or halt automotive production, not just serve 500 errors. Second, the software stack has 5 distinct layers (OS, middleware, control software, ML models, firmware) each with different release cadences, test requirements, and rollback complexity. Third, at ~400 people with no established SRE team, Staff SRE means you own the roadmap, write the runbooks, and make architectural decisions that will be load-bearing for years — not executing within a mature system.

---

**Q2. The JD says "Transition SaaS solutions to self-hosted alternatives for enhanced security." What is the driving concern and how do you prioritize which tools to migrate first?**
A: The concern is IP exfiltration risk: robot control software, ML model weights, and manufacturing processes on a third-party cloud create breach surface Figure cannot control. Prioritize by sensitivity × dependency: (1) SCM (all source code lives here, plus CI/CD pipelines run against it), (2) container registry (images deployed to robots), (3) secrets management (credentials used by all automated systems), (4) issue tracker and documentation (important but lower blast radius if breached). Communication tools (Slack, Zoom) carry low-sensitivity payload and are lowest priority.

---

## Infrastructure and Linux

**Q3. You're taking over a hybrid cloud + on-prem environment with minimal documentation. What is your first week's priority?**
A: Inventory before changing anything. Map every system: (1) run `nmap` sweeps and review cloud provider resource lists to identify all running instances, (2) export all DNS zones and review what resolves to what, (3) pull Terraform state (if it exists) and identify what's drift vs. what's documented, (4) interview each team that owns a critical system about their failure modes and change procedures, (5) document what you find in a shared wiki immediately. Nothing breaks faster than an inherited environment you misunderstood. The second priority is finding the "bus factor 1" systems — things only one person knows about.

---

**Q4. A manufacturing supplier portal is down and the procurement team can't submit purchase orders. Walk me through your incident response.**
A: Start by declaring an incident with a severity level (P1 if production is halted, P2 if degraded). Open a dedicated communication channel and assign an incident commander if available. Gather symptoms without assuming cause: is the portal returning errors or timing out? What changed in the last 24 hours (deployments, certs, firewall rules)? Check the obvious first — SSL cert expiry, DNS resolution, upstream dependency health. Simultaneously notify the procurement team of status and ETA. For a supplier portal specifically, check authentication layer (SSO), database connectivity, and whether the on-prem network path to cloud is intact. Document timeline in real time; you will need this for the postmortem. Resolve, then write a blameless postmortem with action items and owners before the week ends.

---

**Q5. What is the USE method and how would you apply it to debug a slow robot telemetry pipeline?**
A: USE: Utilization, Saturation, Errors — applied per resource (CPU, memory, disk, network, etc.). For a slow telemetry pipeline: (1) Utilization — is the aggregation server CPU or memory near ceiling? Is the WAN link to cloud near capacity? (2) Saturation — is there a queue building up in the message broker? Are Prometheus scrape targets timing out? (3) Errors — are there dropped packets on the factory WiFi? Log ingestion errors in Loki? Database write failures in VictoriaMetrics? Trace the data path: robot → factory edge collector → WAN → cloud ingest → storage, and apply USE at each hop. The bottleneck is usually the first resource that shows high saturation with errors.

---

## CI/CD and Deployment

**Q6. Why can't you use a standard blue-green deployment for robot software updates?**
A: Blue-green deployment assumes you can instantaneously switch all traffic from the old version to the new version and back. Robots cannot do this: (1) a robot mid-task cannot be "switched" without interrupting the task, potentially damaging a workpiece or causing a safety event, (2) some components (firmware) cannot be hot-swapped at all and require the robot to be idle, (3) there is no load balancer equivalent for physical robots — you must coordinate deployment per robot based on its operational state, (4) rollback for firmware may require physical intervention. The correct pattern is progressive fleet rollout with shift-aware scheduling: deploy during maintenance windows, to robots that are idle, in batches, with explicit per-robot sign-off at each stage.

---

**Q7. Why must ML model updates be decoupled from control software releases in a robotics CI/CD pipeline?**
A: ML models (neural network weights) and control software (compiled binaries) have different development cadences, different testing requirements, and different rollback characteristics. A model retraining cycle may happen daily; a control software release may take weeks of HIL testing. Coupling them means a new model always requires a full software release cycle, dramatically slowing ML iteration. Decoupling allows: (1) model weights to be versioned and deployed independently through a separate artifact store and rollout policy, (2) the control software to declare a "model interface" (input/output specification) and accept any model that satisfies it, (3) targeted rollback — if a model causes degraded grasp performance, roll back the model without touching the control software. The tradeoff is maintaining compatibility contracts between model versions and software versions.

---

**Q8. How do you safely deploy an OS patch to 1,000 robots in a BMW factory without halting production?**
A: Shift-aware staged rollout. Step 1: coordinate with manufacturing ops to identify the shift schedule and maintenance windows (typically 30-minute gaps between shifts or planned weekend downtime). Step 2: during a window, select a batch of robots that are docked (not mid-task) — 2-5% of fleet. Step 3: deploy patch, run automated health checks post-reboot (can the robot stand, report sensor data, complete a test cycle?). Step 4: if healthy, hold for 24-48 hours of normal operation before expanding. Step 5: progressively expand to 10%, 25%, 50%, 100%. Emergency stop: if any safety metric breaches at any batch size, stop rollout, do not expand further. For critical CVEs, compress timelines but never skip the HIL validation gate.

---

## Monitoring and SLOs

**Q9. What SLOs would you define for a supplier portal at a robotics manufacturing company?**
A: Start by asking: what business process depends on this portal and what is the cost of downtime? For a procurement portal used 8am–6pm by 50 vendors: (1) Availability SLO: 99.5% during business hours (allows ~18h downtime/year — manufacturing has some tolerance for planned maintenance unlike 24/7 consumer apps), (2) Latency SLO: P95 < 3 seconds for form submissions (vendors are external, bandwidth varies), (3) Error rate SLO: < 0.5% of form submissions result in data loss or 5xx errors. Error budget: 0.5% of hours in a month = ~3.5 hours. Burn through it twice in a month = reliability investigation before adding features. Also define incident severity: outage during active procurement cycle (P1) vs. during off-hours (P3).

---

**Q10. A robot in the BMW factory falls and stops operating. You get paged at 2am. What is your first action, and what data do you immediately look for?**
A: My first action is to confirm the safety status — has the robot been powered down and isolated? Safety takes precedence over root cause investigation. Assuming yes: open the observability stack and pull the robot's Tier 1 metrics and software logs from 5 minutes before the event. Look for: (1) any safety alert that fired before the fall (joint overtemperature, abnormal force reading, software watchdog trigger?), (2) the last software version deployed to that robot and when, (3) any anomaly in joint position or velocity data that preceded the fall (Tier 2 telemetry if available), (4) whether other robots showed similar anomalies or if this is isolated. If a software deploy correlates with the event, trigger fleet-wide rollback for that robot's software version immediately while continuing investigation. Document every action with timestamps — BMW's safety team and Figure's legal team will need this record.

---

## Self-Hosting and Security

**Q11. You migrated from GitHub.com to self-hosted GitLab. Three weeks later, a developer reports they accidentally pushed to the old GitHub remote. What do you do?**
A: This is a data exposure event — sensitive code may have been pushed to an external SaaS after migration intent was to keep it internal. Respond immediately: (1) confirm what was pushed (which repo, which branch, what code) — code containing credentials or robot control algorithms is highest severity, (2) revoke any credentials present in the pushed code immediately, (3) delete the commit from GitHub.com (GitHub supports force-push or repo deletion to remove; if it was already cached externally, that is harder), (4) add a pre-push hook to the old GitHub remote that rejects pushes and redirects to GitLab, (5) investigate why the developer still had the old remote configured — this reveals a gap in the migration runbook or org-wide tooling. Add a post-migration step that audits all developer machines for old remote configurations.

---

**Q12. How would you implement zero-trust access for Figure's hybrid infrastructure (cloud + factory on-prem)?**
A: Zero-trust means: never trust the network, always verify identity, always enforce least privilege. Implementation: (1) Replace VPN with identity-aware proxy (Cloudflare Access, BeyondCorp-style, or Pomerium) — engineers authenticate with SSO before accessing internal tools, regardless of network location, (2) All machine-to-machine communication uses mutual TLS (mTLS) with short-lived certificates issued by an internal PKI (HashiCorp Vault PKI engine), (3) Service accounts use Vault's dynamic credentials — no long-lived static passwords, (4) Factory network is segmented: robots on one VLAN, manufacturing IT on another, engineering workstations on another; firewall rules are explicit and documented in Terraform (no manual firewall rules), (5) All access is logged and reviewed via SIEM. The practical first step: get SSO coverage on every internal tool before any other zero-trust work — it's the identity foundation everything else builds on.

---

## Startup SRE Mindset

**Q13. You're the only SRE at Figure and you receive three urgent requests simultaneously: (a) SCM is slow for all engineers, (b) a manufacturing supplier portal cert expired, (c) a new robotics team needs CI/CD setup for a new project. How do you triage?**
A: (b) > (a) > (c). The expired cert is a P1 — the supplier portal is down entirely (HTTPS handshake fails with cert error), and procurement may be halted; this has a financial and operational impact on production. The SCM slowness is P2 — engineers are unblocked but degraded; active development continues, just slowly. CI/CD setup for a new project is P0 on nobody's urgency scale — it is planned work and can wait. For (b): rotate the cert immediately (Let's Encrypt automate, or push the cert from Vault PKI); total fix time should be under 15 minutes for a practiced SRE. Then address (a): check what changed (recent deployment? traffic spike? disk full?). For (c): schedule a meeting for tomorrow. Separately, identify why a cert expired without alert — the remediation for (b) includes adding cert expiry monitoring.

---

**Q14. A VP of Engineering asks you to skip the HIL testing gate for a robot software update because a customer demo is in 48 hours and the new feature isn't deployed yet. How do you respond?**
A: I decline to skip the gate and offer an alternative. My response: "Skipping HIL testing means deploying software to production robots that has only been validated in simulation. The risk is a robot behaving unexpectedly during or after the demo — which is far more damaging to the customer relationship than the feature not being available. Here is what I can do: (1) expedite the HIL test slot — I will reschedule other jobs and run the test suite tonight, with a human QA present, (2) if the feature can be limited to a subset of robots used in the demo, we can prioritize those for expedited validation, (3) if there is a configuration flag to enable/disable the feature, we can deploy the build with the feature off, complete HIL testing, and enable the feature flag only after validation passes." The goal is to find a path to the business outcome without compromising the safety gate. I would also document this conversation — a VP pressuring to bypass safety gates is a cultural signal that should be visible to leadership.

---

**Q15. How do you introduce SLO discipline to a robotics engineering team that has never used SLOs?**
A: Start with one system, not all systems. Choose a system that recently had a high-visibility outage — the pain is fresh and there is motivation to do better. Steps: (1) Define SLIs together with the owning team — not imposed by SRE. "What makes this system healthy from a user's perspective?" usually gives 2-3 measurable signals, (2) Look at 30 days of historical data and propose an SLO slightly below the current best-effort reliability — this ensures the first SLO is achievable and creates positive momentum, (3) Create a dashboard showing SLO compliance and error budget; make it visible in the team's engineering review, (4) Run a monthly error budget review — is the budget being spent on user-impacting incidents or on planned maintenance? (5) After 90 days, the team will have experienced the value (catching degradation early, prioritizing reliability work) and will want to expand coverage. Avoid starting with a strict SLO and an error budget policy that blocks feature work — the team needs to trust the system before it governs their velocity.

---

## Sources
- [[figure/company-context]]
- [[figure/staff-sre-role]]
- [[figure/scenarios/robot-fleet-cicd]]
- [[figure/scenarios/robot-telemetry]]
- [[figure/scenarios/saas-to-self-hosted]]
- [[sre/concepts/slo-sli-sla]]
