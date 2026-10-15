# Site Reliability Engineering

**A personal journey into Site Reliability Engineering.**

I'm exploring Site Reliability Engineering (SRE) to deepen my understanding of how reliable, scalable, and resilient systems are built, operated, and maintained.

With a background in DevOps, cloud infrastructure, Kubernetes, CI/CD, and Infrastructure as Code, I've become increasingly interested in the engineering principles behind production reliability understanding failures, improving observability, reducing operational toil, and building systems that behave predictably under pressure.

This repository is where I document that journey: the concepts I study, the resources I learn from, the experiments I run, and the lessons I discover along the way.

---

## What I'm Exploring

### 01. SRE Fundamentals
- [ ] What is Site Reliability Engineering?
- [ ] SRE vs DevOps vs Platform Engineering
- [ ] The role and responsibilities of an SRE
- [ ] Reliability as an engineering discipline
- [ ] Toil, automation, and operational excellence
- [ ] The principles behind Google's SRE approach

### 02. Service Reliability
- [ ] Service Level Indicators (SLIs)
- [ ] Service Level Objectives (SLOs)
- [ ] Service Level Agreements (SLAs)
- [ ] Error budgets and release decisions
- [ ] Availability, reliability, and durability
- [ ] Latency percentiles: p50, p95, and p99
- [ ] The four golden signals: latency, traffic, errors, and saturation

### 03. Monitoring and Observability
- [ ] Monitoring vs observability
- [ ] Metrics, logs, and distributed traces
- [ ] Prometheus and Grafana
- [ ] OpenTelemetry and distributed tracing
- [ ] Alerting strategies and actionable alerts
- [ ] Dashboards and service health indicators
- [ ] Troubleshooting with telemetry

### 04. Linux and Networking
- [ ] Linux processes, services, and system resources
- [ ] CPU, memory, disk, and I/O troubleshooting
- [ ] DNS resolution and connectivity
- [ ] TCP/IP, ports, routing, and firewalls
- [ ] HTTP, HTTPS, TLS, and status codes
- [ ] Load balancers, reverse proxies, and health checks
- [ ] Diagnosing slow or unavailable services

### 05. Incident Management
- [ ] Incident detection, triage, and response
- [ ] Severity levels and escalation
- [ ] Mean Time to Detect (MTTD)
- [ ] Mean Time to Recover (MTTR)
- [ ] Root Cause Analysis (RCA)
- [ ] Blameless postmortems
- [ ] Incident documentation and runbooks
- [ ] Learning from recurring incidents

### 06. Distributed Systems and Resilience
- [ ] Service dependencies and failure modes
- [ ] Timeouts, retries, and exponential backoff
- [ ] Circuit breakers and bulkheads
- [ ] Graceful degradation and fault isolation
- [ ] Message queues and asynchronous processing
- [ ] Database reliability and connection pooling
- [ ] Caching and rate limiting
- [ ] Avoiding cascading failures

### 07. Cloud, Containers, and Kubernetes
- [ ] Cloud availability and fault domains
- [ ] High availability and disaster recovery
- [ ] Kubernetes scheduling and resource management
- [ ] Pod failures, OOMKills, and CrashLoopBackOff
- [ ] Startup, readiness, and liveness probes
- [ ] Requests, limits, and autoscaling
- [ ] Capacity planning and load balancing
- [ ] Deployment strategies, rollbacks, and safe releases

### 08. Automation and Infrastructure
- [ ] Bash and Python for operational automation
- [ ] Infrastructure as Code with Terraform
- [ ] CI/CD reliability and deployment automation
- [ ] GitOps and configuration management
- [ ] Automated health checks and remediation
- [ ] Secrets management and access control
- [ ] Reducing repetitive operational work

### 09. Capacity, Performance, and Cost
- [ ] Load testing and performance analysis
- [ ] Resource utilization and saturation
- [ ] Capacity forecasting
- [ ] Horizontal vs vertical scaling
- [ ] Bottleneck identification
- [ ] Cloud cost optimization
- [ ] Balancing reliability, performance, and cost

---

## Learning Resources

A growing collection of documentation, books, articles, videos, courses, and other resources I use to understand SRE concepts and apply them in practice.

### Books and Structured Guides

| Resource | Focus | Link |
|---|---|---|
| Google SRE Book | Core SRE principles and practices | [Read online](https://sre.google/sre-book/table-of-contents/) |
| Google SRE Workbook | Practical SRE implementation | [Read online](https://sre.google/workbook/table-of-contents/) |
| The Site Reliability Workbook: Companion resources | Applying SRE principles | [Google SRE](https://sre.google/) |

### Official Documentation

| Resource | Focus | Link |
|---|---|---|
| Prometheus | Metrics collection and alerting | [Documentation](https://prometheus.io/docs/introduction/overview/) |
| Grafana | Dashboards and visualization | [Documentation](https://grafana.com/docs/) |
| OpenTelemetry | Metrics, logs, and traces | [Documentation](https://opentelemetry.io/docs/) |
| Kubernetes | Container orchestration and troubleshooting | [Documentation](https://kubernetes.io/docs/home/) |
| HashiCorp Terraform | Infrastructure as Code | [Documentation](https://developer.hashicorp.com/terraform/docs) |
| Google Cloud Architecture Framework | Reliability and operational excellence | [Documentation](https://cloud.google.com/architecture/framework) |
| Microsoft Azure Well-Architected Framework | Reliability, operations, and cloud design | [Documentation](https://learn.microsoft.com/en-us/azure/well-architected/) |
| AWS Well-Architected Framework | Reliable cloud architecture | [Documentation](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) |

### Linux and Networking

| Resource | Focus | Link |
|---|---|---|
| Linux man-pages | Linux commands and system calls | [Documentation](https://www.kernel.org/doc/man-pages/) |
| Julia Evans / Wizard Zines | Linux, networking, and systems concepts | [Website](https://wizardzines.com/) |
| Cloudflare Learning Center | DNS, HTTP, TLS, and networking | [Explore](https://www.cloudflare.com/learning/) |

### Practical Learning and Experiments

| Resource | Focus | Link |
|---|---|---|
| Killercoda | Interactive infrastructure labs | [Explore](https://killercoda.com/) |
| Play with Docker | Container experiments | [Explore](https://labs.play-with-docker.com/) |
| Kubernetes interactive tutorials | Deploying and operating
