# 01. SRE Fundamentals

A study guide covering the foundations of Site Reliability Engineering (SRE), how it relates to DevOps and Platform Engineering, the responsibilities of an SRE, and the principles used to operate reliable systems.

> **Learning goal:** Understand not only what SRE is, but how an SRE approaches engineering decisions, operational work, and production failures.

## Contents

- [1. What is Site Reliability Engineering?](#1-what-is-site-reliability-engineering)
- [2. SRE vs DevOps vs Platform Engineering](#2-sre-vs-devops-vs-platform-engineering)
- [3. The role and responsibilities of an SRE](#3-the-role-and-responsibilities-of-an-sre)
- [4. Reliability as an engineering discipline](#4-reliability-as-an-engineering-discipline)
- [5. Toil, automation, and operational excellence](#5-toil-automation-and-operational-excellence)
- [6. Principles behind Google's SRE approach](#6-principles-behind-googles-sre-approach)
- [7. Putting the ideas together](#7-putting-the-ideas-together)
- [8. Key takeaways](#8-key-takeaways)
- [9. Learning resources](#9-learning-resources)
- [10. Questions to check my understanding](#10-questions-to-check-my-understanding)

---

## 1. What is Site Reliability Engineering?

**Site Reliability Engineering (SRE)** is an approach to operating and maintaining systems that applies software engineering principles to operations problems.

Instead of relying only on manual operational work, SRE uses engineering, automation, measurement, and disciplined incident response to make services more reliable and easier to operate.

The term is strongly associated with Google, where SRE was developed as a way to apply software engineering to the work traditionally done by operations teams.

### What does an SRE care about?

An SRE helps ensure that a service:

- Is available when users need it.
- Responds within an acceptable amount of time.
- Behaves correctly under expected and unexpected load.
- Recovers effectively when something fails.
- Can be monitored and troubleshot.
- Can be changed and deployed without unnecessary risk.
- Does not require an unsustainable amount of manual work to maintain.

### Example

Imagine an online shopping API. It works well under normal traffic, but during a major sale its response time increases and some requests fail.

A traditional reactive approach might involve restarting services or adding servers whenever users complain.

An SRE approach would investigate the evidence:

1. Check service metrics, logs, traces, and recent changes.
2. Identify whether the bottleneck is CPU, memory, the database, a dependency, or insufficient capacity.
3. Mitigate the immediate impact on users.
4. Restore service and communicate the status.
5. Investigate contributing causes.
6. Introduce a lasting improvement, such as better capacity planning, query optimization, autoscaling, or more useful alerts.
7. Review whether the service's reliability objectives were met.

The aim is not merely to make the service work again. It is to reduce the likelihood and impact of similar failures in the future.

### SRE is more than monitoring

Monitoring tools, cloud platforms, Kubernetes, and automation are useful parts of the toolkit, but they do not by themselves make a team an SRE team.

SRE also involves setting reliability goals, making trade-offs, managing incidents, learning from failures, and continuously improving the system.

---

## 2. SRE vs DevOps vs Platform Engineering

These disciplines overlap, but they emphasize different outcomes. Their responsibilities vary by organization, so the boundaries are not universal.

| Area | Main emphasis | Typical questions |
|---|---|---|
| DevOps | Improving collaboration and flow between software development and operations | How can we build, test, and release changes reliably and efficiently? |
| Site Reliability Engineering | Applying engineering practices to service reliability and operations | Is the service reliable enough? How do we detect, mitigate, and prevent failures? |
| Platform Engineering | Providing reusable internal platforms and tools for software teams | How can developers safely deploy and operate services without reinventing infrastructure? |

### DevOps

DevOps is a culture and set of practices that bring development and operations closer together. It encourages shared responsibility, automation, continuous integration and delivery, feedback, and shorter delivery cycles.

Typical activities include:

- Building CI/CD pipelines.
- Automating infrastructure provisioning.
- Improving deployment processes.
- Managing configuration and environments.
- Encouraging collaboration between development and operations.

### Site Reliability Engineering

SRE focuses particularly on the reliability of services in production. It uses software engineering and measurable objectives to manage operational risk.

Typical activities include:

- Defining and measuring SLIs and SLOs.
- Designing useful monitoring and alerts.
- Responding to incidents and coordinating recovery.
- Investigating recurring failures.
- Automating repetitive operational work.
- Improving capacity, resilience, and recovery.

### Platform Engineering

Platform Engineering builds and maintains internal capabilities that make it easier for development teams to deliver and operate software.

Examples include:

- Standardized Kubernetes clusters and deployment templates.
- Self-service infrastructure provisioning.
- Reusable CI/CD workflows.
- Internal developer portals.
- Shared observability, security, and secrets-management capabilities.

### How they work together

Consider a company running dozens of microservices on Kubernetes.

- **DevOps practices** help teams build, test, and deploy changes consistently.
- **Platform Engineering** provides the reusable deployment workflows, cluster capabilities, and self-service tools.
- **SRE** helps define reliability targets, improve observability, respond to incidents, and reduce systemic operational risk.

One person or team may perform work across all three areas, particularly in smaller organizations.

**Key distinction:** DevOps emphasizes the way development and operations work together; SRE emphasizes measurable reliability through engineering; Platform Engineering emphasizes reusable capabilities and developer experience.

---

## 3. The role and responsibilities of an SRE

An SRE's exact responsibilities depend on the organization, the services they support, and how the team is structured.

### Core responsibilities

#### Reliability objectives

- Identify the user-visible characteristics that matter.
- Define appropriate SLIs and SLOs.
- Review service performance against objectives.
- Help teams make informed decisions about reliability and release risk.

#### Monitoring and observability

- Build or improve dashboards.
- Create actionable alerts that indicate meaningful problems.
- Use metrics, logs, and traces to investigate behavior.
- Improve visibility into service dependencies and failure modes.

#### Incident response

- Detect and assess service disruptions.
- Help mitigate customer impact and restore service.
- Coordinate with application, infrastructure, and other relevant teams.
- Communicate clearly during incidents.
- Document timelines, findings, and follow-up actions.

#### Automation and toil reduction

- Replace repetitive manual tasks with reliable automation.
- Build tools for deployments, diagnostics, recovery, or routine maintenance.
- Improve operational workflows and runbooks.
- Reduce avoidable human error.

#### Capacity and performance

- Understand workload patterns and resource consumption.
- Identify performance bottlenecks.
- Plan capacity and validate scaling behavior.
- Test system behavior under load.

#### Resilience and recovery

- Identify single points of failure.
- Improve health checks, timeouts, retries, and graceful degradation.
- Validate backup and restore procedures.
- Participate in disaster recovery planning and testing.

#### Learning and continuous improvement

- Analyze incidents without defaulting to blame.
- Find contributing technical and organizational factors.
- Track corrective actions through completion.
- Share knowledge through documentation and reviews.

### What does an SRE do during an incident?

A useful order of priorities is:

1. **Understand impact:** What is failing, and which users or services are affected?
2. **Establish evidence:** Check alerts, dashboards, logs, traces, and recent changes.
3. **Mitigate:** Reduce customer impact using a safe rollback, traffic shift, scaling change, or other appropriate action.
4. **Verify recovery:** Confirm that service behavior has returned to acceptable levels.
5. **Communicate:** Keep stakeholders informed through the incident.
6. **Learn:** Investigate causes and contributing factors, then document actionable improvements.

The safest mitigation depends on the incident. Changes should be deliberate, and recovery should be verified rather than assumed.

### Skills that support SRE work

- Linux and operating-system fundamentals.
- Networking, DNS, HTTP, and TLS.
- Scripting and software engineering.
- Cloud infrastructure and infrastructure as code.
- Containers and Kubernetes.
- Monitoring and observability.
- Distributed-systems fundamentals.
- Troubleshooting, communication, and incident coordination.

---

## 4. Reliability as an engineering discipline

Reliability is not simply the absence of incidents. It is the ability of a service to deliver its intended behavior consistently under defined conditions.

Failures cannot always be eliminated. Systems depend on hardware, networks, software, people, and external services, all of which can fail or behave unexpectedly.

The engineering goal is to understand risk, reduce avoidable failures, limit the impact of unavoidable failures, and recover effectively.

### Reliability involves trade-offs

Perfect reliability is usually neither achievable nor economically justified. Additional redundancy, capacity, testing, and operational controls have costs and can introduce complexity of their own.

A team therefore needs to understand:

- What level of reliability users actually need.
- Which failures have the greatest impact.
- How much risk a proposed change introduces.
- Whether the system can detect and recover from failure.
- Where engineering effort will provide the greatest benefit.

### Measure reliability

Reliability discussions become more useful when they use measurable indicators rather than vague statements such as "the system is stable."

Examples of measurements include:

- Successful requests as a proportion of eligible requests.
- Request latency, including p95 and p99.
- Error rates.
- Saturation of CPU, memory, connection pools, or other constrained resources.
- Recovery time following an incident.

A **Service Level Indicator (SLI)** is a measurement of a service characteristic. A **Service Level Objective (SLO)** is a target for that measurement over a defined period. A **Service Level Agreement (SLA)** is an agreement that may include consequences if specified commitments are not met.

These concepts are explored in more detail in the service-reliability section.

### Design for failure

Reliable systems assume that components can fail.

Useful techniques include:

- Redundancy across appropriate failure domains.
- Health checks and traffic routing.
- Timeouts to avoid waiting indefinitely.
- Bounded retries with backoff and jitter.
- Rate limiting and load shedding.
- Graceful degradation when optional dependencies fail.
- Backups and tested recovery procedures.
- Safe deployments and rollback strategies.

These techniques must be applied thoughtfully. For example, aggressive retries can amplify an overloaded service rather than help it recover.

### Reliability across the lifecycle

Reliability is influenced by design, implementation, testing, deployment, monitoring, incident response, and subsequent improvement. It is not a task that begins only after an application reaches production.

---

## 5. Toil, automation, and operational excellence

In SRE, **toil** refers to operational work that tends to be manual, repetitive, automatable, reactive, and limited in lasting value. Google's SRE guidance uses a more specific definition; not every operational task is toil.

Examples may include:

- Manually repeating the same deployment steps.
- Running the same diagnostic commands for every incident.
- Repeatedly cleaning up a resource because a process is not automated.
- Manually processing routine alerts that do not require human judgment.

Some operational work is necessary and valuable. Incident investigation, complex troubleshooting, and planned maintenance are not automatically toil just because they are operational.

### Why toil matters

Excessive toil can:

- Consume time that could be spent on engineering improvements.
- Increase the risk of human error.
- Slow response and delivery.
- Create dependency on a small number of people.
- Grow as services and workloads scale.

### A practical automation example

**Problem:** An engineer manually checks whether a service is responding several times a day.

**Short-term solution:** Use a health-check command or script to standardize the check.

**Long-term improvement:** Add appropriate service telemetry and actionable alerting, document the diagnostic procedure, and investigate why repeated manual checks are necessary.

Automation should make the operation safer and more repeatable. Automating a poorly understood process can simply make mistakes happen faster.

### An improvement cycle

1. Identify repetitive or error-prone work.
2. Understand the underlying cause and expected behavior.
3. Decide whether to eliminate, simplify, or automate it.
4. Implement the change with appropriate safeguards.
5. Measure whether it reduced effort or improved reliability.
6. Review and maintain the automation over time.

Operational excellence means continuously improving how systems are operated, not merely accumulating scripts and tools.

---

## 6. Principles behind Google's SRE approach

Google's SRE publications describe several influential principles. These are foundational ideas rather than a mandatory checklist for every organization.

### 6.1 Embrace risk

The objective is not to eliminate every possible failure. Teams should understand the level of risk that is acceptable for their service and manage it deliberately.

**Practical implication:** A low-risk internal service may need different reliability investment from a critical customer-facing payment service.

### 6.2 Service Level Objectives

Define measurable reliability targets that reflect what users need. SLOs provide a shared basis for evaluating service performance and deciding where engineering effort is needed.

**Practical implication:** Instead of saying "the API should be fast," define an appropriate latency objective and measure it consistently.

### 6.3 Error budgets

An error budget represents the amount of unreliability permitted by an SLO over its measurement window.

For example, if an illustrative availability SLO is 99.9% over a 30-day period, the corresponding budget is 0.1% of eligible time or requests, depending on how the SLI is defined.

A team can use error-budget consumption to discuss release risk and reliability improvements. A depleted budget may trigger a review or a policy to prioritize reliability work, depending on the organization's agreed process.

### 6.4 Eliminate toil

Use engineering and automation to reduce repetitive operational work, freeing time for work that improves reliability and system design.

### 6.5 Monitor distributed systems

Use monitoring to understand user-visible service behavior and identify problems that require action. Good monitoring should help answer what is broken, how severe it is, and where investigation should begin.

### 6.6 Automate to reduce work and risk

Automation can improve consistency, repeatability, and response speed. It should have clear behavior, appropriate safeguards, and enough observability to detect when it fails.

### 6.7 Release engineering

Treat software releases as an engineering process. Build repeatable deployment mechanisms, validate changes, limit risk where appropriate, and make rollback or recovery practical.

### 6.8 Simplicity

Complexity makes systems harder to understand, test, and operate. Prefer designs that satisfy requirements without introducing unnecessary moving parts.

### A note on Google's model

Google's SRE practices evolved in the context of its own scale, systems, and organizational structure. Other organizations can adopt the principles while adapting implementation to their own needs.

---

## 7. Putting the ideas together

Consider a service whose error rate increases immediately after a deployment.

An SRE-minded response connects several of the concepts in this guide:

1. **Monitoring:** An actionable alert identifies an increase in failed requests.
2. **Impact assessment:** The team checks the affected endpoints, users, and dependencies.
3. **Reliability objectives:** The team assesses the impact on its SLO and remaining error budget.
4. **Incident response:** Engineers investigate the deployment and other plausible causes while mitigating the impact.
5. **Recovery:** The team rolls back or applies a safe fix, then verifies service health.
6. **Learning:** The team documents contributing factors and agrees on follow-up actions.
7. **Automation:** Where appropriate, deployment checks or progressive rollout controls are improved to detect similar problems earlier.
8. **Continuous improvement:** The team reviews whether the changes actually reduce risk.

This example shows how reliability measurement, technical troubleshooting, operational practices, and engineering improvements work together.

---

## 8. Key takeaways

- SRE applies software engineering to operations and reliability problems.
- DevOps, SRE, and Platform Engineering overlap but emphasize different outcomes.
- Reliability must be defined in terms of user needs and measurable service behavior.
- Incidents should be handled by prioritizing impact reduction, recovery, communication, and learning.
- Toil is operational work that can often be reduced through better engineering.
- Automation needs safeguards and should address the underlying problem.
- SLOs and error budgets help teams make explicit reliability trade-offs.
- Reliable systems are designed to detect, tolerate, and recover from failures.
- The right practices depend on the service, its users, and its risk profile.

---

## 9. Learning resources

These are starting points for further study. This list records useful resources to explore; it does not imply that every resource has been completed.

### Primary references

1. **Google: Site Reliability Engineering: The Book**
   - Covers SRE fundamentals, risk, SLOs, toil, monitoring, and incident response.
   - https://sre.google/sre-book/table-of-contents/

2. **Google: The Site Reliability Workbook**
   - Practical guidance on implementing SRE practices.
   - https://sre.google/workbook/table-of-contents/

3. **Google: SRE Resources**
   - Articles, books, and additional SRE material.
   - https://sre.google/

### Further reading

4. **Google Cloud: Architecture Framework**
   - Reliability and operational considerations for cloud systems.
   - https://cloud.google.com/architecture/framework

5. **Microsoft Azure: Well-Architected Framework**
   - Reliability and operational guidance for Azure workloads.
   - https://learn.microsoft.com/en-us/azure/well-architected/

6. **AWS: Well-Architected Framework**
   - Reliability principles and design considerations for AWS workloads.
   - https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html


