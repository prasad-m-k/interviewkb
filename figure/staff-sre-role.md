# Figure AI — Staff SRE Role Analysis

**JD Source:** Job ID 4614747006 (Greenhouse)
**Related:** [[figure/index]], [[figure/company-context]]

---

## The Role in Plain Language

The JD is titled "Staff Site Reliability Engineer" but reads more like "Head of Infrastructure." The responsibilities span cloud, on-prem, CI/CD, monitoring, security, manufacturing systems, and vendor management simultaneously. At this stage of the company, this person is not joining a mature SRE team — they are a foundational hire expected to define what reliable infrastructure looks like for a physical AI company.

---

## Job Posting — Decoded

### "Serve as primary infrastructure owner for mission-critical systems including SCM, CI/CD, software distribution, supplier portals, and manufacturing"

**What they mean:** You own *all* of this. There is no team. SCM = the source control system (likely GitHub today; self-hosted tomorrow). Software distribution = how robot software gets to robot fleets. Manufacturing = ERP/MES systems on the production floor. Supplier portals = vendor-facing web applications.

**What to demonstrate:** Examples where you owned multiple infrastructure domains simultaneously, made architectural decisions, and dealt with the consequences of those decisions in production.

---

### "Transition SaaS solutions to self-hosted alternatives for enhanced security"

**What they mean:** They currently use SaaS tools (GitHub.com, Jira Cloud, Confluence Cloud, possibly Datadog, possibly others) and want to bring them inside their security perimeter. This is driven by IP protection — their robot software and manufacturing data should not live in a third-party cloud.

**What to demonstrate:** Experience migrating between SCM platforms (even one migration — GitHub to GitLab, or Bitbucket to GitHub), experience running self-hosted open-source tools in production (GitLab, Harbor, Vault, Grafana, Prometheus), understanding of the security motivations (blast radius of a SaaS breach, data residency).

**Likely first project in this role:** Evaluating and migrating to self-hosted SCM. GitLab is the most probable candidate. See [[figure/scenarios/saas-to-self-hosted]].

---

### "Design and maintain monitoring, alerting systems, and incident response procedures"

**What they mean:** They have minimal formal monitoring today. You design it from scratch, calibrated for both cloud services and physical robots.

**What to demonstrate:** You have designed a monitoring architecture before (not just added dashboards to an existing Datadog org). You understand the difference between infrastructure metrics (USE model: utilization, saturation, errors) and service metrics (RED model: rate, errors, duration). You can write SLOs — see [[sre/concepts/slo-sli-sla]].

**Figure-specific angle:** Monitoring for robots is not just server CPU/memory. You also need to think about: robot health metrics (joint temperatures, battery state, error codes), fleet-level aggregations (what % of the fleet is operational right now?), and time-series at high frequency (robot sensors produce data at 50–1000 Hz per sensor).

---

### "Automate deployment and scaling to reduce manual workload"

**What they mean:** Toil reduction. Right now, some deployments are manual. They want an SRE who will automate these away.

**What to demonstrate:** Specific examples of automation you wrote that eliminated manual operations. Quantify: "reduced deployment time from 2 hours to 15 minutes," "eliminated 4 hours/week of manual validation steps." See [[devops/concepts/deployment-strategies]].

**Figure-specific nuance:** Automated scaling for robot software is unusual. Robots don't scale horizontally like web services. "Scaling" here means: provisioning new robots into the fleet, managing configuration at fleet scale, and scaling the cloud infrastructure that supports robot training and telemetry.

---

### "Partner with stakeholders to define infrastructure needs and Service Level Objectives"

**What they mean:** At a startup, SLOs don't exist yet. You will have to convince robotics engineers, manufacturing ops, and leadership to adopt SLO discipline. This is a political challenge as much as a technical one.

**What to demonstrate:** A story about introducing SLOs to a team that had never used them. How did you get buy-in? How did you set the initial targets? What happened when you breached them?

---

### "Apply data-driven approaches to demonstrate service robustness"

**What they mean:** Reports, dashboards, and error budget reviews. They want someone who can take a quarterly infrastructure review to leadership and show reliability trends, not just talk about incidents.

**What to demonstrate:** You have built reliability dashboards or produced reliability reports at a previous company. You can interpret error budget burn rates and translate them into capacity/staffing arguments.

---

### "Coordinate with security teams on timely updates and remediations"

**What they mean:** Patch management and vulnerability response. Self-hosting means you own the OS and application update cycle — there is no SaaS vendor patching things for you.

**What to demonstrate:** Experience with CVE triage, patch management tooling (Ansible, Puppet, Chef for fleet-wide patching), and coordinating a patch deployment that required maintenance windows on critical systems.

---

## Required Skills — What "Extensive" Actually Means

| JD Phrase | Figure Bar | What to Prepare |
|---|---|---|
| "Strong Linux/Unix" | Can diagnose production issues from CLI alone, without a GUI | [[sre/topics/linux-cli]], [[sre/patterns/troubleshooting-framework]] |
| "Extensive cloud experience (Azure, AWS, GCP)" | Deep in at least one, working knowledge of the others; multi-cloud networking | Cloud networking (VPC peering, VPN, ExpressRoute/Direct Connect) |
| "Infrastructure as Code mastery" | Has led IaC refactors, not just written Terraform | [[devops/topics/infrastructure-as-code]] — Terraform state management, modules, remote backends |
| "High-availability, fault-tolerant distributed systems" | Can design for N+2 redundancy and explain why | [[solution-arch/topics/scalability-and-reliability]] |
| "Monitoring/logging/alerting" | Has set up Prometheus from scratch; written custom alert rules; built Grafana dashboards | [[devops/concepts/observability-pillars]] |
| "Networking fundamentals" | TCP/IP, DNS, HTTPS, VPN, firewall rules; can trace a network path between on-prem and cloud | [[sre/concepts/networking-fundamentals]] |
| "SLO definition, runbook development, incident response" | Has written SLOs, not just enforced them | [[sre/concepts/slo-sli-sla]], [[devops/patterns/incident-response]] |

---

## What the JD Does Not Say (But You Should Raise)

These are gaps worth probing — they signal Figure is early-stage and you can influence these answers:

- **What is the current on-call structure?** No mention of on-call or paging. At a company where manufacturing never stops, someone must be paged. What are the current rotation and escalation paths?
- **What is the robot software deployment process today?** If they're hiring an SRE to own it, it's probably still manual or semi-automated. Understanding the current pain point is essential.
- **How many SREs are on the team?** Staff SRE at a 400-person company might be the only SRE. Know what you're walking into.
- **What is the current state of IaC coverage?** "IaC mastery required" often means "we have some Terraform, it's a mess, fix it."
- **What does the manufacturing deployment lifecycle look like?** They mention manufacturing systems but not the change management process around them. Production floors have change windows, compliance requirements (ISO 9001 for automotive), and approval chains.

---

## Interview Process (Expected)

Figure typically runs:
1. **Recruiter screen** (30 min) — compensation, timeline, role fit
2. **Technical screen** (60 min) — Linux troubleshooting, IaC, cloud architecture, or a design problem
3. **On-site / virtual loop** (4–5 hours):
   - **Systems design** — one of the three scenarios in this wiki
   - **Technical depth** — Linux, networking, Terraform, monitoring deep-dive
   - **Coding** — scripting (Python/Bash), likely infrastructure-adjacent (parse a log, write a health check)
   - **Cross-functional** — how you work with non-SRE stakeholders (robotics eng, manufacturing ops)
   - **Leadership/values** — startup culture fit, ownership mindset, how you handle ambiguity

---

## Behavioral Framing for a Startup SRE Role

FAANG behavioral questions probe process adherence. Startup behavioral questions probe **ownership and agency**. Figure will want to hear:

- "I owned it" not "the team did it"
- "I built it from scratch" over "I maintained it"
- "I convinced leadership to invest in X" over "leadership told us to do X"
- Comfort with ambiguity: "When the requirements were unclear, I did Y to make progress"
- Speed vs. reliability tradeoffs: "I chose to ship a working solution in 2 days over a perfect solution in 3 months because..."

**Key stories to prepare:**
- A time you owned infrastructure end-to-end with minimal support
- A time you migrated a critical system to a new platform without downtime
- A time you introduced reliability discipline (SLOs, postmortems, runbooks) to a team that didn't have it
- A time you made a hard tradeoff between security/reliability and velocity
- A time you worked with hardware/physical constraints rather than just software

---

## Sources
- [[figure/company-context]]
- [[figure/scenarios/robot-fleet-cicd]]
- [[figure/scenarios/saas-to-self-hosted]]
- [[figure/scenarios/robot-telemetry]]
- [[sre/concepts/slo-sli-sla]]
