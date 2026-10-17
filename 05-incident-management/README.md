# 05. Incident Management

A study guide covering how incidents are detected, assessed, managed, investigated, and resolved in Site Reliability Engineering (SRE). It explores incident response workflows, severity levels, recovery metrics, Root Cause Analysis (RCA), blameless postmortems, runbooks, and continuous improvement.

> **Learning goal:** Understand how an SRE responds to production incidents, minimizes user impact, coordinates recovery, investigates contributing factors, and uses lessons learned to improve system reliability.

---

## Contents

- [1. What is Incident Management?](#1-what-is-incident-management)
- [2. Incident Detection, Triage, and Response](#2-incident-detection-triage-and-response)
- [3. Severity Levels and Escalation](#3-severity-levels-and-escalation)
- [4. Mean Time to Detect (MTTD)](#4-mean-time-to-detect-mttd)
- [5. Mean Time to Recover (MTTR)](#5-mean-time-to-recover-mttr)
- [6. Root Cause Analysis (RCA)](#6-root-cause-analysis-rca)
- [7. Blameless Postmortems](#7-blameless-postmortems)
- [8. Incident Documentation and Runbooks](#8-incident-documentation-and-runbooks)
- [9. Learning from Recurring Incidents](#9-learning-from-recurring-incidents)
- [10. Putting the Ideas Together](#10-putting-the-ideas-together)
- [11. Key Takeaways](#11-key-takeaways)
- [12. Learning Resources](#12-learning-resources)

---

## 1. What is Incident Management?

**Incident Management** is the process of identifying, coordinating, responding to, and recovering from events that disrupt or threaten the normal operation of a service.

In SRE, an incident is not simply an error appearing in a log or an alert firing. It is an event that requires coordinated action because a service is experiencing, or is at significant risk of experiencing, an unacceptable impact.

Examples include:

- An application becoming unavailable.
- An API returning a large number of HTTP 5xx responses.
- A database becoming overloaded.
- A deployment introducing a production defect.
- A Kubernetes service experiencing widespread pod failures.
- A network or DNS problem preventing users from reaching an application.
- A dependency failure causing several services to become unhealthy.

The objective of incident management is to reduce user impact, restore acceptable service behavior, communicate effectively, and learn from what happened.

### Why does incident management matter?

Production systems are made up of many interconnected components. Even a small failure can affect multiple services when dependencies, capacity limits, or recovery mechanisms behave unexpectedly.

Without a structured response process, engineers may investigate the same problem independently, make conflicting changes, overlook important evidence, or delay recovery.

A structured incident management process helps teams:

- Detect problems before they cause widespread impact.
- Prioritize incidents according to their severity.
- Assign clear responsibilities during recovery.
- Coordinate communication between technical teams and stakeholders.
- Restore service safely and verify that it has recovered.
- Preserve evidence for later investigation.
- Identify improvements that prevent similar incidents.

### Incident management versus problem management

These activities are related but serve different purposes.

| Activity | Main objective | Example |
|---|---|---|
| Incident management | Restore acceptable service as quickly and safely as possible | Roll back a deployment causing API failures |
| Problem management | Investigate and address the underlying causes of incidents | Identify why a deployment passed testing despite introducing a production defect |
| Change management | Control the risk associated with changes to systems | Introduce progressive rollouts and deployment verification |

An incident may be resolved before its underlying causes are fully understood. The investigation and permanent corrective work can continue after service has recovered.

### Example

Imagine an online shopping API that begins returning HTTP 503 responses shortly after a deployment.

A structured response might look like this:

1. Monitoring detects an increase in failed requests.
2. The on-call engineer acknowledges the alert and assesses its impact.
3. The incident is assigned an appropriate severity.
4. The team investigates the recent deployment and current service health.
5. Engineers roll back the deployment if the evidence supports that action.
6. Monitoring confirms that successful requests have returned to acceptable levels.
7. The incident timeline and technical findings are documented.
8. A follow-up investigation examines why the defect reached production.
9. Corrective actions improve deployment checks and release safety.

The immediate goal is to restore service. The longer-term goal is to reduce the likelihood and impact of a similar incident.

**Key distinction:** Incident management focuses on coordinated response and recovery, while post-incident investigation focuses on understanding what happened and improving the system.

---

## 2. Incident Detection, Triage, and Response

Incident response can be organized into three closely related activities: detection, triage, and response.

### 2.1 Incident detection

**Incident detection** is the process of identifying that a service is failing or behaving outside its expected operating conditions.

Incidents can be detected through automated monitoring, alerts, customer reports, internal teams, synthetic checks, or unusual system behavior.

Common detection signals include:

- Increased request error rates.
- Higher response latency.
- Unavailable endpoints.
- CPU or memory saturation.
- Database connection failures.
- Kubernetes pods repeatedly restarting.
- Failed background jobs or delayed message processing.
- Unexpected changes in traffic or resource consumption.

#### Monitoring-based detection

Suppose an API normally processes requests with a low error rate. Following a configuration change, its HTTP 5xx rate increases significantly.

A monitoring system evaluates the relevant metrics and triggers an alert when an agreed condition is met.

The alert directs the on-call engineer toward a potential service problem.

However, an alert does not automatically prove that an incident exists. The engineer must assess whether the signal represents a genuine problem, a temporary fluctuation, or an issue that does not require immediate intervention.

#### Customer-reported detection

Not every incident is detected by monitoring.

Users may report problems that existing alerts fail to identify, such as a broken checkout workflow or incorrect application behavior.

Customer reports are therefore valuable signals and may reveal gaps in monitoring coverage.

#### Good incident detection

Effective detection should:

- Identify meaningful user-facing failures.
- Minimize unnecessary alert noise.
- Provide enough context to begin investigation.
- Route alerts to the appropriate responders.
- Detect problems before they become more severe when possible.

Detection quality depends on both the telemetry available and the conditions used to trigger alerts.

### 2.2 Incident triage

**Incident triage** is the process of quickly assessing an incident to determine its impact, urgency, scope, and immediate response requirements.

The goal is not necessarily to identify the exact root cause immediately. It is to understand how serious the situation is and what needs to happen first.

During triage, an engineer should consider:

- **Impact:** Are users unable to complete important tasks?
- **Scope:** Is the problem affecting one endpoint, one service, or an entire platform?
- **Duration:** When did the problem begin, and is it continuing?
- **Severity:** How serious is the disruption?
- **Recent changes:** Were there deployments or configuration changes before the incident?
- **Dependencies:** Are databases, networks, external APIs, or other services affected?
- **Risk of escalation:** Could the failure spread or cause additional damage?

#### Example

An application generates two alerts:

- Alert A: CPU utilization reaches 90% on one non-critical worker.
- Alert B: The checkout API begins returning HTTP 503 responses to a large proportion of requests.

Alert B will generally require a higher response priority because it directly affects an important user workflow.

High resource utilization is not automatically an incident of the same severity as a user-visible outage. The consequences of the condition matter.

### 2.3 Incident response

**Incident response** is the coordinated work performed to contain the impact, mitigate the failure, restore service, and communicate relevant information.

A practical response workflow is:

1. **Acknowledge:** Confirm that someone is responsible for investigating the alert.
2. **Assess:** Determine the affected services, users, and critical functions.
3. **Classify:** Assign an initial severity based on impact and urgency.
4. **Coordinate:** Bring in the appropriate responders and assign responsibilities.
5. **Investigate:** Collect evidence from metrics, logs, traces, deployment history, and system state.
6. **Mitigate:** Apply a safe action to reduce or eliminate the immediate impact.
7. **Verify:** Confirm that the service has returned to acceptable behavior.
8. **Communicate:** Keep relevant stakeholders informed.
9. **Document:** Record important decisions, observations, and timestamps.
10. **Follow up:** Investigate contributing factors and track corrective actions.

The exact order can vary. For example, a well-understood outage may justify immediate rollback while a more detailed investigation continues in parallel.

#### Mitigation versus permanent resolution

These terms describe different outcomes.

- **Mitigation** reduces the immediate impact of an incident.
- **Recovery** means the service has returned to an acceptable operating condition.
- **Permanent remediation** addresses the underlying technical or organizational factors that contributed to the incident.

For example, restarting an unhealthy service may restore availability temporarily. If the service repeatedly becomes unhealthy because of a memory leak, the underlying problem remains.

A successful incident response should not assume that a temporary recovery eliminates the need for further investigation.

### 2.4 Incident coordination

As incidents become more complex, assigning clear responsibilities helps reduce confusion.

Common incident response roles include:

| Role | Responsibility |
|---|---|
| Incident Commander | Coordinates the response, sets priorities, and manages overall progress |
| Technical Lead | Guides technical investigation and mitigation |
| Communications Lead | Provides updates to stakeholders and affected users |
| Responders | Investigate components, execute mitigation steps, and provide technical findings |
| Scribe | Records the incident timeline, decisions, and important observations |

In a small incident, one person may perform several roles. During a major incident, separating these responsibilities helps technical responders focus on diagnosis and recovery.

These roles are a useful model rather than a universal requirement. Teams should adapt them to their size and incident procedures.

### 2.5 Safe mitigation and verification

Mitigation actions can include:

- Rolling back a problematic deployment.
- Shifting traffic away from an unhealthy instance.
- Scaling a constrained service.
- Disabling a faulty feature using a feature flag.
- Reducing non-essential workload.
- Failing over to a healthy replica or region.
- Temporarily disabling an affected integration.

The correct action depends on the evidence and the service architecture.

Before applying a change, consider whether it could worsen the problem. For example, adding application replicas may increase database connection pressure if the database is already overloaded.

After mitigation, verify recovery using service-level signals such as successful request rates, latency, error rates, and critical user workflows.

**Key takeaway:** A useful incident response prioritizes impact reduction and safe recovery. Root cause investigation should not unnecessarily delay an appropriate mitigation.

---

## 3. Severity Levels and Escalation

### 3.1 What are incident severity levels?

**Incident severity levels** classify incidents according to their impact, urgency, and potential consequences.

Severity helps teams decide how quickly to respond, which resources to involve, and how frequently to communicate updates.

A severity classification should reflect the consequences of the incident rather than simply the technical complexity of the problem.

A complicated failure affecting a small internal tool may be less severe than a simple configuration error that makes a critical customer-facing service unavailable.

### 3.2 Example severity model

The following is an illustrative severity model. Actual definitions, response targets, and escalation rules should be established by each organization.

| Severity | Description | Example |
|---|---|---|
| SEV-1 — Critical | Major outage or severe impact on a critical business function | The main payment service is unavailable for most users |
| SEV-2 — High | Significant service degradation affecting many users or an important capability | Checkout requests frequently fail, but a partial workaround exists |
| SEV-3 — Medium | Limited impact or degradation with a reasonable workaround | One non-critical feature is intermittently unavailable |
| SEV-4 — Low | Minor issue with little immediate user impact | A non-critical background report is delayed |

Severity should be based on the team's documented criteria. The labels above are examples, not universal definitions.

### 3.3 Factors used to determine severity

When classifying an incident, consider:

**User impact**

- How many users are affected?
- Are users completely blocked or experiencing partial degradation?
- Are critical workflows unavailable?

**Service criticality**

- Does the affected service support payments, authentication, data processing, or another essential function?
- Are other services dependent on it?

**Duration and urgency**

- Is the problem ongoing?
- Is its impact increasing?
- How quickly must action be taken to avoid further harm?

**Data and security implications**

- Is there evidence of data loss, data corruption, or unauthorized access?
- Are security, privacy, or compliance teams required?

**Available workarounds**

- Can users continue with an alternative workflow?
- Is the workaround reliable and safe?

Severity may change as more evidence becomes available. An initially limited incident can be escalated if the impact spreads, and a higher-severity incident can be downgraded once its scope is understood.

### 3.4 What is escalation?

**Escalation** is the process of involving additional expertise, authority, or response resources when an incident requires more attention than the current responder can provide.

Escalation can be triggered by:

- A response deadline being missed.
- The incident exceeding the current responder's expertise.
- The severity increasing.
- The failure affecting additional services.
- A mitigation attempt failing.
- A suspected security or data-integrity issue.
- A need for decisions outside the responder's authority.

#### Types of escalation

**Functional escalation**

The incident is passed to someone with the required technical expertise.

Example: An application engineer escalates a database connection failure to the database operations team.

**Hierarchical escalation**

A manager or incident lead is involved because the incident requires broader coordination, resources, or business decisions.

**Automatic escalation**

An alerting or on-call system notifies additional responders when an acknowledgment or response condition is not met.

### 3.5 On-call responsibilities

An on-call engineer is responsible for responding to assigned alerts during a defined period.

An effective on-call process should include:

- Clear ownership of services.
- Reliable alert delivery.
- Documented severity and escalation policies.
- Access to dashboards and diagnostic tools.
- Up-to-date runbooks.
- A clear handover process between shifts.
- Backup responders for situations requiring additional expertise.

An alert that reaches nobody is not an effective incident detection mechanism.

### 3.6 Example escalation workflow

Imagine a production API begins failing and the on-call engineer cannot identify the cause.

1. The engineer acknowledges the alert.
2. Initial checks identify widespread request failures.
3. The incident is classified according to the team's severity policy.
4. The incident commander or appropriate lead is engaged if required.
5. The engineer checks recent changes, dependencies, and service health.
6. Database specialists are contacted when evidence points toward database saturation.
7. Additional responders are brought in if the incident affects other services.
8. The team applies a safe mitigation and verifies recovery.
9. The escalation and response timeline are documented.

Escalation should not be interpreted as failure by the original responder. It is a normal mechanism for getting the right expertise involved quickly.

**Key takeaway:** Severity determines the urgency of the response, while escalation ensures that the incident receives the appropriate expertise and authority.

---

## 4. Mean Time to Detect (MTTD)

### 4.1 What is MTTD?

**Mean Time to Detect (MTTD)** measures the average time between an incident beginning and the system or organization detecting it.

It helps evaluate how effectively monitoring, alerting, and other detection mechanisms identify problems.

A lower MTTD generally means that incidents are detected sooner, giving responders more opportunity to limit user impact.

### 4.2 Formula

For a collection of incidents:

\[
\text{MTTD} =
\frac{\sum \text{Detection Time}_i}{N}
\]

Where:

- Detection Time is the time between the incident's start and its detection.
- \(N\) is the number of incidents included in the calculation.

For each incident:

\[
\text{Detection Time} =
\text{Detection Timestamp} -
\text{Incident Start Timestamp}
\]

The calculation requires a consistent definition of when an incident began and when it was detected.

### 4.3 Example calculation

Consider three incidents:

| Incident | Started | Detected | Detection time |
|---|---|---|---|
| Incident A | 10:00 | 10:02 | 2 minutes |
| Incident B | 11:00 | 11:05 | 5 minutes |
| Incident C | 14:00 | 14:03 | 3 minutes |

Therefore:

\[
\text{MTTD} = \frac{2+5+3}{3}
\]

\[
\boxed{\text{MTTD} = 3.33\text{ minutes}}
\]

The average detection time across these incidents is approximately 3 minutes and 20 seconds.

This average does not show the full distribution. Some incidents may be detected almost immediately, while others remain unnoticed for much longer.

### 4.4 Why MTTD matters

Long detection delays can increase:

- The number of users affected.
- The duration of degraded service.
- The amount of failed or delayed work.
- The difficulty of reconstructing what happened.
- The operational and business impact of the incident.

Reducing MTTD helps responders begin investigation sooner, although faster detection alone does not guarantee faster recovery.

### 4.5 How can MTTD be improved?

**Improve service-level monitoring**

Monitor user-visible service behavior, such as request success rates and latency, instead of relying exclusively on infrastructure health.

**Create actionable alerts**

Alert on conditions that require investigation or intervention. Avoid excessive alerts for harmless fluctuations.

**Monitor critical dependencies**

Track the health of databases, queues, external APIs, and other components that can disrupt important workflows.

**Add synthetic checks**

Periodically execute representative user journeys or API requests to identify failures that internal infrastructure metrics may miss.

**Improve telemetry**

Collect useful metrics, logs, and traces so that abnormal behavior can be recognized.

**Review missed incidents**

If users report a failure before monitoring detects it, investigate why the existing detection mechanisms did not identify the problem.

### 4.6 MTTD limitations

MTTD is useful, but it does not describe the entire incident response process.

A service may detect an incident immediately while requiring several hours to restore functionality. Another service may detect a problem later but recover quickly once the alert fires.

Also, detection timestamps can be difficult to establish for incidents that begin gradually or are discovered through retrospective analysis.

For these reasons, MTTD should be considered alongside recovery time, user impact, and service-level indicators.

**Key takeaway:** MTTD measures how quickly an incident is detected, not how quickly it is resolved.

---

## 5. Mean Time to Recover (MTTR)

### 5.1 What is MTTR?

**Mean Time to Recover (MTTR)** measures the average time required to restore a service to an acceptable operating condition following an incident.

Recovery time is important in SRE because incidents cannot always be prevented. The ability to detect failures, mitigate their effects, and restore service safely is a major part of reliability engineering.

Some organizations use MTTR to mean Mean Time to Repair or Mean Time to Resolve. These definitions can differ, so a team should explicitly document which events mark the beginning and end of its measurement.

This guide uses MTTR to mean Mean Time to Recover.

### 5.2 Formula

\[
\text{MTTR} =
\frac{\sum \text{Recovery Time}_i}{N}
\]

For each incident:

\[
\text{Recovery Time} =
\text{Recovery Timestamp} -
\text{Incident Start Timestamp}
\]

### 5.3 Example calculation

Suppose three production incidents have the following recovery durations:

| Incident | Start time | Recovery time | Duration |
|---|---|---|---|
| Incident A | 09:00 | 09:20 | 20 minutes |
| Incident B | 11:00 | 11:45 | 45 minutes |
| Incident C | 15:00 | 15:15 | 15 minutes |

Therefore:

\[
\text{MTTR} = \frac{20+45+15}{3}
\]

\[
\boxed{\text{MTTR} = 26.67\text{ minutes}}
\]

The average recovery time is approximately 26 minutes and 40 seconds.

### 5.4 Why MTTR matters

A lower recovery time can indicate that a team is becoming more effective at restoring service.

It may reflect improvements in:

- Incident detection and triage.
- On-call readiness.
- Diagnostic tools and telemetry.
- Runbook quality.
- Rollback and failover procedures.
- Service architecture and fault isolation.
- Coordination between engineering teams.

However, a lower MTTR does not automatically prove that reliability has improved. A system might recover quickly from frequent incidents while still creating a poor user experience.

MTTR should therefore be interpreted alongside incident frequency, severity, duration of user impact, and SLO performance.

### 5.5 How can MTTR be reduced?

**Improve observability**

Provide dashboards, logs, traces, and metrics that help responders identify the affected components.

**Make rollback safe and repeatable**

Ensure that deployments can be reversed when a release introduces a defect, with appropriate consideration for database and data compatibility.

**Maintain useful runbooks**

Document the checks, commands, and recovery procedures needed for common failure scenarios.

**Practice recovery**

Test failover, restoration, and rollback procedures before they are needed during a real incident.

**Improve service isolation**

Prevent a failure in one component from unnecessarily affecting unrelated components.

**Reduce escalation delays**

Make ownership, contact paths, and escalation policies clear.

**Automate carefully**

Automated remediation can reduce recovery time for well-understood failures, provided that it includes appropriate safeguards and verification.

### 5.6 MTTD versus MTTR

| Metric | Measures | Main question |
|---|---|---|
| MTTD | Time from incident start to detection | How quickly do we notice a problem? |
| MTTR | Time from incident start to recovery, under this guide's definition | How long does recovery take from the start of the incident? |

Consider an incident that begins at 10:00, is detected at 10:05, and is recovered at 10:30.

- MTTD = 5 minutes.
- MTTR = 30 minutes.
- Time from detection to recovery = 25 minutes.

These are different measurements. The recovery time here includes the initial five-minute detection delay.

When reporting operational metrics, teams should define their timestamps consistently to avoid mixing time-to-detect, time-to-acknowledge, time-to-mitigate, and time-to-recover.

### 5.7 MTTD and MTTR are not enough

Averages can hide unusually long incidents. They can also change when the mix of incidents changes.

For a more complete picture, teams may examine:

- Median detection and recovery times.
- High-percentile recovery durations.
- Incident frequency by severity.
- Total user-impact duration.
- Percentage of incidents detected automatically.
- SLO violations and error-budget consumption.
- Recurring incident categories.

The aim is not simply to optimize a number. It is to understand and reduce the impact of service failures.

**Key takeaway:** MTTD measures detection delay, while MTTR measures recovery duration under a clearly defined metric. Both help identify opportunities to improve operational reliability.

---

## 6. Root Cause Analysis (RCA)

### 6.1 What is Root Cause Analysis?

**Root Cause Analysis (RCA)** is an investigation process used to understand why an incident occurred, how it developed, and which technical or organizational conditions contributed to it.

RCA aims to move beyond the immediate symptom and identify improvements that reduce the likelihood or impact of similar failures.

In complex production systems, an incident may not have one simple root cause. It may result from several contributing factors interacting with one another.

For that reason, it is often more useful to investigate causal chains and system weaknesses than to search for one individual or one line of code to blame.

### 6.2 Symptom versus cause

Consider a service that repeatedly crashes because it runs out of memory.

| Observation | Meaning |
|---|---|
| The application is unavailable | User-visible symptom |
| The process is terminated after exceeding its memory limit | Immediate failure mechanism |
| A memory leak causes memory consumption to grow over time | Technical contributing cause |
| Load tests do not cover long-running workloads | Possible testing gap |
| Monitoring does not alert on sustained memory growth | Possible detection gap |

Restarting the application may restore service temporarily, but it does not necessarily address the memory leak or the conditions that allowed it to reach production.

A thorough investigation considers both the direct technical mechanism and the surrounding conditions.

### 6.3 Goals of RCA

An RCA should help answer:

- What happened?
- When did it happen?
- Which users and services were affected?
- How was the incident detected?
- What changed before the incident?
- Which technical conditions caused or contributed to the failure?
- Why did existing safeguards fail to prevent, detect, or contain it?
- How was service restored?
- What changes will reduce the risk of recurrence?

The investigation should be evidence-based. Hypotheses should be distinguished from confirmed findings.

### 6.4 A practical RCA workflow

**Step 1: Establish the timeline**

Collect important timestamps, including the first observed symptoms, alerts, deployments, mitigation actions, and recovery.

**Step 2: Define the impact**

Identify affected services, endpoints, users, requests, or data.

**Step 3: Gather evidence**

Useful evidence may include:

- Metrics and dashboards.
- Application and infrastructure logs.
- Distributed traces.
- Deployment and configuration history.
- Kubernetes events.
- Database metrics.
- Network and dependency health.
- Relevant code changes.

**Step 4: Form hypotheses**

Use the evidence to identify plausible explanations.

For example, a sudden rise in HTTP 5xx responses immediately following a deployment may suggest a release-related issue, but the timing alone does not prove causation.

**Step 5: Test the hypotheses**

Compare the service before and after the event, inspect errors, check dependencies, and examine whether a rollback or controlled reproduction supports the suspected explanation.

**Step 6: Identify contributing factors**

Determine which technical weaknesses, missing safeguards, or operational conditions allowed the incident to occur or increased its impact.

**Step 7: Define corrective actions**

Choose improvements that are specific, actionable, and proportionate to the risk.

**Step 8: Verify the improvements**

Use testing, monitoring, exercises, or follow-up reviews to determine whether the corrective actions are effective.

### 6.5 The Five Whys technique

The **Five Whys** technique investigates a problem by repeatedly asking why an event occurred.

The name suggests asking "why" several times, but five is not a strict requirement. The purpose is to uncover useful causal relationships without stopping at a superficial explanation.

#### Example: A production API becomes unavailable

**Problem:** The API returns a large number of HTTP 503 responses.

1. **Why did the API return 503 responses?**  
   The application could not obtain database connections.

2. **Why were database connections unavailable?**  
   The database connection pool was exhausted.

3. **Why was the pool exhausted?**  
   Requests held connections longer than expected during a workload increase.

4. **Why did the system not handle the increase safely?**  
   Connection acquisition, request concurrency, and overload behavior were not adequately bounded.

5. **Why were these conditions not identified earlier?**  
   Load testing and monitoring did not adequately cover connection-pool saturation under representative traffic.

Possible corrective actions include improving query performance, bounding request concurrency, adding connection-pool monitoring, testing under realistic load, and defining safe overload behavior.

This example illustrates a plausible investigation, not a universal explanation for HTTP 503 responses. Actual conclusions must be supported by evidence.

### 6.6 Fishbone diagrams

A **fishbone diagram**, also called an Ishikawa diagram, helps organize possible causes into categories.

For an application outage, categories might include:

- **Application:** Bugs, memory leaks, or incorrect error handling.
- **Infrastructure:** Resource exhaustion or unavailable nodes.
- **Network:** DNS failures, routing problems, or connection timeouts.
- **Dependencies:** Database saturation or external API failures.
- **Deployment:** Defective releases or incorrect configuration.
- **Process and tooling:** Missing alerts, inadequate tests, or unclear runbooks.

The diagram helps teams explore different explanations rather than prematurely committing to one hypothesis.

It does not prove that any listed cause contributed to the incident. Evidence is still required.

### 6.7 Common RCA mistakes

**Stopping at the symptom**

A restart restored the service, but the reason it failed remains unexplained.

**Assuming correlation proves causation**

A deployment happened shortly before an incident, but other changes or dependencies may also be responsible.

**Blaming an individual**

Statements such as "the engineer made a mistake" do not explain why the system permitted the mistake to produce the observed impact.

**Collecting evidence without producing improvements**

An investigation that identifies important weaknesses but results in no corrective action has limited long-term value.

**Treating the first hypothesis as fact**

Investigators should update their understanding when new evidence contradicts the initial explanation.

**Key takeaway:** RCA should explain how the failure occurred and identify changes that improve prevention, detection, containment, or recovery.

---

## 7. Blameless Postmortems

### 7.1 What is a postmortem?

A **postmortem** is a structured review conducted after an incident to document what happened, assess its impact, understand contributing factors, and identify improvements.

In SRE, a **blameless postmortem** focuses on learning and improving systems rather than assigning personal blame.

Blamelessness does not mean ignoring mistakes or avoiding accountability. It means investigating why an action made sense in its context, what information and safeguards were available, and how the system can be made safer.

### 7.2 Why blamelessness matters

Production incidents often arise from interactions between software, infrastructure, tooling, processes, and human decisions.

If engineers expect to be punished for reporting mistakes or describing uncertainty, they may hesitate to share important details. That makes it harder for the organization to understand the actual conditions that led to failure.

A blameless approach encourages engineers to:

- Report problems honestly.
- Share incomplete or contradictory evidence.
- Explain decisions made under time pressure.
- Identify gaps in testing, monitoring, and documentation.
- Propose improvements without fear of personal humiliation.
- Learn from near misses as well as major incidents.

The objective is to improve the system and its operating environment.

### 7.3 What should a postmortem contain?

A useful postmortem typically includes the following sections.

**1. Incident summary**

A concise explanation of the incident and its outcome.

**2. Impact**

Describe the affected services, user workflows, duration, and measurable consequences.

**3. Severity**

Record the assigned severity and explain the impact that justified it.

**4. Timeline**

List the major events in chronological order.

**5. Detection**

Explain how the incident was discovered and whether existing monitoring behaved as expected.

**6. Response and recovery**

Describe the investigation, coordination, mitigation, and recovery verification.

**7. Contributing factors**

Document the technical and organizational conditions supported by the evidence.

**8. What went well**

Identify effective alerts, useful tools, successful coordination, and other practices that helped.

**9. What could be improved**

Identify delayed detection, missing safeguards, confusing procedures, or other weaknesses.

**10. Action items**

Record concrete improvements, ownership, priority, and an appropriate completion target.

### 7.4 Example incident timeline

Imagine a deployment introduces a configuration error that prevents an API from connecting to its database.

| Time | Event |
|---|---|
| 10:00 | New application version begins deployment |
| 10:03 | Database connection errors increase |
| 10:05 | Monitoring triggers an alert |
| 10:07 | On-call engineer acknowledges the alert |
| 10:12 | Investigation links the failure to the recent release |
| 10:15 | Rollback begins |
| 10:20 | Successful requests return to acceptable levels |
| 10:25 | Recovery is verified and the incident is stabilized |

These timestamps are illustrative. In a real postmortem, the timeline should use recorded evidence and clearly identify any uncertain timestamps.

The timeline helps the team understand the order of events and where detection, investigation, mitigation, or verification could be improved.

### 7.5 Writing blameless findings

Avoid framing findings around personal fault.

Instead of:

> The engineer deployed an incorrect configuration and caused the outage.

Use a systems-focused explanation:

> The deployment process allowed an invalid database configuration to reach production. Existing validation did not detect the problem before traffic was shifted to the new version.

The second statement provides a more useful basis for corrective action: configuration validation, deployment safeguards, and release verification.

### 7.6 Action items should be actionable

Weak action item:

- Improve monitoring.

Stronger action item:

- Add an alert for sustained database connection failures and verify that it reaches the on-call rotation during a controlled test.

Weak action item:

- Be more careful with deployments.

Stronger action item:

- Add a pre-deployment configuration validation step and a post-deployment check that verifies database connectivity before the rollout continues.

A good action item should identify:

- The problem it addresses.
- The intended improvement.
- The person or team responsible.
- The priority and expected completion date, where appropriate.
- How its completion or effectiveness will be verified.

### 7.7 When should a postmortem be written?

Organizations should define postmortem criteria appropriate to their risk profile.

A formal postmortem is particularly useful for:

- Major production outages.
- Significant SLO violations.
- Data loss or data-integrity incidents.
- Security incidents requiring a coordinated review.
- Repeated incidents with a common failure pattern.
- Near misses that reveal important system weaknesses.

Not every small alert requires a formal document. The review process should be proportional to the impact and learning opportunity.

### 7.8 From postmortem to improvement

A postmortem is not complete simply because a document has been published.

Follow-up actions should be tracked, prioritized, and reviewed. If corrective actions repeatedly remain unfinished, the organization may need to improve ownership, prioritization, or the way reliability work is planned.

**Key takeaway:** A blameless postmortem turns incident experience into documented learning and verifiable improvements.

---

## 8. Incident Documentation and Runbooks

### 8.1 Why incident documentation matters

Incident documentation records the information needed to understand, respond to, and learn from service failures.

During an incident, engineers may be working under pressure, handling incomplete information, and coordinating with multiple teams. Clear documentation reduces confusion and preserves important details.

Useful incident records include:

- Incident ID and summary.
- Start, detection, acknowledgment, mitigation, and recovery timestamps.
- Severity and affected services.
- User impact.
- Alerts and relevant telemetry.
- Investigation findings and hypotheses.
- Actions taken and their outcomes.
- Escalation and communication history.
- Recovery verification.
- Follow-up actions and postmortem links.

Documentation should distinguish confirmed facts from assumptions. It should also avoid unnecessary sensitive information, such as secrets or credentials in logs and notes.

### 8.2 What is a runbook?

A **runbook** is a documented procedure that guides an engineer through a known operational task or failure scenario.

Runbooks help engineers perform common checks consistently, especially when the issue occurs outside normal working hours or is handled by someone unfamiliar with the service.

Examples include:

- Diagnosing an API health-check failure.
- Investigating high CPU or memory utilization.
- Checking database connection exhaustion.
- Troubleshooting DNS resolution.
- Recovering a failed background worker.
- Rolling back a problematic deployment.
- Verifying service recovery after an outage.

A runbook should provide a clear path through investigation and recovery, while recognizing when an engineer must stop and escalate.

### 8.3 What should a runbook contain?

**1. Purpose**

Explain the problem the runbook addresses.

**2. Scope and prerequisites**

Identify the relevant service, environment, permissions, tools, and assumptions.

**3. Symptoms and triggers**

Describe the alerts or observations that indicate the procedure may apply.

**4. Initial checks**

Provide safe commands, dashboards, or queries to establish the current state.

**5. Diagnostic steps**

Explain how to distinguish likely failure modes.

**6. Mitigation and recovery**

Describe approved actions, their risks, and the expected outcome.

**7. Verification**

Explain how to confirm that the service has recovered.

**8. Escalation**

State when to involve another team or stop the procedure.

**9. References**

Link to service documentation, dashboards, incident history, and relevant configuration.

**10. Maintenance information**

Record the owner, last review date, and any version-specific assumptions.

### 8.4 Example runbook: Investigating an unhealthy Kubernetes pod

**Scenario:** An alert indicates that an application pod is repeatedly restarting.

**Objective:** Identify the likely failure mode and restore service safely.

#### Step 1: Identify the affected pod

```bash
kubectl get pods -n <namespace>
```

Look for repeated restarts, a `CrashLoopBackOff` state, or other signs of unhealthy execution.

#### Step 2: Inspect pod details

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Review container states, restart counts, events, probe failures, and resource-related termination information.

#### Step 3: Review current logs

```bash
kubectl logs <pod-name> -n <namespace>
```

Look for startup errors, failed dependency connections, configuration problems, or application exceptions.

#### Step 4: Review logs from the previous container instance

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

This can be particularly useful when a container restarts before its logs can be examined.

#### Step 5: Check recent changes and resources

Review recent deployments, configuration updates, resource requests and limits, and relevant cluster events.

If the pod was terminated due to memory pressure, inspect memory usage and termination details. If logs indicate a database connection failure, investigate that dependency as well.

#### Step 6: Choose an appropriate mitigation

Depending on the evidence, the response might involve rolling back a defective release, correcting configuration, restoring a dependency, or adjusting resource allocation.

Do not automatically delete or restart a pod without understanding the likely impact and whether the action will help.

#### Step 7: Verify recovery

```bash
kubectl get pods -n <namespace>
```

Confirm that the workload is healthy, restart behavior has stabilized, and the application is serving requests successfully.

A running pod does not necessarily mean the application is healthy. Validate service behavior through readiness status, relevant metrics, logs, and application-level health checks.

#### Step 8: Escalate and document

If the problem remains unresolved or the impact increases, involve the appropriate service owner or platform team. Record the findings and actions taken.

**Safety note:** The commands above are diagnostic examples. Recovery actions should follow the environment's access controls and approved operational procedures.

### 8.5 Runbooks versus playbooks

The terms overlap and are not used consistently across every organization.

A useful distinction is:

| Document | Typical purpose |
|---|---|
| Runbook | Step-by-step instructions for a known operational task or failure |
| Incident response playbook | Guidance for coordinating a broader incident scenario, including roles, communications, and escalation |
| Postmortem | Retrospective record of what happened, why it happened, and what will improve |

A runbook might explain how to diagnose a failed database connection, while an incident response playbook describes how the organization coordinates a widespread database outage.

### 8.6 Keeping documentation useful

Documentation becomes unreliable when commands, dashboard links, ownership details, or configuration assumptions become outdated.

To keep it useful:

- Review runbooks after relevant incidents.
- Validate instructions in a safe environment.
- Assign ownership.
- Remove obsolete steps.
- Make escalation paths easy to find.
- Link runbooks directly from relevant alerts where possible.
- Automate repeatable diagnostic steps when appropriate.

A runbook that has never been tested may contain steps that no longer work.

**Key takeaway:** Incident records preserve what happened; runbooks help responders perform operational tasks consistently and safely.

---

## 9. Learning from Recurring Incidents

### 9.1 What are recurring incidents?

Recurring incidents are failures that happen repeatedly, often because the same underlying weakness remains unresolved or a mitigation addresses only the symptoms.

Examples include:

- An API repeatedly becoming unavailable because of connection-pool exhaustion.
- Kubernetes workloads repeatedly being terminated due to memory pressure.
- Deployments repeatedly failing because of configuration errors.
- DNS failures recurring because of incorrect records or configuration management.
- Background queues repeatedly becoming overloaded.
- Alerts repeatedly firing because of the same dependency failure.

Repeated incidents consume engineering time, affect users, and may indicate that a service has not received the reliability improvements it needs.

### 9.2 Why do incidents recur?

Common contributing factors include:

**Temporary fixes without permanent remediation**

Restarting a service restores it, but the underlying defect remains.

**Incomplete investigation**

The team identifies an immediate failure mechanism but does not investigate why the system allowed it to happen.

**Unfinished action items**

Corrective actions are documented but never implemented or verified.

**Insufficient testing**

Tests do not cover the relevant workload, failure mode, configuration, or dependency behavior.

**Missing detection**

Monitoring fails to identify the conditions that precede the incident.

**Weak operational documentation**

Responders repeat the same investigation because findings and recovery procedures are not documented.

**Shared dependencies**

Several services experience the same problem because they depend on one overloaded or unreliable component.

### 9.3 Track incidents by category

Incident categorization helps teams recognize patterns across individual failures.

Useful categories may include:

- Deployment and configuration.
- Application defects.
- Resource exhaustion.
- Database and storage.
- Network and DNS.
- External dependencies.
- Capacity and traffic spikes.
- Security and access control.
- Monitoring and alerting.
- Human or process-related conditions.

Categories should be specific enough to reveal useful patterns without becoming so detailed that every incident belongs to a unique category.

### 9.4 Example: Recurring memory-related incidents

Imagine a Kubernetes application experiences three incidents in one month.

| Incident | Immediate symptom | Response |
|---|---|---|
| Incident A | Application pod restarts | Pod is restarted and service recovers |
| Incident B | Pod is terminated after memory usage increases | Workload is restarted |
| Incident C | Multiple replicas become unhealthy under load | Traffic is reduced and replicas recover |

The individual mitigations may restore service, but the repeated pattern suggests that the team should investigate further.

Potential investigation areas include:

- Memory consumption over time.
- Recent application changes.
- Workload size and traffic patterns.
- Container memory limits.
- Garbage collection or memory-management behavior.
- Whether requests and limits match observed workload requirements.
- Whether alerts identify sustained memory growth before failure.

Possible long-term improvements include fixing a memory leak, improving workload testing, introducing appropriate memory monitoring, and validating resource settings under representative traffic.

The correct fix depends on the evidence. Increasing memory limits alone may delay the next incident without addressing the underlying problem.

### 9.5 Create an incident trend review

A recurring-incident review can examine:

1. Which incident categories appear most frequently?
2. Which incidents cause the greatest user impact?
3. Which failures repeatedly consume on-call time?
4. Which incidents have similar contributing factors?
5. Which postmortem actions remain incomplete?
6. Which engineering changes could prevent multiple incident types?
7. Are detection and recovery times improving?
8. Are the implemented fixes reducing recurrence?

The goal is to prioritize improvements according to risk and expected benefit, not simply to count incidents.

### 9.6 Prioritize corrective actions

Not every incident needs the same amount of follow-up work.

Prioritization can consider:

- Severity and user impact.
- Frequency of recurrence.
- Likelihood of another failure.
- Potential data or security consequences.
- Engineering effort required.
- Whether the fix improves several services.
- Whether a temporary workaround remains necessary.

For example, eliminating a shared database bottleneck may prevent incidents across several services, while fixing a minor issue in one internal report may have a more limited benefit.

### 9.7 Automate repeated operational work

If engineers repeatedly perform the same safe diagnostic steps, automation may improve consistency and reduce response time.

Examples include:

- Collecting diagnostic information when an alert fires.
- Checking service health after a deployment.
- Validating configuration before release.
- Detecting abnormal resource consumption.
- Running approved recovery procedures under defined conditions.

Automation should be tested and monitored. Uncontrolled automated remediation can make a problem worse if it repeatedly restarts healthy workloads, increases resource pressure, or hides evidence needed for investigation.

### 9.8 Measure whether improvements work

After implementing a corrective action, examine the results.

Possible indicators include:

- Fewer incidents in the affected category.
- Reduced recurrence of the same failure mode.
- Lower user-impact duration.
- Improved detection and recovery times.
- Fewer repeated manual interventions.
- Better compliance with service-level objectives.
- Successful completion of recovery exercises.

A completed ticket does not necessarily mean the problem has been solved. Follow-up evidence should demonstrate that the intended improvement occurred.

**Key takeaway:** Recurring incidents are signals that a system, process, or operational safeguard may need improvement. The long-term goal is to reduce repeated failures rather than repeatedly restore the same broken condition.

---

## 10. Putting the Ideas Together

Consider an online shopping platform where the checkout API begins returning HTTP 503 responses after a deployment.

The platform uses Kubernetes to run the application, Prometheus to collect metrics, Grafana to visualize service health, and an on-call alerting process to notify engineers.

### Stage 1: Detection

Prometheus records an increase in failed requests. An alert fires when the error rate exceeds the team's configured threshold.

The on-call engineer acknowledges the alert and opens the relevant dashboard.

**Related concepts:** Monitoring, actionable alerts, MTTD.

### Stage 2: Triage

The engineer confirms that checkout requests are failing and that the incident affects a critical user workflow.

The incident is classified using the organization's severity policy. Because the impact is significant, additional responders are engaged as appropriate.

**Related concepts:** Severity levels, impact assessment, escalation.

### Stage 3: Investigation

The responders examine:

- Request success and latency metrics.
- Application logs.
- Database connection metrics.
- Kubernetes events and pod status.
- Recent deployment history.
- Changes to application configuration.

The evidence suggests that the latest release introduced a database configuration error.

**Related concepts:** Observability, evidence-based troubleshooting, RCA.

### Stage 4: Mitigation

The technical lead recommends rolling back the deployment.

The team verifies that the rollback is safe, performs the rollback, and monitors the service.

If the error rate returns to acceptable levels and critical checkout requests succeed, the team can confirm that service recovery has occurred.

**Related concepts:** Incident response, safe mitigation, MTTR.

### Stage 5: Communication and documentation

The incident commander, where assigned, coordinates the response and keeps stakeholders informed.

The scribe records the timeline, impact, investigation findings, decisions, and recovery verification.

**Related concepts:** Incident coordination, communication, documentation.

### Stage 6: Root Cause Analysis

After recovery, the team investigates why the configuration error reached production.

The investigation identifies that configuration validation did not verify database connectivity before the rollout continued.

The team also reviews whether the deployment's health checks adequately represented the checkout service's readiness.

**Related concepts:** RCA, contributing factors, release safety.

### Stage 7: Blameless postmortem

The team documents what happened, how the service was affected, which safeguards failed, and what helped recovery.

The findings focus on improving configuration validation and deployment controls rather than assigning personal blame.

**Related concepts:** Blameless postmortems, organizational learning.

### Stage 8: Corrective actions

The team creates follow-up work to:

- Validate required database configuration before deployment.
- Add a deployment verification check for database connectivity.
- Review readiness checks and rollout behavior.
- Add or improve alerts for the relevant failure signals.
- Update the deployment runbook.
- Test the new safeguards before relying on them in production.

Each action has an owner and a way to verify its effectiveness.

**Related concepts:** Runbooks, automation, recurring-incident prevention.

### Stage 9: Review the outcome

The team checks whether the improvements reduce similar failures and whether future incidents are detected and recovered from more effectively.

MTTD and MTTR provide useful operational measurements, while incident recurrence and SLO performance help assess the broader reliability outcome.

This example demonstrates how incident management connects detection, response, recovery, investigation, documentation, and continuous improvement.

---

## 11. Key Takeaways

- Incident management coordinates the detection, response, mitigation, and recovery of service disruptions.
- Detection identifies abnormal service behavior; triage assesses impact and urgency.
- Incident severity should reflect user impact, service criticality, and the risk of further damage.
- Escalation brings in the expertise and authority required to manage an incident effectively.
- MTTD measures the average delay between an incident beginning and its detection.
- MTTR measures average recovery duration under an explicitly defined metric.
- Recovery should be verified using service behavior, not assumed from a successful restart or deployment.
- Root Cause Analysis investigates the technical and organizational factors that contributed to a failure.
- Blameless postmortems help teams learn from incidents without defaulting to personal blame.
- Runbooks provide repeatable operational guidance, while incident records preserve evidence and decisions.
- Recurring incidents should lead to investigation and lasting improvements rather than repeated temporary fixes.
- Corrective actions need ownership, appropriate priority, and verification.
- The ultimate objective is not merely to resolve incidents faster, but to reduce user impact and improve system reliability over time.

---

## 12. Learning Resources

These resources provide starting points for understanding incident management, incident response, postmortems, and operational reliability.

### Primary references

**1. Google: Site Reliability Engineering: Being On-Call**

- Focus: On-call responsibilities, incident response, and operational practices.
- Link: https://sre.google/sre-book/being-on-call/

**2. Google: Site Reliability Engineering: Managing Incidents**

- Focus: Incident response, coordination, and effective incident management.
- Link: https://sre.google/sre-book/managing-incidents/

**3. Google: Site Reliability Engineering: Postmortem Culture: Learning from Failure**

- Focus: Blameless postmortems, incident learning, and organizational improvement.
- Link: https://sre.google/sre-book/postmortem-culture/

**4. Google: The Site Reliability Workbook**

- Focus: Practical implementation of SRE principles and operational practices.
- Link: https://sre.google/workbook/table-of-contents/

### Incident response and operations

**5. PagerDuty: Incident Response Documentation**

- Focus: Incident response processes, roles, coordination, and operational workflows.
- Link: https://response.pagerduty.com/

**6. Atlassian: Incident Management**

- Focus: Incident response practices, coordination, and incident management workflows.
- Link: https://www.atlassian.com/incident-management

### Monitoring and troubleshooting

**7. Prometheus Documentation**

- Focus: Metrics collection, alerting rules, and monitoring service behavior.
- Link: https://prometheus.io/docs/introduction/overview/

**8. Kubernetes Documentation**

- Focus: Diagnosing application and cluster issues, pod failures, resource management, and workload health.
- Link: https://kubernetes.io/docs/home/

### Further reading

**9. Google SRE Resources**

- Focus: Additional material on reliability engineering, operational practices, and incident learning.
- Link: https://sre.google/

**10. Cloudflare Learning Center**

- Focus: Networking, DNS, HTTP, and infrastructure concepts useful for investigating service incidents.
- Link: https://www.cloudflare.com/learning/


