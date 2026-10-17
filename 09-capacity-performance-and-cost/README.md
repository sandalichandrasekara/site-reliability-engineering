# 09. Capacity, Performance, and Cost

## Overview

Capacity, performance, and cost are closely connected in Site Reliability Engineering. A service must have enough resources to handle its workload, respond within acceptable time limits, and remain financially sustainable.

A system with insufficient capacity may experience high latency, errors, resource exhaustion, and outages. A system with excessive capacity may remain reliable but waste money on underutilized infrastructure.

SREs work to find the right balance between reliability, performance, and cost. This involves measuring actual workload behavior, identifying bottlenecks, forecasting future demand, scaling resources appropriately, and continuously evaluating infrastructure spending.

The goal is not simply to make an application faster or cheaper. It is to deliver the required level of service reliability and performance at an appropriate cost.

This section explores the methods and tools used to understand system capacity, investigate performance problems, plan for growth, and optimize cloud resources without compromising reliability.

## Learning Objectives

By the end of this section, I aim to understand:

- How load testing and performance analysis reveal system limitations.
- How resource utilization and saturation affect reliability.
- How to forecast future capacity requirements.
- The differences between horizontal and vertical scaling.
- How to identify application, infrastructure, and dependency bottlenecks.
- How to optimize cloud costs without creating reliability risks.
- How to balance availability, latency, throughput, and infrastructure spending.
- How to use performance and cost measurements to make evidence-based SRE decisions.

## Table of Contents

1. [Load Testing and Performance Analysis](#1-load-testing-and-performance-analysis)
2. [Resource Utilization and Saturation](#2-resource-utilization-and-saturation)
3. [Capacity Forecasting](#3-capacity-forecasting)
4. [Horizontal vs. Vertical Scaling](#4-horizontal-vs-vertical-scaling)
5. [Bottleneck Identification](#5-bottleneck-identification)
6. [Cloud Cost Optimization](#6-cloud-cost-optimization)
7. [Balancing Reliability, Performance, and Cost](#7-balancing-reliability-performance-and-cost)
8. [Putting It All Together](#8-putting-it-all-together)
9. [Key Takeaways](#9-key-takeaways)
10. [Learning Resources](#10-learning-resources)

---

## 1. Load Testing and Performance Analysis

### 1.1 What Is Performance Testing?

Performance testing evaluates how an application behaves under different workload conditions.

It helps determine whether a system can meet its performance and reliability requirements when users, requests, or background jobs increase.

Performance testing can reveal:

- Slow response times.
- Limited request-processing capacity.
- CPU or memory exhaustion.
- Database bottlenecks.
- Network limitations.
- Connection pool exhaustion.
- Unexpected application errors.
- Performance degradation under sustained load.

Without performance testing, teams may discover capacity limitations only after real users experience problems.

### 1.2 Types of Performance Testing

| Test type | Purpose |
|---|---|
| Load testing | Evaluates behavior under an expected or specified workload |
| Stress testing | Pushes a system beyond expected operating conditions to discover its limits |
| Spike testing | Evaluates behavior when workload increases or decreases sharply |
| Soak testing | Runs a sustained workload to reveal long-term problems |
| Scalability testing | Evaluates how performance changes as resources or workload increase |
| Benchmark testing | Measures performance under a defined, repeatable test configuration |

Each test answers a different question. A successful load test does not automatically prove that a system can tolerate a prolonged resource leak or a sudden traffic spike.

### 1.3 Important Performance Metrics

**Latency**

The time required to complete a request or operation.

Common measurements include:

- Average latency.
- Median latency, or p50.
- p95 latency.
- p99 latency.

Percentiles are especially useful because averages can hide slow experiences affecting a smaller portion of requests.

For example, p99 latency of 800 milliseconds means that approximately 99% of measured requests completed within 800 milliseconds, subject to the measurement method and dataset.

**Throughput**

The amount of work completed during a given period.

Examples include:

- Requests per second.
- Transactions per second.
- Messages processed per second.
- Files processed per minute.

**Error rate**

The proportion of operations that fail.

For example, if 100 out of 10,000 requests fail:

\[
\text{Error Rate} =
\frac{100}{10{,}000}\times100
=1\%
\]

**Resource consumption**

The resources used while processing the workload, such as CPU, memory, disk I/O, and network bandwidth.

### 1.4 Load Testing Example

Suppose a web service normally handles 200 requests per second. The team expects traffic to increase significantly during a promotional event.

A load test can gradually increase traffic while measuring:

- Request latency.
- Successful and failed requests.
- CPU and memory usage.
- Database latency.
- Connection pool utilization.
- Queue backlog.
- Autoscaling behavior.

The objective is to identify the workload level at which the system no longer meets its service requirements.

### 1.5 Example Using k6

[k6](https://grafana.com/docs/k6/latest/) is a tool for load testing applications.

The following example sends a controlled number of virtual-user requests to a test endpoint.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '30s',
};

export default function () {
  const response = http.get('https://test.example.com/');

  check(response, {
    'status is successful': (r) =>
      r.status >= 200 && r.status < 400,
  });

  sleep(1);
}
```

Run the script with:

```bash
k6 run load-test.js
```

This example uses ten virtual users for 30 seconds, with a one-second pause between iterations.

Use an environment you own or have explicit permission to test. A load test can consume significant resources or disrupt a service, even when the test duration is short.

### 1.6 Establish Performance Targets

A useful performance test begins with clear acceptance criteria.

For example:

- p95 latency below 300 milliseconds.
- Error rate below 0.5%.
- At least 500 requests per second.
- CPU utilization below a defined operational threshold.
- No sustained increase in queue backlog.

These are illustrative targets, not universal recommendations. Actual targets should come from user expectations, service-level objectives, and measured system behavior.

### 1.7 Performance Analysis Workflow

```text
Define Workload
      |
      v
Set Performance Targets
      |
      v
Run Controlled Test
      |
      v
Collect Metrics and Traces
      |
      v
Identify Degradation Point
      |
      v
Investigate Bottlenecks
      |
      v
Apply an Improvement
      |
      v
Repeat the Test
```

A repeatable testing process allows teams to compare results before and after a change.

### 1.8 Common Load Testing Mistakes

- Testing against unrealistic workloads.
- Using test data that differs significantly from production data.
- Ignoring background jobs and downstream dependencies.
- Measuring only average latency.
- Running tests without monitoring the database.
- Assuming that more virtual users always represent more requests per second.
- Testing a system without permission.
- Treating one successful test as proof of long-term reliability.

**Key principle:** Load testing should identify the workload a system can sustain while meeting its defined reliability and performance targets.

---

## 2. Resource Utilization and Saturation

### 2.1 What Is Resource Utilization?

Resource utilization describes how much of a resource's available capacity is being used.

Examples include:

- CPU utilization.
- Memory consumption.
- Disk space utilization.
- Disk I/O utilization.
- Network bandwidth utilization.
- Database connection utilization.

Utilization helps explain how heavily a system is being used, but it does not tell the whole story.

A service can experience high latency even when CPU utilization is low, particularly if it is waiting for a database, a network response, a lock, or another dependency.

### 2.2 What Is Saturation?

Saturation occurs when a resource cannot handle additional demand without queuing, delays, rejection, or degraded performance.

Examples include:

- CPU work accumulating faster than it can be processed.
- A database connection pool reaching its maximum size.
- A disk becoming unable to keep up with write requests.
- A message queue accumulating an increasing backlog.
- A thread pool becoming fully occupied.

Utilization measures resource use. Saturation helps describe the pressure caused by demand approaching or exceeding available capacity.

### 2.3 The Four Golden Signals

Google's SRE guidance identifies four important monitoring signals:

| Signal | Meaning | Example |
|---|---|---|
| Latency | Time taken to serve requests | p95 response time |
| Traffic | Demand placed on the system | Requests per second |
| Errors | Failed or unsuccessful requests | HTTP 5xx rate |
| Saturation | How close the system is to its limits | Queue backlog or resource pressure |

These signals provide a useful starting point for understanding service health.

### 2.4 CPU Utilization

High CPU utilization can indicate:

- Increased traffic.
- Expensive computations.
- Inefficient code.
- Excessive background processing.
- Insufficient CPU capacity.

However, high CPU utilization is not automatically a problem. Some workloads are designed to use available CPU efficiently.

The important question is whether the service can still meet its latency and error-rate targets.

On Linux, inspect CPU and system activity using:

```bash
top
```

Or:

```bash
vmstat 1
```

These commands help inspect resource usage and system activity. Interpretation depends on the operating system, workload, and measurement context.

### 2.5 Memory Utilization

High memory usage can result from:

- Memory leaks.
- Large caches.
- Large in-memory datasets.
- Too many concurrent requests.
- Excessive buffering.
- Inefficient object allocation.

Memory pressure can lead to swapping, application instability, or out-of-memory termination.

Inspect memory using:

```bash
free -h
```

For Kubernetes workloads, if resource metrics are available:

```bash
kubectl top pods
kubectl top nodes
```

### 2.6 Disk and Network Saturation

Disk-related problems may include:

- High storage latency.
- Slow reads and writes.
- Exhausted disk space.
- High I/O wait.
- Limited storage throughput.

Network-related problems may include:

- Bandwidth saturation.
- Packet loss.
- Connection exhaustion.
- DNS delays.
- Network latency.
- Retransmissions.

Useful Linux commands include:

```bash
df -h
```

For disk I/O, where the tool is installed:

```bash
iostat -xz 1
```

For network sockets:

```bash
ss -s
```

These tools help gather evidence, but no single command explains every performance issue.

### 2.7 Queue Backlog as a Saturation Signal

A queue backlog is the amount of work waiting to be processed.

If messages arrive faster than workers can process them, the backlog grows.

For example:

```text
Incoming Messages: 1,000 per second
Processing Capacity: 700 per second

Backlog Growth:
1,000 - 700 = 300 messages per second
```

If this imbalance continues, message processing latency will increase.

The service may need more workers, faster processing, reduced incoming demand, or a change to the bottleneck dependency.

### 2.8 SRE Considerations

- Is resource usage increasing alongside latency?
- Is the resource actually saturated, or merely busy?
- Are queues growing faster than they drain?
- Are CPU and memory limits creating artificial constraints?
- Is the application waiting on downstream services?
- Are metrics collected at a useful level of detail?

**Key principle:** Resource utilization shows how much capacity is used; saturation reveals whether demand is overwhelming the system's ability to process work.

---

## 3. Capacity Forecasting

### 3.1 What Is Capacity Forecasting?

Capacity forecasting estimates the resources and throughput a system will need in the future.

It helps SRE teams prepare for:

- Business growth.
- Seasonal traffic.
- Promotional events.
- New product launches.
- Increasing datasets.
- Higher transaction volumes.
- Infrastructure maintenance.
- Failure scenarios that reduce available capacity.

Without capacity forecasting, organizations may scale too late or pay for resources they do not need.

### 3.2 What Should Be Forecast?

Capacity planning may include:

- CPU and memory.
- Application replicas.
- Database throughput and connections.
- Storage capacity.
- Network bandwidth.
- Queue processing capacity.
- Cloud spending.
- Available capacity during component failures.

Forecasting should consider both normal operation and the conditions the system is expected to tolerate.

### 3.3 Historical Trend Analysis

Historical data can reveal patterns such as:

- Daily traffic peaks.
- Weekly usage cycles.
- Seasonal growth.
- Increased storage consumption.
- Rising database load.
- Resource usage after software releases.

For example, if traffic increases every weekday morning, the team can prepare for that demand rather than responding after latency increases.

Historical patterns are useful, but they do not fully predict sudden events or architectural changes.

### 3.4 A Simple Linear Forecast

Suppose an application currently receives 10,000 requests per minute and demand is expected to grow by 10% each month.

A simple growth model is:

\[
D_n = D_0(1+r)^n
\]

Where:

- \(D_n\) is the forecast demand after \(n\) periods.
- \(D_0\) is the current demand.
- \(r\) is the growth rate per period.
- \(n\) is the number of periods.

After six months:

\[
D_6 = 10{,}000(1.10)^6
\]

\[
D_6 \approx 17{,}716
\]

The forecast is approximately 17,716 requests per minute.

This is an illustrative model. Real demand may grow unevenly, and the assumed growth rate should be validated against historical data and business expectations.

### 3.5 Capacity Headroom

Capacity headroom is the spare capacity available above the current workload.

Suppose a service can safely handle 1,000 requests per second and currently handles 650 requests per second.

The remaining nominal capacity is:

\[
1{,}000 - 650 = 350
\]

That is 350 requests per second of nominal headroom.

However, this calculation assumes that the service can sustain its tested capacity under the current workload and that no other dependency becomes the bottleneck.

Headroom should also account for:

- Traffic spikes.
- Deployment overhead.
- Failed replicas.
- Maintenance events.
- Variations in request complexity.
- Downstream dependency capacity.

### 3.6 Forecasting Under Failure Conditions

Suppose an application has four replicas, each capable of serving 250 requests per second under the required latency target.

Nominal combined capacity is:

\[
4 \times 250 = 1{,}000
\]

If one replica becomes unavailable, nominal capacity becomes:

\[
3 \times 250 = 750
\]

If expected peak demand is 800 requests per second, the application may not meet its target during this failure.

This is why capacity forecasting should include failure scenarios instead of assuming that every replica will always be available.

### 3.7 Capacity Planning Workflow

```text
Collect Historical Metrics
          |
          v
Understand Traffic Patterns
          |
          v
Estimate Future Demand
          |
          v
Measure Sustainable Capacity
          |
          v
Model Peak and Failure Scenarios
          |
          v
Estimate Required Resources
          |
          v
Validate with Load Testing
          |
          v
Review and Update Forecast
```

### 3.8 SRE Considerations

- What is the expected traffic growth?
- How much capacity is available during peak demand?
- Can the service handle the loss of a replica or node?
- Are database and storage requirements growing at the same rate?
- When will capacity become insufficient?
- How frequently should forecasts be reviewed?

**Key principle:** Capacity planning combines historical evidence, expected growth, measured performance, and realistic failure assumptions.

---

## 4. Horizontal vs. Vertical Scaling

Scaling increases the resources available to a system or changes how work is distributed across existing resources.

### 4.1 Vertical Scaling

Vertical scaling increases or decreases the resources assigned to a single machine or instance.

Examples include:

- Increasing CPU cores.
- Increasing memory.
- Moving to a larger virtual machine.
- Increasing the capacity of a database instance.

```text
Before:
+-------------------+
| Application       |
| 2 CPU / 4 GB RAM  |
+-------------------+

After:
+-------------------+
| Application       |
| 8 CPU / 16 GB RAM |
+-------------------+
```

**Advantages**

- Often simpler than distributing application state.
- Can improve performance for resource-bound workloads.
- May require fewer application-level architectural changes.

**Disadvantages**

- Limited by the maximum size of the instance.
- May require downtime or a restart.
- Can create a larger single failure domain.
- May become expensive at higher sizes.

### 4.2 Horizontal Scaling

Horizontal scaling adds or removes instances to distribute work.

Examples include:

- Adding application replicas.
- Increasing the number of Kubernetes Pods.
- Adding worker processes across multiple machines.
- Adding database read replicas for suitable workloads.

```text
Before:
+-------------+
| Application |
+-------------+

After:
+-------------+   +-------------+   +-------------+
| Application |   | Application |   | Application |
| Instance 1  |   | Instance 2  |   | Instance 3  |
+-------------+   +-------------+   +-------------+
```

**Advantages**

- Can increase total throughput.
- Can improve resilience when instances are distributed across failure domains.
- Often supports automatic scaling.
- May allow individual instances to be replaced independently.

**Disadvantages**

- Requires effective traffic distribution.
- Can introduce coordination and distributed-state complexity.
- May increase pressure on shared dependencies.
- Requires enough underlying infrastructure capacity.

### 4.3 Comparison

| Feature | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| Method | Increase resources on an instance | Add or remove instances |
| Complexity | Often simpler initially | Requires coordination and distribution |
| Upper limit | Limited by instance size | Limited by architecture and total capacity |
| Fault tolerance | Does not inherently add replicas | Can improve resilience with independent replicas |
| Stateful workloads | Often convenient | May require replication or partitioning |
| Common use | Resource-heavy individual workloads | Distributed application tiers |

### 4.4 When to Use Each Approach

Vertical scaling may be appropriate when:

- An application needs more memory on one instance.
- A database is limited by the current instance size.
- The application cannot easily distribute state.
- A moderate resource increase is sufficient.

Horizontal scaling may be appropriate when:

- Requests can be processed independently.
- The application is stateless or state is externalized.
- Traffic varies substantially.
- Additional replicas improve both throughput and resilience.

Many systems use both approaches.

For example, a team may increase the size of worker nodes while also scaling the number of application Pods.

### 4.5 Kubernetes Autoscaling

Kubernetes supports several scaling mechanisms.

**Horizontal Pod Autoscaler (HPA)**

Adjusts the number of Pod replicas based on configured metrics.

**Vertical Pod Autoscaler (VPA)**

Can recommend or adjust CPU and memory requests, depending on its configuration and operating mode.

**Cluster autoscaling**

Adds or removes worker nodes when supported and configured, based on scheduling demand and cluster conditions.

These mechanisms are related but not interchangeable.

Adding more Pods will not solve every bottleneck. If all Pods depend on a database that has reached its connection or throughput limit, scaling the application may worsen the incident.

### 4.6 SRE Considerations

- Can the workload be distributed effectively?
- Is the bottleneck within an individual instance or a shared dependency?
- Does scaling require state replication?
- Is there enough capacity to schedule more replicas?
- Will additional instances increase database connections or downstream traffic?
- Does the chosen scaling method meet the service's availability and latency requirements?

**Key principle:** Scale the constrained part of the system, not simply the component that is easiest to scale.

---

## 5. Bottleneck Identification

### 5.1 What Is a Bottleneck?

A bottleneck is a component or resource that limits the overall performance or throughput of a system.

Examples include:

- CPU-intensive application code.
- Insufficient memory.
- Slow database queries.
- Database connection limits.
- Disk I/O constraints.
- Network latency.
- Lock contention.
- A slow external API.
- A queue worker that cannot process messages quickly enough.

Improving a component that is not limiting the workload may produce little or no improvement in end-to-end performance.

### 5.2 Understand the Request Path

Consider a request passing through several components:

```text
Client
  |
  v
Load Balancer
  |
  v
Application
  |
  +------> Cache
  |
  +------> Database
  |
  +------> External API
  |
  v
Response
```

The request may be delayed by any component along this path.

The application itself may consume very little CPU while spending most of its time waiting for the database or external API.

### 5.3 Common Bottleneck Symptoms

| Symptom | Possible causes |
|---|---|
| High CPU and rising latency | Expensive computation or insufficient CPU |
| Low CPU and high latency | Dependency waits, locking, I/O, or network delays |
| High memory pressure | Large data structures, memory leaks, excessive concurrency |
| Growing queue backlog | Incoming work exceeds processing capacity |
| Slow database queries | Missing indexes, inefficient queries, contention, or storage limits |
| Connection timeouts | Connection pool exhaustion or unavailable dependencies |
| High disk latency | Storage saturation or inefficient I/O |
| Errors after scaling out | Shared dependency limits or increased contention |

These are diagnostic clues rather than definitive proof of a root cause.

### 5.4 A Structured Investigation

```text
Observe Degradation
        |
        v
Measure Latency and Error Rate
        |
        v
Inspect Resource Utilization
        |
        v
Trace Slow Requests
        |
        v
Identify the Slowest Component
        |
        v
Form a Testable Hypothesis
        |
        v
Change One Relevant Factor
        |
        v
Measure the Result
```

Distributed tracing is particularly useful for identifying how much time a request spends in each service or dependency.

Metrics help identify when and where degradation occurs. Logs provide event details, while traces show the path and timing of individual operations.

### 5.5 Example: Database Bottleneck

A web application has the following symptoms:

- CPU utilization is moderate.
- Request latency has increased.
- Database query duration is high.
- Database connections are near their configured limit.

Adding application replicas may increase the number of concurrent database requests, potentially making the database problem worse.

A better investigation would examine:

- Slow query logs.
- Query execution plans.
- Database CPU and I/O.
- Connection pool configuration.
- Query concurrency.
- Caching opportunities.
- Whether requests are waiting for connections.

The correct improvement depends on the evidence. It could involve query optimization, connection management, caching, additional database capacity, or reduced concurrency.

### 5.6 Little's Law

Little's Law relates the average number of items in a system, throughput, and average time spent in that system:

\[
L = \lambda W
\]

Where:

- \(L\) is the average number of items in the system.
- \(\lambda\) is the average arrival or completion rate in a stable system.
- \(W\) is the average time spent in the system.

For example, if a service completes 100 requests per second and the average time in the system is 0.2 seconds:

\[
L = 100 \times 0.2 = 20
\]

There are approximately 20 requests in the system on average under those assumptions.

This relationship helps reason about concurrency and queueing. It does not mean that increasing concurrency will automatically improve throughput.

### 5.7 SRE Considerations

- Which component consumes the largest share of request time?
- Is the bottleneck consistent or workload-dependent?
- Does the issue affect every request or only certain operations?
- Is a shared dependency limiting the entire service?
- Does the proposed fix address the root cause?
- Can the improvement be verified through a repeatable test?

**Key principle:** Measure the complete request path before deciding which component to optimize.

---

## 6. Cloud Cost Optimization

### 6.1 What Is Cloud Cost Optimization?

Cloud cost optimization aims to deliver the required service quality while using cloud resources efficiently.

Cloud expenses can include:

- Virtual machines and worker nodes.
- Managed databases.
- Object and block storage.
- Network transfer.
- Load balancers.
- Monitoring and logging.
- Backup storage.
- Managed platform services.
- Idle development and testing environments.

Cost optimization is not simply reducing resource usage. Reducing resources below the level needed for reliable service can increase latency, errors, and incident costs.

### 6.2 Common Sources of Cloud Waste

- Oversized virtual machines.
- Idle resources.
- Underutilized Kubernetes nodes.
- Excessive replica counts.
- Unnecessary storage retention.
- Unoptimized data transfer.
- Overly verbose logs retained for too long.
- Forgotten test environments.
- Overprovisioned databases.
- Inefficient queries that increase compute usage.

A useful first step is to understand which services and workloads account for the largest share of spending.

### 6.3 Rightsizing

Rightsizing means adjusting resource allocations to better match actual workload requirements.

Examples include:

- Choosing an appropriately sized virtual machine.
- Adjusting Kubernetes CPU and memory requests.
- Reducing unnecessary replica counts.
- Selecting an appropriate database tier.
- Changing storage capacity or performance tiers when supported.

Rightsizing should use observed workload behavior, realistic peak demand, and required failure headroom.

A resource that appears idle during quiet periods may still be necessary during traffic peaks or failures.

### 6.4 Autoscaling

Autoscaling can reduce waste by adjusting capacity to match workload demand.

For example:

```text
Low Demand
    |
    v
Fewer Replicas
    |
    v
Lower Resource Consumption

High Demand
    |
    v
More Replicas
    |
    v
More Available Capacity
```

Autoscaling needs appropriate minimum and maximum limits, useful metrics, sufficient startup time, and adequate cluster capacity.

Scaling down too aggressively can cause instability, cold starts, or capacity shortages during sudden traffic increases.

### 6.5 Storage and Data Transfer Costs

Storage cost optimization may include:

- Removing verified obsolete data.
- Using appropriate storage tiers.
- Applying retention policies.
- Compressing data where appropriate.
- Reviewing backup retention.
- Eliminating unnecessary duplicate data.

Network costs may be reduced by:

- Avoiding unnecessary cross-region transfers.
- Placing communicating services appropriately.
- Caching frequently accessed content.
- Reducing redundant data transfer.
- Reviewing logging and telemetry volume.

Changes to storage or retention policies must preserve required backups, recovery objectives, and compliance obligations.

### 6.6 Cost Visibility and Allocation

Useful practices include:

- Applying consistent resource tags or labels.
- Allocating costs to services or teams.
- Reviewing spending trends.
- Setting budgets and alerts.
- Investigating unexpected cost increases.
- Tracking cost per request, transaction, or customer operation.

A total cloud bill may hide important inefficiencies. Unit cost helps show how efficiently the system serves actual demand.

### 6.7 Unit Economics for SRE

Unit cost measures the cost associated with a defined unit of work.

For example:

\[
\text{Cost per Request}
=
\frac{\text{Infrastructure Cost}}
{\text{Successful Requests}}
\]

If monthly infrastructure costs are \$2,000 and the service successfully handles 20 million requests:

\[
\frac{2{,}000}{20{,}000{,}000}
=
\$0.0001
\]

The example gives a cost of \$0.0001 per successful request.

This measure is most useful when compared over time with workload complexity, latency, error rate, and reliability. A lower cost per request is not an improvement if the system is failing more often or delivering an unacceptable user experience.

### 6.8 Cost Optimization Workflow

```text
Collect Cost and Usage Data
           |
           v
Identify Major Cost Drivers
           |
           v
Find Underutilized Resources
           |
           v
Estimate Savings and Risks
           |
           v
Test a Targeted Change
           |
           v
Measure Cost and Performance
           |
           v
Keep, Adjust, or Reverse the Change
```

### 6.9 SRE Considerations

- Which workloads account for most spending?
- Are resources sized for measured demand?
- Is enough capacity retained for failures and traffic peaks?
- Are idle resources identified safely?
- Do cost reductions affect latency or availability?
- Is the cost per successful operation improving?
- Are backups and observability being reduced below acceptable levels?

**Key principle:** Optimize the cost of delivering the required service level, not cost in isolation.

---

## 7. Balancing Reliability, Performance, and Cost

### 7.1 The Three-Way Trade-Off

SRE teams often need to balance three objectives:

- **Reliability:** The service remains available and behaves correctly.
- **Performance:** The service responds within acceptable time limits and handles expected demand.
- **Cost:** The resources required to operate the service remain financially sustainable.

These objectives can conflict.

For example, adding replicas may improve capacity and resilience but increase infrastructure spending. Reducing replicas may save money but leave insufficient headroom during traffic peaks or failures.

The objective is to find an appropriate balance based on the service's requirements.

### 7.2 Service-Level Objectives

A Service-Level Objective (SLO) defines a target for a service-level indicator over a specified period.

Examples include:

- A target percentage of successful requests.
- A target percentage of requests completed within a latency threshold.
- A target availability level.

SLOs help make reliability requirements measurable.

Cost optimization should be evaluated against these objectives. If a cost reduction causes the service to miss its SLO, the change may not be acceptable.

### 7.3 Error Budgets

An error budget represents the amount of unreliability permitted by an SLO.

For example, if an availability SLO is 99.9% over a 30-day period, the corresponding error budget is 0.1% of that period.

Assuming a 30-day period of 43,200 minutes:

\[
43{,}200 \times 0.001 = 43.2
\]

That corresponds to 43.2 minutes of unavailability under a simple time-based model.

The actual interpretation depends on the SLI and how the service measures availability. A request-based SLO, for example, uses request outcomes rather than simply counting minutes.

Error budgets help teams decide when to prioritize reliability work over new features or higher-risk changes.

### 7.4 Reliability Has a Cost

Improving reliability may require:

- Additional replicas.
- Multi-zone deployment.
- More database capacity.
- Redundant network paths.
- Backup storage.
- Disaster recovery environments.
- Monitoring and incident response.
- More extensive testing.

Not every service needs the same degree of redundancy. Criticality, user impact, recovery requirements, and business priorities should guide the investment.

### 7.5 Performance Has a Cost

Reducing latency may require:

- More CPU capacity.
- Better database indexing.
- Caching.
- Faster storage.
- Improved algorithms.
- Reduced network round trips.
- More efficient serialization.
- Additional replicas.

Sometimes the best performance improvement is a code or query optimization that reduces resource consumption and operating cost at the same time.

### 7.6 A Decision Framework

When considering an infrastructure or performance change, evaluate:

| Question | Why it matters |
|---|---|
| What user problem are we solving? | Keeps the change tied to a measurable outcome |
| Which SLO or performance target is affected? | Defines the reliability requirement |
| What evidence identifies the bottleneck? | Avoids unnecessary resource changes |
| What is the expected cost impact? | Makes the financial trade-off visible |
| What happens during a failure or traffic spike? | Tests the design beyond normal conditions |
| How will the result be measured? | Confirms whether the change actually helped |
| Can the change be reversed? | Reduces risk if assumptions are wrong |

### 7.7 Example: Reducing Infrastructure Spending

Suppose a service is running more application replicas than it needs during normal demand.

Reducing the replica count could lower costs, but the team should first determine:

- Whether the current replicas are needed during peak traffic.
- Whether the remaining replicas can handle a node or zone failure.
- Whether autoscaling reacts quickly enough.
- Whether the database can handle the resulting traffic distribution.
- Whether the change affects latency or error rates.

A safe approach is to test a gradual reduction, monitor service-level indicators, and preserve a way to restore capacity.

### 7.8 SRE Considerations

- What level of reliability does the service actually require?
- Is the current performance problem caused by insufficient capacity or inefficient code?
- Are resources being added to address a symptom rather than a bottleneck?
- Can cost be reduced without breaching SLOs?
- Is there sufficient headroom for peak traffic and component failures?
- Are cost and performance evaluated together?

**Key principle:** The best system is not necessarily the fastest, cheapest, or most redundant. It is the system that meets its defined requirements at an appropriate operational cost.

---

## 8. Putting It All Together

Consider a web service experiencing increased traffic and cloud spending.

The service is using more CPU, request latency is increasing, and the cloud bill is rising.

A structured SRE investigation might follow this process:

1. **Measure the user impact:** Review latency, traffic, error rates, and SLO performance.
2. **Inspect resource usage:** Determine whether CPU, memory, storage, network, or connection pools are saturated.
3. **Trace the request path:** Identify slow application operations and downstream dependencies.
4. **Load-test the relevant workload:** Reproduce the issue in an authorized test environment.
5. **Identify the bottleneck:** Confirm whether the limitation is in the application, database, network, or another component.
6. **Forecast future demand:** Estimate expected traffic and resource needs.
7. **Choose the right scaling approach:** Scale vertically, horizontally, or through a combination of both.
8. **Evaluate cost:** Estimate the effect of the change on compute, storage, network, and managed services.
9. **Validate reliability:** Confirm that latency and error rates meet the intended SLOs.
10. **Review the outcome:** Measure cost per successful operation and update capacity forecasts.

The important point is that performance, capacity, and cost should be investigated together.

Adding resources without identifying the bottleneck may waste money. Removing resources without considering failure scenarios may reduce reliability. Optimizing code without measuring the result may not improve the actual user experience.

A reliable SRE approach uses evidence to identify the constraint, changes one relevant factor at a time where practical, and measures the result.

---

## 9. Key Takeaways

- Load testing reveals how a system behaves under defined workloads.
- Latency percentiles provide insight into slow requests that averages can hide.
- Throughput measures completed work, while error rate measures failed operations.
- Resource utilization and saturation are related but different concepts.
- CPU, memory, disk, network, and queues should be monitored according to workload behavior.
- Capacity forecasting combines historical trends, expected growth, and measured sustainable capacity.
- Capacity plans should account for peak demand and expected failure scenarios.
- Vertical scaling increases the resources of an instance; horizontal scaling adds or removes instances.
- Scaling the application may not help when a shared dependency is the bottleneck.
- Bottleneck identification should be based on measurements, logs, and traces.
- Cloud cost optimization includes rightsizing, autoscaling, storage management, and cost visibility.
- Unit cost can reveal efficiency trends, but it must be evaluated alongside reliability and performance.
- SLOs make service requirements measurable, while error budgets help guide reliability decisions.
- Cost reductions should preserve required availability, performance, and recovery capabilities.
- The best optimization is the one that improves the overall service outcome, not just an isolated metric.

---

## 10. Learning Resources

### Performance and Load Testing

- [Grafana k6 Documentation](https://grafana.com/docs/k6/latest/)
- [Apache JMeter Documentation](https://jmeter.apache.org/usermanual/index.html)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

### Monitoring and Resource Analysis

- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Linux `top` Manual](https://man7.org/linux/man-pages/man1/top.1.html)
- [Linux `vmstat` Manual](https://man7.org/linux/man-pages/man8/vmstat.8.html)
- [Kubernetes Resource Metrics](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)

### Capacity and Reliability

- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

### Cloud Cost Optimization

- [AWS Well-Architected Framework — Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [Google Cloud Architecture Framework — Cost Optimization](https://cloud.google.com/architecture/framework/cost-optimization)
- [Microsoft Azure Well-Architected Framework — Cost Optimization](https://learn.microsoft.com/en-us/azure/well-architected/cost-optimization/)
- [FinOps Foundation](https://www.finops.org/)
