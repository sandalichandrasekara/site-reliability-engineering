# 08. Automation and Infrastructure

## Overview

Automation and infrastructure management are essential parts of Site Reliability Engineering. As systems grow, manually managing servers, deploying applications, checking service health, and responding to repetitive incidents becomes inefficient and error-prone.

SRE teams use automation to make operational tasks consistent, repeatable, and scalable. Tools and practices such as Bash, Python, Terraform, CI/CD pipelines, GitOps, and configuration management help reduce manual effort while improving reliability.

However, automation must be designed carefully. An incorrect script, an unsafe deployment pipeline, or a misconfigured infrastructure change can affect many systems at once.

The goal is not simply to automate everything. It is to automate the right tasks with appropriate validation, observability, access controls, and recovery mechanisms.

This section explores how automation and infrastructure practices help SRE teams manage systems reliably and reduce repetitive operational work.

## Learning Objectives

By the end of this section, I aim to understand:

- How Bash and Python can automate operational tasks.
- How Infrastructure as Code (IaC) manages infrastructure through version-controlled definitions.
- How Terraform provisions and manages infrastructure.
- How CI/CD pipelines improve deployment consistency and reliability.
- How GitOps and configuration management support controlled infrastructure changes.
- How automated health checks and remediation work.
- How secrets management and access control protect infrastructure.
- How to identify repetitive operational work that should be automated.
- How to design automation that is observable, safe, maintainable, and recoverable.

## Table of Contents

1. [Bash and Python for Operational Automation](#1-bash-and-python-for-operational-automation)
2. [Infrastructure as Code with Terraform](#2-infrastructure-as-code-with-terraform)
3. [CI/CD Reliability and Deployment Automation](#3-cicd-reliability-and-deployment-automation)
4. [GitOps and Configuration Management](#4-gitops-and-configuration-management)
5. [Automated Health Checks and Remediation](#5-automated-health-checks-and-remediation)
6. [Secrets Management and Access Control](#6-secrets-management-and-access-control)
7. [Reducing Repetitive Operational Work](#7-reducing-repetitive-operational-work)
8. [Putting It All Together](#8-putting-it-all-together)
9. [Key Takeaways](#9-key-takeaways)
10. [Learning Resources](#10-learning-resources)
11. [Knowledge Check](#11-knowledge-check)
12. [Practical Scenarios](#12-practical-scenarios)
13. [Hands-On Labs](#13-hands-on-labs)

---

## 1. Bash and Python for Operational Automation

### 1.1 What Is Operational Automation?

Operational automation uses scripts and tools to perform routine infrastructure and service-management tasks with minimal manual intervention.

Examples include:

- Checking whether services are running.
- Collecting CPU, memory, and disk information.
- Searching logs for errors.
- Rotating or cleaning up old files.
- Testing application endpoints.
- Processing monitoring data.
- Deploying applications.
- Creating incident reports.
- Checking backup results.

Automation helps reduce repetitive work and ensures that the same procedure can be performed consistently.

### 1.2 Why Automation Matters in SRE

Manual operations can introduce several problems:

- Human error during repetitive tasks.
- Inconsistent execution of procedures.
- Slow response to common incidents.
- Dependence on individual team members.
- Difficulty scaling operational work as systems grow.

Automation can improve consistency and speed, but it also introduces risks if scripts are incorrect or operate without suitable safeguards.

A good automation workflow should be repeatable, understandable, observable, and safe to run.

### 1.3 Bash for Operational Tasks

Bash is a command-line shell commonly used for automating tasks in Linux and Unix-like environments.

It is useful for:

- Executing system commands.
- Combining command-line utilities.
- Managing files and directories.
- Checking processes.
- Running scheduled jobs.
- Automating deployment steps.
- Connecting existing tools into a workflow.

Example: checking disk usage.

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

echo "Checking disk usage..."

df -h

echo "Disk usage check completed."
```

Explanation:

- `#!/usr/bin/env bash` selects Bash through the environment.
- `set -e` causes the script to exit when an eligible command fails.
- `set -u` treats unset variables as errors.
- `set -o pipefail` helps pipelines report failures from commands other than the last command.

These options improve error handling but do not replace explicit validation. Some commands legitimately return non-zero exit codes, so scripts should handle expected failures deliberately.

### 1.4 Example: Checking a Service Endpoint

A simple Bash script can check whether an HTTP endpoint responds successfully.

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

URL="https://example.com/"

if curl --fail --silent --show-error \
    --max-time 5 \
    "$URL" \
    --output /dev/null; then
    echo "Endpoint is responding."
else
    echo "Endpoint check failed."
    exit 1
fi
```

This script:

1. Sends an HTTP request.
2. Sets a request timeout.
3. Suppresses the response body.
4. Returns a non-zero exit status when the check fails.

A single failed request does not always mean a service is down. In production, health checks should account for transient network errors, retry policies, and the impact of the check itself.

### 1.5 Python for Operational Automation

Python is a general-purpose programming language suitable for more complex automation.

It is useful for:

- Calling APIs.
- Parsing structured logs.
- Processing JSON and YAML.
- Collecting metrics.
- Generating reports.
- Integrating cloud services.
- Automating infrastructure workflows.
- Implementing validation and decision logic.

Python is often preferable when automation requires structured data, multiple conditions, reusable functions, or complex error handling.

### 1.6 Example: Checking an HTTP Endpoint with Python

```python
from urllib.request import urlopen
from urllib.error import URLError

URL = "https://example.com/"

try:
    with urlopen(URL, timeout=5) as response:
        if 200 <= response.status < 400:
            print(f"Endpoint responded: HTTP {response.status}")
        else:
            raise RuntimeError(
                f"Unexpected HTTP status: {response.status}"
            )

except (URLError, TimeoutError, RuntimeError) as error:
    print(f"Endpoint check failed: {error}")
    raise SystemExit(1)
```

This example demonstrates:

- Making an HTTP request.
- Setting a timeout.
- Checking the response status.
- Handling network and validation errors.
- Returning a failure exit code.

The example is suitable for learning. Production health checks should also consider authentication, redirects, TLS verification, expected response content, and how failures are reported.

### 1.7 Bash vs. Python

| Feature | Bash | Python |
|---|---|---|
| Best suited for | Shell commands and system workflows | Structured logic and data processing |
| System command integration | Excellent | Supported through libraries and subprocesses |
| API integration | Possible, often less convenient | Extensive library support |
| JSON processing | Possible with external tools | Built-in JSON support |
| Complex error handling | Can become difficult | Generally easier to structure |
| Large automation projects | Can become difficult to maintain | Usually easier to organize and test |

A practical rule is to use Bash for straightforward shell workflows and Python when the task requires substantial data processing, complex logic, or API integration.

### 1.8 Automation Best Practices

**Make scripts repeatable**

Running a script twice should not cause unexpected damage. Where possible, design scripts to be idempotent, meaning repeated execution produces the same intended end state.

**Validate inputs**

Check required parameters, file paths, environment variables, and configuration before making changes.

**Handle errors explicitly**

Do not silently ignore failed commands or API requests.

**Use timeouts**

Network calls and external commands should not wait indefinitely.

**Log useful information**

Record what the script attempted, what succeeded, and what failed. Avoid logging credentials or other secrets.

**Use dry-run modes when appropriate**

Allow users to inspect planned changes before applying them.

**Test before production use**

Start with a development environment and expand gradually.

### 1.9 SRE Considerations

- Does the script handle partial failures?
- Can it be safely run more than once?
- Does it distinguish between a real failure and an expected condition?
- Is its output useful during an incident?
- Can the task be tested without affecting production?
- Does it require more access than necessary?

**Key principle:** Good automation reduces human error without creating uncontrolled machine-driven failures.

---

## 2. Infrastructure as Code with Terraform

### 2.1 What Is Infrastructure as Code?

Infrastructure as Code (IaC) is the practice of defining infrastructure through machine-readable files instead of relying entirely on manual configuration.

IaC can manage resources such as:

- Virtual machines.
- Networks and subnets.
- Firewalls and security groups.
- Load balancers.
- Databases.
- Storage resources.
- Kubernetes infrastructure.
- Identity and access policies.

Infrastructure definitions can be stored in Git, reviewed, tested, and applied consistently.

### 2.2 Why IaC Matters in SRE

Manually created infrastructure can become inconsistent across environments.

For example, a production server might have a network rule that does not exist in staging. Over time, undocumented manual changes can make the infrastructure difficult to reproduce.

IaC helps by providing:

- Version-controlled infrastructure definitions.
- Repeatable provisioning.
- Reviewable infrastructure changes.
- Better visibility into intended configuration.
- Easier environment recreation.
- Reduced manual configuration errors.

IaC does not automatically guarantee secure or reliable infrastructure. The definitions still need review, testing, access control, and ongoing maintenance.

### 2.3 What Is Terraform?

Terraform is an Infrastructure as Code tool that uses declarative configuration to manage infrastructure through providers.

A Terraform configuration describes the desired resources. Terraform compares the configuration with its state and the infrastructure it can observe, then proposes changes.

The usual workflow is:

```text
Write Configuration
        |
        v
Initialize Terraform
        |
        v
Validate Configuration
        |
        v
Review Execution Plan
        |
        v
Apply Approved Changes
        |
        v
Verify Infrastructure
```

### 2.4 Important Terraform Concepts

| Concept | Meaning |
|---|---|
| Provider | Integrates Terraform with a platform or service |
| Resource | Infrastructure object Terraform manages |
| Variable | Input used to customize configuration |
| Output | Value exposed after evaluation or application |
| Module | Reusable collection of Terraform configuration |
| State | Records Terraform's understanding of managed resources |
| Plan | Proposed set of infrastructure changes |
| Backend | Defines where Terraform state is stored and how it is managed |

### 2.5 Example Terraform Configuration

The following example creates a local file to demonstrate Terraform's workflow without provisioning cloud infrastructure.

```hcl
terraform {
  required_version = ">= 1.5.0"
}

resource "local_file" "sre_notes" {
  filename = "${path.module}/sre-notes.txt"
  content  = "Infrastructure managed with Terraform.\n"
}

output "created_file" {
  value = local_file.sre_notes.filename
}
```

This example uses the Local provider, which must be available to Terraform.

Initialize the working directory:

```bash
terraform init
```

Validate the configuration:

```bash
terraform validate
```

Review the proposed changes:

```bash
terraform plan
```

Apply the changes:

```bash
terraform apply
```

Inspect Terraform's state:

```bash
terraform state list
```

For this local demonstration, Terraform creates a text file. In a real infrastructure project, the resources might instead be cloud networks, virtual machines, or managed databases.

### 2.6 Terraform State

Terraform state tracks the relationship between configuration and managed resources.

State is important because Terraform uses it to understand which resources it manages and what changes may be required.

For team environments, remote state storage can support collaboration and state locking where the chosen backend provides those capabilities.

Important precautions include:

- Restricting state access.
- Protecting state files from unauthorized changes.
- Avoiding unnecessary state exposure in logs.
- Using an appropriate remote backend for shared environments.
- Backing up important state.
- Never committing sensitive state files to a public repository.

Terraform state may contain sensitive information even when the configuration itself does not.

### 2.7 Terraform and Idempotency

Terraform is designed to converge managed resources toward the declared configuration.

When the infrastructure already matches the configuration, a subsequent plan should generally show no changes, assuming the configuration, provider behavior, and observed infrastructure remain stable.

However, drift can occur when resources are changed manually outside Terraform or when external conditions change.

For example:

```text
Terraform Configuration
          |
          v
Expected Infrastructure
          |
          | Compare
          v
Actual Infrastructure
          |
          +---- Matches ------> No change needed
          |
          +---- Differs ------> Review planned changes
```

Review unexpected changes before applying them, especially when they involve deletion, replacement, networking, or production data.

### 2.8 Terraform Best Practices

- Store configuration in version control.
- Review `terraform plan` before applying changes.
- Separate environments and manage their state carefully.
- Use reusable modules where they improve consistency.
- Pin provider and module versions appropriately.
- Protect state and credentials.
- Avoid hardcoding secrets in configuration.
- Use automated validation and policy checks.
- Restrict who can apply production changes.
- Detect and investigate infrastructure drift.

### 2.9 SRE Considerations

- Can infrastructure be recreated consistently?
- Is state protected and recoverable?
- Are changes reviewed before application?
- Could a configuration change replace a critical resource?
- Are development, staging, and production environments appropriately separated?
- Can the team detect manual changes that bypass the normal workflow?

**Key principle:** Infrastructure changes should be reviewable, repeatable, and controlled like application code.

---

## 3. CI/CD Reliability and Deployment Automation

### 3.1 What Is CI/CD?

CI/CD refers to practices that automate building, testing, and delivering software.

**Continuous Integration (CI)** involves regularly integrating changes and validating them through automated checks.

**Continuous Delivery** keeps changes in a deployable state so they can be released through a controlled process.

**Continuous Deployment** automatically releases changes that pass the required checks, without a separate manual production-release approval at the final stage.

The exact workflow depends on the organization and its risk requirements.

### 3.2 A Typical CI/CD Pipeline

```text
Developer Pushes Code
         |
         v
    Source Checkout
         |
         v
    Static Analysis
         |
         v
     Automated Tests
         |
         v
    Build Application
         |
         v
   Security and Quality Checks
         |
         v
   Publish Versioned Artifact
         |
         v
   Deploy to Test Environment
         |
         v
   Validate Application Health
         |
         v
    Release to Production
         |
         v
   Monitor and Verify Results
```

A reliable pipeline detects problems early and makes releases consistent.

### 3.3 Why CI/CD Reliability Matters

A pipeline is part of the production delivery system. If it is unreliable, it can slow releases, create inconsistent deployments, or introduce faulty changes.

Common pipeline problems include:

- Flaky tests.
- Missing dependency checks.
- Unpinned tool versions.
- Incorrect environment configuration.
- Unavailable build infrastructure.
- Exposed credentials.
- Deployment steps that partially succeed.
- Insufficient validation after deployment.

### 3.4 Build Once, Promote the Same Artifact

A useful practice is to build a versioned artifact once and promote that same artifact through environments.

For example:

```text
Source Code
    |
    v
Build Artifact: app-1.4.2
    |
    +----> Test Environment
    |
    +----> Staging Environment
    |
    +----> Production Environment
```

This reduces the risk of accidentally testing one build and deploying a different build.

Container images should ideally be referenced by immutable digests when strict artifact identity is required.

### 3.5 Deployment Validation

A successful pipeline command does not necessarily mean the application is healthy.

After deployment, validate:

- Deployment completion.
- Application readiness.
- Error rates.
- Request latency.
- Resource utilization.
- Dependency health.
- Relevant business or functional checks.

For a Kubernetes Deployment, a basic check is:

```bash
kubectl rollout status deployment/web-app
```

This checks rollout progress. Additional health checks and service-level indicators are still needed to verify the release's impact on users.

### 3.6 CI/CD Safety Controls

**Automated testing**

Catch defects before they reach production.

**Artifact versioning**

Make the deployed version identifiable and traceable.

**Approval controls**

Require review or authorization for changes with significant production impact.

**Environment separation**

Prevent accidental use of production credentials or configuration in development workflows.

**Progressive delivery**

Use canaries or staged rollouts when appropriate.

**Rollback or recovery procedures**

Prepare a recovery path before the release begins.

**Pipeline observability**

Track failed builds, deployment duration, test failures, and deployment outcomes.

### 3.7 Measuring Pipeline Reliability

Useful measurements include:

- Build success rate.
- Pipeline failure rate.
- Test flakiness.
- Time from code change to deployment.
- Deployment frequency.
- Change failure rate.
- Time to restore service after a failed change.

These metrics help teams identify bottlenecks and release risks. They should be interpreted in context rather than used to encourage unsafe deployment speed.

### 3.8 SRE Considerations

- Are tests reliable and repeatable?
- Can the deployed artifact be traced back to source code?
- Does the pipeline stop when critical checks fail?
- Are production credentials protected?
- Can a failed deployment be detected quickly?
- Is recovery possible if the pipeline itself becomes unavailable?

**Key principle:** A reliable deployment pipeline makes changes predictable, traceable, and recoverable.

---

## 4. GitOps and Configuration Management

### 4.1 What Is GitOps?

GitOps is an operational approach in which Git repositories hold the declared desired state of systems, and automated controllers reconcile the running environment toward that state.

Instead of manually changing a live environment and hoping the change is preserved, teams update the configuration in Git and use an approved reconciliation workflow.

GitOps is commonly used with Kubernetes and infrastructure configuration.

### 4.2 How GitOps Works

```text
Engineer Changes Configuration
              |
              v
        Pull Request
              |
              v
       Review and Approval
              |
              v
         Merge to Git
              |
              v
      GitOps Controller
              |
              v
    Reconcile Running System
              |
              v
     Verify Actual State
```

The controller periodically or continuously checks the desired state and attempts to reconcile differences.

A GitOps controller may also detect configuration drift and correct it according to its configuration and permissions.

### 4.3 Benefits of GitOps

- Configuration history and change traceability.
- Reviewable changes through pull requests.
- Repeatable environment configuration.
- Clearer separation between desired and actual state.
- Automated reconciliation.
- Easier auditing of configuration changes.

### 4.4 What Is Configuration Management?

Configuration management maintains systems in a known and intended configuration.

It may include:

- Installing packages.
- Managing operating-system settings.
- Creating users and groups.
- Configuring services.
- Managing files and permissions.
- Applying security baselines.
- Enforcing configuration consistency.

Tools such as Ansible are commonly used for configuration management.

Terraform primarily provisions and manages infrastructure resources, while configuration-management tools often focus on configuring systems and software. Their responsibilities can overlap, depending on the architecture.

### 4.5 Example Ansible Playbook

The following example ensures that a package is installed on applicable Debian-based hosts.

```yaml
---
- name: Configure a web server
  hosts: webservers
  become: true

  tasks:
    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
```

The playbook describes the desired condition: Nginx should be installed.

If it is already installed, a properly designed task should not needlessly reinstall it. This is an example of idempotent configuration management.

### 4.6 GitOps vs. Traditional Manual Changes

| Approach | Characteristics |
|---|---|
| Manual changes | Direct changes to running systems; may be difficult to trace |
| Script-based deployment | Automates changes but may still lack a central desired-state model |
| Infrastructure as Code | Defines infrastructure in version-controlled configuration |
| GitOps | Uses Git as the declared source of desired state and automated reconciliation |

These approaches can be combined. For example, Terraform can provision a Kubernetes cluster while a GitOps controller manages application configuration inside it.

### 4.7 Managing Drift

Configuration drift occurs when the actual system differs from the intended configuration.

Causes include:

- Manual production changes.
- Incomplete deployment operations.
- Changes made by external automation.
- Differences between environments.
- Untracked configuration edits.

A reliable process should detect drift, assess its cause, and reconcile it through an approved workflow.

Not every difference should be overwritten automatically. Some systems need an investigation step before correction, especially when the change may be intentional or safety-critical.

### 4.8 SRE Considerations

- Is Git the trusted source of desired configuration?
- Are pull requests reviewed and validated?
- Can unauthorized manual changes be detected?
- Could automatic reconciliation undo an emergency change?
- Are configuration secrets stored securely?
- Is the controller's access limited to the resources it needs?

**Key principle:** GitOps improves consistency when the desired state is correct, protected, and supported by safe reconciliation rules.

---

## 5. Automated Health Checks and Remediation

### 5.1 What Are Automated Health Checks?

Automated health checks regularly evaluate whether a system is functioning as expected.

Examples include:

- HTTP endpoint checks.
- Process availability checks.
- Database connectivity checks.
- Queue backlog checks.
- Disk-capacity checks.
- Certificate-expiration checks.
- Backup completion checks.
- Kubernetes readiness checks.

Health checks can provide early warning of failures and support automated responses.

However, a health check is only as useful as the condition it measures.

### 5.2 Types of Health Checks

**Liveness checks**

Determine whether a process should be restarted, as with Kubernetes liveness probes.

**Readiness checks**

Determine whether a component should receive normal traffic.

**Synthetic checks**

Simulate a user or client operation to verify an important workflow.

**Dependency checks**

Assess whether required downstream systems are available or usable.

**Infrastructure checks**

Inspect capacity, node health, disk space, certificates, and other infrastructure conditions.

### 5.3 Health Checks vs. Monitoring

Health checks evaluate a particular condition at a point in time or at regular intervals.

Monitoring collects and analyzes signals over time, such as:

- Metrics.
- Logs.
- Traces.
- Events.
- Alerts.
- Service-level indicators.

Health checks can feed monitoring systems, but they are not a replacement for broader observability.

### 5.4 What Is Automated Remediation?

Automated remediation takes a predefined action when a known failure condition is detected.

Examples include:

- Restarting a failed process.
- Replacing an unhealthy instance.
- Scaling out a workload.
- Clearing a safely identified temporary resource.
- Failing over to a healthy endpoint.
- Opening an incident or notifying an on-call engineer.

Automated remediation is most appropriate when the failure condition is well understood and the action has predictable consequences.

### 5.5 A Safe Remediation Workflow

```text
Health Check Detects Failure
             |
             v
     Validate the Signal
             |
             v
   Is the Condition Known?
         /          \
       Yes           No
        |             |
        v             v
  Apply Bounded     Alert and
  Remediation       Investigate
        |
        v
   Verify Recovery
        |
        v
   Record the Action
```

### 5.6 Example: A Bounded Health Check

```bash
#!/usr/bin/env bash

set -u

URL="https://example.com/"
LOG_PREFIX="endpoint-check"

if curl --fail --silent --show-error \
    --max-time 5 \
    "$URL" \
    --output /dev/null; then
    printf '%s: endpoint healthy\n' "$LOG_PREFIX"
else
    printf '%s: endpoint check failed\n' "$LOG_PREFIX" >&2
    exit 1
fi
```

This example reports the endpoint's status. It does not automatically restart a service or modify infrastructure.

That separation is intentional: detection and remediation should be designed independently so that an unreliable signal cannot trigger an unsafe action.

### 5.7 Risks of Automated Remediation

Automation can amplify a problem if it reacts to misleading signals.

Examples include:

- Restarting every instance during a brief network slowdown.
- Scaling out an application until a database is overwhelmed.
- Repeatedly recreating resources without fixing the cause.
- Deleting files that were incorrectly classified as temporary.
- Triggering repeated failovers between unstable systems.

These problems can create remediation loops or cascading failures.

### 5.8 Safe Remediation Practices

- Verify the triggering condition.
- Prefer narrowly scoped actions.
- Set retry limits and cooldown periods.
- Make actions idempotent where possible.
- Record the action and its outcome.
- Alert a human when remediation fails.
- Include a manual override or disable mechanism.
- Test the automation in a controlled environment.
- Measure whether the action actually improves service health.

**Key principle:** Automated remediation should reduce the impact of known failures, not blindly react to every alert.

---

## 6. Secrets Management and Access Control

### 6.1 What Are Secrets?

Secrets are sensitive values used to authenticate, authorize, or establish secure communication.

Examples include:

- API tokens.
- Database passwords.
- Cloud credentials.
- Private keys.
- Signing keys.
- Certificates and private certificate material.
- Application service credentials.

If secrets are exposed, an attacker or unauthorized user may gain access to infrastructure, data, or production services.

### 6.2 Common Secrets Management Mistakes

- Committing credentials to Git.
- Embedding secrets in container images.
- Writing secrets to application logs.
- Passing credentials through insecure channels.
- Giving every service the same credentials.
- Keeping old credentials active indefinitely.
- Using production secrets in development environments.
- Storing plaintext secrets in CI/CD variables without suitable protections.

Removing a secret from the latest Git commit does not guarantee that it has been removed from the repository's history, logs, caches, or cloned copies.

If a credential is exposed, it should be treated as compromised and rotated or revoked promptly.

### 6.3 What Is a Secrets Manager?

A secrets manager is a system for securely storing, accessing, and often rotating sensitive values.

Examples include:

- HashiCorp Vault.
- AWS Secrets Manager.
- Azure Key Vault.
- Google Cloud Secret Manager.
- Kubernetes Secrets, with appropriate security controls.

The right choice depends on the platform, application architecture, operational requirements, and security model.

Kubernetes Secrets are not automatically encrypted in every configuration. Production clusters should use appropriate encryption at rest, restrictive RBAC, secure delivery methods, and controlled access to the API.

### 6.4 What Is Access Control?

Access control determines which identities can perform which actions on which resources.

A useful principle is **least privilege**: grant only the permissions needed for a task.

For example:

- A monitoring service may need permission to read metrics.
- A deployment identity may need permission to update a specific application.
- A developer may need access to a development environment but not production secrets.
- An emergency administrator may have elevated permissions through a controlled process.

### 6.5 Identity and Access Management

Identity and Access Management (IAM) controls identities and their permissions across infrastructure and applications.

Common mechanisms include:

- Role-based access control (RBAC).
- Service accounts or workload identities.
- Multi-factor authentication.
- Short-lived credentials.
- Audit logging.
- Credential rotation.
- Environment-specific permissions.

In Kubernetes, RBAC can limit which resources an identity can read or modify.

### 6.6 Example: Kubernetes Secret

A Kubernetes Secret can represent sensitive configuration such as a database password.

An illustrative manifest is:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  username: app-user
  password: replace-with-a-securely-supplied-value
```

**Important:** This is an educational example. Do not commit real credentials or production secret values to Git. `stringData` accepts plaintext input, and the resulting Kubernetes Secret is not automatically safe merely because it is a Secret object.

For production environments, use a secure provisioning workflow or an external secrets manager and apply appropriate encryption, access restrictions, and auditing.

### 6.7 Secure Automation

Automation often needs access to infrastructure, which means its credentials must be protected.

Recommended practices include:

- Use dedicated identities for automation.
- Limit permissions to required resources.
- Prefer short-lived credentials where supported.
- Retrieve secrets from approved secret stores.
- Avoid printing secrets in logs.
- Rotate credentials according to risk and policy.
- Audit sensitive operations.
- Separate permissions across environments.
- Review who can modify CI/CD pipelines and their secret settings.

### 6.8 SRE Considerations

- Which identities can change production infrastructure?
- Can automation access secrets unrelated to its task?
- Are credentials rotated and revoked appropriately?
- Can secret access be audited?
- What happens if a CI/CD runner is compromised?
- Are emergency permissions controlled and reviewed?

**Key principle:** Automation should be powerful enough to do its job, but no more powerful than necessary.

---

## 7. Reducing Repetitive Operational Work

### 7.1 What Is Repetitive Operational Work?

Repetitive operational work includes tasks that require human effort but do not consistently produce new value.

Examples include:

- Manually restarting the same failed service.
- Repeatedly collecting the same incident logs.
- Copying configuration between environments.
- Manually checking the same health endpoints.
- Repeating standard deployment steps.
- Generating routine reports by hand.
- Responding to recurring alerts that have well-understood causes.

In SRE, reducing repetitive work creates more time for engineering improvements, reliability analysis, and prevention.

### 7.2 What Is Toil?

Toil is operational work that tends to be manual, repetitive, automatable, tactical, and grows with the system or its workload.

Not every operational task is toil. Designing a recovery strategy or investigating a novel incident can be valuable engineering work even when it is performed manually.

Examples of potential toil:

| Task | Possible improvement |
|---|---|
| Manually checking service status | Automated health checks |
| Repeating deployment commands | CI/CD pipeline |
| Provisioning infrastructure by hand | Terraform or another IaC tool |
| Manually updating cluster configuration | GitOps |
| Collecting logs during every incident | Standardized diagnostic scripts |
| Repeating routine access changes | Approved, auditable access workflows |
| Manually generating weekly reports | Automated report generation |

### 7.3 How to Identify Automation Opportunities

Start by documenting repeated tasks and measuring their operational cost.

Useful questions include:

- How often does the task occur?
- How much time does each occurrence require?
- How frequently do mistakes happen?
- Is the task predictable?
- Can the outcome be validated automatically?
- What is the risk if automation makes a mistake?
- Does the task require human judgment?

Tasks that are frequent, predictable, and low-risk are often good initial candidates.

### 7.4 Prioritizing Automation

A simple prioritization model is:

\[
\text{Estimated Annual Time Saved}
=
\text{Occurrences per Year}
\times
\text{Minutes per Occurrence}
\div 60
\]

For example, if a task occurs 20 times each week and takes 10 minutes each time:

\[
20 \times 52 \times 10 \div 60
\approx 173.3 \text{ hours/year}
\]

This estimates the time spent on the task, not the net benefit of automating it. Development, testing, maintenance, and incident risk must also be considered.

A useful prioritization framework considers:

- Frequency.
- Time consumed.
- Error rate.
- Risk reduction.
- Implementation effort.
- Long-term maintenance cost.

### 7.5 The Automation Lifecycle

```text
Identify Repetitive Task
          |
          v
Document Current Workflow
          |
          v
Assess Risk and Benefits
          |
          v
Design and Implement Automation
          |
          v
Test in a Safe Environment
          |
          v
Deploy with Monitoring
          |
          v
Measure Time Saved and Failures
          |
          v
Maintain and Improve
```

Automation is not a one-time activity. Scripts, dependencies, APIs, permissions, and operational requirements change over time.

### 7.6 Avoid Automating a Broken Process

Automating a poorly understood workflow can make a bad process faster and more difficult to control.

Before automation:

1. Understand the root cause of the repeated task.
2. Simplify unnecessary steps.
3. Standardize the expected outcome.
4. Define error handling.
5. Decide when human intervention is necessary.
6. Measure whether the automation actually improves the process.

For example, automatically restarting an application every time it slows down may hide a memory leak instead of fixing it.

### 7.7 Measuring the Benefits

Possible measurements include:

- Time spent on repetitive tasks.
- Manual intervention frequency.
- Automation success rate.
- Failed automation runs.
- Mean time to detect and recover from common failures.
- Number of recurring incidents.
- Time spent maintaining automation.

The aim is not to maximize the number of scripts. It is to reduce operational effort while improving reliability and safety.

**Key principle:** Automate repetitive work when the expected long-term benefit exceeds the cost and risk of building and maintaining the automation.

---

## 8. Putting It All Together

Consider a service that runs in Kubernetes and is deployed through a CI/CD pipeline.

The team wants to create a consistent process for provisioning infrastructure, releasing changes, checking service health, and recovering from known failures.

A reliable workflow might look like this:

```text
          Infrastructure Code
                  |
                  v
         Review and Validation
                  |
                  v
          Terraform Plan
                  |
                  v
        Approved Infrastructure
                  |
                  v
             GitOps Config
                  |
                  v
           CI/CD Pipeline
                  |
                  v
       Build and Test Application
                  |
                  v
        Deploy Versioned Artifact
                  |
                  v
        Health Checks and Metrics
                  |
                  v
       Automated or Manual Response
                  |
                  v
          Verify Service Recovery
```

Each component has a distinct responsibility:

1. **Bash and Python** automate operational tasks and integrate tools.
2. **Terraform** manages declared infrastructure resources.
3. **CI/CD** validates, builds, and delivers application changes.
4. **GitOps** reconciles running systems with approved desired configuration.
5. **Health checks** detect specific failure conditions.
6. **Remediation workflows** respond to known failures within defined boundaries.
7. **Secrets management** protects credentials used by applications and automation.
8. **Access control** limits who and what can change infrastructure.
9. **Operational measurement** identifies tasks worth simplifying or automating.

The workflow should also account for failure:

- What happens if a pipeline fails halfway through?
- Can an infrastructure change be reviewed before it is applied?
- Does a health check generate false alarms?
- Can a remediation action worsen an outage?
- Can compromised automation credentials be revoked quickly?
- Can the team identify and recover from a faulty configuration change?

Reliable automation includes both the normal path and the failure path.

---

## 9. Key Takeaways

- Bash is effective for shell-oriented automation, while Python is useful for structured logic and API integration.
- Operational scripts should validate inputs, handle errors, use timeouts, and produce useful logs.
- Infrastructure as Code makes infrastructure changes version-controlled and repeatable.
- Terraform uses configuration and state to plan and manage infrastructure changes.
- Terraform state can contain sensitive information and must be protected.
- CI/CD pipelines improve delivery consistency but require reliable tests, secure credentials, and deployment validation.
- Building an artifact once and promoting it across environments improves traceability.
- GitOps uses version-controlled desired state and automated reconciliation.
- Configuration management helps maintain consistent system configuration.
- Health checks detect specific conditions; observability provides a broader view of system behavior.
- Automated remediation should be bounded, validated, observable, and reversible where possible.
- Secrets must not be committed to Git or exposed in logs.
- Least privilege reduces the impact of compromised identities and automation.
- Toil reduction focuses on removing repetitive operational work, not eliminating necessary engineering judgment.
- Automation should be measured and maintained like any other production system.

---

## 10. Learning Resources

### Bash and Python

- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/)
- [ShellCheck — Shell Script Analysis](https://www.shellcheck.net/)
- [Python Documentation](https://docs.python.org/3/)
- [Python `subprocess` Documentation](https://docs.python.org/3/library/subprocess.html)

### Infrastructure as Code

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Language Documentation](https://developer.hashicorp.com/terraform/language)
- [Terraform CLI Documentation](https://developer.hashicorp.com/terraform/cli)
- [Terraform State Documentation](https://developer.hashicorp.com/terraform/language/state)

### CI/CD and GitOps

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ci/)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Flux Documentation](https://fluxcd.io/docs/)

### Configuration Management and Security

- [Ansible Documentation](https://docs.ansible.com/)
- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
