# 04. Linux and Networking

> A practical SRE study guide for inspecting Linux systems, diagnosing connectivity problems, and troubleshooting slow or unavailable services.

## Learning objectives

- [ ] Inspect processes, services, logs, and system resources.
- [ ] Diagnose CPU, memory, disk-space, inode, and I/O problems.
- [ ] Explain DNS resolution and investigate name-resolution failures.
- [ ] Troubleshoot IP connectivity, ports, routing, and firewalls.
- [ ] Explain TCP/IP, HTTP/HTTPS, TLS, and common status codes.
- [ ] Describe load balancers, reverse proxies, and health checks.
- [ ] Use a structured, evidence-based process to troubleshoot outages.

---

## 1. Linux processes, services, and system resources

### 1.1 Processes

A **process** is a running instance of a program. Each process has a process ID (PID), user, resource usage, and lifecycle.

```bash
ps aux                         # List running processes
ps -ef                         # Alternative process listing
ps -p <PID> -o pid,ppid,cmd,%cpu,%mem
top                            # Interactive process/resource view
pgrep -af nginx                # Find processes by name/pattern
pstree -p                      # Process hierarchy (if installed)
```

- **PID:** process identifier.
- **PPID:** parent process identifier.
- **Signal:** notification sent to a process, such as a termination request.
- **Zombie:** a process that has exited but whose parent has not collected its exit status.

Prefer graceful termination when appropriate:

```bash
kill -TERM <PID>
```

`SIGTERM` asks a process to stop cleanly. `kill -KILL <PID>` / `kill -9 <PID>` forcibly terminates it and prevents cleanup, so reserve it for cases where graceful termination fails and the impact is understood. Avoid killing production processes blindly.

### 1.2 Services and systemd

A **service** is typically a long-running background program managed by a service manager. Many Linux distributions use `systemd`.

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-units --type=service --state=failed
```

- `start` starts a service.
- `stop` stops it.
- `restart` stops and starts it again.
- `reload` asks it to reload configuration, if supported.
- `enable` configures it to start at boot; it does not necessarily start it immediately.

Inspect service logs:

```bash
journalctl -u nginx
journalctl -u nginx --since "30 minutes ago"
journalctl -u nginx -f              # Follow new log entries
journalctl -p err -b                # Errors from the current boot
```

**Practical flow:** check status → read logs → validate configuration → check dependencies, ports, and resources → make the smallest safe change → verify recovery.

### 1.3 Resource overview

```bash
uptime                  # Uptime and load averages
free -h                 # Memory and swap
df -h                   # Filesystem capacity
df -i                   # Inode capacity
du -sh /var/log/*       # Approximate log-directory usage
lsblk                   # Block devices
vmstat 1                # CPU, memory, run queue, and I/O activity
iostat -xz 1             # Detailed disk I/O (sysstat package)
```

Some tools, such as `iostat`, may need to be installed. Use commands available on the host and follow production access policies.

---

## 2. CPU, memory, disk, and I/O troubleshooting

Do not just notice a high metric. Correlate it with the incident timeline, traffic, application behavior, logs, and other resources.

### 2.1 CPU pressure

Symptoms may include high latency, slow commands, timeouts, and a growing run queue.

```bash
top
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head
uptime
vmstat 1
```

- **CPU utilization** indicates how busy the CPUs are.
- **Load average** estimates runnable tasks and tasks in uninterruptible sleep over 1, 5, and 15 minutes. It is not a CPU percentage.
- Interpret load relative to the number of CPU cores and the workload. A load average of 8 means something different on a 2-core host than on a 16-core host.
- High CPU can result from inefficient code, traffic spikes, tight loops, expensive queries, garbage collection, or background jobs.

**SRE approach:** identify the busiest process or thread, compare with baseline and traffic, inspect application logs/profiles, and mitigate safely. Adding CPU is not always the correct root-cause fix.

### 2.2 Memory pressure

```bash
free -h
ps -eo pid,comm,rss,%mem --sort=-%mem | head
vmstat 1
dmesg -T | grep -i -E 'out of memory|oom|killed process'
```

- Linux uses otherwise-free RAM for filesystem cache; **available memory** is often more useful than looking at the `free` column alone.
- **Swap** moves memory pages to disk. Heavy swapping can severely affect latency.
- The **OOM killer** may terminate a process when the system cannot satisfy memory needs.
- A **memory leak** occurs when an application keeps retaining memory it no longer needs.

If a service was killed, inspect kernel and service logs, memory trends, and any container/pod limits. In Kubernetes, a container can be OOM-killed because it exceeds its memory limit even when the host still has available memory.

### 2.3 Disk space and inodes

```bash
df -h
df -i
sudo du -xhd1 /var 2>/dev/null | sort -h
sudo find /var/log -type f -size +500M -ls
```

Two separate failure modes:

- **Filesystem full:** there is no space left for writes.
- **Inodes exhausted:** there are no free file entries, even if the filesystem still has free bytes. This can happen when an application creates huge numbers of small files.

Possible causes include runaway logs, old backups, container images, temporary files, and unbounded application output. Do not delete files blindly on a production host. Determine what consumes the space and use the approved cleanup/rotation procedure.

### 2.4 Disk I/O bottlenecks

A disk can have free space but still be slow because reads/writes are queued or the storage device is saturated.

```bash
iostat -xz 1
vmstat 1
iotop                  # If installed and permitted
```

Look for sustained I/O wait, queueing, high latency, and throughput near device limits. Interpret `%util` carefully, especially on SSDs and devices supporting parallel I/O; no single metric proves a bottleneck.

Possible causes: slow storage, excessive writes, database checkpoints, large backups, noisy neighbors, or many small random I/O operations.

### 2.5 Symptom-to-resource matrix

| Symptom | Inspect first | Possible causes |
|---|---|---|
| CPU consistently high | `top`, per-process CPU, traffic | CPU-heavy code, traffic spike, loop |
| Memory falling / swap rising | `free -h`, process RSS, OOM logs | Leak, workload growth, insufficient limit |
| “No space left on device” | `df -h`, `df -i`, `du` | Full filesystem or inode exhaustion |
| High latency with I/O wait | `iostat`, `vmstat`, app metrics | Storage saturation, slow queries, heavy writes |
| Process repeatedly exits | `systemctl status`, `journalctl`, exit code | Crash, bad config, dependency failure |

---

## 3. DNS resolution and connectivity

### 3.1 What DNS does

The **Domain Name System (DNS)** maps names such as `api.example.com` to records such as IP addresses. It allows clients to use stable names instead of remembering IPs.

Common record types:

- `A`: IPv4 address.
- `AAAA`: IPv6 address.
- `CNAME`: alias to another DNS name.
- `MX`: mail exchanger.
- `TXT`: text records, often used for verification and email policies.
- `NS`: authoritative name servers.

A simplified lookup path is: application/resolver cache → configured recursive resolver → DNS hierarchy and authoritative server as needed → response returned and cached according to TTL. Actual paths vary by environment.

### 3.2 DNS commands

```bash
getent hosts api.example.com
dig api.example.com
dig A api.example.com
dig AAAA api.example.com
dig +short api.example.com
dig @1.1.1.1 api.example.com
nslookup api.example.com
cat /etc/resolv.conf
```

`getent` follows the host's configured name-service setup, which may include sources beyond DNS. `dig` helps inspect DNS records and responses.

### 3.3 Common DNS failures

- **NXDOMAIN:** the responding DNS server says the queried name does not exist.
- **SERVFAIL:** the server failed to complete the lookup; possible causes include upstream problems, DNSSEC validation, or authoritative-server issues.
- **Timeout:** no response arrived in time; investigate resolver reachability, firewall rules, or DNS service health.
- **Wrong/stale address:** check TTL, caches, split-horizon DNS, and whether the record points to the intended environment.
- **Works by IP but not by name:** DNS may be involved, but also check TLS hostnames and virtual-host routing before concluding.

```bash
dig api.example.com
getent hosts api.example.com
curl -v --connect-timeout 5 https://api.example.com/
```

Do not change DNS records as the first incident response. Confirm the resolver, record, TTL, environment, and expected destination first.

---

## 4. TCP/IP, ports, routing, and firewalls

### 4.1 IPs, ports, and sockets

- An **IP address** identifies a network interface/address.
- A **port** identifies a service endpoint on a host; HTTPS commonly uses TCP port 443.
- A **socket** is an endpoint used for network communication.
- `IP:port` is a useful way to describe where a service listens or where a client connects.

TCP is connection-oriented and provides ordered, reliable byte-stream delivery. UDP is connectionless and does not itself guarantee delivery or ordering. DNS often uses UDP but can also use TCP. HTTPS commonly uses TCP; HTTP/3 uses QUIC over UDP.

### 4.2 TCP three-way handshake

A simplified TCP connection starts with:

1. Client sends `SYN`.
2. Server replies `SYN-ACK`.
3. Client sends `ACK`.

After the handshake, application data can be exchanged. If a connection cannot be established, the client may see a refusal, timeout, or routing error depending on where it fails.

### 4.3 Listening ports and connections

```bash
ss -lntp                 # Listening TCP sockets (permissions vary)
ss -lunp                 # Listening UDP sockets
ss -ant                  # TCP socket states
sudo lsof -i :8080       # Process using a port (if installed)
```

Common TCP states:

- `LISTEN`: waiting for incoming connections.
- `ESTABLISHED`: connection is active.
- `TIME-WAIT`: recently closed connection retained for protocol correctness.
- `SYN-SENT`: a connection attempt is waiting for a response.

A port not listening can cause connection refusal. A firewall silently dropping traffic often causes a timeout, though actual behavior depends on the network and firewall.

### 4.4 Routes and connectivity tests

```bash
ip addr                 # Interfaces and addresses
ip route                # Routing table
ip route get 10.0.0.10  # Route selected for a destination
ping -c 4 10.0.0.10     # Basic reachability/latency test
traceroute 10.0.0.10    # Path discovery, if installed
tracepath 10.0.0.10     # Alternative diagnostic, if available
nc -vz api.example.com 443
curl -v --connect-timeout 5 https://api.example.com/
```

Important caveats:

- A failed `ping` does not prove a host is down; ICMP may be blocked.
- `traceroute` can show incomplete paths when intermediate devices filter probes.
- `nc -vz` tests TCP connectivity, not whether the application returns a valid response.
- Test from the same network/location as the affected client where possible. A laptop test may not reflect a pod, VM, or private subnet's path.

### 4.5 Firewalls and network rules

Firewalls restrict inbound/outbound traffic by address, protocol, and port. In cloud environments, also inspect network security groups, route tables, network ACLs, private endpoints, and load-balancer rules.

Connection checklist:

1. Is the destination IP/name correct?
2. Does the service listen on the expected interface and port?
3. Is there a valid route to the destination and a return route?
4. Do host and cloud firewalls permit traffic in both directions?
5. Are DNS, proxy, VPN, peering, and private endpoint settings correct?
6. Is the application accepting and processing the connection?

Do not open a port to the entire internet as a quick fix. Use the narrowest source, destination, and protocol permitted by the intended design.

---

## 5. HTTP, HTTPS, TLS, and status codes

### 5.1 What happens when opening an HTTPS URL?

A simplified request can involve:

1. Resolve the hostname using DNS.
2. Establish network connectivity to the destination IP and port.
3. Establish TCP (for HTTP/1.1 or HTTP/2 over TCP).
4. Negotiate TLS, validating certificates and cryptographic parameters.
5. Send the HTTP request.
6. A server or proxy routes it to the application/upstream.
7. Receive the HTTP response.

HTTP/3 differs because it runs over QUIC/UDP rather than TCP. Proxies, caches, and connection reuse can also change the details of this flow.

### 5.2 HTTP vs HTTPS and TLS

- **HTTP** is an application-layer request/response protocol.
- **HTTPS** is HTTP protected by TLS.
- **TLS** provides encryption in transit, integrity protection, and server authentication through certificates. Client certificates can be used for mutual TLS.
- A certificate must be valid for the hostname and chain to a trusted issuer. Expiry, missing intermediate certificates, hostname mismatch, and incorrect system time can cause validation failures.

```bash
curl -v https://example.com
curl -I https://example.com
openssl s_client -connect example.com:443 -servername example.com
```

Use `curl -k` only as a temporary diagnostic to check whether certificate verification is the specific problem. It disables verification and is not a safe production fix.

### 5.3 Common HTTP status codes

| Code | Meaning | Common interpretation |
|---|---|---|
| `200 OK` | Request succeeded | Normal successful response |
| `201 Created` | Resource created | Often returned after a create operation |
| `204 No Content` | Success, no response body | Successful operation without a body |
| `301/302` | Redirect | Client is directed elsewhere |
| `304 Not Modified` | Cached representation still valid | Conditional request |
| `400 Bad Request` | Invalid request | Malformed input or parameters |
| `401 Unauthorized` | Authentication required/failed | Missing or invalid credentials |
| `403 Forbidden` | Request understood but refused | Caller lacks permission or access is blocked |
| `404 Not Found` | Resource not found | Wrong path or absent resource |
| `408 Request Timeout` | Request timed out | Server did not receive a complete request in time |
| `429 Too Many Requests` | Rate limited | Request rate or quota exceeded |
| `500 Internal Server Error` | Server-side failure | Unexpected application error |
| `502 Bad Gateway` | Gateway received invalid upstream response | Upstream crash, reset, or protocol issue |
| `503 Service Unavailable` | Service temporarily unavailable | Overload, maintenance, or no healthy backends |
| `504 Gateway Timeout` | Gateway timed out waiting for upstream | Upstream slow or unreachable; timeout mismatch |

**Interview tip:** 502/503/504 alone do not identify root cause. Determine which component emitted the response—load balancer, reverse proxy, gateway, or application—and inspect its logs and upstream health.

---

## 6. Load balancers, reverse proxies, and health checks

### 6.1 Load balancers

A **load balancer** distributes requests/connections across multiple backends. It can improve availability and capacity by avoiding dependence on a single backend.

Key concepts:

- **Listener/frontend:** address and port accepting traffic.
- **Backend pool:** destinations receiving traffic.
- **Health probe:** test used to decide whether a backend is eligible.
- **Algorithm:** for example, round robin or least connections, depending on product/configuration.
- **Layer 4:** routes mainly using transport details such as IP and port.
- **Layer 7:** routes using application details such as HTTP host, path, or headers.

### 6.2 Reverse proxies

A **reverse proxy** sits in front of backend services and receives requests on their behalf. It may terminate TLS, route requests, add headers, cache/compress responses, apply rate limits, or provide access logs.

Examples include NGINX, HAProxy, Envoy, and Traefik. A reverse proxy and a load balancer are related roles; one product can perform both.

### 6.3 Health checks

- **Liveness check:** asks whether a process should be restarted because it is no longer functioning.
- **Readiness check:** asks whether an instance should receive traffic right now.
- **Startup check:** gives a slow-starting application time to initialize before liveness checks take effect.

A readiness check should indicate whether the instance can serve its intended requests. Avoid making liveness depend on every external dependency: a temporary database outage could otherwise trigger restart loops across all instances.

For a load balancer, confirm that the probe protocol, port, path, expected response, timeout, interval, and success/failure thresholds match the service. A service may be running but marked unhealthy because the probe path is wrong or the application is not ready.

### 6.4 Common load-balancing failures

- No healthy backends → requests may return 503.
- One backend unhealthy → uneven traffic or partial failures.
- Incorrect host/path routing → 404 or wrong service.
- TLS mismatch at proxy/backend → handshake or upstream errors.
- Timeout mismatch between proxy and application → 504s or premature connection closure.
- Sticky sessions or uneven connection distribution → traffic may not be balanced as expected.

---

## 7. Diagnosing slow or unavailable services

Use a repeatable process. Define the user-visible symptom, then narrow the failing layer.

### 7.1 Troubleshooting sequence

**Step 1: Scope the incident**

- What is failing: all requests or a subset?
- When did it begin? Was there a deployment, configuration change, traffic spike, or dependency change?
- Which region, environment, endpoint, client, or tenant is affected?
- Is the symptom latency, connection failure, DNS failure, TLS error, or an HTTP error?

**Step 2: Reproduce and classify**

```bash
curl -v --connect-timeout 5 --max-time 15 https://api.example.com/health
dig api.example.com
nc -vz api.example.com 443
```

These tests answer different questions: DNS lookup, TCP connection, TLS/HTTP response, and timing. One successful test does not prove the next layer works.

**Step 3: Check the service and host**

```bash
systemctl status my-service
journalctl -u my-service --since "15 minutes ago"
ss -lntp
top
free -h
df -h
```

For containers, inspect logs, restart counts, CPU/memory limits, pod events, readiness, and node capacity.

**Step 4: Check the network path**

- Does DNS return the expected address?
- Does the application listen on the expected interface and port?
- Is the selected route correct?
- Do host/cloud firewall rules and security policies permit traffic?
- Can the load balancer reach the backend on the probe and service ports?

**Step 5: Check proxy, load balancer, and dependencies**

Inspect backend health, request/response codes, upstream connect time, upstream response time, TLS errors, database/queue latency, connection pools, and rate limits.

**Step 6: Correlate telemetry**

Use logs, metrics, and traces together. Filter to the incident time window and compare against a healthy baseline. A trace may reveal a slow downstream call; metrics show whether many requests are affected; logs expose specific errors.

**Step 7: Mitigate safely and verify**

Possible mitigations include rolling back a recent change, restoring capacity, draining an unhealthy backend, stopping a runaway job, or reverting a bad configuration. Follow incident procedures and get approval where required.

After mitigation, verify user-facing success rate and latency, check that error rates have returned to baseline, and document root cause and follow-up actions. Avoid making several unrelated changes at once.

### 7.2 Symptom-to-layer guide

| Symptom | First areas to investigate |
|---|---|
| DNS name fails, IP test works | DNS records, resolver, cache, split-horizon DNS |
| Connection refused | Wrong port, service not listening, explicit reject |
| Connection times out | Routing, dropped firewall traffic, unreachable endpoint, overloaded service |
| TLS certificate error | Hostname, expiry, trust chain, system time, SNI |
| HTTP 401/403 | Authentication, authorization, policy, credentials |
| HTTP 404 | URL/path, routing rule, application route |
| HTTP 502 | Proxy/gateway and upstream response/reset |
| HTTP 503 | Capacity, readiness, maintenance, healthy backend count |
| HTTP 504 | Upstream latency/reachability, timeout configuration |
| High latency with normal CPU | Network, disk I/O, database, locks, queues, external dependency |
| Only one instance fails | Instance config, health, resource pressure, local connectivity |

### 7.3 Example: API is slow

Suppose users report that an API takes 8 seconds to respond.

1. Check request rate, error rate, and latency percentiles to see whether the issue is widespread.
2. Compare p50, p95, and p99 latency. If p50 is normal but p99 is high, a smaller set of requests may be affected.
3. Inspect distributed traces for slow spans and downstream calls.
4. Check application logs using a request/trace ID.
5. Check host/pod CPU, memory, restarts, throttling, and I/O.
6. Check database query latency, connection-pool saturation, queues, and third-party dependencies.
7. Mitigate the identified bottleneck, then verify latency and errors recover.

High latency does not always mean insufficient CPU. The application may be waiting on a database, network, lock, queue, or exhausted connection pool.

### 7.4 Example: service unavailable after deployment

1. Confirm deployment time and affected version.
2. Check service/pod status, logs, and startup errors.
3. Verify the process listens on the expected port and configuration/secrets are present.
4. Check readiness probes and whether the load balancer has healthy backends.
5. Check connectivity to required dependencies.
6. If the release is the likely cause and impact is ongoing, follow the rollback procedure.
7. Confirm recovery from the user's perspective and capture the learning.

---

## 8. Safe troubleshooting habits

- Begin with read-only inspection where possible.
- Record the exact error, timestamp, affected endpoint, and scope.
- Compare against a known-good baseline.
- Change one thing at a time and verify the result.
- Avoid restarting or killing processes before checking logs and impact.
- Do not expose secrets, tokens, private keys, or sensitive data in logs or command output.
- Do not disable TLS validation or broadly open firewall rules to “make it work.”
- In production, follow change-control and incident-management procedures.

---

## 09. Quick command reference

| Purpose | Command |
|---|---|
| Processes | `ps aux`, `top`, `pgrep -af <name>` |
| Service status | `systemctl status <service>` |
| Service logs | `journalctl -u <service> --since "15 minutes ago"` |
| Memory | `free -h`, `vmstat 1` |
| Filesystem bytes/inodes | `df -h`, `df -i` |
| Directory usage | `du -sh <path>` |
| Disk I/O | `iostat -xz 1` |
| Interfaces/routes | `ip addr`, `ip route` |
| Listening ports | `ss -lntp`, `ss -lunp` |
| DNS | `dig <host>`, `getent hosts <host>` |
| TCP port test | `nc -vz <host> <port>` |
| HTTP request | `curl -v <url>` |
| TLS handshake | `openssl s_client -connect <host>:443 -servername <host>` |

---

## 12. Learning resources

Prefer official documentation and practise commands in a safe environment.

- [Linux man-pages project](https://www.kernel.org/doc/man-pages/)
- [systemd documentation](https://www.freedesktop.org/software/systemd/man/)
- [Red Hat documentation: Performance and monitoring](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/)
- [Linux kernel documentation](https://docs.kernel.org/)
- [Cloudflare Learning Center: DNS](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [Cloudflare Learning Center: TCP/IP](https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/)
- [MDN: HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [MDN: HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [Let's Encrypt: How HTTPS works](https://letsencrypt.org/how-it-works/)
- [NGINX documentation](https://nginx.org/en/docs/)
- [HAProxy documentation](https://docs.haproxy.org/)
