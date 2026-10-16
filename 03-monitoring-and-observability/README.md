# 03. Monitoring and Observability

A practical study guide to understanding system behavior, collecting telemetry, designing useful dashboards and alerts, and troubleshooting services using evidence.

> **Learning goal:** Understand how to use metrics, logs, and traces together to detect issues, investigate their causes, and assess the impact on users.

## Contents

- [1. Monitoring vs observability](#1-monitoring-vs-observability)
- [2. Metrics, logs, and distributed traces](#2-metrics-logs-and-distributed-traces)
- [3. Prometheus and Grafana](#3-prometheus-and-grafana)
- [4. OpenTelemetry and distributed tracing](#4-opentelemetry-and-distributed-tracing)
- [5. Alerting strategies and actionable alerts](#5-alerting-strategies-and-actionable-alerts)
- [6. Dashboards and service health indicators](#6-dashboards-and-service-health-indicators)
- [7. Troubleshooting with telemetry](#7-troubleshooting-with-telemetry)
- [8. Example: investigating a slow API](#8-example-investigating-a-slow-api)
- [9. Practical exercises](#9-practical-exercises)
- [10. Learning resources](#10-learning-resources)
- [11. Questions to check my understanding](#11-questions-to-check-my-understanding)

---

## 1. Monitoring vs observability

Monitoring and observability are closely related, but they are not exactly the same.

### Monitoring

**Monitoring** means collecting and examining signals about a system to determine whether it is healthy and whether known conditions require attention.

Monitoring commonly answers questions such as:

- Is the service responding?
- Has the error rate increased?
- Is CPU or memory usage too high?
- Is latency above its target?
- Has a disk filled up?
- Are requests or jobs accumulating in a queue?

Monitoring often uses predefined metrics, dashboards, health checks, and alert rules. It is especially effective when teams know which failure conditions they need to detect.

### Observability

**Observability** is the ability to understand a system's internal state from the outputs it produces, commonly through metrics, logs, traces, and related context.

It helps engineers investigate questions they may not have anticipated when creating their dashboards and alerts:

- Why is one endpoint slow while other endpoints are healthy?
- Which dependency is contributing most to request latency?
- Why are only requests from one region failing?
- What changed between successful and unsuccessful requests?
- Which service is causing a queue to grow?

Observability depends on useful instrumentation, consistent context, and the ability to query and correlate telemetry.

| Monitoring | Observability |
|---|---|
| Often focuses on known conditions and expected failure modes | Helps investigate unfamiliar or complex behavior |
| Commonly uses dashboards, thresholds, and alerts | Uses telemetry and context to explore causes |
| Helps detect that something may be wrong | Helps investigate why it is wrong |
| Example: alert when API error rate exceeds a threshold | Example: trace a failed request across services to identify the failing dependency |

This is a useful distinction, but not a strict separation: monitoring is part of an observable system, and good monitoring can also help investigate causes.

### Instrumentation matters

A system is not automatically observable just because a monitoring platform has been installed.

Applications and infrastructure must emit meaningful telemetry. Instrumentation should capture relevant measurements and context while avoiding unnecessary sensitive data, excessive cardinality, or excessive cost.

---

## 2. Metrics, logs, and distributed traces

The three common telemetry signals are metrics, logs, and traces. Each provides a different view of system behavior.

### 2.1 Metrics

**Metrics** are numerical measurements collected over time.

Examples:
- Request rate.
- Request duration.
- Error count or error ratio.
- CPU and memory usage.
- Number of active connections.
- Queue depth.
- Number of running or pending Kubernetes pods.

Metrics are useful for:
- Seeing trends and changes over time.
- Building dashboards.
- Alerting on service health.
- Comparing current behavior with a baseline.
- Understanding resource usage and saturation.

A metric usually has a name, value, timestamp, and optional labels or dimensions.

Example metric names might include:

```text
http_requests_total
http_request_duration_seconds
process_resident_memory_bytes
```

These are illustrative names; the actual metric names depend on the application's instrumentation and conventions.

**Important:** Metrics with too many unique label combinations can create high cardinality, increasing storage and query costs. Avoid labels such as raw user IDs, request IDs, or arbitrary full URLs when their values can be effectively unbounded.

### 2.2 Logs

**Logs** are timestamped records of events that occur in an application or system.

Examples:
- A request failed validation.
- A database connection timed out.
- A user-facing operation returned an error.
- A configuration reload failed.
- A service started or stopped.

Structured logs, commonly represented as JSON, are easier to query consistently than unstructured text.

Example:

```json
{
  "timestamp": "2026-10-16T10:20:30Z",
  "level": "ERROR",
  "service": "checkout-api",
  "message": "Database connection timed out",
  "trace_id": "example-trace-id"
}
```

The trace ID above is illustrative, not a real identifier.

Logs are useful when you need details about a particular event. Good logging includes meaningful context while avoiding passwords, access tokens, personal data, and other secrets.

### 2.3 Distributed traces

A **distributed trace** follows an operation as it travels through multiple components or services.

A trace contains one or more **spans**. Each span represents a unit of work and may include:
- Start time and duration.
- Service or component name.
- Operation name.
- Parent-child relationships.
- Attributes and status.
- Links to related events or context.

Imagine a checkout request passing through:

```text
Client
  └── API Gateway
       └── Checkout API
            ├── Inventory Service
            └── Payment Service
                 └── Payment Database
```

A trace can help reveal that the checkout API itself is fast, but the payment service spends most of its time waiting for the database.

Traces are useful for:
- Understanding request paths across services.
- Finding slow dependencies.
- Investigating intermittent failures.
- Identifying where latency accumulates.
- Correlating events across service boundaries.

### How the signals complement one another

| Signal | Best at answering | Example |
|---|---|---|
| Metrics | What is changing, and how widespread is it? | The API's p95 latency increased |
| Logs | What happened in a specific event? | A database query timed out |
| Traces | Where did time or failure occur across a request path? | The database span accounts for most request latency |

A common investigation begins with metrics, narrows down the problem using traces, and uses logs to inspect specific events. The sequence can vary depending on the symptoms and available telemetry.

---

## 3. Prometheus and Grafana

Prometheus and Grafana are widely used in cloud-native monitoring, but they have different primary roles.

### 3.1 Prometheus

**Prometheus** is an open-source monitoring and alerting toolkit that collects and stores time-series metrics and provides a query language called PromQL.

Common uses include:
- Scraping metrics from instrumented applications and exporters.
- Storing time-series data.
- Querying metrics with PromQL.
- Evaluating alerting rules.
- Sending firing alerts to an alert-management system such as Alertmanager.

A simplified pull-based workflow looks like this:

```text
Application / Exporter
        │
        │ Exposes metrics endpoint
        ▼
    Prometheus
        │
        ├── Stores time-series data
        ├── Evaluates recording and alerting rules
        └── Provides queries
                  │
                  ▼
               Grafana
```

Prometheus typically scrapes configured targets at intervals. Applications often expose metrics on an HTTP endpoint such as `/metrics`, while exporters expose metrics for systems that do not natively provide Prometheus-format metrics.

Prometheus does not automatically discover or understand every application. Targets, service discovery, instrumentation, permissions, and network access must be configured appropriately.

### 3.2 PromQL basics

PromQL is used to query and aggregate Prometheus time-series data. Examples below assume the corresponding metrics exist and their labels match the queries.

**Select a metric:**

```promql
up
```

Shows the scrape status of configured targets. A value of `1` means a target was successfully scraped; `0` means the scrape failed. It does not prove that every user-facing function of the service works correctly.

**Calculate a per-second request rate:**

```promql
rate(http_requests_total[5m])
```

Calculates the average per-second increase of a counter over the previous five minutes, using the counter's observed samples.

**Calculate a 5xx error ratio:**

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

This illustrates a 5xx request ratio if `http_requests_total` is a counter with a `status` label. In a real environment, use the actual metric schema and ensure the numerator and denominator represent compatible request populations. Handle an absent or zero denominator appropriately.

**Check CPU usage on Linux instances:**

```promql
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))
```

This estimates non-idle CPU percentage per instance when the `node_cpu_seconds_total` metric is available from the Prometheus Node Exporter.

### 3.3 Grafana

**Grafana** is an observability and visualization platform used to query and display data from sources such as Prometheus, Loki, and tracing backends.

Common uses include:
- Building dashboards.
- Creating panels for metrics and trends.
- Exploring telemetry during an incident.
- Configuring alert rules and notification workflows, depending on the setup.
- Bringing multiple data sources into a shared view.

A typical workflow:
1. Configure Prometheus as a data source in Grafana.
2. Create a dashboard.
3. Add a panel and write a PromQL query.
4. Select an appropriate visualization and time range.
5. Add useful labels, descriptions, and units.
6. Review the dashboard during normal operation and incidents.

Grafana visualizes data; it does not automatically create the instrumentation or guarantee that the underlying metrics are meaningful.

### Prometheus vs Grafana

| Prometheus | Grafana |
|---|---|
| Collects and stores time-series metrics | Visualizes and explores data |
| Provides PromQL queries | Queries configured data sources |
| Evaluates Prometheus alerting rules | Displays dashboards and can manage alerts, depending on configuration |
| Commonly scrapes metrics endpoints | Can visualize data from many backends |

---

## 4. OpenTelemetry and distributed tracing

**OpenTelemetry (OTel)** is an open-source, vendor-neutral set of APIs, SDKs, conventions, and tools for generating, collecting, and exporting telemetry.

It supports traces, metrics, and logs. OpenTelemetry is not itself a monitoring backend. It helps applications and infrastructure produce and transport telemetry so that compatible backends can store, query, and visualize it.

### 4.1 OpenTelemetry components

- **API:** Interfaces used to create or access telemetry.
- **SDK:** Implementation that configures sampling, processing, and exporting.
- **Instrumentation:** Code or libraries that generate telemetry for applications and dependencies.
- **Semantic conventions:** Shared naming conventions for common attributes and operations.
- **OTel Collector:** A service that can receive, process, and export telemetry to one or more destinations.
- **Exporter:** A component that sends telemetry to a supported backend or protocol endpoint.

A simplified architecture:

```text
Application
   │
   ├── Metrics
   ├── Logs
   └── Traces
        │
        ▼
 OpenTelemetry SDK / Instrumentation
        │
        ▼
 OpenTelemetry Collector (optional)
        │
        ├── Metrics backend
        ├── Log backend
        └── Trace backend
```

The Collector is optional; applications can export directly to a supported backend. A Collector is useful when a team needs centralized processing, batching, filtering, routing, or consistent export configuration.

### 4.2 How distributed tracing works

When a request enters an instrumented service, a trace may begin. As the request calls other instrumented services, trace context is propagated so their spans can be connected to the same trace.

A trace commonly includes:
- A trace ID shared by spans in the same trace.
- A span ID for each individual span.
- Parent-child relationships or other links between spans.
- Timing and operation information.
- Attributes that help identify the service, endpoint, or dependency.

Context propagation is important. If trace context is not passed between services, the trace may appear fragmented.

### 4.3 Trace IDs and logs

Adding trace and span identifiers to structured logs makes it easier to move between a trace and the log records associated with it.

For example, a trace may show that a database call took two seconds. The related logs may reveal a timeout, a connection-pool issue, or a query error.

Do not use trace IDs as metric labels: their high cardinality can make metrics expensive and difficult to operate.

### 4.4 Sampling

Tracing every request may be expensive for high-volume services. Sampling determines which traces are recorded or retained.

Common approaches include:
- **Head sampling:** The decision is made near the beginning of a trace.
- **Tail sampling:** The decision is made after spans are collected, allowing decisions based on trace outcomes or duration, subject to the Collector's configuration and resource constraints.

Sampling must be chosen carefully. If error traces are important, a strategy that discards too many of them can make incidents harder to diagnose.

### 4.5 Instrumentation choices

- **Automatic instrumentation:** Libraries or agents capture common framework and dependency operations with limited code changes.
- **Manual instrumentation:** Developers add spans, metrics, or events around business-specific operations.

Automatic instrumentation can be a useful starting point, but manual instrumentation is often needed to understand important business operations and application-specific behavior.

---

## 5. Alerting strategies and actionable alerts

An alert should tell someone that a condition needs attention and should lead to a useful action.

Too few alerts can leave important incidents undetected. Too many noisy alerts create alert fatigue, making it easier to miss genuinely important signals.

### 5.1 What should trigger an alert?

Prefer alerts that represent meaningful symptoms, such as:
- A sustained increase in failed user requests.
- A service's latency objective being violated.
- A critical endpoint becoming unavailable.
- A queue growing without recovering.
- A resource nearing a limit in a way that threatens service health.
- A service-level objective consuming its error budget unusually quickly.

Not every unusual metric needs a page. Some conditions are better handled by dashboards, tickets, or routine review.

### 5.2 Symptom-based vs cause-based alerts

**Symptom-based alerts** focus on user-visible impact, such as a high error rate or failed requests.

**Cause-based alerts** focus on a potential underlying problem, such as high CPU usage or a disk nearing capacity.

Both can be useful. Symptom-based alerts help establish impact, while cause-based alerts can provide early warning or context. Resource alerts should be interpreted according to whether the condition threatens service behavior.

### 5.3 Alert on sustained conditions

A single brief spike may not justify waking someone up. Alert rules often use a duration or other logic to distinguish transient changes from persistent problems.

Example Prometheus alert rule:

```yaml
groups:
  - name: example-service-alerts
    rules:
      - alert: HighServerErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Elevated server error ratio"
          description: "The observed 5xx ratio has been above 5% for 10 minutes."
```

This is an illustrative rule, not a production-ready universal alert. It assumes a compatible request counter and `status` label. A real rule should handle absent or zero traffic, select the relevant service and request population, validate the query, and set thresholds based on service expectations.

### 5.4 What makes an alert actionable?

A useful alert should include:
- **Clear name:** What is happening?
- **Severity:** How urgently does it need attention?
- **Impact:** What user or service behavior may be affected?
- **Evidence:** Which metric or condition triggered it?
- **Context:** Links to relevant dashboards, logs, traces, or recent changes.
- **Next steps:** A runbook or initial diagnostic checks.
- **Ownership:** Which team is expected to respond?

### 5.5 Severity and routing

Organizations define severity differently, but a practical approach might be:

- **Critical / page:** Severe user impact or an urgent threat to service availability; requires immediate response.
- **Warning / ticket or notification:** A developing issue that needs timely investigation but may not require immediate paging.
- **Informational:** Useful operational context without a direct response requirement.

Severity should reflect impact and urgency, not merely the numerical size of a metric.

### 5.6 Alert fatigue

Alert fatigue can develop when:
- Alerts fire for conditions that do not require action.
- Thresholds are too sensitive.
- Duplicate alerts are generated for the same incident.
- Notifications lack context or ownership.
- Alerts repeatedly fire for known, unresolved issues.

Improve alert quality by reviewing alert history, removing redundant rules, tuning thresholds, grouping related notifications, and ensuring every paging alert has a clear response path.

---

## 6. Dashboards and service health indicators

A dashboard should help a specific audience answer useful questions quickly. It should not be a collection of every metric available.

### 6.1 Start with the purpose

Before creating a dashboard, decide:
- Who will use it?
- Which service or system does it cover?
- What questions should it answer?
- What actions might follow from the information?

A service overview dashboard may be designed for on-call engineers, while a capacity dashboard may focus on infrastructure trends.

### 6.2 Suggested service overview

For a user-facing API, consider these panels:

| Panel | What it helps answer |
|---|---|
| Request rate | How much traffic is the service handling? |
| Success and error ratio | Are requests succeeding? |
| p50, p95, and p99 latency | How long are requests taking, including the slow tail? |
| Availability or SLO compliance | Is the service meeting its reliability target? |
| CPU and memory | Are compute resources constrained? |
| Restarts and unhealthy instances | Are application processes or pods failing? |
| Dependency latency and errors | Is a downstream service contributing to the issue? |
| Queue depth and age | Is asynchronous work accumulating? |
| Recent deployments | Did behavior change after a release? |

Choose panels relevant to the service rather than adding all of them automatically.

### 6.3 Service health indicators

Useful service health indicators often include:
- Success rate or error ratio.
- Latency percentiles.
- Traffic and workload volume.
- Saturation and queueing.
- Dependency health.
- SLO performance and error-budget consumption.

Infrastructure metrics are valuable, but they should be interpreted alongside user-visible service outcomes.

### 6.4 Dashboard design tips

- Put the most important user-facing indicators near the top.
- Use consistent units and meaningful labels.
- Choose time ranges appropriate to the question.
- Make it easy to compare current behavior with a baseline.
- Avoid unnecessary panels and visually noisy displays.
- Include links to runbooks, logs, and traces when possible.
- Test the dashboard during a simulated incident.
- Ensure access controls and data handling are appropriate.

A dashboard supports investigation; it does not replace clear SLOs or actionable alerts.

---

## 7. Troubleshooting with telemetry

Troubleshooting is the process of narrowing down possible causes using evidence, testing hypotheses, and verifying the result.

Telemetry is most useful when it helps answer a specific question rather than encouraging random exploration.

### A practical investigation workflow

#### Step 1: Establish the symptom and impact

Clarify what users are experiencing:
- Is the service unavailable, slow, or returning errors?
- Which endpoints, regions, tenants, or operations are affected?
- When did the issue begin?
- Is the issue ongoing or intermittent?

Use alerts and service-level dashboards to understand the scope.

#### Step 2: Check the timeline

Compare the start of the problem with:
- Recent deployments or configuration changes.
- Traffic changes.
- Infrastructure events.
- Dependency incidents.
- Scheduled jobs or maintenance.

Correlation is a clue, not proof that a change caused the incident.

#### Step 3: Compare the golden signals

Review:
- **Latency:** Are requests slower?
- **Traffic:** Has demand changed?
- **Errors:** Are failures more frequent?
- **Saturation:** Is a resource or queue constrained?

Look for patterns across the affected and unaffected components.

#### Step 4: Narrow the scope

Break down measurements by useful, bounded dimensions such as:
- Service.
- Endpoint or operation.
- Region or availability zone.
- Response class.
- Dependency.
- Deployment version.

Avoid dimensions that create excessive metric cardinality, such as arbitrary user IDs.

#### Step 5: Follow a request through traces

If distributed traces are available:
- Find an affected request.
- Identify the slowest or failing span.
- Inspect downstream calls.
- Compare successful and failed traces.
- Check whether the delay is in the application, network, database, or another dependency.

A trace shows timing and recorded context; it may not by itself prove the root cause.

#### Step 6: Inspect relevant logs

Use the appropriate time window and context to find:
- Exceptions and timeouts.
- Connection failures.
- Unexpected response codes.
- Configuration errors.
- Resource or dependency failures.

Use trace IDs or other correlation fields when available. Avoid exposing secrets or sensitive user information in queries and shared incident notes.

#### Step 7: Form and test a hypothesis

State a specific hypothesis, such as:

"The API's latency increased because database connections are exhausted."

Then check evidence:
- Connection-pool utilization.
- Database query latency.
- Active and waiting connections.
- Application traces.
- Recent code or configuration changes.

If the evidence does not support the hypothesis, revise it rather than continuing to assume it is correct.

#### Step 8: Mitigate and verify

Choose a safe action based on the evidence and the potential impact. After the action, verify that the user-facing symptoms improve and that the change did not create a different problem.

#### Step 9: Document and improve

Record the timeline, impact, evidence, mitigation, and follow-up actions. Improve instrumentation, alerts, automation, or documentation where the investigation revealed a gap.

---

## 8. Example: investigating a slow API

### Scenario

Users report that a checkout API is taking several seconds to respond. The service is still reachable, but some requests time out.

### Initial telemetry

- Request rate is approximately normal.
- p95 latency has increased significantly.
- The 5xx ratio is rising.
- Application CPU is moderate.
- Database connection-pool usage is close to its limit.

### Investigation

1. Check the latency and error dashboards to confirm when the problem started.
2. Compare the timeline with recent releases and dependency changes.
3. Inspect traces for slow checkout requests.
4. Discover that many affected traces spend most of their time waiting for a database operation.
5. Review application logs and database metrics for timeouts, slow queries, and connection-pool behavior.
6. Determine whether the issue is due to a slow query, a connection leak, increased concurrency, or another cause before choosing a mitigation.

### Possible response

If evidence identifies a newly introduced query regression, a safe rollback or code fix may be appropriate. If the cause is connection exhaustion, simply increasing the pool size may only delay the problem or overload the database. The team should understand why connections remain occupied and validate any capacity change.

After mitigation:
- Verify that latency and error rates recover.
- Confirm that database pressure returns to an acceptable level.
- Document the cause and contributing factors.
- Add a useful alert or diagnostic view if needed.
- Improve testing or rollout controls to reduce recurrence.

This scenario illustrates why metrics, logs, and traces are complementary.

---

## 9. Practical exercises

### Exercise 1: Understand the three signals

- [ ] Explain the difference between a metric, a log, and a trace.
- [ ] Choose a sample API and list five useful metrics.
- [ ] Write an example structured error log.
- [ ] Draw a request path across three services and identify where spans would be created.

### Exercise 2: Explore Prometheus

- [ ] Run Prometheus locally or in a disposable lab environment.
- [ ] Configure or inspect a metrics target.
- [ ] Query `up`.
- [ ] Query a request counter using `rate()`.
- [ ] Build a request-rate or error-ratio query using metrics that actually exist in your environment.
- [ ] Learn how labels affect aggregation and cardinality.

### Exercise 3: Build a Grafana dashboard

- [ ] Configure a Prometheus data source.
- [ ] Create panels for traffic, errors, and latency.
- [ ] Add a resource or saturation panel.
- [ ] Set appropriate units and time ranges.
- [ ] Document what each panel tells an on-call engineer.

### Exercise 4: Explore distributed tracing

- [ ] Instrument a sample application with OpenTelemetry.
- [ ] Send traces to a compatible trace backend.
- [ ] Follow a request through multiple services, if possible.
- [ ] Identify a slow span.
- [ ] Correlate a trace with application logs.

### Exercise 5: Design an actionable alert

- [ ] Choose a user-impacting symptom.
- [ ] Define the metric and its calculation.
- [ ] Set a threshold based on an explicit objective or baseline.
- [ ] Decide how long the condition should persist before notification.
- [ ] Add a description, severity, and diagnostic link.
- [ ] Test the alert and review whether the notification provides enough context.

### Exercise 6: Investigate a simulated incident

- [ ] Create or use a scenario involving high latency or an increased error rate.
- [ ] Identify the initial symptom using metrics.
- [ ] Narrow the problem using logs and traces.
- [ ] Write a hypothesis and the evidence supporting or disproving it.
- [ ] Document the mitigation and how recovery was verified.

---

## 10. Learning resources

These are recommended starting points. Add notes for specific pages, chapters, courses, videos, and labs as you work through them.

### Monitoring and observability fundamentals

1. **Google SRE Book — Monitoring Distributed Systems**
   - Monitoring philosophy, signals, and alerting.
   - https://sre.google/sre-book/monitoring-distributed-systems/

2. **Google SRE Workbook**
   - Practical SRE implementation guidance.
   - https://sre.google/workbook/table-of-contents/

3. **OpenTelemetry documentation**
   - Concepts, instrumentation, collection, and export.
   - https://opentelemetry.io/docs/

### Prometheus and Grafana

4. **Prometheus documentation**
   - https://prometheus.io/docs/introduction/overview/

5. **PromQL basics**
   - https://prometheus.io/docs/prometheus/latest/querying/basics/

6. **Prometheus histograms and summaries**
   - https://prometheus.io/docs/practices/histograms/

7. **Grafana documentation**
   - https://grafana.com/docs/

8. **Grafana tutorials**
   - https://grafana.com/tutorials/

### Distributed tracing

9. **OpenTelemetry: Getting Started**
   - https://opentelemetry.io/docs/getting-started/

10. **OpenTelemetry Collector**
    - https://opentelemetry.io/docs/collector/

11. **OpenTelemetry: Observability Primer**
    - https://opentelemetry.io/docs/concepts/observability-primer/

### Practical labs

12. **Prometheus: Getting started**
    - https://prometheus.io/docs/prometheus/latest/getting_started/

13. **Grafana: Get started**
    - https://grafana.com/docs/grafana/latest/getting-started/

14. **OpenTelemetry Demo**
    - A sample microservices application designed to demonstrate observability.
    - https://opentelemetry.io/docs/demo/
