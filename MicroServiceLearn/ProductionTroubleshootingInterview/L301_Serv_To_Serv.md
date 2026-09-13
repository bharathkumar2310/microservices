# Microservices Production Troubleshooting - Service-to-Service Connectivity

## Purpose

This document is a detailed interview and production troubleshooting guide for these 15 service-to-service communication questions:

1. Service A is unable to connect to Service B.
2. Service A receives `Connection refused`.
3. Service A receives a connection timeout.
4. Service A receives a read timeout.
5. Requests work sometimes but time out intermittently.
6. Service B is running, but Service A cannot reach it.
7. Ping works, but the API call fails.
8. DNS resolution for Service B fails.
9. A hostname works from a laptop but not from Service A.
10. An API gateway returns HTTP 502.
11. An API gateway returns HTTP 503.
12. An API gateway returns HTTP 504.
13. Only one Service B instance is failing.
14. Requests routed to one particular Service B instance fail.
15. Service B's health endpoint is healthy, but its real APIs fail.

The goal is not to memorize a list of commands. The goal is to learn how to identify the exact failing layer, collect evidence, narrow the fault domain, prove the root cause, apply the correct fix, and verify that the problem cannot immediately recur.

> Command examples use placeholder names such as `service-b`, ports such as `8080`, and Kubernetes resources such as `<service-a-pod>`. Replace them with the real values. Run tests from the same runtime environment, network namespace, security identity, and proxy path as Service A whenever possible.

---

# 1. Core Mental Model

## 1.1 The complete request path

When Service A calls Service B, the request normally passes through several independent stages:

```text
Service A application code
        |
        | 1. Read configuration and build the URL
        v
DNS or service discovery
        |
        | 2. Resolve hostname to one or more IP addresses
        v
Routing, firewall, security group, network policy, NAT, proxy
        |
        | 3. Deliver packets to the destination network
        v
TCP listener on Service B host and port
        |
        | 4. Complete the TCP handshake
        v
TLS listener, when HTTPS or mTLS is used
        |
        | 5. Negotiate TLS and validate certificates
        v
Gateway, ingress, load balancer, sidecar, or reverse proxy
        |
        | 6. Select and forward to a backend
        v
Service B HTTP server
        |
        | 7. Match route, authenticate, authorize, and parse the request
        v
Service B business logic
        |
        | 8. Wait for threads, connection pools, CPU, and local resources
        v
Database, cache, queue, Kafka, external API, or Service C
        |
        | 9. Complete downstream work
        v
Serialize and return the response
```

A failure at one stage does not prove that the later stages are broken. For example:

- A DNS error means TCP was never attempted.
- A connection refusal means name resolution may have succeeded, but the destination did not accept the TCP connection.
- A TLS error means TCP generally succeeded, but secure-session negotiation failed.
- HTTP 401, 403, 404, or 500 means an HTTP responder was reached. Basic network connectivity therefore worked far enough to receive an HTTP response.
- A read timeout means a connection was established, but the expected response did not arrive within the caller's deadline.
- HTTP 502, 503, and 504 usually come from an intermediary, but the exact meaning depends on that product's configuration.

## 1.2 Error classification table

| Symptom | What it normally proves | First fault domain to investigate |
|---|---|---|
| `UnknownHostException`, `Name or service not known`, `NXDOMAIN` | The requested hostname did not resolve | Hostname, DNS, service discovery, search domain |
| DNS query timeout or `SERVFAIL` | A resolver did not answer successfully | Resolver health, DNS network path, authoritative DNS |
| `No route to host` or `Network is unreachable` | The local networking stack could not find or use a route, or a device returned an unreachable response | Route table, subnet, gateway, network policy, firewall |
| `Connection refused` | A TCP attempt reached something that actively rejected it | Listener, host, port, bind address, rejected target |
| Connection timeout | TCP did not complete within the connect deadline | Packet drop, routing, firewall, security rule, unreachable target |
| `Connection reset by peer` | A TCP connection existed and was forcefully closed | Service B, proxy, load balancer, TLS/protocol mismatch |
| TLS handshake or certificate error | TCP normally succeeded; TLS negotiation or trust failed | Certificate, hostname/SAN, CA trust, SNI, TLS version, mTLS |
| HTTP 400 | HTTP worked; the request was considered invalid | Request syntax, headers, payload, gateway validation |
| HTTP 401 or 403 | HTTP worked; authentication or authorization failed | Token, identity, scope, role, clock, policy |
| HTTP 404 | HTTP worked; the requested route or resource was not found | Path, base URL, route mapping, version, resource ID |
| HTTP 429 | HTTP worked; a rate or concurrency limit rejected the request | Client rate, gateway quota, server throttling |
| HTTP 500 | Service B or an intermediary executed code and failed | Application exception, dependency, data, configuration |
| HTTP 502 | A gateway/proxy could not obtain or accept a valid upstream response | Gateway-to-backend connection, TLS, protocol, reset, malformed response |
| HTTP 503 | The responder considered the service temporarily unavailable | No healthy backends, overload, maintenance, circuit breaker |
| HTTP 504 | A gateway/proxy waited too long for an upstream response | Backend latency, queueing, dependencies, timeout budget |
| Read timeout | The connection was established, but the client did not receive enough response data in time | Service queueing, slow processing, downstream delay, response transfer |

These are starting points, not universal laws. Always identify which component generated the error and check that component's documented semantics.

## 1.3 Six rules that prevent random troubleshooting

1. **Capture the exact error.** "Service B is down" is a conclusion, not evidence.
2. **Test from Service A's execution environment.** A laptop, bastion, or unrelated debug pod may use different DNS, routes, policies, proxies, certificates, and identities.
3. **Separate the hops.** Test `A -> gateway`, `gateway -> B`, and `B -> dependency` independently.
4. **Change one variable at a time.** Otherwise, a successful retest does not prove which change fixed the issue.
5. **Correlate across time and instance.** Record timestamp, request ID, source instance, destination instance, endpoint, status, and latency.
6. **Prove recovery at the same layer that failed.** A healthy ping does not prove an HTTP fix, and a healthy `/health` response does not prove a business transaction works.

## 1.4 Evidence to collect before changing anything

Record the following in an incident worksheet:

```text
Caller:
Caller instance/pod/host:
Destination URL:
Resolved destination IPs:
Protocol and port:
Proxy/gateway/load balancer path:
Exact exception or HTTP status:
Error message and nested/root exception:
First failure time:
Last failure time:
Failure percentage:
Affected endpoints/tenants/regions:
Request ID or trace ID:
Destination instance, if known:
Client connect/read/overall timeouts:
Recent deployment/config/network/certificate changes:
```

Do not paste credentials, bearer tokens, secrets, private keys, or sensitive payloads into notes or commands. Redact them from logs.

## 1.5 Baseline diagnostic commands

### Windows PowerShell

```powershell
Resolve-DnsName service-b
Test-NetConnection -ComputerName service-b -Port 8080 -InformationLevel Detailed
curl.exe -v --connect-timeout 5 --max-time 15 http://service-b:8080/health
Get-NetTCPConnection -LocalPort 8080 -State Listen
netstat -ano | findstr :8080
```

### Linux

```bash
getent ahosts service-b
dig service-b
nc -vz -w 5 service-b 8080
curl -v --connect-timeout 5 --max-time 15 http://service-b:8080/health
ss -lntp
```

### TLS

```bash
curl -v --connect-timeout 5 --max-time 15 https://service-b/health
openssl s_client -connect service-b:443 -servername service-b -showcerts </dev/null
```

`curl -k` disables certificate verification. It can be used once to prove that trust validation is the failing stage, but it is not a production fix. Correct the certificate, hostname, trust store, or mTLS configuration instead.

### Kubernetes

```bash
kubectl exec -n <namespace> <service-a-pod> -- getent hosts service-b
kubectl exec -n <namespace> <service-a-pod> -- curl -v --connect-timeout 5 --max-time 15 http://service-b:8080/health
kubectl get service -n <namespace> service-b -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o wide
kubectl get pods -n <namespace> -l app=service-b -o wide
kubectl describe pod -n <namespace> <service-b-pod>
kubectl get networkpolicy -n <namespace>
```

Minimal or distroless containers may not contain DNS or HTTP tools. An approved ephemeral debug container can help, but remember that a separate debug pod may have different labels, service account, sidecar, egress rules, or network policies. It is not automatically equivalent to Service A.

---

# 2. Question 1 - Service A Is Unable to Connect to Service B

## Interview question

> Service A is unable to connect to Service B. How would you investigate?

## What the statement does and does not tell you

"Unable to connect" is too broad to diagnose. It can describe any failure from a malformed URL to a slow database. The first job is to replace that sentence with a precise observation:

```text
Service A instance A-3 called https://service-b.example/orders at 14:32:08 UTC.
DNS resolved 10.20.4.18.
TCP connection to 10.20.4.18:443 succeeded in 12 ms.
TLS failed because the certificate hostname did not match.
```

That observation identifies a TLS problem. Without it, teams often check the database, restart pods, or change timeouts even though the request never reached Service B.

## Where the problem can occur

| Fault domain | Example causes |
|---|---|
| Service A code/configuration | Wrong base URL, port, scheme, path, proxy, environment variable, stale configuration |
| DNS/service discovery | Missing record, wrong namespace, stale IP, resolver failure, split-horizon DNS |
| Network path | Missing route, firewall drop, security group, network ACL, Kubernetes NetworkPolicy, service-mesh egress rule |
| TCP destination | Service down, wrong port, listener bound to localhost, full accept queue |
| TLS/mTLS | Expired certificate, hostname mismatch, missing CA, SNI mismatch, client certificate rejected |
| Gateway/load balancer | Wrong upstream, no healthy targets, protocol mismatch, stale target registration |
| HTTP/API contract | Wrong route, method, headers, content type, authentication, API version |
| Service B application | Startup failure, exception, overload, deadlock, resource exhaustion |
| Service B dependency | Slow or unavailable DB, cache, Kafka, Service C, external provider |
| One instance only | Version drift, bad secret, node issue, local resource exhaustion |

## Step-by-step debugging

### Step 1 - Establish the exact scope

Determine:

- Is every request failing or only a percentage?
- Is every Service A instance affected?
- Is every Service B instance affected?
- Is one endpoint, tenant, payload size, region, or protocol affected?
- Did the issue begin immediately after a deployment, certificate rotation, DNS change, firewall change, or autoscaling event?
- Is the call direct, or does it pass through a proxy, sidecar, gateway, ingress, or load balancer?

Scope is diagnostic evidence. A 25 percent failure rate across four equally weighted backends strongly suggests one bad backend. Failures only for large payloads suggest limits, serialization cost, or transfer time rather than basic reachability.

### Step 2 - Read the complete client error

Inspect the root exception, not only the top-level wrapper. Many HTTP libraries wrap the useful cause:

```text
HTTP client exception
  caused by retry exception
    caused by SocketTimeoutException
      caused by Connect timed out
```

Record whether failure occurred during:

1. URL construction.
2. DNS lookup.
3. TCP connect.
4. TLS handshake.
5. Request write.
6. Response wait/read.
7. HTTP status handling.
8. Response parsing.

### Step 3 - Confirm the effective destination

Verify the value Service A actually loaded at runtime:

```text
scheme: https
host: service-b.internal
port: 443
base path: /api/v2
proxy: none or expected proxy
```

Do not rely only on a configuration repository. A running process may have an old environment variable, mounted ConfigMap, secret, feature flag, or cached discovery result. Compare the effective configuration of a failing instance with a working instance. Redact secrets.

### Step 4 - Reproduce from Service A's runtime boundary

Execute a bounded diagnostic request from the Service A host, pod, container, or equivalent network namespace:

```bash
curl -v --connect-timeout 5 --max-time 15 https://service-b.internal/health
```

If the application uses a proxy, mTLS sidecar, custom trust store, or service identity, a plain shell `curl` may take a different path. Compare both tests and account for those differences.

### Step 5 - Test DNS independently

```powershell
Resolve-DnsName service-b.internal
```

```bash
getent ahosts service-b.internal
dig service-b.internal
```

Check:

- Did resolution succeed?
- Did it return the expected IP type, region, environment, and number of addresses?
- Does every Service A instance get the same answer?
- Is an old IP still cached?
- Does the application prefer an unreachable IPv6 address?

### Step 6 - Test TCP independently

```powershell
Test-NetConnection -ComputerName service-b.internal -Port 443 -InformationLevel Detailed
```

```bash
nc -vz -w 5 service-b.internal 443
```

Interpret the result:

- Immediate refusal: destination was reached but rejected the port.
- Timeout: handshake did not complete; investigate packet path and drops.
- Success: DNS, routing, and TCP worked for this test. Continue to TLS or HTTP.

### Step 7 - Test TLS, if applicable

```bash
openssl s_client -connect service-b.internal:443 -servername service-b.internal -showcerts </dev/null
```

Inspect:

- Certificate validity dates.
- Subject Alternative Name contains the requested hostname.
- Issuer is trusted by Service A.
- Full intermediate chain is supplied.
- Client certificate is present and accepted for mTLS.
- SNI selects the expected virtual host.
- Client and server support a common TLS version and cipher.

### Step 8 - Test HTTP behavior

Use the real method, path, required headers, and a safe test payload when permitted:

```bash
curl -v --connect-timeout 5 --max-time 15 \
  -H "Content-Type: application/json" \
  https://service-b.internal/api/v2/orders
```

Interpret HTTP responses before returning to network checks:

- 401/403: transport worked; investigate identity and authorization.
- 404: transport worked; investigate route, path prefix, method, or version.
- 415: content type or body format mismatch.
- 429: throttling or quota.
- 500: Service B or an intermediary executed the request and failed.
- 502/503/504: identify the intermediary that generated it.

### Step 9 - Inspect Service B

Confirm:

- Process/container/pod is running.
- Application startup completed successfully.
- The expected port is listening.
- Listener is bound to a remotely reachable interface such as `0.0.0.0` or the correct host IP, not only `127.0.0.1`.
- Readiness is true and the instance is registered in discovery.
- Service B received the test request.
- Logs, metrics, and traces show the same request ID and timestamp.

### Step 10 - Inspect every intermediary

For each gateway, load balancer, ingress, proxy, or service-mesh sidecar, verify:

- Route match and upstream name.
- Upstream DNS result.
- Target IP and port.
- Backend health and reason.
- HTTP versus HTTPS protocol.
- TLS SNI, trust, and client-certificate settings.
- Connection, idle, and response timeout.
- Retry and circuit-breaker state.
- Request/response size limits.

### Step 11 - Inspect Service B's dependencies

If the request reached B but did not complete, trace:

```text
A -> B queue -> B code -> DB/cache/Kafka/C -> response serialization
```

Check latency, errors, pool utilization, thread state, CPU throttling, memory pressure, GC pauses, locks, and downstream timeouts.

### Step 12 - Correlate with changes

Compare the first failure time with:

- Service A or B deployment.
- Configuration or secret rotation.
- Certificate change.
- DNS or service-discovery update.
- Firewall, route, security group, or NetworkPolicy change.
- Node replacement or autoscaling.
- Database migration.
- Traffic increase.

Correlation is a clue, not proof. Verify the mechanism before reverting or changing production.

### Step 13 - Verify the fix

After the root cause is corrected:

1. Repeat the same failing test from the same Service A environment.
2. Test the real business endpoint, not only `/health`.
3. Confirm normal success rate and latency across all instances.
4. Confirm queues, retries, and resource saturation return to normal.
5. Monitor for at least the relevant traffic cycle.
6. Record evidence and add an alert, test, or configuration control that detects recurrence.

## Interview-ready answer

> I would first replace "cannot connect" with the exact exception, timestamp, affected scope, and request path. I would classify the failure as configuration, DNS, TCP, TLS, HTTP, application processing, or a downstream dependency. Then I would reproduce it from Service A's actual runtime environment, verify the effective host, port, scheme, path, and proxy, and test DNS, TCP, TLS, and HTTP separately. I would confirm that Service B is listening and ready, inspect every gateway or load balancer hop, correlate logs and traces by request ID and backend instance, and check B's dependencies if the request reached it. Finally, I would verify the correction with the original business request across all instances and add prevention rather than only restarting a service.

---

# 3. Question 2 - Connection Refused

## Interview question

> Service A is getting `Connection refused` while calling Service B. What could be the reasons, and how would you debug it?

## What `Connection refused` means

During a normal TCP handshake:

```text
Service A                         Service B
   | -------- SYN -----------------> |
   | <----- SYN-ACK ---------------- |
   | -------- ACK -----------------> |
   |        connection established   |
```

A refusal commonly looks like:

```text
Service A                         Destination
   | -------- SYN -----------------> |
   | <--------- RST ---------------- |
   |        connection refused       |
```

The refusal often proves that packets reached the destination IP or a network device that actively rejected the connection. It does **not** prove that the Service B application was reached.

## Where the refusal can originate

- The operating system on the Service B host because no process listens on that port.
- A container or pod IP where the application has not started listening.
- A load balancer listener with no matching port or target.
- A reverse proxy or sidecar that rejects the connection.
- A firewall configured to `REJECT` rather than silently `DROP`.
- A stale IP belonging to a different host.

## Detailed causes

| Cause | Why it produces a refusal | Evidence |
|---|---|---|
| Service B process is stopped or crashed | The host has no listener for the port | No listening socket; startup/crash logs |
| Wrong destination port | Another port is used or nothing listens on the configured port | Config says `8080`; `ss` shows `8081` |
| Application startup is incomplete | Container is running before the web server binds | Startup logs stop before "server started"; readiness false |
| Listener is bound only to loopback | Remote packets arrive on another interface with no matching listener | `127.0.0.1:8080` instead of `0.0.0.0:8080` |
| Wrong or stale IP | DNS/discovery points to a host that does not run B | DNS answer differs from current service endpoints |
| Kubernetes `targetPort` is wrong | Service forwards to a port the pod does not expose | Service `targetPort` differs from pod listener |
| Load balancer includes a dead target | Some connections are sent to an instance with no listener | Failures correlate with one target IP |
| Rolling deployment race | Traffic reaches a terminating or not-yet-ready instance | Refusals occur during rollout; endpoint removal is delayed |
| Firewall actively rejects | A network device sends a reset or reject response | Packet capture or firewall logs identify reject |
| Protocol endpoint is wrong | Client connects to a port not configured for that service | Listener inventory and gateway configuration mismatch |
| Host port is not published | Container listens internally, but host/NAT mapping is absent | Container works locally; host port has no listener |
| Service-mesh sidecar is not ready | Traffic is redirected to a proxy listener that is unavailable | Sidecar startup/readiness and redirect rules |

## Step-by-step debugging

### Step 1 - Confirm that this is a refusal

Capture the exact host, resolved IP, port, and nested exception:

```text
ConnectException: Connection refused
destination=10.20.4.18:8080
```

An immediate failure strongly supports refusal. A failure only after the configured connect timeout is more likely a packet drop or reachability issue.

### Step 2 - Resolve and record every destination IP

```powershell
Resolve-DnsName service-b
```

```bash
getent ahosts service-b
```

If multiple IPs are returned, test each one. One stale address can make a refusal intermittent.

### Step 3 - Reproduce the TCP connection from Service A

```powershell
Test-NetConnection service-b -Port 8080 -InformationLevel Detailed
```

```bash
nc -vz -w 5 service-b 8080
```

Test the hostname first, then each resolved IP. Record whether all destinations refuse or only one does.

### Step 4 - Check the listener on Service B

Windows:

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen
netstat -ano | findstr :8080
```

Linux:

```bash
ss -lntp | grep ':8080'
```

Confirm:

- A listener exists.
- The owning process is Service B.
- The port is correct.
- The bind address is reachable remotely.
- Both IPv4 and IPv6 behavior match the client.

Examples:

```text
127.0.0.1:8080   -> local callers only
0.0.0.0:8080     -> all IPv4 interfaces
[::]:8080        -> IPv6 and sometimes dual-stack, depending on OS settings
```

### Step 5 - Inspect application startup

Look for:

- Port already in use.
- Invalid configuration.
- Missing secret.
- Failed dependency initialization.
- Certificate/key load failure.
- Framework started but embedded HTTP server failed.
- Crash loop or out-of-memory termination.

A `Running` container state is not enough. Confirm the application emitted its normal "ready/listening" event.

### Step 6 - Check container or Kubernetes port mapping

For Kubernetes:

```bash
kubectl get service -n <namespace> service-b -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o yaml
kubectl get pods -n <namespace> -l app=service-b -o wide
```

Verify:

```text
Service port -> targetPort -> container listener
selector -> intended pods
endpoint IPs -> current ready pods
```

The `containerPort` field documents a port but does not itself make the application listen. The actual process listener is what matters.

### Step 7 - Check load balancer and service discovery

Confirm that:

- The rejected IP is a current backend.
- Health checks use the correct port and protocol.
- Draining targets are removed before their listener closes.
- Registration is not stale.
- Traffic is not sent to a startup-incomplete instance.

### Step 8 - Check reject rules only after listener checks

Inspect host firewall, security appliance, Kubernetes policy, or proxy logs for an explicit reject. Most dropped firewall traffic causes a timeout, while an explicit reject can cause an immediate refusal.

### Step 9 - Correct and prove

Typical permanent fixes:

- Start or stabilize the Service B process.
- Correct host/port configuration.
- Bind to the intended interface.
- Correct Service `targetPort`, load balancer target port, or container publishing.
- Make readiness depend on the HTTP listener being ready.
- Add graceful shutdown and endpoint draining.
- Remove stale discovery records.
- Alert on missing listeners, failed readiness, and target-health changes.

Verify from Service A, through the normal hostname and intermediary path, and across every returned backend.

## Interview-ready answer

> Connection refused usually means the TCP attempt reached a host or device that actively rejected the port, commonly because nothing is listening. I would capture the resolved IP and port, reproduce from Service A, and test every resolved address. On B I would verify the process, startup completion, listening socket, bind address, and owning process. In containers or Kubernetes I would trace Service port to `targetPort` to the real pod listener and inspect endpoint readiness. I would also check for stale load-balancer targets, rollout races, sidecar readiness, and explicit firewall rejects. I would fix the incorrect listener or routing state and verify the real path from A rather than treating a restart as the root cause.

---

# 4. Question 3 - Connection Timeout

## Interview question

> Service A is getting a connection timeout while calling Service B. How would you troubleshoot it?

## What a connection timeout means

A connection timeout occurs before the TCP connection is established. Service A sends connection attempts, but the handshake does not complete before the client connect deadline.

```text
Service A                         Service B or network
   | -------- SYN -----------------> |
   |                                  |
   | -------- retry SYN ------------> |
   |                                  |
   | -------- retry SYN ------------> |
   |                                  |
   X connect timeout
```

This differs from:

- `Connection refused`: an active rejection normally arrives quickly.
- Read timeout: a connection was established, but response data was late.
- TLS timeout: TCP may have succeeded, but secure negotiation stalled.

## Detailed causes

| Cause | Where it occurs | Why the handshake does not finish |
|---|---|---|
| Firewall or security group drops traffic | Source, destination, or intermediate network | SYN or SYN-ACK is silently discarded |
| Kubernetes NetworkPolicy blocks A-to-B traffic | Cluster network | Policy denies ingress or egress |
| Missing or incorrect route | Host, subnet, VPC/VNet, VPN, peering | Packet cannot reach the destination or return path |
| Wrong IP from config, DNS, or discovery | Client/discovery | Client targets an unused or unreachable address |
| Wrong port with silent filtering | Network/device | Traffic to that port is dropped instead of rejected |
| Asymmetric routing | Network | SYN reaches B, but SYN-ACK returns through a path that drops it |
| NAT/SNAT or ephemeral-port exhaustion | Egress gateway, node, host | New outbound connections cannot obtain usable translation state |
| Connection-tracking table exhaustion | Host, node, firewall | New flows are dropped |
| Destination or load balancer overload | Backend/LB | Accept path, SYN backlog, or appliance capacity is exhausted |
| Proxy required but bypassed, or unwanted proxy used | Service A runtime | Request goes to an unreachable direct path or proxy |
| Private endpoint used from an unconnected network | Cross-environment networking | No valid route or permission exists |
| IPv6/IPv4 mismatch | DNS/client/network | Client selects an address family not supported end to end |
| Cloud network ACL mismatch | Subnet boundaries | Stateless return-path rule blocks ephemeral traffic |
| Node or zone network fault | Infrastructure | Only workloads on a node/AZ experience timeouts |

## Step-by-step debugging

### Step 1 - Verify the timeout phase and value

Confirm the exception says **connect** timeout and record:

- Connect timeout configured by Service A.
- Number of retries and retry delay.
- Total elapsed time.
- Resolved destination IP.

Retries can make a 3-second connect timeout appear as a 12-second application failure. Understand the whole retry timeline.

### Step 2 - Compare scope

Ask:

- Do all Service A instances time out?
- Does one node, subnet, region, or availability zone fail?
- Do all destination IPs fail?
- Did the issue begin with a network, DNS, scaling, or deployment event?
- Are only new connections failing while existing keep-alive connections work?

If existing connections work but new ones time out, investigate NAT ports, conntrack, accept queues, load balancer capacity, and new-connection rate.

### Step 3 - Resolve DNS and test every address

```powershell
Resolve-DnsName service-b
Test-NetConnection service-b -Port 8080 -InformationLevel Detailed
```

```bash
getent ahosts service-b
nc -vz -w 5 service-b 8080
```

Test from each affected Service A instance when the problem is instance-specific.

### Step 4 - Compare direction and return path

Network communication requires both directions:

```text
A -> B : SYN
B -> A : SYN-ACK
```

An ingress rule may allow the SYN while an egress, network ACL, route, NAT, or asymmetric path drops the SYN-ACK. Check both source and destination routes and policies.

### Step 5 - Inspect network controls

Trace the actual path and evaluate:

- Source host firewall.
- Destination host firewall.
- Cloud security groups.
- Stateless network ACLs in both directions.
- Kubernetes ingress and egress NetworkPolicies.
- Service-mesh authorization and egress policy.
- VPC/VNet route tables, peering, VPN, transit gateway, private-link rules.
- NAT gateway and SNAT utilization.
- Corporate proxy and `NO_PROXY` behavior.

Do not assume a rule applies because its name sounds correct. Verify source identity/CIDR, destination CIDR, protocol, destination port, and return ephemeral ports.

### Step 6 - Inspect the destination

Even though a missing listener often refuses immediately, a destination under severe load or protected by a dropping firewall may time out. Check:

- Listener and bind address.
- SYN backlog and accept queue.
- Host CPU and network saturation.
- Load balancer target and listener capacity.
- Node conntrack usage.
- Pod/node health.

### Step 7 - Use packet evidence when standard checks are inconclusive

With authorization, capture narrowly filtered packets on the source and destination:

```bash
tcpdump -nn -i any host <destination-ip> and port 8080
```

Interpretation:

| Source capture | Destination capture | Likely conclusion |
|---|---|---|
| SYN leaves; B sees no SYN | No inbound SYN | Drop or route problem before B |
| SYN leaves; B sees SYN and sends SYN-ACK; A sees no SYN-ACK | Return packet missing | Return-route, ACL, firewall, NAT, or asymmetry |
| A sees SYN and immediate RST | Active rejection | Treat as refusal, inspect listener/rejecting device |
| Handshake completes | Not a connect timeout at this layer | Move to TLS, HTTP, or read phase |

Packet capture should be targeted, approved, short-lived, and protected because payloads and addresses can be sensitive.

### Step 8 - Check saturation and new-connection behavior

Review:

- New connections per second.
- Active and failed connections.
- NAT/SNAT port use.
- Conntrack table use.
- Load balancer rejected connections.
- TCP retransmissions.
- SYN backlog drops.
- File descriptor usage.

A retry storm can worsen exhaustion. If many clients retry simultaneously, use capped exponential backoff with jitter and a strict overall deadline.

### Step 9 - Fix and verify

Permanent fixes may include:

- Correct routes, security groups, ACLs, firewall, or NetworkPolicy.
- Correct the destination IP, hostname, port, or address family.
- Add or repair peering/private connectivity.
- Increase NAT or load-balancer capacity after proving exhaustion.
- Reuse connections and configure pools to reduce connection churn.
- Fix proxy and `NO_PROXY` configuration.
- Repair node networking or replace a faulty node through the normal operational process.
- Add synthetic TCP checks from the same network zones as Service A.

Verify new connections, not only already-open pooled connections.

## Interview-ready answer

> A connection timeout means the TCP handshake did not finish before the connect deadline. I would first confirm the phase, timeout, retries, destination IP, and affected source and destination instances. From Service A I would resolve DNS and test each address and port. Then I would check both forward and return routes, source and destination firewalls, security groups, stateless ACLs, Kubernetes ingress and egress policies, proxy rules, NAT/SNAT capacity, and load-balancer state. If necessary, a narrow packet capture would show whether the SYN reaches B and whether the SYN-ACK returns. I would also investigate new-connection saturation such as conntrack, NAT ports, or SYN backlog. After correcting the actual path or capacity issue, I would verify fresh connections from every affected zone.

---

# 5. Question 4 - Read Timeout

## Interview question

> Service A connects to Service B, but the response takes too long and eventually gets a read timeout. What could be happening?

## What a read timeout means

The client established a connection and then waited too long for response data. Depending on the HTTP library, the timeout may apply to:

- Time until the first response byte.
- Maximum idle time between response bytes.
- A socket read operation.
- The entire response body.

Confirm the library's exact definition. An overall request deadline is different from a socket read timeout.

```text
A -- TCP/TLS connected --> B
A -- request -----------> B
                          B queues or processes
                          B waits for DB/C/resource
A <--- no response within read timeout
X read timeout
```

## Where time can be spent

```text
Client pool wait
  + proxy/gateway queue
  + Service B accept/request queue
  + Service B thread queue
  + application code
  + DB pool wait
  + DB execution/lock wait
  + downstream HTTP pool wait
  + downstream service latency
  + serialization/compression
  + response network transfer
  = end-to-end latency
```

## Detailed causes

| Cause | Mechanism | Evidence |
|---|---|---|
| Slow business logic | CPU-heavy algorithm, lock, synchronous work, inefficient loop | Long in-process trace span; CPU or profiler evidence |
| Request queueing | All worker threads/event loops are busy | Queue depth and active threads at maximum |
| Thread-pool exhaustion | Request waits for a worker | Pool active=max, queued tasks rising |
| DB connection-pool exhaustion | B waits to borrow a DB connection | Pool wait time and pending borrowers rise |
| Slow query or missing index | DB execution exceeds normal latency | Query span, execution plan, slow-query log |
| DB blocking or lock contention | Query waits behind another transaction | DB lock/wait diagnostics |
| Downstream Service C is slow | B's deadline is consumed by C | Trace shows long B-to-C span |
| HTTP client pool exhaustion | B waits for a reusable downstream connection | Pending connection requests and pool max |
| CPU throttling or saturation | Work receives too little CPU time | CPU limit/throttle metrics, run queue |
| GC pause or memory pressure | Application pauses or spends time collecting | GC pause metrics/logs, heap pressure |
| Deadlock or blocked thread | Work never progresses | Thread dump or runtime diagnostics |
| Large request or response | Parse, serialize, compress, and transfer take longer | Duration correlates with payload size |
| Slow external system | Provider latency holds B open | Downstream metrics and provider logs |
| Network stall after connection | Packets are lost or flow is interrupted | Retransmissions, proxy/network metrics |
| Timeout is below valid service objective | Healthy long-running operation exceeds caller limit | Baseline p99 is greater than configured timeout |
| Retry amplification | A or B repeats slow operations and increases load | Multiple attempts per request ID; load spikes |
| Synchronous work should be asynchronous | Long job is incorrectly held inside HTTP request | Endpoint waits for batch/report/export completion |

## Step-by-step debugging

### Step 1 - Identify who timed out

Determine whether the message came from:

- Service A HTTP client.
- Service A's inbound gateway.
- A service-mesh sidecar.
- API gateway or load balancer.
- Service B's downstream client.

Record connect, read, write, idle, and total/request deadlines at every hop. The shortest deadline normally fires first.

### Step 2 - Confirm that Service B received the request

Use request ID, trace ID, timestamp, method, and path.

- No B access log or trace: delay may be before B, in a gateway queue, connection reuse problem, request-write phase, or wrong target.
- B start log exists but no completion: investigate B and its dependencies.
- B completed before A timed out: investigate response delivery, proxy buffering, stale connection, client parsing, or mismatched clocks/timestamps.
- B completed after A timed out: investigate B latency and cancellation handling.

### Step 3 - Build a latency waterfall

Distributed tracing is ideal:

```text
Total request:                5.02 s
  Gateway queue:              0.04 s
  Service B queue:            0.82 s
  Service B application:      0.11 s
  DB connection wait:         1.20 s
  DB query:                   2.73 s
  Serialization/response:     0.12 s
```

The waterfall prevents blaming the longest-looking component without accounting for queue time.

Without tracing, correlate logs with monotonic durations where available:

```text
A call start
B request start
B downstream start/end
B request end
A timeout
```

### Step 4 - Separate queue time from execution time

Low CPU does not prove the service is healthy. A service can be idle while all threads wait on locks, pools, or downstream I/O.

Inspect:

- Request queue length.
- Active, idle, maximum, and rejected worker threads.
- Event-loop blocked time for nonblocking runtimes.
- DB/HTTP pool active, idle, pending, acquire time, and timeout.
- Semaphore or bulkhead utilization.
- Lock waits and thread states.

### Step 5 - Inspect resource metrics per instance

Check the same time window and instance:

- CPU utilization and CPU throttling.
- Memory working set/heap and allocation rate.
- GC frequency and pause duration.
- Thread count and blocked threads.
- File descriptors and sockets.
- Network errors/retransmissions.
- Disk latency if local I/O is involved.
- Container restarts and node pressure.

Use percentiles rather than averages. A normal average can hide a severe p99 tail.

### Step 6 - Inspect the database

Look for:

- Pool wait before query execution.
- Slow query duration.
- Lock/blocking time.
- Missing/unused index.
- Bad execution plan or parameter sensitivity.
- Large result set.
- Connection errors or failover.
- Database CPU, I/O, memory, and connection saturation.
- Transaction held open too long.

Do not add an index or terminate a query solely from an application timeout. Confirm with database evidence and follow the database change process.

### Step 7 - Inspect every downstream call

For each `B -> C` call, record:

- DNS/connect/TLS/request/response duration.
- Timeout and retry policy.
- Circuit-breaker state.
- Pool wait.
- Status code.
- Downstream instance.

Ensure B's downstream timeout fits inside B's own remaining deadline. Otherwise, A can time out while B is still waiting on C.

### Step 8 - Check request-specific patterns

Compare fast and slow requests by:

- Endpoint and method.
- Tenant/customer.
- Record ID and data shape.
- Payload and response size.
- Cache hit/miss.
- Query plan.
- Feature flag.
- Destination instance.
- Time of day and traffic level.

### Step 9 - Check timeout and retry design

A good timeout budget is layered:

```text
Caller total deadline
  > gateway timeout plus network margin
  > Service B internal deadline plus response margin
  > each downstream timeout and bounded retries
```

Do not automatically increase the timeout. That can:

- Keep threads and connections occupied longer.
- Increase queue length.
- Hide a regression.
- Cause more work to continue after callers abandon it.
- Turn a fast failure into a large cascading outage.

Increase a timeout only if the operation's valid service-level objective requires it and the system has enough capacity.

### Step 10 - Fix the bottleneck and verify under representative load

Possible fixes:

- Optimize query/code or reduce result size.
- Remove lock contention.
- Correct pool sizing based on dependency capacity.
- Add bounded concurrency and backpressure.
- Move legitimately long work to an asynchronous job.
- Propagate deadlines and cancellation.
- Tune CPU/memory only after proving resource constraint.
- Correct retries and add exponential backoff with jitter.
- Cache suitable data with correct invalidation.
- Align timeout budgets with the service objective.

Verify p50, p95, p99, timeout rate, pool wait, queue depth, dependency latency, and resource saturation.

## Interview-ready answer

> A read timeout means the connection was established but response data did not arrive within the client's read deadline. I would identify which component timed out and its exact timeout semantics, then use a request or trace ID to confirm whether B received and completed the request. I would build a latency waterfall that separates gateway and service queueing, application execution, pool waits, database time, downstream calls, and response transfer. I would inspect per-instance CPU, throttling, GC, threads, connection pools, locks, query plans, and downstream latency, and compare slow requests with successful ones by endpoint, data, payload, and instance. I would fix the measured bottleneck and align deadlines; I would not simply increase the timeout because that can worsen saturation and hide the root cause.

---

# 6. Question 5 - Intermittent Timeouts

## Interview question

> Service A to Service B works sometimes but times out sometimes. How would you investigate?

## Why intermittent failures are different

A complete outage tells you the path is consistently broken. An intermittent outage proves that at least one combination of source, destination, request, time, or dependency works. The investigation should therefore find the dimension that differs between success and failure.

Think in a matrix:

```text
source A instance
destination B instance
node / zone / region
resolved IP
endpoint / method
tenant / record / payload
time / traffic level
connection reused or new
dependency selected
software/config version
```

## Detailed causes

| Pattern | Common cause |
|---|---|
| Failure rate is close to `1 / number of backends` | One bad Service B instance |
| Failures occur only on one Service A pod/node | Source-side DNS, route, proxy, certificate, node, NAT, or config issue |
| Failures occur during traffic peaks | Thread/pool/CPU/DB capacity exhaustion |
| Failures occur at regular intervals | GC, scheduled job, cache refresh, connection eviction, certificate task |
| Failures follow deployment/scaling | Startup readiness, draining, mixed versions, cold cache |
| Failures affect certain tenants or records | Data-specific query, lock, code path, payload size |
| Failures affect new connections only | NAT/conntrack/SYN backlog/TLS handshake capacity |
| Failures affect reused connections only | Stale keep-alive connection or mismatched idle timeouts |
| Failures map to one DNS answer | Stale or unhealthy address in a multi-record response |
| Failures occur after retry bursts | Retry amplification and cascading saturation |
| Failures occur in one zone | Network path, node, zone-local dependency, or uneven routing |
| Failures occur on cache misses | Slow DB or downstream path hidden by cache hits |

Other causes include:

- Uneven load-balancer weights or sticky sessions.
- Connection-pool exhaustion under bursts.
- Thread-pool queue spikes.
- DB lock contention or plan changes for particular parameters.
- Periodic long GC pauses.
- External provider variability.
- Autoscaling that reacts too slowly.
- DNS TTL/cache differences between instances.
- Rolling deployment with old and new versions.
- Packet loss or flaky network interface.
- Service-mesh sidecar resource starvation.

## Step-by-step debugging

### Step 1 - Quantify instead of sampling one failure

Measure:

- Requests and failures per minute.
- Timeout rate and latency percentiles.
- Start/end time.
- Burst or continuous pattern.
- Affected percentage.
- Retry attempts and final failures.

An average latency graph is insufficient. Inspect p95/p99/max and timeout count.

### Step 2 - Build success and failure datasets

For a representative time window, collect:

```text
timestamp
request/trace ID
source instance/node/zone
resolved destination IP
destination instance/node/zone
endpoint/method
tenant or safe data category
payload/response size
status or exception
connect duration
time to first byte
total duration
retry attempt
```

Do not log sensitive payload contents merely to diagnose latency.

### Step 3 - Group failures by dimension

Examples:

```text
failure rate by destination instance
failure rate by source instance
failure rate by endpoint
failure rate by zone
failure rate by five-minute interval
failure rate by payload-size bucket
```

The strongest correlation determines the next investigation. Avoid changing all instances before identifying whether one is different.

### Step 4 - Test the one-bad-instance hypothesis

Inspect load-balancer access logs, trace attributes, response headers, or application logs for backend identity. Compare direct, approved tests to each backend while preserving required `Host` and TLS SNI values.

If one instance is bad:

1. Drain it from traffic to protect users.
2. Preserve logs, metrics, runtime state, and configuration before replacing it when safe.
3. Compare it with a healthy instance.
4. Find why it became different.

### Step 5 - Correlate with saturation

Overlay timeout rate with:

- Request rate and concurrency.
- Worker queue and active threads.
- DB and HTTP pool pending counts.
- CPU throttling.
- GC pauses.
- DB latency/locks/connections.
- NAT/SNAT and conntrack use.
- Load balancer active/new/rejected connections.
- Downstream latency and error rate.

Temporal alignment is essential. Current healthy metrics do not explain a spike from 30 minutes earlier.

### Step 6 - Check DNS and endpoint rotation

Record all answers and test each IP. Check:

- Multi-A/AAAA record contains a bad destination.
- Clients cache answers for different durations.
- A stale record survived deployment.
- Headless Kubernetes service returns an unready pod.
- Service discovery registration/deregistration is delayed.

### Step 7 - Check connection reuse

Compare:

- Forced new connection versus normal keep-alive.
- Idle age of failed connections.
- Client pool idle timeout.
- Gateway/load-balancer idle timeout.
- Server keep-alive timeout.

If an intermediary closes idle connections before the client evicts them, the client can occasionally reuse a stale socket. Align idle settings or validate connections before reuse according to the client library's supported mechanisms.

### Step 8 - Check retry behavior

Find whether one user request creates multiple backend attempts. Verify:

- Only idempotent operations are retried unless protected by an idempotency key.
- Retries are bounded.
- Backoff includes jitter.
- Retry budget is below the overall deadline.
- Multiple layers are not each retrying.

Retries may hide the first failures while multiplying load until the service collapses.

### Step 9 - Reproduce safely

Use production-safe, rate-limited diagnostics or reproduce in a representative nonproduction environment. Do not generate uncontrolled load during an incident. A test must retain the suspected variables: instance, network path, headers, payload size, auth identity, and cache state.

### Step 10 - Fix, monitor, and prevent

The fix depends on the proven dimension:

- Remove configuration/version drift.
- Correct load-balancer health/draining.
- Fix query, dependency, pool, or resource bottleneck.
- Correct DNS/discovery lifecycle.
- Align keep-alive/idle timeouts.
- Repair a node/zone network issue.
- Add backpressure, deadline propagation, and safe retry policy.
- Improve autoscaling using a leading saturation signal rather than CPU alone.

Verify the failure distribution becomes zero or returns to the accepted baseline across all dimensions, not only in aggregate.

## Interview-ready answer

> Because some requests succeed, I would compare successful and failed requests rather than treat B as globally down. I would collect request ID, timestamp, source and destination instance, node and zone, resolved IP, endpoint, payload-size category, latency phase, and retry attempt. I would group failures by each dimension and correlate them with traffic, queues, pools, CPU throttling, GC, database and downstream latency, DNS answers, and load-balancer targets. I would specifically test for one bad backend, one bad source node, peak-load saturation, stale keep-alive connections, DNS rotation, and retry amplification. I would drain an unhealthy instance if needed, preserve evidence, fix the actual difference, and verify every instance and traffic period.

---

# 7. Question 6 - Service B Is Running, but Service A Cannot Reach It

## Interview question

> Service B is running, but Service A cannot reach it. What would you check?

## "Running" is not the same as "reachable"

These are separate states:

```text
Process exists
    -> application initialized
    -> server socket listens
    -> correct interface and port
    -> instance is ready
    -> service discovery includes it
    -> network path permits A
    -> proxy/load balancer routes to it
    -> TLS succeeds
    -> HTTP route accepts the request
```

A container may be `Running` while the application is in a crash loop, blocked during startup, listening only on loopback, not ready, or exposed through an incorrectly configured service.

## Detailed causes by location

### On Service B

- Process exists but HTTP server failed to start.
- Server listens on a different port.
- Server binds to `127.0.0.1`.
- Server listens only on IPv6 while the path uses IPv4, or the reverse.
- Local firewall blocks remote traffic.
- Application is alive but not ready.
- Host is resource-starved and does not accept promptly.

### In containers or Kubernetes

- Container port is not published.
- Kubernetes Service selector does not match pod labels.
- Service `targetPort` does not match the application listener.
- No ready EndpointSlices exist.
- Pod IP is stale in discovery.
- NetworkPolicy denies ingress or Service A egress.
- Sidecar interception/mTLS policy rejects the path.
- Service exists in another namespace and the short name resolves incorrectly.

### In the network or intermediary

- Wrong DNS record or environment.
- Wrong route, subnet, peering, VPN, or private endpoint.
- Firewall, security group, or ACL blocks the port.
- Load balancer listener or backend pool is wrong.
- Health check marks the backend unavailable.
- Proxy is required but Service A bypasses it, or internal traffic is incorrectly sent through it.

## Step-by-step debugging

### Step 1 - Define what "running" evidence exists

Ask whether "running" means:

- OS process is visible.
- Container state is running.
- Startup completed.
- Port is listening.
- Liveness passed.
- Readiness passed.
- Real business endpoint succeeded locally.

Do not accept one of these as proof of the others.

### Step 2 - Test locally on B

Check the listener:

```bash
ss -lntp | grep ':8080'
curl -v --connect-timeout 2 --max-time 10 http://127.0.0.1:8080/health
curl -v --connect-timeout 2 --max-time 10 http://<b-host-ip>:8080/health
```

Interpretation:

- Loopback works, host IP fails: bind address or host firewall.
- Both fail: listener/application problem.
- Both work: move outward to service exposure and network path.

### Step 3 - Test from progressively closer locations

Use a hop-by-hop approach:

```text
B local loopback
B host/pod IP
another pod/host in B's subnet or namespace
Service A pod/host
gateway/load balancer path
```

The first boundary where the test changes from success to failure identifies the likely fault domain.

### Step 4 - Validate effective Service A configuration

Confirm Service A uses the correct:

- Environment.
- Hostname/FQDN.
- Namespace.
- Port.
- Scheme.
- Base path.
- Proxy bypass.
- TLS trust and client identity.

### Step 5 - Validate Kubernetes service wiring

```bash
kubectl get service -n <namespace> service-b -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o wide
kubectl get pods -n <namespace> -l app=service-b --show-labels
```

Trace:

```text
DNS name
  -> Service ClusterIP and port
  -> selector
  -> ready EndpointSlice addresses and target ports
  -> pod IP
  -> actual process listener
```

### Step 6 - Validate network permissions in both directions

Check routes, security groups, ACLs, NetworkPolicies, mesh policies, and host firewalls using the real source and destination identities. A broad test from an administrator workstation may bypass the restriction affecting Service A.

### Step 7 - Validate TLS and HTTP

If TCP succeeds, stop saying "cannot reach." State the later failure precisely:

- TLS certificate failure.
- mTLS client identity failure.
- 401/403 authorization.
- 404 route mismatch.
- 500 business failure.

### Step 8 - Correct and prevent

Typical fixes:

- Make startup fail visibly if the server cannot bind.
- Configure correct bind address and port.
- Correct container/service/load-balancer mapping.
- Make readiness represent traffic acceptance.
- Use graceful startup/shutdown and endpoint draining.
- Correct network and mesh policy.
- Add an end-to-end synthetic check from Service A's network zone.

## Interview-ready answer

> I would challenge what "running" means. A process or running container does not prove that the application initialized, listens on the expected interface and port, is ready, is registered, or is reachable from A. I would test B locally on loopback and its host or pod IP, verify the socket and bind address, then test progressively from the same subnet and finally from A. In Kubernetes I would trace DNS to Service port, selector, EndpointSlice, pod IP, and actual listener, and inspect ingress and egress policies and sidecars. Once TCP succeeds I would classify any TLS or HTTP failure separately. The first boundary where success changes to failure identifies the owning layer.

---

# 8. Question 7 - Ping Works, but the API Call Fails

## Interview question

> Service A can ping Service B's server, but the API call fails. Why, and how would you troubleshoot it?

## What ping proves

`ping` normally uses ICMP echo. An API normally uses DNS, TCP, optionally TLS, and HTTP.

```text
ICMP success != TCP port success != TLS success != HTTP/API success
```

Ping can prove that one IP answers ICMP from one source. It does not prove:

- The API port is open.
- A process listens.
- A firewall allows TCP.
- TLS certificates are valid.
- The correct virtual host is selected.
- Authentication and authorization work.
- The URL path exists.
- Service B or its dependencies are healthy.

Some environments block ICMP even when APIs work, so ping failure is also not conclusive.

## Detailed causes

- ICMP is allowed, but TCP port `8080` or `443` is blocked.
- The server is reachable, but no process listens on the API port.
- Service A uses the wrong port or protocol.
- Service binds only to localhost.
- DNS used by ping resolves a different address than the application uses or caches.
- HTTPS is sent to an HTTP port, or HTTP is sent to a TLS port.
- TLS hostname, trust, SNI, version, or mTLS fails.
- Proxy or load balancer route is incorrect.
- Required `Host` header or virtual-host mapping is missing.
- API path, method, base path, or version is wrong.
- Authentication token is absent, expired, wrong audience, or unauthorized.
- Request content type or payload is invalid.
- API responds with 500 due to application/dependency failure.
- Response is slow and times out.

## Step-by-step debugging

### Step 1 - Record what ping actually tested

Capture the hostname and IP shown by ping:

```text
Pinging service-b [10.20.4.18]
```

Compare it with Service A's DNS result and application logs. A hostname can return multiple addresses, and the application may select another one.

### Step 2 - Test the API's TCP port

```powershell
Test-NetConnection service-b -Port 8080 -InformationLevel Detailed
```

```bash
nc -vz -w 5 service-b 8080
```

- TCP fails: investigate listener, port, route, firewall, or policy.
- TCP succeeds: continue to TLS/HTTP; do not keep using ping as evidence.

### Step 3 - Test protocol and TLS

```bash
curl -v http://service-b:8080/health
curl -v https://service-b:443/health
```

Use only the protocol the service is intended to expose. Errors such as "wrong version number" often indicate HTTPS was sent to a plain HTTP port or vice versa.

For TLS:

```bash
openssl s_client -connect service-b:443 -servername service-b -showcerts </dev/null
```

### Step 4 - Test the real HTTP contract

Verify:

- Method: GET, POST, PUT, and so on.
- Full path and gateway prefix.
- `Host` header/virtual host.
- Content type and accept header.
- Authentication token and scope.
- Required correlation or tenant headers.
- Payload schema and size.

Use safe credentials and do not expose tokens in command history or shared logs.

### Step 5 - Interpret the HTTP result

| Result | Next area |
|---|---|
| No TCP connection | Network/listener/port |
| TLS error | Certificate/trust/SNI/mTLS/protocol |
| 400/415 | Request format/content type |
| 401/403 | Authentication/authorization |
| 404/405 | Path, route, version, or method |
| 429 | Rate/concurrency policy |
| 500 | Application or dependency |
| 502/503/504 | Gateway and backend relationship |
| Read timeout | Queue, application, dependency, response |

### Step 6 - Compare command and application paths

If `curl` succeeds but Service A fails, compare:

- URL after configuration substitution.
- Proxy and `NO_PROXY`.
- DNS cache.
- Trust store and client certificate.
- HTTP library protocol/version.
- Connection pool.
- Headers and payload.
- Timeout and retry policy.
- Service-mesh interception.

## Interview-ready answer

> Ping only tests ICMP reachability to an IP; it does not test the API's TCP port, TLS, HTTP route, identity, or application. I would compare the IP that ping used with Service A's actual DNS result, then test the exact TCP port from Service A. If TCP succeeds, I would test TLS with the correct SNI and then the real HTTP method, path, headers, authentication, and payload. I would use the resulting status or exception to move to the correct layer. A 401, 404, or 500 proves much more HTTP progress than ping does.

---

# 9. Question 8 - DNS Resolution Is Failing

## Interview question

> DNS resolution for Service B is failing. How would you troubleshoot it?

## DNS failure types

Do not group every DNS problem under `UnknownHostException`.

| DNS result | Meaning |
|---|---|
| `NXDOMAIN` | The queried name is reported not to exist |
| `NOERROR` with no useful answer | Name may exist but requested record type is absent |
| `SERVFAIL` | Resolver could not complete the lookup, often due to upstream, DNSSEC, or server failure |
| Query timeout | Resolver was unreachable, overloaded, or DNS traffic was blocked/dropped |
| Correct query returns wrong/stale IP | Record, cache, split-horizon view, or registration is wrong |
| Some queries work, some fail | Multiple resolvers, packet loss, load, truncation/TCP fallback, or per-node issue |
| Short name fails, FQDN works | Search suffix, namespace, or `ndots` behavior |

## Where DNS can fail

```text
Application DNS cache
  -> OS resolver/cache
  -> container resolver configuration
  -> node-local DNS cache
  -> cluster/corporate recursive resolver
  -> authoritative DNS or service discovery
```

## Detailed causes

- Typo or wrong environment/namespace in the hostname.
- DNS record was never created or was deleted.
- Service discovery registration is missing or stale.
- Wrong record type: A, AAAA, CNAME, SRV, or private endpoint.
- Incorrect `/etc/resolv.conf`, Windows DNS adapter, search suffix, or resolver order.
- Kubernetes Service name or namespace is wrong.
- CoreDNS or node-local DNS is unavailable, overloaded, throttled, or misconfigured.
- UDP/TCP port 53 is blocked by firewall or NetworkPolicy.
- DNS response is truncated and TCP fallback is blocked.
- Split-horizon DNS gives different answers inside and outside the network.
- Negative result is cached after a record was created.
- Positive cache retains an old IP after backend replacement.
- Application DNS cache ignores or extends TTL.
- Multiple resolvers have inconsistent data.
- Search-domain expansion queries an unintended name.
- IPv6 AAAA is returned but the runtime cannot reach IPv6.
- Too many search attempts or DNS queries cause delay.
- Cloud private DNS zone is not linked to Service A's network.
- Headless service exposes pod addresses that are stale or unready.

## Step-by-step debugging

### Step 1 - Capture the exact requested name

Log or inspect the exact hostname after configuration is resolved:

```text
service-b
service-b.orders
service-b.orders.svc.cluster.local
service-b.prod.internal.example
```

Trailing dots, hidden whitespace, wrong case in non-DNS discovery systems, a URL accidentally passed as a hostname, or an unresolved variable such as `${SERVICE_B_HOST}` can all matter.

### Step 2 - Query from the affected Service A instance

Windows:

```powershell
Resolve-DnsName service-b
nslookup service-b
Get-DnsClientServerAddress
```

Linux:

```bash
getent ahosts service-b
dig service-b
cat /etc/resolv.conf
```

`getent` is useful because it follows the host's normal name-service configuration, while `dig` directly tests DNS and may bypass hosts-file or other name-service sources.

### Step 3 - Record resolver, response code, answer, TTL, and timing

Check:

- Which resolver answered?
- Was it `NXDOMAIN`, `SERVFAIL`, timeout, or a wrong answer?
- Which A and AAAA records were returned?
- What are the TTLs?
- Does querying the FQDN change the result?
- Do repeated queries alternate between resolvers or answers?

### Step 4 - Compare working and failing instances

Compare:

- Resolver addresses.
- Search domains and `ndots`.
- Hosts-file entries.
- Node or subnet.
- DNS cache state.
- Container runtime DNS settings.
- Application/runtime DNS caching.
- NetworkPolicy and firewall.

If only one node's pods fail, node-local DNS or that node's network path becomes a strong suspect.

### Step 5 - Query a specific configured resolver

```bash
dig @<resolver-ip> service-b.example A
dig @<resolver-ip> service-b.example AAAA
```

If permitted and relevant, compare recursive and authoritative answers. Do not bypass approved DNS architecture as a permanent workaround.

### Step 6 - Check Kubernetes-specific objects

```bash
kubectl get service -n <namespace> service-b -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o wide
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=200
```

Check:

- The Service exists in the expected namespace.
- Service name is correct.
- CoreDNS is ready and not restarting.
- DNS error/latency metrics and upstream resolver health.
- Network policy allows DNS egress.
- Cluster domain matches configuration.

For a normal ClusterIP service, empty endpoints do not usually make the service name fail DNS; they make traffic have no usable backend. Headless service behavior is different because DNS answers are derived from endpoints.

### Step 7 - Check caching carefully

Identify caches at the application, JVM/runtime, OS, node, and recursive resolver. Flushing a cache can prove staleness, but first preserve the old answer and TTL as evidence.

Do not solve DNS by permanently hardcoding an IP. That bypasses failover, scaling, certificate hostname validation, and service discovery, and it creates a future outage.

### Step 8 - Confirm the answer is usable

After DNS succeeds:

1. Verify the returned address belongs to the intended environment/service.
2. Test every returned address and the intended port.
3. Check IPv4/IPv6 selection.
4. Confirm old addresses disappear after the expected TTL.

### Step 9 - Fix and prevent

Possible fixes:

- Correct hostname, namespace, record, or private-zone link.
- Repair resolver/CoreDNS health and capacity.
- Allow DNS UDP and TCP traffic.
- Correct search-domain or resolver configuration.
- Remove stale service registration.
- Set appropriate TTL and application cache behavior.
- Monitor DNS error codes, latency, saturation, and record freshness.

## Interview-ready answer

> I would identify whether DNS returns NXDOMAIN, SERVFAIL, timeout, an empty answer, or a wrong/stale address. I would query the exact name from the affected Service A instance, record the configured resolver, search domains, response code, A/AAAA answers, TTL, and timing, and compare those with a working instance. I would trace application cache, OS/container resolver, node-local or CoreDNS, recursive resolver, and authoritative/service-discovery data. In Kubernetes I would verify the Service and namespace, CoreDNS health, DNS egress policy, and EndpointSlice behavior for headless services. After correcting the DNS layer I would test every returned IP and the real API port; I would not hardcode an IP as the fix.

---

# 10. Question 9 - Hostname Works From a Laptop but Not From Service A

## Interview question

> The hostname works from your laptop but does not work from Service A. What could be wrong?

## Key principle

The laptop and Service A are different clients. They can have different:

- DNS resolvers and search domains.
- VPN and private-network access.
- Hosts-file entries and caches.
- Routes, proxies, and `NO_PROXY`.
- Firewalls, security groups, and NetworkPolicies.
- TLS trust stores and client certificates.
- Service identities and authorization policies.
- IPv4/IPv6 preference.
- Service-mesh interception.
- Environment-specific configuration.

A laptop success proves only that the laptop's path works.

## Common scenarios

| Laptop | Service A | Likely difference |
|---|---|---|
| Resolves private name through VPN | Pod resolver returns NXDOMAIN | Private DNS zone or resolver forwarding |
| Uses hosts-file override | Service A has no override | Laptop-only local configuration |
| Reaches public endpoint | A uses private endpoint | Different effective URL or split DNS |
| Bypasses proxy | A sends internal call to proxy | `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` |
| Trusts corporate CA | Container trust store lacks CA | TLS trust bundle |
| Has user credentials | Workload identity lacks permission | Authentication/authorization |
| Uses IPv4 | Runtime prefers unreachable IPv6 | Address-family selection |
| Is outside service mesh | A is subject to mTLS/egress policy | Sidecar or mesh authorization |

## Step-by-step debugging

### Step 1 - Compare the exact request

On both clients record:

```text
effective URL
resolved IPs
source IP/network
proxy used
TLS server name
HTTP method/path
authentication identity
```

Do not compare a browser request on the laptop with a different health URL from Service A and call them equivalent.

### Step 2 - Compare DNS

Run DNS tests from both environments and compare:

- Resolver server.
- Search suffix.
- A/AAAA answers.
- CNAME chain.
- TTL.
- response code.

If laptop is connected to a VPN, test whether the VPN supplies a private resolver or route unavailable to Service A's network.

### Step 3 - Compare route and TCP access

Test the exact destination port from both environments. Review Service A's subnet route, private-zone linkage, security identity, firewall, NetworkPolicy, and service-mesh egress rules.

### Step 4 - Compare proxy behavior

Inspect effective environment or runtime settings:

```text
HTTP_PROXY
HTTPS_PROXY
NO_PROXY
application-specific proxy configuration
```

Common errors:

- Internal hostname omitted from `NO_PROXY`.
- CIDR or domain matching behaves differently in the HTTP library.
- Service A is expected to use an egress proxy but connects directly.
- Proxy cannot resolve the private hostname even though A can.

### Step 5 - Compare TLS trust and identity

The laptop may trust a corporate CA installed by device management while a container has only a default public trust store. Compare:

- CA bundles.
- Certificate hostname.
- SNI.
- mTLS client certificate and key.
- Workload identity.
- Certificate validity and clock.

### Step 6 - Compare application behavior

If a shell request in the Service A container succeeds but the application fails, compare:

- Runtime DNS cache.
- Custom trust store.
- HTTP client proxy.
- Headers/token.
- URL construction.
- Connection pool.
- Timeouts.

### Step 7 - Fix the environment, not the symptom

Correct private DNS linkage, resolver forwarding, routes, policies, proxy bypass, trust bundles, or workload identity through managed configuration. Avoid copying a laptop hosts-file entry or disabling TLS verification.

## Interview-ready answer

> A laptop success does not prove Service A's environment works. I would run equivalent DNS, TCP, TLS, and HTTP tests from both clients and compare the effective URL, resolver and answers, route, proxy, trust store, client identity, and IPv4/IPv6 choice. VPN split DNS, laptop hosts-file entries, private-zone linkage, Kubernetes DNS, NetworkPolicy, `NO_PROXY`, corporate CA trust, and service-mesh mTLS are common differences. I would correct the managed runtime configuration and verify from Service A itself rather than using the laptop as the final test.

---

# 11. Question 10 - HTTP 502 From an API Gateway

## Interview question

> Service A receives HTTP 502 from the API gateway. How would you investigate?

## What 502 usually means

`502 Bad Gateway` normally means an intermediary acting as a gateway or proxy could not obtain or accept a valid response from its configured upstream.

```text
Service A -> Gateway -> Service B
                  |
                  X upstream connection/response problem
Service A <- 502
```

The exact reason is product-specific. Always identify the component that generated the response using headers, body format, access logs, route ID, and request ID.

## Detailed causes by phase

### Before connecting to B

- Upstream hostname cannot be resolved.
- Gateway resolves a stale or wrong IP.
- Wrong upstream host or port.
- No route or policy from gateway to B.
- Connection refused or reset.
- One backend target is unhealthy.

### During TLS to B

- HTTPS configured against an HTTP backend or the reverse.
- Certificate expired or not yet valid.
- Backend certificate hostname does not match.
- Gateway lacks the issuing CA.
- SNI selects the wrong virtual host.
- Backend requires mTLS but gateway has no valid client certificate.
- TLS version/cipher mismatch.

### During HTTP exchange

- Backend closes the connection before a complete response.
- Malformed HTTP status line or headers.
- Truncated/chunked response error.
- HTTP/1.1, HTTP/2, gRPC, or WebSocket protocol mismatch.
- Invalid upstream response headers.
- Response header exceeds gateway limit.
- Backend resets while serializing or streaming.
- Stale pooled connection was reused after backend/LB idle timeout.

### Routing/deployment causes

- Gateway route points to wrong service/version/port.
- Backend pool contains a terminating or startup-incomplete instance.
- Health check is too shallow or checks a different port.
- Deployment mixes incompatible protocol versions.
- Service-mesh sidecar rejects gateway traffic.

## Step-by-step debugging

### Step 1 - Identify the 502 generator

Capture:

- Response headers such as `Server`, `Via`, gateway request ID, or vendor error code.
- Gateway route/upstream name.
- Timestamp.
- Service A request/trace ID.
- Response body, redacted if necessary.

Service B can itself return a 502 if it acts as a gateway, so do not assume the first visible gateway generated it.

### Step 2 - Read the gateway's detailed error

Gateway logs often distinguish:

```text
DNS resolution failed
upstream connect error
connection refused
connection reset before headers
TLS handshake failure
upstream sent invalid header
upstream closed connection
no route to host
```

That subreason is more useful than the public 502 status.

### Step 3 - Determine whether B saw the request

- No B request log/trace: investigate gateway DNS, TCP, TLS, routing, target choice, or logging gaps.
- B saw request and crashed/reset: inspect B exception and process/resource state.
- B completed successfully but gateway returned 502: inspect response protocol, headers, streaming, gateway limits, connection reuse, and target correlation.

### Step 4 - Test gateway-to-B connectivity

Run the diagnostic from the gateway's network/runtime or an approved equivalent:

```text
resolve upstream hostname
connect to exact upstream port
perform TLS with gateway's SNI/trust/client identity
send expected Host header and route
```

A test from Service A directly to B can be useful for comparison, but it does not prove the gateway path works.

### Step 5 - Inspect all backend targets

Map 502s to:

- Upstream IP/instance.
- Node/zone.
- Application version.
- Connection reuse state.

If only one target fails, drain it and investigate as an instance-specific incident.

### Step 6 - Validate protocol configuration

Check both sides agree on:

- HTTP versus HTTPS.
- HTTP/1.1 versus HTTP/2 or h2c.
- gRPC support.
- SNI and `Host` header.
- WebSocket upgrade.
- Response header/body limits.
- Keep-alive and idle timeout.

### Step 7 - Check rollout and lifecycle

Inspect whether:

- Listener closes before target deregistration.
- Readiness becomes true before the full proxy/app stack is ready.
- Gateway keeps stale backend connections through deployment.
- Old and new versions use incompatible response behavior.

### Step 8 - Fix and prevent

Possible fixes:

- Correct upstream host, port, protocol, SNI, or trust.
- Repair gateway-to-B network permissions.
- Remove stale/unhealthy targets.
- Fix backend resets or malformed responses.
- Align keep-alive and idle behavior.
- Add proper readiness and graceful draining.
- Validate gateway routes and TLS in deployment tests.
- Alert on 502 subreason and backend target, not only total 5xx.

## Interview-ready answer

> I would first identify which proxy generated the 502 and read its product-specific upstream error. A 502 usually means the gateway failed to connect to B or did not receive a valid upstream response. I would correlate the gateway request ID with B and determine whether B saw the request. Then I would test DNS, TCP, TLS, SNI, protocol, `Host` header, and route from the gateway's environment, inspect every backend target, and check for resets, malformed or oversized headers, protocol mismatch, stale keep-alive connections, and rollout/draining races. I would fix the exact upstream failure and verify through the gateway path.

---

# 12. Question 11 - HTTP 503 From a Gateway

## Interview question

> Service A receives HTTP 503 from the gateway. What could be the cause, and how would you debug it?

## What 503 usually means

`503 Service Unavailable` means the component generating the response considers the service temporarily unable to handle the request. A gateway often returns it when no usable backend is available, but Service B can also intentionally return 503 because it is overloaded, in maintenance, not ready, or protected by a circuit breaker.

## Detailed causes

| Generator | Cause |
|---|---|
| Gateway/load balancer | No healthy backend targets |
| Gateway/load balancer | Route exists but backend pool/endpoints are empty |
| Gateway/load balancer | All targets failed health checks |
| Gateway/load balancer | Circuit breaker or outlier detection ejected all targets |
| Gateway/load balancer | Connection/concurrency queue or gateway capacity exhausted |
| Service discovery | No registered or ready instances |
| Kubernetes | Service selector mismatch or no ready EndpointSlices |
| Deployment | All old instances drained before new ones became ready |
| Service B | Explicit overload/load-shedding response |
| Service B | Maintenance mode or readiness gate |
| Service B | Critical dependency unavailable and application chooses 503 |
| Sidecar/service mesh | No healthy upstream, policy rejection mapped to 503, or circuit open |
| Rate/concurrency layer | Limit is mapped to 503 instead of 429 |

## Step-by-step debugging

### Step 1 - Identify who generated 503

Use response headers/body, gateway access logs, B access logs, and request IDs.

- Gateway generated it without contacting B: focus on pool health, discovery, policy, and gateway capacity.
- B generated it: inspect B's explicit rejection reason, readiness, overload, and dependencies.

### Step 2 - Inspect backend availability

Check:

- Expected versus healthy target count.
- Health-check status and exact failure reason.
- Endpoint/service-discovery registration.
- Target port, protocol, path, `Host` header, and expected status.
- Readiness of each instance.
- Availability by zone and version.

For Kubernetes:

```bash
kubectl get service -n <namespace> service-b -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o wide
kubectl get pods -n <namespace> -l app=service-b -o wide
kubectl describe pod -n <namespace> <service-b-pod>
```

### Step 3 - Investigate why health checks fail

Do not stop at "unhealthy." Determine:

- Connection refused?
- Timeout?
- TLS failure?
- Wrong path or port?
- Unexpected 301/401/403/404/500?
- Health-check source blocked by firewall/policy?
- Required `Host` header missing?
- Check interval/threshold too aggressive during startup?

### Step 4 - Check deployment and scaling timeline

Look for:

- Desired replicas dropped to zero.
- New instances not ready.
- Rolling-update surge/unavailable settings.
- Readiness regression.
- Autoscaler lag.
- Pod disruption, node drain, or zone failure.
- Discovery deregistration delay.

### Step 5 - Check overload and protective mechanisms

Inspect:

- Gateway active requests, connections, queue, and rejected requests.
- Service B concurrency, queue, thread pool, and pool saturation.
- Rate limits and quotas.
- Circuit-breaker state and reason.
- Service-mesh outlier ejection.
- Load-shedding logic.
- Retry storms.

An overload 503 may be the service protecting itself. Disabling the protection without adding capacity or reducing work can cause a larger failure.

### Step 6 - Check dependencies when B returns 503

Some services intentionally mark themselves unavailable if a critical DB, cache, or downstream service fails. Confirm:

- Which dependency is considered critical.
- Whether readiness should fail for that dependency.
- Whether partial functionality could remain available.
- Whether dependency recovery automatically restores readiness.

### Step 7 - Restore service safely

Immediate actions may include:

- Roll back a bad deployment.
- Restore healthy replica count.
- Correct health check or route.
- Drain a bad target.
- Scale within known dependency capacity.
- Reduce traffic through rate limiting or feature control.
- Repair the critical dependency.

Permanent prevention:

- Deployment availability guards.
- Meaningful readiness and startup probes.
- Capacity and saturation alerts.
- Multi-zone backend distribution.
- Bounded retries and circuit breakers.
- Health-check contract tests.
- Alerts on healthy-target count and endpoint emptiness.

## Interview-ready answer

> I would first identify whether the gateway, service mesh, or B generated the 503. If it is the gateway, I would inspect the expected and healthy target counts, service-discovery or EndpointSlice data, and the exact health-check failure reason, including port, path, protocol, TLS, `Host` header, and policy. I would correlate with deployments, scaling, node events, and circuit-breaker or outlier-ejection state. If B generated it, I would inspect overload/load-shedding, readiness, maintenance mode, and critical dependencies. I would restore healthy capacity or correct routing/health checks, but I would not disable protective 503 behavior without addressing the load.

---

# 13. Question 12 - HTTP 504 From a Gateway

## Interview question

> Service A receives HTTP 504 from the gateway. How would you investigate?

## What 504 usually means

`504 Gateway Timeout` normally means a gateway or proxy did not receive the required upstream response within its configured time.

```text
A -> Gateway -> B -> DB/C
       |
       | waits for upstream
       X timeout expires
A <- 504
```

Service B can still be alive and may even finish the operation after the gateway returns 504. This is important for non-idempotent requests: a client retry could create duplicate work.

## Detailed causes

- Service B request queue is long.
- B's worker/event-loop capacity is exhausted.
- B has high CPU, CPU throttling, memory pressure, or long GC pauses.
- B waits for a DB pool connection.
- Query is slow, blocked, or returns excessive data.
- B waits for Service C, cache, Kafka, storage, or an external provider.
- HTTP client pool in B is exhausted.
- Response serialization, compression, or streaming is slow.
- Network packet loss or stalled upstream connection.
- Gateway queueing consumes part of the timeout.
- Gateway upstream timeout is shorter than valid B latency.
- B retries a downstream operation inside the request.
- Gateway and B timeout budgets are misordered.
- One backend instance is slow.
- Large requests or responses cross a size/processing threshold.

## Step-by-step debugging

### Step 1 - Identify the 504 generator and timeout

Record:

- Gateway/proxy name and route.
- Upstream response timeout.
- Idle timeout.
- Total observed duration.
- Service A's own deadline.
- Request ID/trace ID.
- Backend instance.

If a 504 consistently occurs at exactly 30.0 seconds, a configured 30-second boundary is a strong clue.

### Step 2 - Confirm whether B received and completed the request

Build one of these timelines:

```text
Gateway starts request at 10:00:00.000
B starts request       at 10:00:00.015
Gateway returns 504    at 10:00:05.000
B completes request    at 10:00:08.200
```

This proves B exceeded the gateway budget.

Or:

```text
Gateway starts request
No B request record
Gateway returns 504
```

This points to gateway queueing, upstream connect/TLS delay, wrong target, request transmission, or missing B telemetry.

### Step 3 - Use a trace waterfall

Example:

```text
Gateway total             5.00 s
  Gateway queue           0.20 s
  B request queue         0.90 s
  B application           0.15 s
  DB pool wait            1.10 s
  DB query                2.45 s
  Response                0.20 s
```

The correct fix is likely pool/query/capacity work, not a blanket gateway timeout increase.

### Step 4 - Compare successful and 504 requests

Group by:

- B instance.
- Endpoint.
- Tenant/data key.
- Payload/response size.
- Cache hit/miss.
- DB query or plan.
- Downstream selected.
- Time and traffic.

### Step 5 - Check queueing and saturation

At the exact incident time inspect:

- Gateway queue and connection pool.
- B worker queue/thread pool/event loop.
- DB and HTTP pool pending requests.
- CPU throttling, memory, GC.
- Dependency latency and errors.
- DB locks, slow queries, I/O, connections.
- Network retransmissions.

### Step 6 - Audit the complete timeout budget

Example of a coherent budget:

```text
Service A total deadline:       10.0 s
Gateway upstream timeout:        8.5 s
Service B internal deadline:     7.5 s
B -> C timeout with retries:     3.0 s total
DB statement timeout:            4.0 s
Response/network margin:         1.0 s
```

The actual numbers depend on the service objective, but inner work must stop early enough to return a useful response before outer deadlines expire.

### Step 7 - Check cancellation and duplicate-work risk

When the gateway times out:

- Does it close/cancel the upstream request?
- Does B detect cancellation?
- Does B continue DB or downstream work?
- Is the operation idempotent?
- Will Service A retry?
- Is an idempotency key used for create/payment/order operations?

This is both a reliability and data-correctness concern.

### Step 8 - Fix the measured latency source

Possible fixes:

- Optimize query/code and remove blocking.
- Correct pool/concurrency sizing.
- Add backpressure or load shedding.
- Scale the constrained tier within dependency limits.
- Make long operations asynchronous.
- Reduce payload/result size or stream correctly.
- Bound retries and propagate deadline/cancellation.
- Repair network loss.
- Increase timeout only when valid latency legitimately requires it.

### Step 9 - Verify

Confirm:

- No 504s under representative load.
- p95/p99 fit within the gateway budget with margin.
- Queues and pools do not trend to saturation.
- Timed-out operations stop or remain idempotent.
- All backend instances perform consistently.

## Interview-ready answer

> A 504 generally means a gateway waited longer than its upstream deadline. I would identify which gateway generated it and the exact timeout, then correlate its request ID with B to see whether B received and completed the request. I would create a latency waterfall across gateway queueing, B queueing and code, pool waits, DB, downstream calls, and response transfer, and compare successful and failed requests by backend and data shape. I would audit nested timeout and retry budgets and check whether work continues after the gateway cancels, especially for non-idempotent operations. I would fix the measured bottleneck or make long work asynchronous; I would increase the timeout only if the service objective justifies it.

---

# 14. Question 13 - Only One Service B Instance Is Failing

## Interview question

> Only one instance of Service B is failing while the other instances work. How would you identify the problem?

## What this pattern tells you

If identical traffic succeeds on B1, B2, and B4 but fails on B3, shared components are less likely to be the sole cause. Focus on what is unique to B3:

```text
instance configuration
application version/build
secret/certificate
node/host
zone/subnet
local cache/state/disk
resource pressure
dependency path/DNS
startup lifecycle
```

A shared dependency can still be involved if B3 uses a different connection, shard, credential, DNS answer, or network path.

## Detailed causes

- Different application image, build, or partially completed deployment.
- Configuration, environment variable, feature flag, secret, or certificate drift.
- Failed startup initialization hidden by shallow readiness.
- CPU throttling, memory pressure, GC, thread exhaustion, or pool exhaustion.
- Node disk, network interface, conntrack, DNS, or clock issue.
- B3 is in a different zone/subnet with a broken dependency route.
- B3 resolves a dependency to a different or stale IP.
- Local cache corruption or poisoned entry.
- Stale file, temporary state, or local disk full.
- Certificate expired only on B3.
- Connection pool contains stale/broken connections.
- B3 receives disproportionate traffic because of weight or sticky sessions.
- Clock skew causes token/certificate validation failures.
- Different runtime/JVM flags or resource limits.
- Readiness check is cached or does not exercise the failing capability.

## Step-by-step debugging

### Step 1 - Prove correlation with B3

Use:

- Load-balancer/gateway upstream target.
- Trace attribute.
- Pod name/hostname/instance ID in logs.
- Response header added for internal diagnostics.
- Source and destination IP.

Calculate B3's failure and latency rate versus other instances. Do not rely on one anecdotal request.

### Step 2 - Protect users while preserving evidence

If impact is significant:

1. Drain B3 through the normal load-balancer/orchestrator process.
2. Avoid abruptly killing in-flight non-idempotent work.
3. Preserve logs, events, metrics, config hashes, thread/heap/runtime diagnostics, and node data as appropriate.
4. Keep enough healthy capacity before removing it.

Replacing B3 may restore service but erase the evidence needed to prevent recurrence.

### Step 3 - Compare immutable identity

Compare B3 with a healthy instance:

```text
image digest
application version/commit
deployment revision
runtime version
startup arguments
resource requests/limits
node and zone
creation/start time
```

Use image digest rather than a mutable tag alone.

### Step 4 - Compare effective configuration safely

Compare names and hashes or redacted values for:

- Environment variables.
- Config files/ConfigMaps.
- Feature flags.
- Secret/certificate versions.
- Service URLs.
- Proxy and DNS settings.
- Database/cache endpoints.
- Runtime flags.

Never dump secret values into shared logs.

### Step 5 - Compare runtime and resource state

At the same timestamp compare:

- Request rate and latency.
- CPU and throttling.
- Memory/heap and GC.
- Threads/event loops/queues.
- DB and HTTP connection pools.
- File descriptors and sockets.
- Disk space/I/O.
- Network errors/retransmissions.
- Restarts and termination reason.

### Step 6 - Test dependencies from B3 and a healthy instance

Use equivalent DNS, TCP, TLS, and application-level tests for DB, cache, Kafka, and downstream services. Compare resolved IPs, certificates, route, and identity.

### Step 7 - Inspect node and zone

If other workloads on B3's node or zone also fail, investigate:

- Node networking and DNS.
- CNI or service-mesh agent.
- NAT/conntrack.
- Disk/memory/PID pressure.
- Clock synchronization.
- Zone-local routes or dependencies.

### Step 8 - Recreate only after collecting evidence

If a clean replacement works:

- Compare replacement with failed instance evidence.
- Determine whether the problem was reproducible configuration drift, node state, local corruption, or transient infrastructure.
- Do not close with "pod restart fixed it." Record why replacement changed the failing condition.

### Step 9 - Prevent recurrence

- Enforce immutable images and configuration checksums.
- Add startup/readiness validation.
- Alert on per-instance error/latency outliers.
- Use automated outlier detection carefully.
- Add graceful drain and lifecycle hooks.
- Eliminate mutable local state.
- Spread replicas across nodes/zones.
- Monitor certificate/secret rollout consistency.

## Interview-ready answer

> I would first prove that failures correlate with one instance using gateway logs, trace attributes, pod name, or destination IP. I would drain that instance if necessary while preserving capacity and evidence. Then I would compare it with a healthy instance across image digest, deployment revision, effective configuration, secret/certificate version, node and zone, resource limits, runtime metrics, pools, logs, DNS answers, and dependency connectivity. I would inspect local cache/disk state and node networking as well. Recreating the instance can mitigate impact, but I would still explain why the replacement differs and add per-instance outlier monitoring and drift prevention.

---

# 15. Question 14 - Requests Routed to One Particular Instance Fail

## Interview question

> Requests routed to one particular Service B instance are failing. What could cause this, and how would you debug the routing?

## How this differs from Question 13

Question 13 focuses on comparing a bad instance with healthy instances. This question additionally asks why the routing layer continues to select that bad instance.

There may be two defects:

1. The instance cannot serve the request.
2. The gateway/load balancer/service discovery incorrectly considers it eligible.

## Detailed causes

### Instance-side causes

- Different version/configuration/secret.
- Resource or dependency failure.
- Local corrupted state.
- Listener accepts connections but business endpoint fails.
- Readiness is stale, cached, or too shallow.

### Routing-side causes

- Health check tests only `/health` while business API is broken.
- Health check uses a different port, protocol, host, or network path.
- Failure threshold is too high, so bad target remains eligible too long.
- Target registration is stale.
- Terminating target is not drained before listener/dependency shutdown.
- Readiness changes do not propagate promptly to the load balancer.
- Sticky session repeatedly pins a user to the bad target.
- Weight is incorrect or target receives disproportionate traffic.
- DNS round-robin still advertises a removed target.
- Cross-zone routing sends traffic through a broken path.
- Service-mesh outlier detection is disabled or misconfigured.
- Gateway route selects the wrong subset/version.
- Direct pod registration uses stale pod IP.

## Step-by-step debugging

### Step 1 - Prove target selection

For failed and successful requests capture:

```text
gateway route
upstream cluster/pool
upstream IP and port
target instance/pod
routing weight
session/stickiness key
zone
health state at request time
```

### Step 2 - Inspect the routing layer's target list

Compare:

- Expected instances.
- Registered/discovered instances.
- Healthy/eligible instances.
- Draining instances.
- Removed but cached instances.
- Weight and zone.

In Kubernetes:

```bash
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o yaml
kubectl get pod -n <namespace> <bad-pod> -o yaml
```

Check readiness conditions and endpoint `ready`, `serving`, and `terminating` state where supported.

### Step 3 - Reproduce directly and through the router

When approved, test:

1. Normal gateway/service URL.
2. The specific target IP and port while preserving required HTTP `Host` and TLS SNI.
3. A known healthy target with the same request.

Direct target tests can bypass authentication, mesh, or policy. Use them only as controlled diagnostics, not as a production workaround.

### Step 4 - Validate the health-check contract

Verify:

- Path exercises traffic acceptance, not only process existence.
- Port and protocol match real routing.
- Required `Host` header and TLS SNI are correct.
- Check source is permitted by policy.
- Success status range is correct.
- Interval, timeout, healthy threshold, and unhealthy threshold fit startup and failure behavior.
- Check cannot remain successful from a stale cache.

### Step 5 - Inspect lifecycle timing

Build a deployment timeline:

```text
readiness false
endpoint marked terminating
load balancer starts drain
application rejects new work
listener closes
process exits
```

The ordering should prevent new traffic after the application can no longer serve it.

### Step 6 - Check stickiness and weighting

If only certain users fail:

- Inspect cookie/header/source-IP affinity.
- Confirm sticky-session TTL.
- Determine whether one shard/tenant maps to the target.
- Check canary/version weights.

### Step 7 - Mitigate and correct

- Drain/remove the bad target.
- Correct readiness or health check.
- Fix deregistration and graceful shutdown timing.
- Remove stale DNS/discovery registration.
- Correct weights, subsets, and stickiness.
- Fix the instance root cause from Question 13.
- Add per-target error-rate and latency outlier alerts.

## Interview-ready answer

> I would prove which target handled each request and inspect the load balancer or service-discovery target list, health state, weights, stickiness, zone, and lifecycle state at that time. I would compare a normal routed request with controlled direct tests to the bad and healthy targets while preserving `Host` and SNI. Then I would validate that health checks use the real port, protocol, route, and traffic capability, and inspect readiness propagation, stale registration, and graceful draining during rollout. I would remove the bad target to protect users, fix both the instance issue and the reason it remained eligible, and monitor errors per target.

---

# 16. Question 15 - Health Endpoint Is Healthy, but Actual APIs Fail

## Interview question

> Service B is healthy according to its health endpoint, but actual API requests are failing. Why, and how would you investigate?

## Health does not equal business availability

Health endpoints answer only the questions they were designed to answer.

```text
GET /health -> 200
```

may prove only:

```text
the process can accept one lightweight unauthenticated request
```

It may not prove:

```text
authentication works
the orders route exists
the database query works
the correct tenant data is available
the thread and connection pools have capacity
Kafka/cache/Service C is usable
large responses can be generated
the service meets its latency objective
```

## Detailed causes

| Cause | Why health remains green |
|---|---|
| Liveness-only endpoint | It checks only that the process/event loop responds |
| Health bypasses authentication | Business route fails token validation or authorization |
| Health uses a trivial DB query | Real query hits missing table/index, lock, permissions, or bad data |
| Health result is cached | Endpoint reports old state |
| Specific endpoint code bug | Health route does not execute that code |
| Data-specific failure | Only a tenant/order/payload triggers the defect |
| Dependency omitted from readiness | DB/cache/Kafka/C is unavailable but health does not check it |
| Pool/thread saturation | Lightweight health request uses reserved path or succeeds while real work queues |
| Different port/path | Platform health checks management port, traffic uses application port |
| Feature flag/version mismatch | Health is common but business path differs |
| Certificate/auth clock issue | Public health is unauthenticated; secured APIs reject requests |
| Payload/response limit | Small health response avoids gateway and serialization limits |
| Partial outage is intentional | Service remains ready for unaffected endpoints |
| Health-check identity has extra access | Probe succeeds while normal workload identity fails |

## Step-by-step debugging

### Step 1 - Define the health endpoint's contract

Read the implementation and deployment configuration. Determine:

- Is it liveness, readiness, startup, or a generic health endpoint?
- Which dependencies does it check?
- Does it use cached results?
- Which port and network path invokes it?
- Does it require authentication?
- What timeout and success status does the platform use?

### Step 2 - Reproduce the real failing business request

Capture:

- Method and full path.
- Safe representation of headers and auth identity.
- Payload/data category.
- Exact status/exception.
- Request ID.
- Source and destination instance.

Do not replace the business test with `/health`.

### Step 3 - Compare request paths

```text
Health:
gateway -> management port -> health handler -> 200

Business:
gateway -> application port -> auth -> route -> DB -> C -> serialization
```

Find the first component present only in the failing path.

### Step 4 - Interpret the real response

- 401/403: identity, token claims, roles, clock, or authorization policy.
- 404/405: route, path prefix, method, API version, or deployment mismatch.
- 400/415: validation, content type, or schema.
- 429/503: throttling, overload, readiness, or load shedding.
- 500: code, data, configuration, or dependency exception.
- Timeout/504: queueing, resource saturation, DB, downstream, or timeout budget.

### Step 5 - Trace business dependencies

Use traces and logs to inspect:

- Auth provider/key discovery.
- Database query and pool.
- Cache.
- Kafka/queue.
- Service C or external provider.
- Feature-flag service.
- Object storage.
- Serialization and response size.

### Step 6 - Check capacity and isolation

A health endpoint can stay fast because it:

- Does almost no work.
- Uses a separate management port/thread pool.
- Avoids the saturated DB/HTTP pool.
- Has higher priority.

Inspect business request queues, pools, concurrency limits, CPU throttling, GC, and dependency capacity.

### Step 7 - Check data and endpoint specificity

Compare:

- One endpoint versus all endpoints.
- One tenant/record versus all data.
- One write versus reads.
- Large versus small requests.
- Cache hit versus miss.
- One version/feature flag versus another.

### Step 8 - Improve health design without creating a restart loop

Use distinct probes:

- **Startup:** Has initialization completed?
- **Liveness:** Is the process making progress, or must it be restarted?
- **Readiness:** Should this instance receive new traffic now?
- **Synthetic business check:** Can a safe representative transaction complete end to end?

Do not make liveness depend on every remote service. A temporary database outage could cause all application instances to restart repeatedly, adding load without repairing the database. Critical dependency checks generally belong in readiness or external synthetic monitoring, with deliberate failure semantics.

### Step 9 - Fix and verify

- Correct the endpoint-specific code, auth, data, dependency, route, or capacity issue.
- Improve readiness if the instance should not receive traffic in that condition.
- Add safe synthetic tests for critical business flows.
- Alert on business success rate and latency, not only health status.
- Verify a representative business transaction through the normal gateway and identity.

## Interview-ready answer

> A green health endpoint proves only what that endpoint checks. I would identify whether it is startup, liveness, or readiness and inspect its exact dependencies, port, path, cache, and identity. Then I would reproduce the real business request and classify its status or timeout, compare the health and business call paths, and trace the components used only by the business route: authentication, route mapping, data-specific logic, pools, DB, cache, Kafka, downstream services, and serialization. I would fix the real failure, improve readiness or synthetic monitoring where appropriate, and keep liveness shallow enough to avoid dependency-driven restart loops.

---

# 17. Liveness, Readiness, Startup, and Synthetic Checks

## Liveness

Question:

> Is the process alive and making progress, or should the platform restart it?

Liveness should generally be local and stable. It should not fail merely because a remote database has a temporary outage.

## Readiness

Question:

> Should this instance receive new traffic now?

Readiness can consider critical initialization, listener state, local saturation, required configuration, and carefully selected dependencies. If readiness fails, routing should remove the instance without necessarily restarting it.

## Startup

Question:

> Has this slow-starting application completed initialization?

A startup probe prevents liveness from killing an application that legitimately needs more initialization time.

## Synthetic business check

Question:

> Can a safe, representative user transaction complete through the real path?

Synthetic checks are external monitoring, not necessarily pod probes. They can validate DNS, gateway, authentication, service code, and dependencies end to end.

| Check | Failure action | Typical scope |
|---|---|---|
| Startup | Continue waiting or restart after startup budget | Initialization |
| Liveness | Restart instance | Process progress |
| Readiness | Stop routing new traffic | Traffic capability |
| Synthetic | Alert/investigate; possibly automate controlled response | Business availability |

---

# 18. Fast Comparison: Refused, Connect Timeout, Read Timeout, 502, 503, 504

| Symptom | Connection established? | Typical owner | First checks |
|---|---:|---|---|
| DNS failure | No | DNS/discovery | Exact name, resolver, answer, TTL |
| Connection refused | No | Destination listener/rejecting device | Host, port, listener, bind, target |
| Connection timeout | No | Network path or capacity | Route, firewall, policies, NAT, packet direction |
| TLS failure | TCP yes | TLS endpoints | Trust, SAN, SNI, mTLS, protocol |
| Read timeout | Yes | B/dependency/response path | Trace, queues, pools, DB, downstream, timeout |
| 502 | Varies between gateway and B | Gateway-upstream communication | Gateway subreason, DNS/TCP/TLS/protocol/reset |
| 503 | HTTP responder reached | Gateway or service availability | Generator, healthy targets, readiness, overload |
| 504 | Gateway connected or attempted upstream work but deadline expired | Gateway/backend latency path | Timeout owner, trace waterfall, B/dependencies |

---

# 19. Detailed Command and Interpretation Guide

## DNS

### Windows

```powershell
Resolve-DnsName service-b
Resolve-DnsName service-b -Type A
Resolve-DnsName service-b -Type AAAA
Get-DnsClientServerAddress
```

### Linux

```bash
getent ahosts service-b
dig service-b A
dig service-b AAAA
cat /etc/resolv.conf
```

Use `getent` to test the normal host-resolution path and `dig` to inspect DNS details. Record the resolver and answer instead of reporting only "DNS works."

## TCP

### Windows

```powershell
Test-NetConnection service-b -Port 8080 -InformationLevel Detailed
```

### Linux

```bash
nc -vz -w 5 service-b 8080
```

Interpret:

- Success: this test completed TCP.
- Immediate refusal: inspect listener/target/reject.
- Timeout: inspect packet path/drop/capacity.

## HTTP timing

```bash
curl -sS -o /dev/null \
  --connect-timeout 5 \
  --max-time 15 \
  -w 'remote_ip=%{remote_ip} code=%{http_code} dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} first_byte=%{time_starttransfer} total=%{time_total}\n' \
  https://service-b/health
```

Important fields:

- `time_namelookup`: DNS stage.
- `time_connect`: time through TCP connect.
- `time_appconnect`: TLS completion time.
- `time_starttransfer`: time to first response byte.
- `time_total`: complete transfer time.

On Windows, use `curl.exe` to avoid confusion with any PowerShell alias in older environments.

## Listener

### Windows

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen
netstat -ano | findstr :8080
```

### Linux

```bash
ss -lntp | grep ':8080'
```

## TLS

```bash
openssl s_client -connect service-b:443 -servername service-b -showcerts </dev/null
```

Check certificate chain, validity, SAN, SNI-selected certificate, verification result, and mTLS request.

## Route

### Windows

```powershell
Get-NetRoute
tracert service-b
```

### Linux

```bash
ip route get <destination-ip>
traceroute service-b
```

Traceroute can be incomplete because intermediate devices often drop its probes. It does not replace a test to the real TCP port.

## Kubernetes service path

```bash
kubectl get service -n <namespace> service-b -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=service-b -o yaml
kubectl get pods -n <namespace> -l app=service-b -o wide
kubectl describe pod -n <namespace> <pod>
kubectl logs -n <namespace> <pod> --since=30m
kubectl logs -n <namespace> <pod> -c <sidecar> --since=30m
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Check Service selector, ports, ready endpoints, pod readiness/restarts, application logs, sidecar logs, and events.

## Packet capture

Use only when authorized and when higher-level evidence is insufficient:

```bash
tcpdump -nn -i any host <destination-ip> and port <port>
```

Capture only the minimum duration and filter needed. Protect and delete captures according to operational policy because they may contain sensitive data.

---

# 20. How to Use Logs, Metrics, and Traces Together

## Logs answer "what happened?"

Useful fields:

```text
timestamp in UTC
request/trace ID
service and instance
source/destination
method and normalized route
status or exception class
latency
retry attempt
dependency name
```

Avoid logging tokens, secrets, and sensitive payloads.

## Metrics answer "how much, how often, and where?"

Use the RED method for services:

- **Rate:** request throughput.
- **Errors:** error/timeout/rejection rate.
- **Duration:** latency percentiles.

Use saturation metrics for resources:

- Queue depth.
- Active/max threads.
- Pool active/pending/max.
- CPU throttling.
- Memory/GC.
- Connection/NAT/conntrack utilization.
- DB locks/I/O/connections.

Do not rely only on averages or aggregate metrics. Break down by instance, route, zone, and dependency with controlled label cardinality.

## Traces answer "where was time or failure spent?"

An effective trace shows:

```text
gateway span
  -> Service A client span
     -> Service B server span
        -> DB span
        -> Service C client span
```

Add target instance, route, retry count, and meaningful error attributes without recording sensitive data.

## Without distributed tracing

Reconstruct the path using request ID and timestamps:

```text
Gateway accepted request
A called B
B accepted request
B called DB/C
B completed or failed
A received response or timed out
Gateway returned response
```

Synchronize system clocks. Prefer duration measurements based on monotonic clocks within a process because wall-clock skew can make cross-system timelines misleading.

---

# 21. Production Debugging Workflow

Use this order during an incident:

1. **Protect users and data.** Drain a bad instance, stop unsafe retries, or reduce load through approved controls when necessary.
2. **Record evidence.** Exact error, time, request ID, source, destination, route, change timeline.
3. **Classify the failing stage.** Configuration, DNS, TCP, TLS, HTTP, gateway, application, dependency, response.
4. **Reproduce from the correct boundary.** Same environment and path as Service A.
5. **Narrow by scope.** All versus some sources, targets, endpoints, tenants, zones, or times.
6. **Form one testable hypothesis.** Example: "B3's listener closes before load-balancer draining completes."
7. **Collect evidence that can disprove it.** Compare healthy and failing instances and packet/request timelines.
8. **Apply the smallest safe correction.**
9. **Verify the original failure path and business transaction.**
10. **Monitor for recurrence and secondary effects.**
11. **Document root cause, contributing factors, detection gap, and prevention.**

## Root cause versus contributing factors

Example:

```text
Root cause:
  New release used the wrong Service targetPort.

Contributing factors:
  Readiness checked a separate management port.
  Deployment allowed all old pods to terminate before new pods were usable.
  Alerting watched pod Running state instead of healthy backend count.

Mitigation:
  Roll back release.

Permanent corrections:
  Fix targetPort.
  Test Service-to-pod wiring in deployment validation.
  Make readiness represent traffic port.
  Add healthy-target-count alert.
```

"Restarted the pod" is a mitigation unless the reason a restart repaired the state is understood.

---

# 22. Common Interview Mistakes

## Mistake 1 - "I will check the logs"

Better:

> I will inspect Service A's complete root exception and use it to decide whether to test DNS, TCP, TLS, HTTP, or post-connection processing.

## Mistake 2 - "Connection refused means B is down"

Better:

> It commonly means the selected destination actively rejected the port. I will check host, port, listener, bind address, target registration, container/service mapping, and explicit reject rules.

## Mistake 3 - "Ping works, so the network is fine"

Better:

> Ping tests ICMP. I will test the actual TCP port and then TLS and HTTP from Service A.

## Mistake 4 - "Increase the timeout"

Better:

> I will identify where the time is spent and correct queueing, pool, query, dependency, resource, or retry behavior. I will change the timeout only if the valid service objective requires it.

## Mistake 5 - "Health is green, so B is healthy"

Better:

> I will identify exactly what the health endpoint checks and test the real business path and dependencies.

## Mistake 6 - Testing only from a laptop

Better:

> I will reproduce from Service A's runtime because DNS, routes, proxy, policies, trust, and identity differ.

## Mistake 7 - Restarting before collecting evidence

Better:

> I will mitigate impact safely, preserve the failing instance's evidence, and determine why replacement or restart changes the condition.

## Mistake 8 - Listing every possible cause without narrowing

Better:

> I will use the exact error, scope, hop, and success/failure comparison to select the next test and eliminate fault domains.

---

# 23. Reusable Interview Answer Template

For most service-to-service incidents:

> First, I would capture the exact exception or HTTP response, timestamp, request ID, affected scope, effective URL, and the component that generated the error. I would map the call path and classify whether failure occurs in configuration, DNS, TCP, TLS, gateway routing, HTTP handling, application processing, or a downstream dependency. I would reproduce from Service A's actual runtime environment and test each layer independently. I would correlate logs, metrics, and traces by source and destination instance, compare successful and failed requests, and check recent deployments or configuration/network changes. I would use that evidence to prove one root cause, apply the smallest safe fix, retest the original business path across all instances, monitor the relevant latency/error/saturation metrics, and add a preventive control.

Customize the middle of this answer to the exact symptom:

```text
Refused       -> listener, host, port, bind, target
Connect time  -> routes, drops, policies, NAT, handshake
Read time     -> queue, code, pools, DB, downstream, response
502           -> gateway-to-upstream DNS/TCP/TLS/protocol/response
503           -> generator, healthy capacity, readiness, overload
504           -> timeout owner, latency waterfall, deadline budget
Intermittent  -> compare dimensions and instances
Health green  -> compare health contract with business path
```

The strongest interview answers do not merely list tools. They explain:

1. What the symptom proves.
2. What it does not prove.
3. Where the issue can occur.
4. Which test comes next and why.
5. How the result changes the next step.
6. How the root cause is fixed and recurrence is prevented.
