# Problem

A connection timeout means Service A could not complete the TCP connection within its configured connect budget. DNS may already have produced an IP, but A did not receive the required handshake response.

```text
DNS -> IP -> SYN -----------------> expected SYN-ACK
             retransmit, retry, then timeout
TLS and HTTP have not started
```

It differs from fast refusal (an active rejection), TLS failure (TCP succeeded), read timeout (request sent), and gateway 504 (an HTTP intermediary timed out waiting upstream).

# Production Situation

At `2026-09-13T11:27:22Z`, new Order A version `4.19.0` calls Inventory B.

* route `POST /v1/inventory/reservations`
* requestId `ord-18b44e`
* traceId `03f92f3577b34da6a3ce929d0e0e0003`
* source `order-a-4.19.0-r4w9p`, zone `eu-west-1c`
* target `10.42.7.18:8080`, B `inventory-b-7.5-r8x2p`
* normal old A connect p99 9 ms, route p99 175 ms, success 99.95%
* abnormal new A: 61 connect timeouts/s exactly at 5.000 s, success 0%
* B continues accepting 154/s from old A, p99 171 ms
* SYN retransmits rise only for new A pods
* CPU/heap/GC are normal on both services

# Architecture

```text
Old A labels app=order-api -------------------------+
                                                     | allowed
New A labels app.kubernetes.io/name=order-api --X   | NetworkPolicy
                                                     v
                                            Inventory B :8080
```

The direct pod-to-B path is blocked before TCP completes. If A first reached a gateway, separate the two connects. Envoy normally maps `UF` upstream connection failure to 503; NGINX may generate 502 with an upstream connect error; Envoy `UT` maps to 504 after its upstream request deadline.

# What I Check FIRST

1. **Timeout phase and exact plateau.** WHAT: connect versus pool/read/overall duration. WHY: all can surface as "timeout." LOOK FOR: connect span ending at 5.000 s.
2. **Source/target scope.** WHAT: version, pod labels, node, zone, IP, target. WHY: old A succeeds against the same B. LOOK FOR: failures only from 4.19.0.
3. **Handshake evidence.** WHAT: successful connects, RSTs, retransmits, flow logs. WHY: no RST distinguishes drop from refusal. LOOK FOR: SYN retransmissions and no SYN-ACK.
4. **Boundary counts.** WHAT: A attempts and B accepts. WHY: B cannot execute requests it never accepts. LOOK FOR: no B traffic from new A.
5. **Recent policy/rollout changes.** WHAT: pod labels, service accounts, NetworkPolicy selectors. WHY: identity changes can alter reachability. LOOK FOR: selector mismatch.

# Step-by-Step Investigation

### Step 1 - Confirm connect timeout

* **What I check:** deepest Java exception and phase timing.
* **Why:** Feign, WebClient, and RestTemplate wrap different timeout causes.
* **Example:** trace request `ord-18b44e`.
* **Expected result:** pool acquire 2 ms, connect under 15 ms, response under 200 ms.
* **Bad result:** `ConnectTimeoutException` after 5000 ms with no TLS span.
* **Meaning:** TCP did not complete before A's budget.
* **Next branch:** retain selected IP and compare source populations.

### Step 2 - Rule DNS in or out

* **What I check:** A's actual resolver, answer, TTL, selected IP, and duration.
* **Why:** DNS delay can consume time before a connect is attempted.
* **Expected result:** `10.42.7.18` in 3 ms, same for old/new pods.
* **Bad result:** timeout/NXDOMAIN/SERVFAIL or different stale IP.
* **Meaning:** investigate DNS if bad; do not call it a TCP timeout.
* **Next branch:** with a good answer, test exact IP/port from both pod versions.

### Step 3 - Compare bounded TCP probes

* **What I check:** one safe connection from old A and one from new A.
* **Why:** same destination controls B health and isolates source path.
* **Expected result:** both connect around 8 ms.
* **Bad result:** old connects in 8 ms; new reaches the 5 s cap.
* **Meaning:** source identity/network state predicts failure.
* **Next branch:** split by node, zone, labels, and service account.

### Step 4 - Inspect packet/flow outcome

* **What I check:** SYN retransmits, RSTs, accepted sockets, and authorized flow logs for exact five-tuple.
* **Why:** timeout alone cannot name the dropping device.
* **Expected result:** SYN, SYN-ACK, ACK and an accepted B socket.
* **Bad result:** repeated SYN, no RST, flow action `DENY` for new source.
* **Meaning:** a policy or route silently drops the handshake.
* **Next branch:** identify the first differing policy selector; if flow says ALLOW, inspect routing/reverse path.

### Step 5 - Check Kubernetes policy identity

* **What I check:** pod labels, namespace labels, service account, NetworkPolicy selectors, ports, and policy types.
* **Why:** policies select identities, not application versions.
* **Expected result:** new pod matches `app=order-api`.
* **Bad result:** new pod has only `app.kubernetes.io/name=order-api`.
* **Meaning:** the allow rule misses every new pod.
* **Next branch:** compare rendered manifests and admission validation.

### Step 6 - Check alternatives before changing policy

* **What I check:** B listener, destination port, CNI/node health, route, firewall, NAT/conntrack, and reverse path.
* **Why:** multiple mechanisms yield missing SYN-ACK.
* **Expected result:** old source proves listener/target work; failures follow version across nodes.
* **Bad result:** one node fails for all destinations or every source fails to B.
* **Meaning:** one-node CNI/route or shared destination path is more likely.
* **Next branch:** use the dimension that predicts outcome; do not broaden policy speculatively.

### Step 7 - Verify later layers are absent

* **What I check:** TLS count, gateway upstream rate, B HTTP rate, B resources, and dependency rate.
* **Why:** failed TCP should produce no later work.
* **Expected result:** zero failed-call samples later; B stays normal for old traffic.
* **Bad result:** B has the request ID or a server span.
* **Meaning:** the timeout may be an outer deadline after connection, or telemetry target is wrong.
* **Next branch:** reclassify using the trace waterfall.

### Connect-timeout branch matrix

| Result | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| New A fails; old A succeeds to same IP | Source population controls outcome | The application version code is faulty | Compare labels, service account, proxy, node, and policy identity |
| One node fails for all destinations | Node path is suspect | CNI is definitely the cause | Compare routes, conntrack, host firewall, interface errors, and a peer node |
| One zone fails to every B target | Zone path/policy is suspect | B is healthy from all zones | Compare cross-zone routes, ACLs, and a same-zone control destination |
| Every source times out to one B IP | Destination/path-specific issue | B listener is absent | Check whether B sees SYN, sends SYN-ACK, and has a valid reverse route |
| B sees no SYN and flow says DENY | Drop occurs before B capture point | Which policy owner introduced it | Resolve the exact matching policy/rule and change event |
| B sees SYN but sends no SYN-ACK | Destination kernel/listener/backlog may be involved | Java code is slow | Inspect listen socket, SYN backlog, host policy, and kernel counters |
| B sends SYN-ACK but A never sees it | Reverse path loses the response | Forward path is healthy end to end | Inspect return route, asymmetric firewall, and destination NAT |
| Flow says ALLOW but connect times out | That observed device allowed the packet | Every later hop allowed it | Move capture/flow observation to the next boundary |
| TCP connects but TLS times out | TCP path works; secure negotiation stalls | Certificate validation is the cause | Inspect TLS alerts, server handshake load, SNI, and packet loss |
| Gateway emits 504 after 3 s | An HTTP gateway deadline expired | Gateway upstream connect timed out | Inspect gateway connect and upstream response phase separately |

### Packet evidence interpretation

A rising SYN retransmit counter supports loss or silent drop, but the counter is
node-wide unless labeled; I correlate source/destination and incident window.

A fast RST converts this scenario to refusal and sends me to listener, port,
sidecar, or reject policy.

No RST does not identify a firewall. Route black holes, security groups,
NetworkPolicy, host ACLs, reverse-path filtering, and dead next hops can all
produce the same client timer.

Packet capture is sensitive and requires authorization. I prefer existing flow
telemetry and bounded counters before capturing payload-bearing traffic.

If capture is approved, only handshake metadata for the exact five-tuple and
short UTC window is needed; application payload collection is unnecessary.

### Timeout-budget reasoning

A 5 s connect timeout inside a 6 s end-to-end deadline leaves almost no budget
for TLS, HTTP, B, or a safe retry.

Reducing the connect timeout can improve failure speed after the path is fixed,
but it does not repair packet drops.

Increasing it retains A threads/connections longer and worsens saturation.

Retries must fit the remaining deadline, target an idempotent operation, use
backoff/jitter, and stop when failure is deterministic by source policy.

I compare original request rate with socket attempts; 61 failed calls/s with
two retries would produce 183 connection attempts/s without one B request.

### Kubernetes-specific next checks

If policy selectors match, I compare namespace selectors and service account
identity because both can participate in an allow rule.

If all policy objects look correct, I verify the CNI's actual policy model and
status; YAML intent alone does not prove dataplane programming.

If only newly scheduled pods fail on one node, I compare CNI agent readiness,
node routes, and host-network state before changing application config.

If a debug pod succeeds, I do not clear policy until it uses the same labels,
namespace, service account, sidecar, node class, and destination path as A.

# Metrics to Check

| Metric | High / low / flat / deployment meaning |
|---|---|
| Connect timeouts | High at exact 5 s shows client connect budget; low moves to later phases |
| Connect successes | Zero for new A but normal for old A isolates source; split destination too |
| SYN retransmits | High supports loss/drop; low with fast error suggests rejection/config |
| TCP resets | High suggests active close/refusal; zero here fits silent drop |
| DNS duration/errors | High means stop at DNS; low/flat 3 ms rules it out for samples |
| A attempts vs B accepts | New-A attempts high, B accepts flat means pre-B loss |
| Request rate | Flat rejects traffic surge; high could expose conntrack/NAT capacity |
| Per-version/zone/node | After-deploy discontinuity and all-node failure point to identity labels |
| B CPU/queue/pools | Low/flat expected because calls do not arrive; high old-traffic load is separate |
| Retry attempts | High multiplies five-second waits; retries cannot repair a deterministic drop |
| Circuit breaker | Open is a consequence; verify threshold and source before tuning |
| p50/p95/p99/max | A mixed fleet may show normal p50 and 5 s p99; split version and outcome |

# Distributed Trace Investigation

```text
traceId=03f92f3577b34da6a3ce929d0e0e0003
Order A server                       5,006ms span=c001 ERROR
  Inventory client                  5,002ms span=c002 ERROR
    pool.acquire                         2ms span=c003
    dns.lookup                           3ms span=c004
    tcp.connect                      5,001ms span=c005 connect_timeout
  [no TLS, gateway, B, or DB span]
```

The parent shows user impact; the connect child owns nearly all time. Attributes must include source pod/version/zone, resolved IP, port, target, attempt, and deadline.

No B span supports a pre-B branch only after checking head/tail sampling, propagation, exporter health, and B access logs. A missing child can be instrumentation loss. A successful old-A control trace to the same B establishes that B's 171 ms path is healthy.

# Distributed Logs

```text
2026-09-13T11:27:22.417Z level=ERROR service=order-service
instance=order-a-4.19.0-r4w9p version=4.19.0 zone=eu-west-1c
traceId=03f92f3577b34da6a3ce929d0e0e0003 spanId=c002 requestId=ord-18b44e
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
target=10.42.7.18:8080 phase=connect timeout_ms=5000 attempt=1
error="io.netty.channel.ConnectTimeoutException: connection timed out"
```

Join this with CNI flow records using timestamp, source/destination IP and port, then compare old A. The application log proves its local timer expired. It does not prove NetworkPolicy caused the loss. Flow action, source labels, policy selector, and recovery after label correction create the causal chain.

# Commands / Tools

```powershell
Resolve-DnsName inventory-b.shop.svc.cluster.local -DnsOnly
Test-NetConnection 10.42.7.18 -Port 8080 -InformationLevel Detailed
curl.exe -v --connect-timeout 5 --max-time 7 http://10.42.7.18:8080/actuator/health
```

A five-second false result proves one source could not open one port then; it does not identify firewall, route, reverse path, or listener.

```bash
dig +time=2 +tries=1 inventory-b.shop.svc.cluster.local
nc -vz -w 5 10.42.7.18 8080
curl -sS -v --connect-timeout 5 --max-time 7 http://10.42.7.18:8080/actuator/health
ss -s
```

`ss -s` may reveal broad socket pressure but not a specific policy. Ping can succeed while TCP is dropped.

```bash
kubectl get pods -n shop -l app.kubernetes.io/name=order-api -o wide --show-labels
kubectl get networkpolicy -n shop -o yaml
kubectl exec -n shop order-a-4.19.0-r4w9p -- nc -vz -w 5 10.42.7.18 8080
kubectl exec -n shop order-a-4.18.2-good -- nc -vz -w 5 10.42.7.18 8080
```

These are bounded inspections. NetworkPolicy evaluation depends on CNI implementation. A debug pod is not equivalent unless labels, namespace, service account, sidecar, and node path match.

# Root Cause

The egress NetworkPolicy allowed pods selected by `app=order-api`. Deployment 4.19.0 supplied only `app.kubernetes.io/name=order-api`.

```text
new deployment
-> source identity label changes
-> allow policy no longer selects new pods
-> SYN packets are dropped
-> connect waits 5 seconds and expires
-> TLS/HTTP/B never run
-> new-version orders fail
```

Old/new probes to the same target, flow logs, labels, selector, and post-fix recovery support every arrow.

# Fix

**Immediate mitigation:** pause rollout and restore the approved stable identity label. Do not broaden ingress to all namespaces. Do not raise connect timeout or add retries.

**Root cause correction:** align deployment and NetworkPolicy selectors using an immutable identity label and explicit port 8080.

**Permanent fix:** admission checks ensure every protected workload matches intended policies; canaries run caller-context connectivity; policy changes deploy before dependent identity changes where safe; flow telemetry preserves reason/source.

# Verification

Before: new A connect p99 5.001 s, timeout 61/s, B accepts 0/s from new A, business success 0% for new pods.

After: new A connect p99 11 ms, timeout 0/s, retransmits return to baseline, B accepts the expected 61/s, combined business success 99.96%, attempts/request 1.00 for 30 minutes. Each new pod and zone completes a real reservation once.

# Prevention

* Alert on connect-timeout rate and exact timeout plateaus by source version/zone.
* Alert on A outbound attempts without B accepts.
* Policy-as-code tests validate selectors, namespace/service account, ports, and default-deny behavior.
* Runbook branches refusal versus timeout using timing, RST, retransmit, and flow action.
* Canary tests DNS, TCP, TLS, HTTP, and business outcome from the real identity.
* Keep connect timeout short and distinct from read deadline; propagate an end-to-end deadline.
* Bounded retries require idempotency, jitter, remaining budget, and downstream capacity.

# Interview Answer

### What I would say in an interview

I first prove it is a TCP connect timeout, not DNS or a read timeout. Here only A 4.19.0 failed: DNS returned the same B IP in 3 ms, but connect spans ended exactly at five seconds with SYN retransmits and no TLS or B span. Old A reached the same B in 8 ms. Comparing pod identity with NetworkPolicy showed the rollout changed the label the allow rule selected. I paused the rollout, restored the stable label, then added policy contract tests. I verified sub-15-ms connects, matching B accepts, zero retransmits/timeouts, and business success above 99.9%.

### Common interviewer traps

Do not call every timeout a slow B, confuse a connect timeout with 504, or increase timeout first. A timeout does not identify the dropping device. A successful probe from a laptop or unlabeled debug pod is not equivalent.

### Quick memory flow

Phase -> DNS -> exact IP/port -> old/new source comparison -> RST/retransmit/flow -> policy/route/listener -> targeted mitigation -> full-path verify.

# Interview Follow-up Questions

1. **Why does a firewall drop time out?** No RST or SYN-ACK returns, so the client waits and retransmits.
2. **Could B still be responsible?** Yes: wrong route, listener backlog, host firewall, or reverse path; old-source success narrows it here.
3. **Gateway status versus direct connect timeout?** A direct client sees a socket exception. Envoy normally returns 503 `UF`; NGINX can return 502 with its own upstream connect error.
4. **504 versus connect timeout?** 504 is an HTTP response from an intermediary whose upstream deadline expired.
5. **What if one node fails?** Compare CNI, route, conntrack, and node network rather than workload label.
6. **Why not retry?** Deterministic policy drops only multiply waits and consume the deadline.
7. **How do WebClient timeouts differ?** Connect timeout covers socket establishment; response timeout covers waiting after connection.
8. **What does successful ping prove?** Only ICMP reachability for that sample, not TCP port 8080.
