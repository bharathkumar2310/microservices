# Problem

Service A gets `Connection refused` while opening a TCP connection to Service B. DNS may be correct and the host reachable, but no process accepts the selected address/port, or a host/sidecar actively rejects it. The failure occurs before TLS and HTTP.

```text
DNS succeeds -> IP selected -> TCP SYN -> RST/refusal
                                  X
TLS and HTTP never start
```

Unlike a connect timeout, refusal is usually immediate. Unlike a read timeout, no established request waits for a response. It is not a 502/503/504 unless an intermediary converts the transport failure into HTTP.

# Production Situation

At `2026-09-13T10:02:22Z`, Order Service A calls `POST /v1/inventory/reservations`.

* requestId `ord-a61d09`, traceId `02f92f3577b34da6a3ce929d0e0e0002`
* source `order-a-4.18.2-v7n4j`, zone `eu-west-1a`
* three B targets: `.21`, `.22`, `.23`, version `7.5`
* failed target `10.42.7.23:8080`, instance `inventory-b-3`
* normal: 225 requests/s, p99 170 ms, error 0.05%
* abnormal: 74 refusals/s in 1-3 ms, error 32.9%
* B-1/B-2 each accept about 75/s; B-3 accepts 0/s
* B-3 is Running and locally healthy

The roughly one-third rate suggests one of three targets, but target labels prove it.

# Architecture

```text
Order A
   |
   v
Kubernetes Service / Envoy
   +------> B-1 10.42.7.21:8080 LISTEN 0.0.0.0
   +------> B-2 10.42.7.22:8080 LISTEN 0.0.0.0
   `------> B-3 10.42.7.23:8080 X only 127.0.0.1
```

Envoy normally returns HTTP 503 with `UF` for an upstream refusal, while a direct Feign/WebClient call exposes `ConnectException`. NGINX can generate HTTP 502 with `connect() failed (111: Connection refused) while connecting to upstream`. These are product-specific presentations of the same B-facing TCP event.

# What I Check FIRST

1. **Exact exception and elapsed time.** WHAT: nested `ConnectException`, target, and connect duration. WHY: 2 ms refusal differs from a 5 s timeout. LOOK FOR: `ECONNREFUSED 10.42.7.23:8080`.
2. **Target distribution.** WHAT: failures by backend/instance. WHY: a fleet average hides one bad target. LOOK FOR: B-3 predicts all failures.
3. **B acceptance.** WHAT: A attempts, gateway upstream attempts, B server requests. WHY: refusal happens before the handler. LOOK FOR: B-3 server rate zero.
4. **Listener and bind.** WHAT: process, namespace, bind address, and service targetPort. WHY: Running does not mean externally listening. LOOK FOR: `127.0.0.1:8080`.
5. **Change state.** WHAT: pod config hash and rollout events. WHY: restart may erase drift evidence. LOOK FOR: stale ConfigMap on B-3.

# Step-by-Step Investigation

### Step 1 - Confirm refusal rather than a generic connect error

* **What I check:** deepest exception, phase timer, selected IP/port, and retry attempt.
* **Why:** wrappers such as Feign `RetryableException` can hide the socket cause.
* **Example:** inspect request `ord-a61d09`.
* **Expected result:** healthy target connects in 8 ms.
* **Bad result:** B-3 returns `Connection refused` in 2 ms.
* **Meaning:** an RST or local active rejection came back; packets are not silently dropped.
* **Next branch:** split by target and inspect its listener.

### Step 2 - Verify DNS/discovery correctness

* **What I check:** all answers, EndpointSlice addresses, readiness, TTL, and selected backend.
* **Why:** a retired or wrong IP can refuse even while current B pods are healthy.
* **Example:** compare `10.42.7.23` with the EndpointSlice.
* **Expected result:** three current ready addresses including `.23`.
* **Bad result:** `.23` absent or terminating.
* **Meaning:** stale discovery/target lifecycle, not necessarily an application bind.
* **Next branch:** reconcile discovery if stale; here `.23` is current, so inspect the socket.

### Step 3 - Compare exact TCP behavior

* **What I check:** bounded port probe to each target from A's context.
* **Why:** a Service-level probe can randomly select a healthy peer.
* **Expected result:** all targets report connected under 15 ms.
* **Bad result:** `.23` refuses immediately while `.21/.22` connect.
* **Meaning:** the fault is destination-specific, after routing but before TLS.
* **Next branch:** inspect B-3 process and namespace.

### Step 4 - Inspect listener ownership and binding

* **What I check:** `ss`/`netstat`, PID, bind address, port, container/sidecar namespace.
* **Why:** listening on loopback serves local probes but rejects pod-IP traffic.
* **Expected result:** Java owns `0.0.0.0:8080` or `[::]:8080`.
* **Bad result:** Java owns only `127.0.0.1:8080`.
* **Meaning:** B-3 cannot accept non-loopback traffic.
* **Next branch:** compare effective `server.address` against a good peer.

### Step 5 - Validate Service and gateway configuration

* **What I check:** Service `port`/`targetPort`, mesh cluster port, target health reason, and sidecar listener.
* **Why:** a correct application listener on 8080 still refuses if routed to 8081.
* **Expected result:** every layer targets 8080.
* **Bad result:** gateway reports upstream connect failure on 8081.
* **Meaning:** routing contract mismatch rather than B bind.
* **Next branch:** fix the contract; in this incident ports match, so continue to config drift.

### Step 6 - Compare runtime configuration

* **What I check:** B-3 environment, ConfigMap resource version, image, startup arguments, config hash, node, and zone versus B-2.
* **Why:** the same image can load different runtime state.
* **Expected result:** `SERVER_ADDRESS=0.0.0.0` everywhere.
* **Bad result:** B-3 has `127.0.0.1` from `legacy-71c`.
* **Meaning:** stale node-mounted configuration caused the listener scope.
* **Next branch:** preserve state, drain B-3, and correct the source.

### Step 7 - Check later layers only as controls

* **What I check:** TLS samples, B HTTP rate, CPU, GC, queue, DB pool, and DB rate.
* **Why:** a refusal prevents all later work.
* **Expected result:** no failed-call samples and normal resources.
* **Bad result:** matching B access log means the connection did not actually refuse at this B hop.
* **Meaning:** telemetry boundaries or an intermediary need re-evaluation.
* **Next branch:** identify the actual generator and target.

### Refusal result branches

| Result | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| All B targets refuse | Shared port/listener/config is likely | Every B process crashed | Compare Service `targetPort`, rollout config, and listeners on one target |
| Only B-3 refuses | Target-specific state predicts failure | The B-3 JVM is stopped | Compare process uptime, bind, sidecar, and runtime config with B-2 |
| Pod IP refuses; loopback works | Listener exists only locally or routing reaches another namespace | Spring handler is healthy for remote traffic | Run `ss` in each container namespace and inspect `server.address` |
| Both loopback and pod IP refuse | Nothing accepts that port in the tested namespace | The process is absent | Check process/startup logs, actual configured port, and sidecar ownership |
| Service IP refuses; every pod IP works | Service/virtual-IP/port mapping is wrong | B listeners are faulty | Inspect Service port/targetPort, EndpointSlice ports, and kube-proxy/CNI path |
| Envoy returns 503 `UF`; direct refuses | Envoy translated upstream connection failure | Envoy caused the listener defect | Read `upstream_transport_failure_reason`, then inspect that target's socket |
| NGINX returns 502 with `connect() failed (111)` | NGINX could not open the upstream socket | NGINX caused B's closed port | Confirm upstream address, then inspect that target's listener |
| Refusal follows new B version | Rollout state is implicated | New code logic is the cause | Compare effective bind/port/startup flags and rendered manifests |
| Refusal follows one node | Node-local path or injected config is implicated | Every pod on the node shares the same cause | Move only a controlled canary or compare CNI, host firewall, and mounts |
| Refusal appears during drain | Target may still be selected after listener closure | Drain controller definitely failed | Compare readiness, endpoint withdrawal, LB weight, and termination times |
| Reused socket resets, new sockets work | This is a stale-connection problem, not initial refusal | The target is healthy for all work | Inspect connection age, drain state, and keep-alive compatibility |

### Listener ownership checks

`ss -lntp` showing `0.0.0.0:8080` proves an IPv4 listener in that network
namespace. It does not prove Kubernetes selects the pod or that HTTP succeeds.

`[::]:8080` may accept IPv4 too, depending on `bindv6only`; I verify rather than
assuming dual-stack behavior.

`127.0.0.1:8080` proves only loopback reachability. A local readiness check can
therefore remain green while every pod-IP connection is refused.

No `ss` line may mean the application is still starting, crashed, bound another
port, runs in another container namespace, or the diagnostic lacks permission.

An Envoy sidecar owning 8080 changes the next check: I inspect its inbound
listener and cluster, then the application port behind it.

A successful TCP probe proves one handshake. It does not prove HTTP route,
authentication, business dependencies, or sustained availability.

### Spring Boot evidence

I compare the effective `server.address`, `server.port`, management port,
active profile, command-line arguments, and environment property precedence.

Spring Boot startup log `Tomcat started on port 8080` identifies a port but may
not make the bind address obvious; kernel listener output is the runtime truth.

Actuator liveness on loopback is a process signal, not remote reachability.

A startup assertion can fail fast when production config requests loopback.

A remote readiness gate should be bounded and should not turn an unrelated
dependency outage into a restart loop.

### Recovery safety branches

Before draining B-3, I verify B-1/B-2 active requests, CPU, worker queue, HTTP
pool, DB pool, and zone redundancy can absorb an extra 50% each.

If peers have no headroom, I first stop retries or shed noncritical work and add
a known-good instance from the immutable configuration.

If recreating B-3 loads the same stale ConfigMap, replacement will fail again;
the configuration source must be corrected before scale-out.

After B-3 returns, I send a controlled business request specifically through
that target and confirm its request ID in B-3 logs and trace.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| `tcp_connect_errors{reason="refused"}` | High and fast means active rejection; low/zero moves elsewhere; split target/source |
| Connect duration | Low 1-3 ms with error supports refusal; high at exact budget supports timeout |
| A attempts vs B accepts | A high/B-3 flat localizes before B HTTP; equal counts contradict refusal at B |
| Per-target error rate | One target at 100% explains fleet 33%; aggregate hides it |
| B server rate | Low on B-3 is expected; sudden fall after deploy suggests listener/readiness/selection |
| Listener/target health | Unhealthy reason `connection_refused` selects port/listener, not business logic |
| New/reused connections | New failures suggest listener; reused resets suggest draining/stale pool |
| CPU/memory/GC | Low/flat B-3 is expected because requests never execute; high is not needed for refusal |
| Retry count | High can shift load to peers and mask first-attempt defect; compare original requests |
| p50/p95/p99/max | Fast failures may lower latency percentile; always separate outcome and counts |

# Distributed Trace Investigation

```text
traceId=02f92f3577b34da6a3ce929d0e0e0002
Order A server                         21ms span=b001
  Inventory client                     3ms span=b002 ERROR
    tcp.connect target=10.42.7.23       2ms span=b003 refused
  [no TLS]
  [no B server span]
```

The parent shows the user operation; the client child identifies target B-3; the TCP child ends quickly. A gateway path may instead show Envoy `503 UF -> upstream refused` or NGINX `502 -> connect() failed (111)`, depending on the product.

Missing TLS/HTTP/B spans are expected, but missing spans can also result from head sampling, exporter loss, broken propagation, or absent instrumentation. Confirm sampling and B access logs. Compare a successful trace to B-2 with the same route, version, identity, and minute.

# Distributed Logs

```text
2026-09-13T10:02:22.417Z level=ERROR service=order-service
instance=order-a-4.18.2-v7n4j version=4.18.2 zone=eu-west-1a
traceId=02f92f3577b34da6a3ce929d0e0e0002 spanId=b002 requestId=ord-a61d09
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
target=10.42.7.23:8080 backend=inventory-b-3 latency_ms=2 attempt=1
error="java.net.ConnectException: Connection refused"
```

Correlate with gateway upstream logs and B-3 startup/config logs. The A log proves A observed a refusal to that labeled target. It does not prove the Java process was down; wrong port, loopback bind, stale IP, sidecar reject, or host policy can all emit refusal. Listener and runtime evidence select the cause.

# Commands / Tools

```powershell
Resolve-DnsName inventory-b.shop.svc.cluster.local -DnsOnly
Test-NetConnection 10.42.7.23 -Port 8080 -InformationLevel Detailed
curl.exe -v --connect-timeout 2 --max-time 5 http://10.42.7.23:8080/actuator/health
Get-NetTCPConnection -State Listen -LocalPort 8080
```

`TcpTestSucceeded=False` with quick completion supports refusal but does not name the process. `Get-NetTCPConnection` is meaningful only on the destination host/namespace.

```bash
getent hosts inventory-b
nc -vz -w 2 10.42.7.23 8080
ss -lntp | grep ':8080'
curl -sS -v --connect-timeout 2 --max-time 5 http://127.0.0.1:8080/actuator/health
curl -sS -v --connect-timeout 2 --max-time 5 http://10.42.7.23:8080/actuator/health
```

Success `127.0.0.1` plus refusal on pod IP demonstrates bind scope. `ping` is not useful proof of TCP listener or HTTP.

```bash
kubectl get pods -n shop -l app=inventory -o wide
kubectl get endpointslice -n shop -l kubernetes.io/service-name=inventory-b -o wide
kubectl describe pod -n shop inventory-b-3
kubectl logs -n shop inventory-b-3 --since=15m
```

These read state; logs may contain secrets, so use bounded, authorized queries. `Running` is not `Ready`, listening, selected, or business-capable.

# Root Cause

B-3 loaded `SERVER_ADDRESS=127.0.0.1` from stale ConfigMap revision `legacy-71c`.

```text
stale configuration
-> Spring Boot binds Tomcat to loopback only
-> local health succeeds
-> pod IP has no accepting listener
-> kernel rejects TCP in 1-3 ms
-> B-3 receives no HTTP
-> one of three calls fails
```

The target split, socket output, config comparison, and recovery after replacement establish causality.

# Fix

**Immediate mitigation:** preserve config/socket evidence, verify B-1/B-2 capacity, then drain B-3 from the gateway/Service. Avoid blind restarts; they can erase drift evidence.

**Root cause correction:** remove the stale override and bind the application to the intended non-loopback interface. Recreate B-3 through the managed deployment.

**Permanent fix:** use one immutable config source, publish config hashes, validate bind address at startup, and gate rollout on a remote caller-context readiness probe. Do not enlarge pools or timeouts; neither creates a listener.

# Verification

Before: refusals 74/s, fleet error 32.9%, B-3 accepted 0/s, B-3 external TCP false.

After: refusals 0/s, business success 99.96%, each target accepts 74-76/s, connect p99 11 ms, B-3 route p99 168 ms, attempts/request 1.00 for 30 minutes. A real reservation through B-3 commits once.

# Prevention

* Alert on per-target refusal rate and A-attempt/B-accept mismatch.
* Block deployments when EndpointSlice points at a target without a remote listener.
* Test Service port, targetPort, sidecar cluster port, and Spring `server.address`.
* Runbook includes target split, socket namespace, loopback-versus-pod-IP test, evidence preservation, and drain.
* Canary one target and exercise a business route from an A-equivalent identity.
* Keep retries bounded and only for safe idempotent operations with remaining deadline.

# Interview Answer

### What I would say in an interview

I distinguish refusal from timeout using the exact exception and duration. Here one-third of calls failed in 1-3 ms, and every failure selected B-3. DNS and discovery were correct, but B-3 accepted no HTTP traffic. Comparing sockets showed B-3 listening only on `127.0.0.1:8080`, while peers used `0.0.0.0:8080`; a stale ConfigMap supplied the bind. I preserved evidence and drained B-3, corrected configuration, and verified zero refusals, balanced per-target traffic, and real reservation success.

### Common interviewer traps

Running is not listening. Local health can pass on loopback. A fast refusal is not a firewall-drop timeout. Do not add replicas until the selector/config contract is known, and do not cite one log as proof.

### Quick memory flow

Exact refusal -> target split -> discovery -> exact port -> listener/bind -> config/port map -> drain -> correct -> balanced verification.

# Interview Follow-up Questions

1. **Can a firewall cause refusal?** Yes, a reject rule may return RST; a drop usually times out.
2. **Why does local health pass?** It uses loopback, the only bound address.
3. **Could port exhaustion cause this?** Client ephemeral-port exhaustion usually has different local errors; inspect socket counters and scope.
4. **What if a gateway translates it?** Envoy normally uses 503 with `UF`; NGINX can use 502 with `connect() failed (111)`. Read the generator's exact reason.
5. **Why no B trace?** TCP failed before B HTTP instrumentation; still check sampling and access logs.
6. **What if only reused connections fail?** Investigate resets and draining, not a missing new listener first.
7. **Would a restart fix it?** It may reload correct config temporarily but is mitigation, not the permanent correction.
8. **How does Spring Boot bind?** `server.address` controls the interface; `127.0.0.1` is loopback only.
