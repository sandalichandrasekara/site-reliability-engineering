# 02. Service Reliability

A practical study guide to measuring service health, setting reliability targets, understanding error budgets, and interpreting the signals that show how a service behaves for its users.

> **Learning goal:** Be able to define meaningful reliability measurements, explain the difference between SLIs, SLOs, and SLAs, and use service telemetry to make informed engineering decisions.

## Contents

- [1. Understanding service reliability](#1-understanding-service-reliability)
- [2. Service Level Indicators (SLIs)](#2-service-level-indicators-slis)
- [3. Service Level Objectives (SLOs)](#3-service-level-objectives-slos)
- [4. Service Level Agreements (SLAs)](#4-service-level-agreements-slas)
- [5. Error budgets and release decisions](#5-error-budgets-and-release-decisions)
- [6. Availability, reliability, and durability](#6-availability-reliability-and-durability)
- [7. Latency percentiles: p50, p95, and p99](#7-latency-percentiles-p50-p95-and-p99)
- [8. The four golden signals](#8-the-four-golden-signals)
- [9. Putting the concepts together](#9-putting-the-concepts-together)
- [10. Practical exercises](#10-practical-exercises)
- [11. Learning resources](#11-learning-resources)
- [12. Questions to check my understanding](#12-questions-to-check-my-understanding)

---

## 1. Understanding service reliability

**Service reliability** describes how consistently a service delivers the behavior users expect over time and under stated conditions.

A service can be running while still being unreliable. For example, its process may be healthy, but requests could be slow, return errors, or fail because a downstream database is unavailable.

To understand reliability, measure behavior that matters to users, set clear targets, and monitor whether those targets are being met.

Useful questions include:

- Can users successfully complete their intended actions?
- How long do requests take?
- Are failures affecting all users or only a subset?
- Does the service remain responsive as traffic increases?
- How quickly can the service recover from a failure?
- Are reliability improvements worth their cost and complexity?

Service reliability measurements help turn broad goals such as "keep the API stable" into explicit, measurable expectations.

---

## 2. Service Level Indicators (SLIs)

A **Service Level Indicator (SLI)** is a quantitative measurement of some aspect of a service's behavior.

An SLI should represent a service characteristic that matters to users, such as successful requests, request latency, or the correctness of a result.

### Common SLIs

| SLI | What it measures | Example |
|---|---|---|
| Availability / success rate | The proportion of eligible requests that succeed | 99.95% of eligible requests succeeded |
| Latency | How long requests take | 99% of requests complete within 300 ms |
| Error rate | The proportion of requests that fail | 0.2% of eligible requests return server errors |
| Throughput | The amount of work handled over time | 1,200 requests per second |
| Freshness | How up-to-date data is | 99% of updates appear within 5 seconds |
| Durability | Whether stored data remains intact and retrievable | Successful verification of stored data and recovery procedures |

Not every metric is a good user-facing SLI. CPU usage, memory usage, and queue depth can be useful diagnostic indicators, but they do not always directly tell you whether a user received a successful, timely result.

### How to define an SLI

A useful request-based SLI normally needs a clear definition of:

1. **Eligible events:** Which requests or operations are included?
2. **Good events:** What counts as a successful outcome?
3. **Measurement source:** Where does the data come from?
4. **Measurement window:** Over what period is it calculated?

For example:

- Eligible events: all valid requests to the checkout API, excluding explicitly documented cases such as client-side validation errors.
- Good events: requests that return the expected successful result within 500 ms.
- Measurement source: application telemetry or a trusted request metric.
- Measurement window: rolling 30 days.

The exclusions must be chosen carefully. Excluding too many failure cases can make an SLI look healthy even when users are having a poor experience.

### Request-based success-rate example

Suppose an API receives 100,000 eligible requests in a measurement window, and 99,700 meet the defined success criteria.

\[
\text{Success rate} = \frac{99{,}700}{100{,}000}\times100 = 99.7\%
\]

The SLI is 99.7% for that window.

A latency SLI may instead count requests completed within a threshold:

\[
\text{Good-event ratio} =
\frac{\text{Requests completed within the threshold}}
{\text{Eligible requests}}
\]

**Key point:** An SLI is the measurement. It is not the target.

---

## 3. Service Level Objectives (SLOs)

A **Service Level Objective (SLO)** is a target value or range for an SLI over a defined measurement window.

An SLO tells a team what level of service it aims to provide. It helps engineers assess service health, prioritize reliability work, and make informed decisions about change.

### Example SLOs

- At least 99.9% of eligible API requests succeed over a rolling 30-day window.
- At least 99% of eligible requests complete within 300 ms over a rolling 30-day window.
- 99.5% of accepted background jobs complete within five minutes.

These are illustrative targets, not universal standards. A suitable objective depends on user expectations, business requirements, technical constraints, and the cost of achieving it.

### What makes a useful SLO?

A good SLO should be:

- **User-relevant:** It measures an outcome that matters.
- **Measurable:** The SLI can be calculated consistently.
- **Clearly scoped:** Eligible requests, endpoints, and exclusions are defined.
- **Time-bound:** The measurement window is specified.
- **Realistic:** The target is based on evidence and service requirements.
- **Actionable:** The team knows what to do when performance is outside the target.

### SLI vs SLO

| Concept | Meaning | Example |
|---|---|---|
| SLI | What is measured | 99.7% of eligible requests succeeded |
| SLO | The target for that measurement | At least 99.9% should succeed over 30 days |

If the SLI is below the SLO, the service is not meeting its objective for that window.

### Why not aim for 100%?

A 100% target can be extremely expensive or practically impossible to guarantee. Systems depend on components and networks that can fail, and extra redundancy can introduce cost and complexity.

The appropriate target is the one that meets user needs and business expectations without wasting resources on reliability that provides little additional value.

An SLO is not an excuse to accept poor reliability. It is a way to make reliability expectations explicit and engineering trade-offs deliberate.

---

## 4. Service Level Agreements (SLAs)

A **Service Level Agreement (SLA)** is an agreement between a service provider and a customer that defines service commitments and may specify consequences when those commitments are not met.

An SLA may cover:

- A promised level of availability or performance.
- How the service is measured.
- The measurement period and exclusions.
- Customer and provider responsibilities.
- Support response commitments.
- Service credits or other remedies if the commitment is breached.

The precise terms depend on the contract.

### SLI vs SLO vs SLA

| Term | Core question | Typical audience |
|---|---|---|
| SLI | What does the service actually do? | Engineers and service owners |
| SLO | What level of service are we aiming for? | Engineering, product, and operations |
| SLA | What service commitment have we agreed with the customer? | Provider and customer |

For example, a team may set an internal availability SLO of 99.95%, while its customer contract promises 99.9% availability with defined service credits if the SLA terms are breached. The figures are illustrative; the relationship and exact definitions depend on the service.

### Important distinctions

- An SLO is an objective, not automatically a contractual promise.
- An SLA is not necessarily just an availability percentage; it can include other commitments.
- A service can miss an internal SLO without breaching its SLA.
- A service can breach an SLA even if some internal dashboards look healthy, if the contract defines measurement differently.

Always read the definitions, time windows, exclusions, and calculation methods rather than comparing percentages alone.

---

## 5. Error budgets and release decisions

An **error budget** is the amount of unreliability permitted by an SLO during its measurement window.

It gives teams a way to reason about the trade-off between shipping changes and maintaining reliability.

### Calculate an error budget

For a simple availability or request-success SLO:

\[
\text{Error budget fraction} = 1 - \text{SLO fraction}
\]

If the SLO is 99.9%:

\[
1 - 0.999 = 0.001 = 0.1\%
\]

For a request-based SLI, this means 0.1% of eligible requests may be bad before the SLO is missed. For a time-based availability SLI, the corresponding interpretation is based on unavailable time. The SLI's definition determines what the budget actually represents.

### Example: request-based budget

Suppose a service has:
- 1,000,000 eligible requests in a 30-day window.
- An SLO requiring 99.9% of eligible requests to be successful.

The permitted number of bad requests is:

\[
1{,}000{,}000 \times 0.001 = 1{,}000
\]

The budget is 1,000 bad requests for that measurement window, assuming every request has equal weight and the SLI is defined as a simple request ratio.

If 700 requests have been bad, 300 bad requests remain before the target is missed for the window.

### Example: time-based availability budget

For an illustrative 30-day window and 99.9% availability target:

\[
30 \times 24 \times 60 \times 0.001 = 43.2\text{ minutes}
\]

That is approximately 43.2 minutes of unavailability in a 30-day period under a simple time-based definition. Actual calculations depend on the SLI, window, and contract or policy definitions.

### How error budgets inform releases

An error budget helps the team evaluate whether its reliability target is being met and how much risk it can reasonably take.

A possible policy might be:

- **Budget consumption is low:** Continue normal delivery, while still using appropriate testing and deployment safeguards.
- **Budget is being consumed unusually quickly:** Investigate the causes and consider safer rollout practices.
- **Budget is exhausted:** Follow the team's agreed policy, which may prioritize reliability work and restrict higher-risk changes until reliability improves.

This is a policy decision, not an automatic universal rule. The response should be defined in advance and proportionate to service criticality.

### Why error budgets are useful

They provide a shared, evidence-based way for product and engineering teams to discuss delivery speed and reliability. Rather than arguing that a service should always be perfectly reliable or that releases should never be delayed, teams can make decisions based on objectives and observed risk.

---

## 6. Availability, reliability, and durability

These terms are related but not interchangeable.

### Availability

**Availability** describes whether a service is usable or able to provide its intended function when required.

A service may be considered available based on a defined health check or, more usefully, on whether it can successfully handle eligible user requests.

A simple time-based formula is:

\[
\text{Availability} =
\frac{\text{Agreed available time}}
{\text{Total agreed time}}\times100
\]

For example, if a service is available for 29 days and 23 hours out of a 30-day window, availability is calculated using the agreed time-based definition. Planned maintenance, partial outages, and measurement rules may change what counts.

### Reliability

**Reliability** is the ability of a system to perform its intended function correctly and consistently over time and under stated conditions.

Availability is one aspect of reliability, but a service can be available and still return incorrect results, lose work, or respond too slowly.

Examples of reliability concerns:
- Requests return correct results.
- Background jobs are not silently dropped.
- Dependencies fail without causing widespread cascading failures.
- The service recovers predictably after faults.

### Durability

**Durability** is the likelihood that stored data remains intact and is not lost or corrupted over the relevant period.

Durability is especially important for databases, object storage, backups, and message queues.

Examples of durability practices:
- Replication and appropriate storage redundancy.
- Backups with tested restore procedures.
- Integrity checks and corruption detection.
- Retention policies and recovery planning.

Replication alone is not a complete backup strategy: accidental deletion or corruption can be replicated too.

### Comparison

| Property | Main question | Example failure |
|---|---|---|
| Availability | Can the user access and use the service? | API returns connection errors |
| Reliability | Does the system consistently perform its intended function correctly? | API returns incorrect results or loses a job |
| Durability | Is stored data preserved without loss or corruption? | A confirmed record is permanently lost |

### High availability is not the same as zero downtime

Redundancy, health checks, load balancing, and failover can reduce downtime, but they cannot guarantee that no disruption will occur. Failover mechanisms also need testing; an untested backup or standby system may not work when needed.

---

## 7. Latency percentiles: p50, p95, and p99

**Latency** is the time taken to complete an operation, such as an HTTP request.

Looking only at average latency can hide slow experiences for a subset of users. Percentiles help show how response times are distributed.

### What the percentiles mean

- **p50 (50th percentile):** Half of measured requests completed at or below this latency. This is the median.
- **p95 (95th percentile):** 95% of measured requests completed at or below this latency; the slowest 5% took longer.
- **p99 (99th percentile):** 99% of measured requests completed at or below this latency; the slowest 1% took longer.

These are percentile thresholds, not percentages of requests that failed.

### Example

Suppose the observed request latencies are:

| Metric | Latency |
|---|---:|
| p50 | 80 ms |
| p95 | 250 ms |
| p99 | 900 ms |

Interpretation:
- A typical request completes in about 80 ms or less.
- 95% of requests complete in 250 ms or less.
- 1% of requests take longer than 900 ms.

The gap between p50 and p99 suggests a slow tail affecting a subset of requests. Investigation may reveal a slow database query, resource contention, queueing, garbage collection, or a slow downstream dependency.

### When should each percentile be used?

| Percentile | Useful for | Limitation |
|---|---|---|
| p50 | Understanding typical request behavior | Can hide problems affecting a minority of users |
| p95 | Tracking user experience beyond the median and identifying common tail latency | Does not show the most extreme tail |
| p99 | Investigating high-tail latency, especially for important or high-volume services | Can be noisy when traffic volume is low |

No single percentile tells the whole story. Select percentiles based on the service, traffic volume, user expectations, and the decisions you need to make.

### Percentiles need context

- Compare the same endpoint or operation over time.
- Use appropriate histogram buckets or a telemetry system that calculates quantiles correctly.
- Consider traffic volume and aggregation method.
- Look at error rate, throughput, and saturation alongside latency.
- Define latency objectives using the relevant request population.

A percentile calculated separately for each instance cannot always be averaged to obtain the true fleet-wide percentile. Aggregatable histogram data is generally more appropriate for fleet-level percentile calculations.

### Latency vs throughput

Latency measures how long an operation takes. Throughput measures how much work is completed over time.

A service can have high throughput and poor latency if it is overloaded or has a queue of waiting work. Both measurements help describe performance.

---

## 8. The four golden signals

Google's SRE guidance identifies four golden signals for monitoring a user-facing system:

1. Latency
2. Traffic
3. Errors
4. Saturation

Together, they help an engineer understand service behavior and recognize many common failure patterns.

### 8.1 Latency

**Question:** How long does it take to serve a request?

Useful measurements:
- Request latency distributions and percentiles.
- Latency split by endpoint or operation.
- Successful-request latency compared with failed-request latency.

**Example signal:** p95 latency rises from 180 ms to 1.4 seconds.

**Possible causes:** A slow dependency, overloaded database, increased queueing, resource contention, or a recent change.

**Investigation:** Compare latency by endpoint, inspect traces and dependency timings, check recent deployments, and correlate with resource metrics.

Do not treat all latency as equivalent. Fast failures and slow successful requests may have very different causes.

### 8.2 Traffic

**Question:** How much demand is the system handling?

Useful measurements:
- Requests per second.
- Active users or concurrent requests.
- Jobs submitted per minute.
- Queue arrival rate and workload size.

**Example signal:** Incoming traffic doubles while the number of healthy application instances remains unchanged.

**Possible causes:** A traffic spike, retry storm, unexpected client behavior, or a legitimate increase in demand.

**Investigation:** Compare traffic with historical patterns, check request sources and endpoint distribution, and verify whether the service has sufficient capacity.

Traffic is a measure of demand, not a measure of success. More requests do not necessarily mean healthier service.

### 8.3 Errors

**Question:** How often do requests fail?

Useful measurements:
- Failed requests as a ratio of eligible requests.
- HTTP 5xx rates.
- Application exceptions.
- Failed jobs or dependency calls.
- Business-operation failures where they can be measured reliably.

**Example signal:** The API's 5xx rate increases from 0.1% to 8%.

**Possible causes:** A bad deployment, unavailable dependency, exhausted connection pool, resource exhaustion, or invalid configuration.

**Investigation:** Check the timing of the increase, group errors by type, inspect logs and traces, and compare affected and unaffected endpoints.

Not every 4xx response indicates a service failure; client errors should be interpreted according to the SLI definition and the service's expected behavior.

### 8.4 Saturation

**Question:** How close is a constrained resource to its limit?

Useful measurements:
- CPU utilization and throttling.
- Memory utilization and OOMKills.
- Disk capacity and I/O latency.
- Database connection-pool usage.
- Queue depth and time spent waiting.
- Thread-pool or worker utilization.

**Example signal:** Request latency increases while the database connection pool remains fully occupied.

**Possible causes:** Insufficient capacity, slow queries, leaked connections, excessive concurrency, or a downstream dependency that is not responding.

**Investigation:** Identify the constrained resource, establish why it is saturated, and decide whether to reduce demand, improve efficiency, or increase capacity.

High utilization alone does not always mean there is a problem. The key is whether the resource is constrained and affecting service behavior.

### How the signals work together

| Observed signals | Possible interpretation | Investigation direction |
|---|---|---|
| Latency rises, saturation rises | The service may be overloaded | Check CPU, memory, pools, queues, and capacity |
| Errors rise immediately after a deployment | The change may have introduced a problem | Review release changes, logs, and rollback options |
| Traffic rises, then latency and errors rise | Demand may exceed available capacity | Check scaling, rate limits, and dependency capacity |
| Latency rises but CPU is low | The service may be waiting on another component | Inspect database, network, queues, and traces |
| Queue depth rises while workers are busy | Processing capacity may be insufficient | Compare arrival rate, processing rate, and worker health |

These patterns are clues, not proof. Confirm hypotheses with evidence before making disruptive changes.

---

## 9. Putting the concepts together

Imagine an online ordering API with the following illustrative objectives:

- Success-rate SLO: at least 99.9% of eligible requests succeed over a rolling 30-day window.
- Latency SLO: at least 95% of eligible successful requests complete within 300 ms over the same window.

During a deployment, the monitoring system reports:
- Traffic is stable.
- p95 latency increases to 1.2 seconds.
- The error rate rises sharply.
- Database connection usage reaches its configured limit.

A reliability-focused response might be:

1. **Validate the signals:** Confirm the latency and error alerts using dashboards and request telemetry.
2. **Assess user impact:** Identify affected operations and estimate how much of the SLO budget is being consumed.
3. **Investigate the change:** Check the deployment, database queries, connection-pool configuration, and dependency timings.
4. **Mitigate safely:** Roll back or apply another appropriate mitigation if evidence points to the change and rollback is safe.
5. **Verify recovery:** Confirm that latency, errors, and resource pressure return to acceptable levels.
6. **Learn from the incident:** Identify contributing factors and agree on corrective actions.
7. **Improve prevention:** Consider deployment safeguards, database optimization, better alerts, or capacity changes.

This example connects SLI measurements, SLO targets, error-budget decisions, latency percentiles, and the four golden signals.

---

## 10. Learning resources

These are recommended references to work through and revisit. 

### Primary SRE references

1. **Google SRE Book: Service Level Objectives**
   - https://sre.google/sre-book/service-level-objectives/

2. **Google SRE Workbook**
   - Practical guidance for implementing SRE practices.
   - https://sre.google/workbook/table-of-contents/

3. **Google SRE Book: Monitoring Distributed Systems**
   - https://sre.google/sre-book/monitoring-distributed-systems/

4. **Google SRE Book: Embracing Risk**
   - https://sre.google/sre-book/embracing-risk/

### Observability and metrics

5. **Prometheus documentation**
   - Metrics, querying, and alerting.
   - https://prometheus.io/docs/introduction/overview/

6. **Prometheus: Histograms and summaries**
   - Understand histogram buckets and quantile calculations.
   - https://prometheus.io/docs/practices/histograms/

7. **Grafana documentation**
   - Dashboards and visualizing service behavior.
   - https://grafana.com/docs/

### Cloud reliability frameworks

8. **Google Cloud Architecture Framework: Reliability**
   - https://cloud.google.com/architecture/framework/reliability

9. **Microsoft Azure Well-Architected Framework: Reliability**
   - https://learn.microsoft.com/en-us/azure/well-architected/reliability/

10. **AWS Well-Architected Framework: Reliability Pillar**
    - https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html

