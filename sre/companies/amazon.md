# Amazon SRE / Systems Development Engineer (SDE) Interview Prep

**Related:** [[sre/patterns/troubleshooting-framework]], [[sre/concepts/slo-sli-sla]], [[sre/concepts/load-balancers]]

## The Role

Amazon doesn't use the "SRE" title extensively — the equivalent roles are:
- **Systems Development Engineer (SDE):** Software engineering focused on infrastructure, automation, and distributed systems. Codes at the same bar as SWE.
- **Cloud Support Engineer (CSE):** Primarily customer-facing; heavy on AWS service knowledge.
- **Operations Engineer:** More ops-focused; heavy on Linux and tooling.

Many SRE skills apply to Amazon's SDE-Infra roles within AWS teams (EC2, S3, Route 53, ELB, RDS). The interview bar and format below apply to SDE-Infra.

## Process

1. **Online Assessment (OA):** 2 coding problems (LeetCode Medium), completed on HackerRank. Time: 90 min.
2. **Phone screen:** 1 coding question + 2–3 behavioral (Leadership Principles).
3. **Virtual Onsite (5 rounds, each 60 min):**
   - **Coding (2 rounds):** LeetCode Medium–Hard. Language: your choice.
   - **System Design (1 round):** Design a distributed system at AWS scale.
   - **Behavioral — Bar Raiser (2 rounds):** Deep dive into past experience using Leadership Principles.
4. **Debrief:** The **Bar Raiser** (an interviewer from outside the team) has veto power and holds the bar independent of hiring manager influence.

## Emphasis

Amazon is unique in weighting **behavioral (Leadership Principles) equally with technical skills**. A great system design answer with weak LP answers fails the loop.

**Technical:** Same depth as Google but with explicit AWS service knowledge expected for AWS-team roles.

**Behavioral:** The 16 Leadership Principles. Every answer must map to one or more LPs using S.T.A.R. format.

## Leadership Principles (LP) — The 16

| # | Principle | SRE/SDE interview interpretation |
|---|---|---|
| 1 | Customer Obsession | "How did you prioritize reliability because users depended on it?" |
| 2 | Ownership | "You own it end-to-end, even if outside your team's scope." |
| 3 | Invent and Simplify | "What did you automate or simplify to reduce toil?" |
| 4 | Are Right, A Lot | "How do you make good decisions with incomplete data?" |
| 5 | Learn and Be Curious | "Tell me about something technical you recently learned." |
| 6 | Hire and Develop the Best | "How have you mentored someone or raised a team's bar?" |
| 7 | Insist on Highest Standards | "Tell me about a time you refused to ship because quality wasn't right." |
| 8 | Think Big | "Describe a vision you had that others thought was unrealistic." |
| 9 | Bias for Action | "Tell me about a time you made a decision with incomplete information." |
| 10 | Frugality | "How did you accomplish X with limited resources?" |
| 11 | Earn Trust | "Tell me about a time you had to admit you were wrong." |
| 12 | Dive Deep | "Tell me about a time you dug into the details of a problem." |
| 13 | Have Backbone; Disagree and Commit | "Tell me about a time you pushed back on a decision." |
| 14 | Deliver Results | "What was the most impactful project you delivered under pressure?" |
| 15 | Strive to be Earth's Best Employer | Rarely asked in technical loops |
| 16 | Success and Scale Bring Broad Responsibility | Rarely asked in technical loops |

**Most commonly tested LPs for SRE/SDE:** Ownership, Dive Deep, Bias for Action, Deliver Results, Have Backbone.

**Prep strategy:** Prepare 5–7 strong S.T.A.R. stories. Each story should map to 2–3 LPs. Don't prepare 16 separate stories.

## Frequently Tested Topics

### Distributed Systems / AWS Scale
Amazon runs the world's largest cloud. Expect deep AWS-specific questions:

- **S3:** Object storage internals — consistency model (strong consistency since 2020), storage classes, multipart upload, presigned URLs, Cross-Region Replication.
- **DynamoDB:** Key-value + document store; partition key → hash ring distribution; hot partition problem; read/write capacity units vs. on-demand; GSI vs. LSI.
- **SQS vs. SNS vs. Kinesis:** When to use each. SQS = work queue (at-least-once, deduplication with visibility timeout). SNS = pub/sub fan-out. Kinesis = ordered, partitioned, replay-capable stream.
- **EC2 Auto Scaling:** Launch templates, scaling policies (target tracking, step, scheduled), lifecycle hooks for graceful scale-in.
- **Route 53:** DNS routing policies — simple, weighted, latency-based, geolocation, failover, multivalue. Health checks and DNS failover.
- **ELB:** Application Load Balancer (L7), Network Load Balancer (L4), connection draining (deregistration delay), sticky sessions, access logs.

### Linux / Systems
Same depth as Apple/Google:
- [[sre/concepts/linux-boot-process]]
- [[sre/concepts/process-signals]]
- [[sre/concepts/memory-management]]
- [[sre/concepts/disk-and-io]]
- [[sre/concepts/networking-fundamentals]]

### Coding

High-frequency DSA for Amazon:
| Problem | Why Amazon asks it |
|---|---|
| [[dsa/problems/lru-cache]] | Caching layer design (ElastiCache) |
| [[dsa/problems/merge-k-sorted-lists]] | Merging distributed logs from multiple sources |
| [[dsa/problems/top-k-frequent-elements]] | Identifying hot keys in DynamoDB/S3 |
| [[dsa/problems/course-schedule-ii]] | Service dependency ordering in deployments |
| [[dsa/problems/number-of-islands]] | Network topology / VPC connectivity |
| [[dsa/problems/design-hit-counter]] | Rate limiting for API Gateway |
| [[dsa/problems/merge-intervals]] | Maintenance window scheduling, on-call scheduling |

## System Design at Amazon Scale

Amazon's system design questions are AWS-native: "Design X using AWS services."

**Framework:**
1. Clarify requirements (SLA, scale, consistency).
2. Choose AWS primitives (compute: EC2/Lambda/ECS; storage: S3/DynamoDB/RDS; messaging: SQS/SNS/Kinesis).
3. Design for failure: AZ failover, circuit breakers, DLQs.
4. Cost optimization: right-size instances, use S3 Intelligent Tiering, Reserved Instances for predictable load.
5. Security: IAM least privilege, VPC isolation, encryption at rest and in transit, Secrets Manager.

**Common design questions:**
- Design S3 (distributed object store with 11-nines durability)
- Design Amazon's shopping cart (high availability, partition tolerance, shopping cart items as CRDTs)
- Design a distributed rate limiter using DynamoDB
- Design a real-time data pipeline (Kinesis → Lambda → DynamoDB)
- Design a multi-region active-active database layer

### Amazon's Dynamo Paper (Required Reading)
The original 2007 Dynamo paper introduced concepts used in many modern systems:
- **Consistent hashing** with virtual nodes for even distribution.
- **Vector clocks** for conflict resolution.
- **Sloppy quorum** (W + R > N with hinted handoff) for high availability.
- **Anti-entropy** (Merkle trees) for replica reconciliation.

## SRE Practices

Amazon uses the term "Operational Excellence" (from the AWS Well-Architected Framework):
- **Runbooks over heroics:** Every recurring operational task has a runbook. Runbooks drive toward automation.
- **Pre-mortems:** Before a launch, run a "pre-mortem" — assume it failed, work backward to find why.
- **Correction of Errors (COE):** Amazon's version of a postmortem. Must include 5 Whys, customer impact, and corrective action.
- **Two-Pizza Teams:** Small autonomous teams own their services end-to-end, including on-call.

### Operational Metrics Amazon Watches
- **TTD (Time to Detect):** How fast does monitoring catch the issue?
- **TTM (Time to Mitigate):** How fast does the team restore service?
- **Error Rate:** Percentage of failed API calls.
- **Availability:** Per-service, per-region, per-AZ.

## Scripting Problems
Amazon frequently tests practical scripting:
- [[sre/problems/log-parsing-script]] — Find top-10 IPs from access logs
- [[sre/problems/disk-space-hogs]] — Find largest directories
- Configuration rollout with error-rate rollback
- DynamoDB item bulk migration script

## Behavioral Tips

- Use "I" not "we" in stories — interviewers need to understand *your* contribution.
- Always quantify impact: "reduced deployment time by 40%," "saved $200K/year in compute costs."
- Stories should show **scope growth**: started with a small problem, saw a bigger opportunity, drove impact beyond original scope.
- For "Dive Deep" questions: go to the level of kernel calls, network packets, or DB query plans. Amazon loves depth.
- Practice the "why" recursively (5 Whys) — this is how Amazon-caliber engineers think about root causes.

## Tips from Sources

- The Bar Raiser's job is to maintain the bar, not to be adversarial. Treat their hard follow-up questions as genuine curiosity, not a gotcha.
- Amazon's system design expects explicit cost analysis. "This design uses 3 AZs × 10 r5.4xlarge instances = ~$X/month" — show you care about frugality (LP #10).
- For coding, write tests for your code before running. Amazon's culture has high standards for code quality even in interviews.
- Always ask about consistency requirements for system design. Amazon's culture defaults to eventual consistency (high availability), but payments and inventory must be strongly consistent.

## Sources

- AWS Well-Architected Framework (Operational Excellence pillar)
- Amazon Dynamo paper (SOSP 2007)
- [[sre/concepts/slo-sli-sla]]
