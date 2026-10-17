# 07. Cloud, Containers, and Kubernetes

## Overview

Modern Site Reliability Engineering relies heavily on cloud infrastructure, containerization, and orchestration platforms such as Kubernetes. These technologies help teams deploy applications, manage infrastructure, scale services, and maintain reliability when components fail or workloads increase.

However, running an application in the cloud does not automatically make it reliable. Applications can still experience outages due to infrastructure failures, resource exhaustion, misconfigured health checks, poor capacity planning, or unsafe deployments.

This section explores how cloud platforms, containers, and Kubernetes work together to support reliable, scalable, and resilient systems.

The goal is to understand not only how these technologies operate, but also how an SRE uses them to prevent failures, investigate incidents, and maintain service availability.

## Learning Objectives

By the end of this section, I aim to understand:

- How cloud availability zones and fault domains affect reliability.
- The difference between high availability and disaster recovery.
- How Kubernetes schedules workloads and manages resources.
- How to troubleshoot pod failures, OOMKills, and CrashLoopBackOff.
- How startup, readiness, and liveness probes affect application health.
- How resource requests, limits, and autoscaling work.
- How capacity planning and load balancing support reliable services.
- How deployment strategies, rollbacks, and safe releases reduce production risk.

## Table of Contents

1. [Cloud Availability and Fault Domains](#1-cloud-availability-and-fault-domains)
2. [High Availability and Disaster Recovery](#2-high-availability-and-disaster-recovery)
3. [Kubernetes Scheduling and Resource Management](#3-kubernetes-scheduling-and-resource-management)
4. [Pod Failures, OOMKills, and CrashLoopBackOff](#4-pod-failures-oomkills-and-crashloopbackoff)
5. [Startup, Readiness, and Liveness Probes](#5-startup-readiness-and-liveness-probes)
6. [Requests, Limits, and Autoscaling](#6-requests-limits-and-autoscaling)
7. [Capacity Planning and Load Balancing](#7-capacity-planning-and-load-balancing)
8. [Deployment Strategies, Rollbacks, and Safe Releases](#8-deployment-strategies-rollbacks-and-safe-releases)
9. [Putting It All Together](#9-putting-it-all-together)
10. [Key Takeaways](#10-key-takeaways)
11. [Learning Resources](#11-learning-resources)

---

## 1. Cloud Availability and Fault Domains

### 1.1 What Is Cloud Availability?

Cloud availability describes how consistently a cloud service or infrastructure component remains accessible and operational.

Cloud providers offer infrastructure across different physical locations and isolated failure domains. These options allow applications to continue operating when individual components or locations become unavailable.

However, cloud availability depends on how the application is architected. Deploying an application on a cloud platform does not automatically protect it from infrastructure failures.

For example, an application running on a single virtual machine may become unavailable if that machine fails, even if the cloud provider's overall infrastructure remains healthy.

### 1.2 What Are Fault Domains?

A fault domain is a group of infrastructure components that may be affected by the same failure.

Examples include:

- A single virtual machine.
- A physical host.
- A rack of servers.
- A network segment.
- An availability zone.
- A cloud region.

The purpose of fault isolation is to reduce the chance that one failure affects the entire application.

For example, if all application replicas run on the same physical host, a host failure may take down every replica at once.

If replicas are distributed across independent fault domains, the application has a better chance of remaining available.

### 1.3 Regions and Availability Zones

A cloud region is a geographic area containing cloud infrastructure.

An availability zone is an isolated infrastructure location within a region. The exact implementation and isolation guarantees depend on the cloud provider.

| Concept | Meaning | Reliability benefit |
|---|---|---|
| Region | Geographic cloud location | Provides a location for deploying services |
| Availability zone | Isolated location within a region | Helps protect against zone-level failures |
| Fault domain | Group of resources sharing potential failure causes | Helps identify and isolate correlated failures |
| Multi-region architecture | Application distributed across multiple regions | Can improve resilience against regional outages |

### 1.4 Single-Zone vs. Multi-Zone Architecture

**Single-zone architecture**

```text
Users
  |
  v
Load Balancer
  |
  v
Availability Zone A
  |
  +-- Application
  +-- Database
```

If the zone becomes unavailable, the application may become unavailable.

**Multi-zone architecture**

```text
                 Users
                   |
                   v
              Load Balancer
                   |
          +--------+--------+
          |                 |
          v                 v
   Availability Zone A  Availability Zone B
          |                 |
          v                 v
      App Replica       App Replica
          |                 |
          +--------+--------+
                   |
                   v
             Shared Data Layer
```

A multi-zone design can improve availability, provided that traffic routing, data replication, dependencies, and failover mechanisms are also designed for resilience.

### 1.5 Common Failure Scenarios

- A virtual machine crashes.
- A physical host becomes unavailable.
- A zone experiences a network or power failure.
- A regional service disruption affects several dependencies.
- Multiple replicas fail together because they share the same underlying infrastructure.

### 1.6 SRE Considerations

An SRE should ask:

- Are application replicas distributed across independent fault domains?
- Could a single infrastructure failure affect every replica?
- Does the load balancer stop sending traffic to unhealthy instances?
- Can the application continue operating if one zone fails?
- Are important dependencies available across the same failure domains?
- What happens if the cloud provider's regional services become unavailable?

**Key principle:** Redundancy is most useful when the redundant components do not share the same failure mode.

---

## 2. High Availability and Disaster Recovery

High availability and disaster recovery are related but solve different problems.

### 2.1 What Is High Availability?

High availability (HA) is the ability of a system to remain operational despite certain component failures.

HA commonly uses:

- Multiple application instances.
- Load balancing.
- Replication.
- Health checks.
- Automatic failover.
- Redundant infrastructure.
- Deployment across multiple availability zones.

For example, if one application instance fails, a load balancer can route requests to healthy instances.

```text
                 Users
                   |
                   v
              Load Balancer
                   |
          +--------+--------+
          |        |        |
          v        v        v
        App 1    App 2    App 3
        Healthy  Failed   Healthy
          |                 |
          +--------+--------+
                   |
             Service continues
```

The failed instance still needs investigation and recovery, but the service may remain available.

### 2.2 What Is Disaster Recovery?

Disaster recovery (DR) focuses on restoring service and data after a major disruptive event, such as:

- A regional outage.
- Significant data corruption.
- Accidental deletion of critical resources.
- Loss of an entire environment.
- A major infrastructure or security incident.

DR may involve backups, replicated data, recovery environments, documented procedures, and tested restoration processes.

### 2.3 RTO and RPO

Two important disaster recovery concepts are Recovery Time Objective and Recovery Point Objective.

**Recovery Time Objective (RTO)**

The target maximum time to restore a service after a disruption.

**Recovery Point Objective (RPO)**

The target maximum amount of data loss measured in time.

Example:

- RTO: 2 hours.
- RPO: 15 minutes.

This means the recovery plan targets restoring the service within two hours and limiting data loss to approximately the most recent 15 minutes of data.

These are targets, not guarantees. They must be supported by tested recovery procedures.

### 2.4 High Availability vs. Disaster Recovery

| Feature | High Availability | Disaster Recovery |
|---|---|---|
| Primary goal | Maintain service during component failures | Restore service after major disruption |
| Typical scope | Instances, zones, local dependencies | Regions, environments, data recovery |
| Common mechanisms | Replicas, health checks, failover | Backups, replication, recovery procedures |
| Recovery approach | Often automatic and rapid | May involve automated or manual steps |
| Key measurements | Availability, failover time | RTO, RPO, recovery success |

### 2.5 Backup vs. Replication

Replication copies data to another system or location. It can improve availability, but it may also copy accidental deletions or corrupted data.

Backups provide recoverable copies of data from an earlier point in time, depending on the backup design and retention policy.

A reliable DR strategy may require both.

```text
Primary Database
       |
       +------> Replica
       |        Supports availability
       |
       +------> Backup Storage
                Supports recovery
```

A replica is not automatically a backup.

### 2.6 SRE Considerations

- Are backups being created successfully?
- Has restoration been tested?
- Are RTO and RPO documented and achievable?
- Can the application recover from the loss of an entire region?
- Are failover procedures automated, documented, and tested?
- Can a replicated corruption event be reversed using retained backups?

**Key principle:** A recovery plan that has never been tested is an assumption, not proven resilience.

---

## 3. Kubernetes Scheduling and Resource Management

### 3.1 What Is Kubernetes?

Kubernetes is a container orchestration platform that automates the deployment, scaling, scheduling, and management of containerized applications.

It helps manage:

- Containerized workloads.
- Application replicas.
- Scheduling across worker nodes.
- Service discovery and networking.
- Health checks and container restarts.
- Resource allocation.
- Rolling updates and rollbacks.

Kubernetes does not eliminate application failures. It provides mechanisms for detecting and responding to many failures.

### 3.2 Core Kubernetes Components

| Component | Responsibility |
|---|---|
| Cluster | Collection of control-plane and worker-node resources |
| Control plane | Coordinates the desired state of the cluster |
| API server | Exposes the Kubernetes API |
| Scheduler | Selects suitable nodes for new Pods |
| Controller manager | Runs controllers that reconcile actual state with desired state |
| Worker node | Runs application workloads |
| Kubelet | Manages Pods and containers on a node |
| Pod | Smallest deployable Kubernetes workload unit |
| Deployment | Manages replicated application Pods and updates |
| Service | Provides a stable way to access a set of Pods |

### 3.3 What Is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It contains one or more containers that share certain resources, such as network identity and potentially storage volumes.

A typical web application may run in multiple Pods:

```text
Deployment: web-app
       |
       +-- Pod 1
       |    +-- Application Container
       |
       +-- Pod 2
       |    +-- Application Container
       |
       +-- Pod 3
            +-- Application Container
```

A Deployment helps maintain the desired number of replicas.

If one managed Pod disappears, Kubernetes can create a replacement, assuming the cluster has sufficient capacity and the workload can be scheduled.

### 3.4 How Kubernetes Scheduling Works

The scheduler selects a worker node for a Pod that has not yet been assigned to a node.

It considers factors such as:

- Available CPU and memory relative to requests.
- Node selectors and affinity rules.
- Taints and tolerations.
- Topology and placement constraints.
- Other scheduling requirements.

A Pod may remain in the `Pending` state if no node satisfies its scheduling requirements.

For example, a Pod requesting 4 CPU cores may not fit on any eligible node even if several nodes have some CPU capacity available.

### 3.5 Desired State and Reconciliation

Kubernetes uses a declarative model.

You specify the desired state, such as three replicas, and Kubernetes controllers work to reconcile the actual state with it.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:stable
          ports:
            - containerPort: 80
```

This manifest requests three replicas of the application.

If one managed Pod disappears, the Deployment's control loop attempts to restore the desired replica count.

The desired replica count does not guarantee that all replicas will become healthy. Scheduling failures, image-pull errors, insufficient capacity, or application crashes can prevent recovery.

### 3.6 Useful Kubernetes Commands

Inspect nodes:

```bash
kubectl get nodes
```

Inspect Pods:

```bash
kubectl get pods -o wide
```

Inspect Deployments:

```bash
kubectl get deployments
```

Inspect a Pod's events and configuration:

```bash
kubectl describe pod <pod-name>
```

Inspect recent cluster events:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 3.7 SRE Considerations

- Is the cluster able to schedule replacement Pods?
- Are replicas distributed across nodes and zones?
- Are resource requests realistic?
- Are workloads competing for limited node capacity?
- Are critical applications protected from unnecessary scheduling constraints?
- Does the desired state match the application's actual availability requirements?

**Key principle:** Kubernetes can reconcile desired state, but successful reconciliation depends on capacity, configuration, and healthy application behavior.

---

## 4. Pod Failures, OOMKills, and CrashLoopBackOff

Understanding Kubernetes failure states is essential for diagnosing incidents.

### 4.1 Common Pod States

| State or condition | Meaning |
|---|---|
| Pending | The Pod has not completed scheduling or setup |
| Running | The Pod has been assigned to a node and at least one container is running or starting |
| Succeeded | All containers completed successfully and will not be restarted |
| Failed | All containers have terminated, and at least one failed |
| ImagePullBackOff | Kubernetes is delaying repeated attempts to pull a container image |
| CrashLoopBackOff | A container repeatedly terminates and Kubernetes is delaying another restart |

A Pod's phase is not the same as every displayed status or condition. For example, `CrashLoopBackOff` is a container waiting reason, not a Pod phase.

### 4.2 What Is CrashLoopBackOff?

`CrashLoopBackOff` occurs when a container repeatedly starts, exits, and is restarted, with Kubernetes applying a delay between restart attempts.

Common causes include:

- Application startup errors.
- Missing environment variables.
- Invalid configuration.
- Incorrect command or entry point.
- Failed dependency connections.
- Permission problems.
- Repeated application crashes.

Example troubleshooting commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

The `--previous` option is especially useful when the current container has restarted and the logs from its previous instance are needed.

### 4.3 What Is an OOMKill?

OOM stands for *Out of Memory*.

An OOMKill occurs when a process or container is terminated due to memory exhaustion. In Kubernetes, a container may be killed after exceeding its memory limit, although the exact circumstances depend on node memory pressure and system behavior.

For example:

```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "256Mi"
```

If the application needs more memory than its configured limit, it may be terminated.

Repeated OOMKills can cause a Pod to restart continuously and may contribute to `CrashLoopBackOff`.

### 4.4 Common OOMKill Causes

- Memory leaks.
- Traffic spikes.
- Large in-memory data structures.
- Inefficient queries or processing.
- Insufficient memory limits.
- Too many concurrent requests.
- Misconfigured runtime memory settings.

### 4.5 Troubleshooting Workflow

```text
Pod is unhealthy
       |
       v
Check Pod status
       |
       v
Describe the Pod and inspect events
       |
       +----> OOMKilled?
       |         |
       |         v
       |    Inspect memory usage,
       |    limits, and application behavior
       |
       +----> Application crash?
       |         |
       |         v
       |    Check current and previous logs
       |
       +----> Pending?
                 |
                 v
          Inspect scheduling events,
          resource requests, and node capacity
```

### 4.6 Diagnostic Commands

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl top pod
kubectl top nodes
```

The `kubectl top` commands require a working resource metrics source, such as Metrics Server.

### 4.7 SRE Considerations

When a Pod fails, do not immediately assume Kubernetes itself is broken.

Investigate:

- The container termination reason and exit code.
- Current and previous logs.
- Recent configuration or deployment changes.
- Memory and CPU usage.
- Dependency availability.
- Node pressure and scheduling events.
- Whether failures affect one replica or the whole service.

**Key principle:** Restarting a failed container may restore it temporarily, but identifying the root cause is necessary to prevent repeated failures.

---

## 5. Startup, Readiness, and Liveness Probes

Kubernetes probes help determine whether an application has started, can receive traffic, or needs to be restarted.

### 5.1 Startup Probe

A startup probe determines whether an application has finished starting.

It is useful for applications that need a long initialization period.

Until the startup probe succeeds, Kubernetes does not run the configured liveness and readiness probes for that container.

### 5.2 Readiness Probe

A readiness probe determines whether a container is ready to serve traffic.

If a container fails its readiness probe, Kubernetes marks it as not ready and ordinarily removes it from the set of ready endpoints used for Service traffic routing.

The container is not necessarily restarted.

Examples of readiness failures include:

- The application is still loading required data.
- The service is temporarily overloaded.
- Required initialization has not finished.
- The application cannot currently serve requests safely.

### 5.3 Liveness Probe

A liveness probe determines whether a container should be considered unhealthy and restarted.

If a configured liveness probe repeatedly fails, Kubernetes restarts the container according to its restart policy.

A poorly designed liveness probe can make an incident worse by restarting an application that is merely slow or temporarily overloaded.

### 5.4 Comparison

| Probe | Main question | Typical response to failure |
|---|---|---|
| Startup | Has startup completed? | Startup-dependent health checks remain gated; continued failure eventually causes a restart |
| Readiness | Can the application serve traffic? | Container is marked not ready |
| Liveness | Should the container be restarted? | Container is restarted after the configured failure threshold |

### 5.5 Example Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
    - name: web
      image: nginx:stable
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 30
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 10
        failureThreshold: 3
```

This example uses HTTP checks against the web server. In a real application, probe endpoints should be chosen according to the service's behavior and failure modes.

### 5.6 Probe Design Mistakes

**Using the same check for every purpose**

An endpoint that checks every downstream dependency may fail when a dependency is temporarily unavailable, causing an unnecessary restart.

**Making liveness too sensitive**

If liveness checks fail during a brief slowdown, repeated restarts may prevent the application from recovering.

**Marking an unready application as ready**

If readiness checks only confirm that a process exists, traffic may be sent to an application that cannot actually serve requests.

**Ignoring startup time**

Applications that require substantial initialization may be restarted before they have a reasonable chance to start.

### 5.7 SRE Considerations

- Does readiness reflect whether the application can handle traffic?
- Does liveness detect a condition that restarting the process can actually fix?
- Does the startup probe allow enough time for initialization?
- Can a temporary dependency outage cause every replica to restart?
- Are probe failures correlated with latency, errors, and traffic drops?

**Key principle:** Readiness protects traffic routing; liveness supports recovery; startup probes protect slow initialization.

---

## 6. Requests, Limits, and Autoscaling

Kubernetes resource management helps applications share cluster capacity while reducing the risk of resource exhaustion.

### 6.1 CPU and Memory Requests

A resource request specifies the amount of a resource that Kubernetes uses when scheduling a container.

For example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
```

This requests 250 millicores of CPU and 128 MiB of memory.

CPU requests are expressed in CPU units:

- `1000m` = 1 CPU core.
- `500m` = half a CPU core.
- `250m` = one quarter of a CPU core.

Requests influence scheduling and, for supported autoscaling configurations, can affect how utilization is calculated.

### 6.2 CPU and Memory Limits

Limits define the maximum resource use enforced by Kubernetes mechanisms.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

In this example:

- The scheduler uses a CPU request of 250m and a memory request of 128Mi.
- CPU usage is limited to 500m.
- Memory usage is limited to 256Mi.

CPU limits generally cause throttling when the container tries to exceed its allowed CPU quota.

Memory limits can result in a container being terminated when it exceeds its memory allowance.

### 6.3 Why Requests and Limits Matter

Incorrect resource configuration can create reliability problems.

| Configuration problem | Possible consequence |
|---|---|
| CPU request too low | Scheduling and CPU-utilization decisions may not reflect actual needs |
| CPU limit too restrictive | CPU throttling and increased latency |
| Memory request too low | Poor placement decisions and increased risk during node pressure |
| Memory limit too low | OOMKills |
| No meaningful capacity planning | Overloaded nodes or inefficient resource use |

Requests and limits should be based on observed behavior, workload requirements, and appropriate safety margins.

### 6.4 What Is Autoscaling?

Autoscaling adjusts the number of application replicas or the available infrastructure capacity based on defined signals or conditions.

Common forms include:

- **Horizontal Pod Autoscaler (HPA):** Adjusts the number of Pod replicas.
- **Vertical Pod Autoscaler (VPA):** Recommends or adjusts CPU and memory requests, depending on configuration and operating mode.
- **Cluster autoscaling:** Adds or removes worker nodes when supported by the environment and configured conditions.

These mechanisms solve different problems and can work together.

### 6.5 Horizontal Pod Autoscaler

An HPA can scale a Deployment based on CPU utilization, memory utilization when configured and supported, or other metrics.

Example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

This configuration requests a minimum of two replicas, allows up to ten, and targets average CPU utilization of 70% of the Pods' CPU requests.

The example requires suitable resource requests and a functioning metrics pipeline.

### 6.6 Why Autoscaling May Not Solve an Incident

Autoscaling takes time and depends on available capacity.

It may not solve the problem when:

- Traffic rises faster than new replicas can become ready.
- The cluster has no capacity for additional Pods.
- A downstream database is already overloaded.
- The bottleneck is a shared dependency.
- The workload is limited by an external API.
- The scaling metric does not reflect the actual bottleneck.

Scaling the application without scaling or protecting its dependencies may shift the failure elsewhere.

### 6.7 SRE Considerations

- Are resource requests based on real workload measurements?
- Are CPU throttling and memory pressure monitored?
- Does autoscaling have sufficient headroom?
- Are scaling thresholds appropriate for the application's response time?
- Can dependencies handle the maximum number of replicas?
- Does the cluster have capacity to schedule additional Pods?

**Key principle:** Autoscaling increases capacity only when the bottleneck can be relieved by adding resources.

---

## 7. Capacity Planning and Load Balancing

### 7.1 What Is Capacity Planning?

Capacity planning estimates the resources needed to operate a service under expected and peak workloads.

It considers:

- Request volume.
- CPU and memory requirements.
- Network throughput.
- Storage capacity and I/O.
- Database connections.
- Queue throughput.
- Traffic growth.
- Failure scenarios.
- Deployment and scaling overhead.

The goal is to provide enough capacity for normal operation and a reasonable safety margin for unexpected demand or component failures.

### 7.2 Capacity Headroom

Capacity headroom is the available capacity above current demand.

For example, if a service can safely handle 1,000 requests per second and currently receives 700 requests per second, it has 300 requests per second of nominal spare capacity.

However, that does not mean it can always sustain the extra capacity under every workload. Request complexity, latency targets, dependency constraints, and traffic distribution also matter.

SREs should evaluate capacity under realistic workloads and failure conditions.

### 7.3 What Is Load Balancing?

Load balancing distributes incoming traffic across multiple backends.

Common approaches include:

- Round robin.
- Least connections.
- Weighted routing.
- Health-aware routing.
- Consistent hashing.
- Geographic or latency-based routing.

The appropriate strategy depends on the application, session behavior, backend capabilities, and traffic patterns.

### 7.4 Load Balancing in Kubernetes

A Kubernetes Service provides a stable way to reach a set of selected Pods.

```text
                 Client Requests
                       |
                       v
                 Kubernetes Service
                       |
              +--------+--------+
              |        |        |
              v        v        v
            Pod A    Pod B    Pod C
```

Service traffic is routed through Kubernetes networking components to eligible endpoints. Exact implementation details depend on the cluster's networking configuration.

Ingress controllers or Gateway API implementations may provide HTTP or HTTPS routing into a cluster.

Cloud load balancers can distribute traffic between instances, nodes, services, or regions depending on the architecture.

### 7.5 The Relationship Between Load Balancing and Capacity

Load balancing does not create additional capacity. It distributes the work that available backends can handle.

If every backend is saturated, distributing traffic more evenly may not be enough.

Likewise, adding replicas may not help if all requests depend on a single overloaded database.

### 7.6 Capacity During Failure

Consider a service with three replicas, each able to handle 300 requests per second under the required latency target.

Its nominal combined capacity is 900 requests per second, assuming the workload distributes effectively and there are no shared bottlenecks.

If one replica fails, the remaining nominal capacity is 600 requests per second.

If traffic is 750 requests per second, the service may become overloaded after that failure.

This is why capacity planning should consider degraded conditions, not just normal operation.

### 7.7 SRE Considerations

- What is the maximum sustainable request rate?
- What happens when one replica or node fails?
- Is traffic distributed evenly enough for the workload?
- Are slow or unhealthy backends removed from routing?
- Are database connections, queues, and external dependencies included in capacity estimates?
- Is there enough spare capacity during maintenance and deployments?

**Key principle:** A reliable system must have enough capacity to handle both expected demand and the failure scenarios it claims to tolerate.

---

## 8. Deployment Strategies, Rollbacks, and Safe Releases

Deployments are a major source of production risk. A reliable release process reduces the impact of faulty code, configuration changes, and infrastructure updates.

### 8.1 What Is a Deployment Strategy?

A deployment strategy defines how a new application version replaces an existing version.

Common strategies include:

- Rolling deployment.
- Recreate deployment.
- Blue-green deployment.
- Canary deployment.

The best strategy depends on service architecture, risk tolerance, state compatibility, and operational requirements.

### 8.2 Rolling Deployment

A rolling deployment gradually replaces old Pods with new Pods.

```text
Initial:
[v1] [v1] [v1]

During rollout:
[v1] [v1] [v2]

Later:
[v1] [v2] [v2]

Complete:
[v2] [v2] [v2]
```

Advantages:

- Gradual replacement.
- Usually avoids stopping the entire application.
- Supported directly by Kubernetes Deployments.

Risks:

- Old and new versions may run simultaneously.
- The new version may be unhealthy.
- Incorrect rollout settings may reduce available capacity.
- Database or API changes may not be backward compatible.

### 8.3 Recreate Deployment

A recreate strategy terminates the existing version before starting the new one.

Advantages:

- Simple version transition.
- Avoids simultaneous old and new versions.

Risks:

- Can cause downtime.
- Requires the application to tolerate the interruption or have another availability mechanism.

### 8.4 Blue-Green Deployment

Blue-green deployment maintains two environments:

- **Blue:** Current production version.
- **Green:** New version being prepared.

After validation, traffic is switched to green.

```text
                 Traffic
                    |
                    v
               Routing Layer
                /         \
               v           v
          Blue (v1)     Green (v2)
          Current       Candidate
```

Advantages:

- Allows validation before switching traffic.
- Can make traffic reversal straightforward.

Risks:

- Requires extra infrastructure or capacity.
- Shared data changes may make reversal difficult.
- The new environment may behave differently under production traffic.

### 8.5 Canary Deployment

A canary deployment exposes a small portion of traffic to the new version before gradually increasing exposure.

```text
Initial:
v1 = 100% traffic
v2 =   0% traffic

Canary:
v1 = 95% traffic
v2 =  5% traffic

Expanded:
v1 = 75% traffic
v2 = 25% traffic

Complete:
v1 =  0% traffic
v2 = 100% traffic
```

The percentages are illustrative, not universal defaults.

During a canary, monitor:

- Error rate.
- Request latency.
- Saturation.
- Application logs.
- Business-critical behavior.
- Dependency failures.

If the new version breaches defined safety criteria, stop the rollout or revert traffic.

### 8.6 Kubernetes Rolling Update Settings

A Deployment can control how many Pods are updated at a time and how much temporary capacity is allowed.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:stable
          ports:
            - containerPort: 80
```

In this example:

- `maxUnavailable: 0` prevents the rolling update from intentionally reducing the desired number of available replicas during the rollout.
- `maxSurge: 1` permits one additional Pod above the desired replica count during the update.

These settings do not guarantee zero downtime. Successful availability also depends on health checks, scheduling capacity, traffic routing, application behavior, and dependency compatibility.

### 8.7 Rollbacks

A rollback restores a previous application version or deployment configuration.

For Kubernetes Deployments:

```bash
kubectl rollout status deployment/web-app
kubectl rollout history deployment/web-app
kubectl rollout undo deployment/web-app
```

A rollback is most effective when the previous version is still compatible with the current infrastructure and data.

A code rollback may not reverse:

- Database schema changes.
- Data migrations.
- External side effects.
- Configuration changes managed separately.
- Messages already processed by an asynchronous system.

### 8.8 Safe Release Principles

**Small changes**

Smaller changes are generally easier to diagnose and reverse.

**Progressive exposure**

Expose a new version to a limited amount of traffic before a full rollout when the architecture supports it.

**Health-based rollout**

Use readiness checks and rollout monitoring to avoid routing traffic to unready instances.

**Automated validation**

Run tests and deployment checks before and during the release.

**Clear rollback criteria**

Define what conditions require a pause or rollback, such as a sustained rise in errors or latency.

**Backward-compatible changes**

Use migration approaches that support old and new application versions operating during a rollout.

**Observability**

Compare errors, latency, traffic, and resource use before and after the release.

### 8.9 SRE Considerations

- Can the application remain available during a rollout?
- Can old and new versions coexist safely?
- Is sufficient capacity available for surge Pods?
- Are rollout health and service-level indicators monitored?
- Can the previous version run against the current database schema?
- Is there a tested procedure for pausing, rolling back, or routing traffic away from a faulty release?

**Key principle:** A safe deployment is not just a successful update. It is a controlled change with measurable health criteria and a viable recovery path.

---

## 9. Putting It All Together

Consider a web application deployed on Kubernetes in a cloud environment.

The application receives a sudden increase in traffic. CPU usage rises, response times increase, and some requests begin to fail.

A reliable architecture and SRE response might involve the following:

1. **Cloud architecture:** Application replicas are distributed across multiple availability zones.
2. **Load balancing:** Traffic is routed to healthy application replicas.
3. **Resource management:** CPU and memory requests and limits reflect observed workload requirements.
4. **Autoscaling:** The HPA increases replicas when configured metrics meet scaling conditions.
5. **Capacity planning:** Worker nodes have enough capacity to schedule new Pods.
6. **Health checks:** Readiness probes prevent unready replicas from receiving normal Service traffic.
7. **Failure isolation:** Timeouts and resource limits prevent some failures from consuming unlimited resources.
8. **Dependency protection:** Database connections and downstream requests are bounded so scaling the application does not overwhelm shared dependencies.
9. **Safe releases:** A recent deployment can be paused or rolled back if the evidence links the regression to the new version.
10. **Disaster recovery:** Backups and tested recovery procedures protect against larger failures that ordinary replica replacement cannot solve.

The key is that reliability comes from multiple mechanisms working together, not from Kubernetes or cloud infrastructure alone.

### Example Incident Investigation

A production Deployment suddenly has several unavailable Pods.

An SRE might investigate in this order:

```text
1. Confirm customer impact
   |
   v
2. Check Deployment and Service health
   |
   v
3. Inspect Pod status and recent events
   |
   v
4. Check logs, restarts, CPU, and memory
   |
   v
5. Check node capacity and scheduling failures
   |
   v
6. Check recent deployments and configuration changes
   |
   v
7. Mitigate the immediate cause
   |
   v
8. Verify recovery using service-level indicators
   |
   v
9. Document findings and prevent recurrence
```

The investigation should follow evidence rather than assume that the first visible symptom is the root cause.

---

## 10. Key Takeaways

- Cloud hosting alone does not guarantee application availability.
- Fault domains help identify which components may fail together.
- Multi-zone and multi-region architectures address different failure scenarios.
- High availability and disaster recovery have different goals.
- Backups and replicas serve different purposes.
- Kubernetes schedules Pods and reconciles desired state, but successful recovery requires capacity and healthy workloads.
- `CrashLoopBackOff` is a restart-backoff condition, not a root cause.
- OOMKills can indicate memory limits, leaks, workload spikes, or other memory-pressure problems.
- Startup, readiness, and liveness probes have distinct responsibilities.
- Resource requests affect scheduling; limits constrain resource use.
- Autoscaling is not a substitute for capacity planning or dependency protection.
- Load balancing distributes traffic but does not create extra capacity.
- Safe deployment strategies reduce the risk of releasing faulty changes.
- Rollbacks may not reverse database or external side effects.
- Reliability must be verified through observability, realistic failure testing, and recovery exercises.

---

## 11. Learning Resources

### Official Documentation

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Kubernetes Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
