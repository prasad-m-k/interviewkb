# Figure AI — Company Context

**Related:** [[figure/index]], [[figure/staff-sre-role]]

---

## What Figure Builds

Figure AI is developing autonomous humanoid robots designed to work alongside humans in physically demanding labor environments. Their robots — Figure 01 and Figure 02 — are full-body humanoids (bipedal, dexterous hands) built for tasks like automotive assembly, warehouse logistics, and manufacturing.

**Key context for the interview:**
- Robots operate inside BMW's Spartanburg assembly plant (production partnership) — a real factory, not a lab
- OpenAI partnership for the language/reasoning layer (the "brain" that interprets instructions)
- Funded by Microsoft, OpenAI, Bezos Expeditions, Intel, NVIDIA, Samsung — infrastructure expectations come with that scale of investment
- ~300–500 employees; fast-paced; the SRE function is not yet mature at FAANG scale
- San Jose, CA headquarters; manufacturing co-located or near automotive partners

**What this means for SRE:** You are not infrastructure-as-a-service for a web product. You are the backbone of a company where hardware and software are tightly coupled, production errors have physical consequences, and the pace of development is startup-fast.

---

## Why Infrastructure is Different in Physical AI

### 1. The Robot is a Production System

In web SRE, "production" means a server farm. At Figure, production means a physical robot operating on a BMW factory floor. Infrastructure failures propagate to physical outcomes:
- A bad CI/CD deploy could push a model that makes a robot move unpredictably around humans.
- SCM downtime stops software teams from merging safety-critical patches.
- A manufacturing portal outage freezes the procurement of robot components.

This is not more severe than a web outage — it is differently severe. Rollback does not always mean reboot. Staged deployment means physical validation, not just traffic percentage.

### 2. Hybrid Is Not a Migration Target — It's the Steady State

Most cloud companies treat "on-prem" as legacy to eliminate. At Figure, on-prem is permanent:
- Manufacturing facilities (San Jose + partner sites) require local compute for real-time control loops
- Factory networks are often air-gapped or heavily firewalled from the internet
- Edge compute on the robot itself (inference for vision, grasping) is on-prem by definition
- Cloud handles training, experiment tracking, and fleet telemetry aggregation

The SRE at Figure must be fluent in both worlds and design systems where they interoperate without either being a single point of failure for the other.

### 3. The Software Stack Has an Unusual Lifecycle

A standard web service has one artifact: the application binary. A Figure robot has at minimum:
- **OS layer** (likely Ubuntu or a hardened derivative for robot hardware)
- **Middleware** (probably ROS 2 — Robot Operating System — or a custom equivalent)
- **Control software** (motion planning, manipulation, gait)
- **ML models** (vision, grasping, language understanding — neural network weights)
- **Firmware** (joint actuators, sensors)

Each layer has a different release cadence, different testing infrastructure, and different rollback complexity. Firmware rollback may require physical intervention. Model updates may require re-evaluation against safety benchmarks. A CI/CD system that treats all of these as "just deployments" will fail.

### 4. Manufacturing Systems Are Enterprise, Not Cloud-Native

The job description specifically mentions SCM (Supply Chain Management), supplier portals, and manufacturing systems. These are not microservices running in Kubernetes:
- **ERP** (Enterprise Resource Planning — think SAP or Oracle) manages procurement, BOM (Bill of Materials), inventory
- **MES** (Manufacturing Execution System) tracks production floor operations in real time
- **PLM** (Product Lifecycle Management) manages hardware design revisions
- **Supplier portals** are externally-facing web apps that vendors use to submit quotes, invoices, and quality data

These systems are often vendor-managed SaaS that Figure wants to self-host for security (the explicit JD motivation). They have different availability profiles — an ERP outage during a production shift costs money per minute, but the tolerance for maintenance windows is different than a consumer app.

### 5. Security Is an Existential Concern

Figure is building technology that will be foundational to the physical economy. Competitors, nation-states, and well-funded adversaries would benefit from their:
- Robot control software and ML model weights
- Manufacturing processes and supply chain data
- Safety mechanisms and failure modes

The JD's explicit callout of "Transition SaaS solutions to self-hosted alternatives for enhanced security" reflects this. SaaS tools expose data to third-party clouds; self-hosted keeps IP within Figure's security perimeter. The SRE is a security co-owner here, not just an availability owner.

---

## What "Staff SRE at a Robotics Startup" Actually Means

At a FAANG, "Staff SRE" means you own a specific service or domain within a mature SRE organization. At Figure (~400 people), it means something closer to:

- You likely **built the SRE function** or are joining to **rebuild it** — you own the roadmap, not just execution
- **No platform team** to hand problems to — if the CI/CD system is broken, that is your problem
- **No runbook library** inherited from five years of operations — you write the runbooks
- **Cross-functional by necessity** — you work with robotics engineers, hardware engineers, manufacturing operations, security, and vendors in the same week
- **Vendor relationships** — self-hosting means you negotiate with GitLab, HashiCorp, Grafana Labs, or whichever vendors you choose
- **On-call is real** — if manufacturing halts because a supplier portal is down at 2am, someone gets paged

The compensation ceiling ($250K base) reflects this scope. You are not executing within a mature system — you are building it while the company's core operations depend on it.

---

## What to Research Before the Interview

1. **Figure 01 / Figure 02 demos** (YouTube) — understand what the robot does physically; reference this in design answers to show you understand the product
2. **Figure + BMW partnership** — know the manufacturing context; BMW Spartanburg is an automotive assembly plant
3. **Figure + OpenAI partnership** — multimodal model for robot reasoning; you may be asked about supporting ML training infrastructure
4. **ROS 2** (Robot Operating System) — not required to be a ROS expert, but knowing it exists and what it does signals robotics infrastructure awareness
5. **Self-hosted alternatives** — know GitLab (vs GitHub), Vault (vs AWS Secrets Manager), Grafana/Prometheus (vs Datadog), Gitea, Plane, Harbor (container registry)

---

## Sources
- [[figure/staff-sre-role]]
- [[figure/scenarios/robot-fleet-cicd]]
- [[sre/concepts/slo-sli-sla]]
