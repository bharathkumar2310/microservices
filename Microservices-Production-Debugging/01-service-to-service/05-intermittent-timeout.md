# Problem

Service A sometimes succeeds and sometimes times out calling Service B. "Intermittent" is a distribution, not a cause. Failures may follow one DNS answer, target, zone, version, node, connection age, retry attempt, or data class.

The central method is to compare matched successes and failures and find the dimension that predicts outcome.

# Production Situation

At `2026-09-13T13:08:22Z`, Order A calls `POST /v1/inventory/reservations`.

* requestId `ord-d50bc4`, traceId `05f92f3577b34da6a3ce929d0e0e0005`
* source `order-a-4.18.2-p8t6d`, zone `eu-west-1a`
* DNS answers `.18`, `.19`, and retired `.37`, TTL 30
* `.18/.19`: connect under 10 ms, route under 120 ms
* `.37`: connect timeout at 3.000 s
* failures: 32-35%, mostly new connections
* original orders 210/s; Inventory attempts 286/s
* attempts/original = 1.36; healthy B CPU rises 36% to 51%

# Architecture

```text
Order A direct discovery
   |
   +-- 10.42.7.18 B-1 -> success
   +-- 10.42.7.19 B-2 -> success
   `-- 10.42.18.37 retired VM -> connect timeout

Gateway inventory separately contains only B-1 and B-2.
```

Gateway success does not clear A's direct path because they use different discovery and pools.

# What I Check FIRST

1. **Failure probability and denominator.** WHAT: first-attempt and final outcomes. WHY: retries hide defects. LOOK FOR: about one of three.
2. **Matched target correlation.** WHAT: selected IP/backend on every success/failure. WHY: averages mix two populations. LOOK FOR: all failures use `.37`.
3. **Connection state.** WHAT: new/reused, age, pool. WHY: stale DNS and stale keep-alive have opposite shapes. LOOK FOR: new `.37` connects fail.
4. **Attempt waterfall.** WHAT: target and duration per retry. WHY: retries amplify and consume deadline. LOOK FOR: 3 s failure then 96 ms success.
5. **Discovery versus actual inventory.** WHAT: DNS answers, registry, EndpointSlice, gateway targets. WHY: control planes can disagree. LOOK FOR: `.37` only in DNS.

# Step-by-Step Investigation

### Step 1 - Create success and failure datasets

* **What I check:** same route/minute/version/identity/payload, grouped by outcome.
* **Why:** one average cannot describe 100 ms and 3 s populations.
* **Expected result:** no single dimension predicts failure.
* **Bad result:** selected IP `.37` predicts 100% timeout.
* **Meaning:** target selection causes intermittency.
* **Next branch:** validate whether `.37` should exist.

### Step 2 - Inspect DNS answers, not only response code

* **What I check:** answer set, order, TTL, cache age, resolver, and chosen IP.
* **Why:** `NOERROR` can return a wrong/stale address.
* **Expected result:** only `.18/.19`, both current.
* **Bad result:** `.37` remains one of three answers.
* **Meaning:** DNS correctness, not availability, is defective.
* **Next branch:** compare discovery ownership and target lifecycle.

### Step 3 - Compare exact TCP outcomes per answer

* **What I check:** bounded connect duration, refusal/timeout/reset, source zone.
* **Why:** each answer can have a different path/state.
* **Expected result:** every answer connects under 15 ms.
* **Bad result:** `.37` retransmits to 3 s; `.18/.19` connect under 10 ms.
* **Meaning:** `.37` is unreachable, not slow B code.
* **Next branch:** verify whether it is retired, routed, or policy-blocked.

### Step 4 - Separate new from reused connections

* **What I check:** connection ID, age, pool target, and reuse flag.
* **Why:** pools delay exposure to changed DNS.
* **Expected result:** outcomes independent of reuse.
* **Bad result:** existing healthy connections work; new resolutions select `.37`.
* **Meaning:** cached healthy sockets mask a stale answer until churn.
* **Next branch:** do not flush all pools before preserving this evidence.

### Step 5 - Analyze retry amplification

* **What I check:** attempts per original, attempt target/duration, remaining deadline, B load.
* **Why:** final success can hide customer latency and extra backend load.
* **Expected result:** 1.00 attempt/request.
* **Bad result:** 1.36; attempt 1 wastes 3 s on `.37`, attempt 2 hits `.18`.
* **Meaning:** retries add 76 attempts/s and threaten healthy targets.
* **Next branch:** bound retries while removing stale discovery.

### Step 6 - Reconcile control planes

* **What I check:** DNS/registry records, EndpointSlice, gateway target inventory, VM lifecycle audit.
* **Why:** direct and gateway paths may use separate inventories.
* **Expected result:** all contain the same live targets.
* **Bad result:** gateway has two live targets; DNS retains terminated VM `.37`.
* **Meaning:** deregistration failed.
* **Next branch:** preserve audit evidence, then remove `.37` through owner control.

### Step 7 - Test alternate intermittent branches

* **What I check:** per-instance runtime, zone loss, stale keep-alive resets, TLS on new sessions, data scope, GC.
* **Why:** intermittency is not always DNS.
* **Expected result:** these do not predict outcome.
* **Bad result:** reused connections reset while new ones succeed, or one B has high GC.
* **Meaning:** follow drain/pool or instance-runtime branch instead.
* **Next branch:** use the strongest matched predictor, never probability alone.

### Intermittency branch matrix

| Failure predictor | What it suggests | What it does not prove | Exact next check |
|---|---|---|---|
| One of three IPs | Per-target path or lifecycle | DNS itself is the owner | Compare registry ownership, port result, and target state |
| One B instance, all connections | Instance runtime/config/node | Its code version is faulty | Compare image/config hash, GC, queue, pools, and node |
| Reused connections only | Stale keep-alive/draining/protocol close | DNS answer is stale | Join connection age and target lifecycle, then test a new connection |
| New connections only | New DNS selection or TLS handshake | Existing sessions are healthy forever | Compare resolved IP, SNI, trust, and pool age |
| One A version | Caller config/client/policy identity | B is fully healthy | Compare effective destination, timeout, labels, proxy, and trust |
| One A node | Node route/CNI/NAT/conntrack | Application instance is bad | Compare other destinations and a same-version pod on another node |
| One zone | Resolver/path/target-zone coupling | Cross-zone networking alone is wrong | Compare same-zone and cross-zone controls and policy |
| One payload/data key | B business dependency or lock/plan | Network is healthy for all calls | Compare matched traces and dependency fingerprints |
| Exactly after connection age | Keep-alive expiry or infrastructure idle timeout | B handler is slow | Compare reset reason, idle duration, and new connection success |
| Random across every dimension | Missing label, shared saturation, or telemetry gap | The issue is truly random | Add target/reuse/attempt/deadline labels and inspect shared resources |

### Retry accounting

I chart original user requests separately from downstream attempts.

At 210 original requests/s and 286 attempts/s, the retry amplification factor
is `286 / 210 = 1.36`.

Final success can remain near 99% while first-attempt success is only 66%.

The failed first attempt consumes 3 s, leaving 2 s of a 5 s order deadline.

If attempt two succeeds in 96 ms, the user still experiences roughly 3.1 s
and healthy targets receive extra load.

If both attempts can mutate inventory, the same idempotency key must flow to B.

If the retry chooses `.37` again, random selection without target ejection
wastes the remaining deadline.

A per-target circuit breaker may reduce impact, but its open state is a
protective consequence; discovery must still remove the retired target.

### DNS cache and connection-pool branches

The DNS TTL of 30 seconds does not guarantee the JVM refreshes every 30 seconds;
JVM and client caches may use different policies.

Connection pools can outlive DNS TTL because an existing socket needs no new
lookup. I inspect connection creation time and resolved address.

Flushing pools may expose the stale answer to every request and cause a
connection storm, so I preserve evidence and remove the bad target first.

If repeated `dig` never returns `.37` but A logs do, I inspect JVM cache,
sidecar DNS cache, and application discovery rather than authoritative DNS.

If A logs current IPs but the gateway selects `.37`, I inspect gateway target
inventory; the two control planes are independent.

### Recovery decision points

After removing `.37`, DNS answer correctness should recover before the old TTL
expires only if caches are refreshed through supported behavior.

If timeouts persist on `.18/.19`, the stale record was not the only cause.

If attempts/request remains 1.36, retry policy or telemetry accounting remains
wrong even when the retired target is gone.

If healthy B CPU stays at 51%, I inspect backlog from retries and allow it to
drain before declaring capacity stable.

I verify every A instance and zone because one long-lived cache can preserve
the incident after the authoritative record is fixed.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| Failure rate | 33% can suggest 1/3 targets; only backend labels prove it |
| First-attempt vs final success | Large gap shows retries masking defect |
| Attempts/original | High 1.36 quantifies amplification; low 1.00 is healthy |
| Per-target connect p99 | `.37` at 3 s versus peers 10 ms localizes target/path |
| DNS rcode/duration | Low and NOERROR do not prove correct answers |
| New/reused split | New-only failure suggests changed DNS/TLS; reused-only suggests stale draining socket |
| B accepted rate | No `.37` accepts means pre-B; peers rise due to retries |
| Healthy-peer CPU/queue | High after errors shows retry impact; verify capacity before draining |
| Circuit state | Uneven per A instance reflects local samples; opening is consequence |
| p50/p95/p99/max | Mixed modes make p50 normal and p99 3 s; split target/outcome |
| After deregistration | First-attempt success and attempts/request should reverse immediately |
| Per-zone/source | One zone suggests route/resolver; all zones plus one IP suggests target |

# Distributed Trace Investigation

```text
traceId=05f92f3577b34da6a3ce929d0e0e0005
Order A server                        3,112ms span=e001
  Inventory attempt=1 target=.37     3,001ms span=e002 ERROR
    dns.lookup                           4ms span=e003 answers=3
    tcp.connect                      3,000ms span=e004 timeout
  Inventory attempt=2 target=.18        96ms span=e005 OK
    Inventory B server                  82ms span=e006
```

The trace exposes hidden retries and different targets. A final 201 alone would hide a three-second near-failure.

A missing `.37` B span fits pre-B timeout, but validate sampling and access logs. Missing retry spans can result from client instrumentation gaps, so compare attempt counters and structured logs.

# Distributed Logs

```text
2026-09-13T13:08:22.417Z level=WARN service=order-service
instance=order-a-4.18.2-p8t6d version=4.18.2 zone=eu-west-1a
traceId=05f92f3577b34da6a3ce929d0e0e0005 spanId=e002 requestId=ord-d50bc4
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
attempt=1 resolved_ip=10.42.18.37 connection_reused=false latency_ms=3000
error="ConnectTimeoutException" remaining_deadline_ms=1900
```

Correlate target and attempt across trace, A logs, registry audit, and B access logs. A log proves one target timed out; it does not prove DNS ownership or retirement. Lifecycle/deregistration evidence and recovery after record removal prove the mechanism.

# Commands / Tools

```powershell
1..6 | ForEach-Object { Resolve-DnsName inventory-b.internal -DnsOnly }
Test-NetConnection 10.42.18.37 -Port 8080
curl.exe -v --connect-timeout 3 --max-time 5 http://10.42.7.18:8080/actuator/health
```

Bound repeated lookups avoid load. Resolver output may not match JVM caching.

```bash
dig +time=2 +tries=1 inventory-b.internal
for ip in 10.42.7.18 10.42.7.19 10.42.18.37; do nc -vz -w 3 "$ip" 8080; done
```

Each probe proves one port/sample only. Do not loop aggressively in production.

```bash
kubectl get endpointslice -n shop -l kubernetes.io/service-name=inventory-b -o wide
kubectl get pods -n shop -l app=inventory -o wide
kubectl exec -n shop order-a-4.18.2-p8t6d -- getent ahosts inventory-b.internal
```

`getent` follows container name-service configuration better than a laptop `dig`; JVM DNS cache can still differ.

# Root Cause

Service discovery retained retired VM `10.42.18.37` after deregistration failed.

```text
retirement -> failed deregistration -> stale third answer
-> random new connection selects dead subnet -> 3 s timeout
-> retry selects healthy B -> extra latency/load -> intermittent customer failure
```

# Fix

**Immediate mitigation:** remove `.37` through the discovery control plane, reduce retry amplification, and confirm two remaining targets have capacity.

**Root cause correction:** make drain and deregistration idempotent and reconcile advertised addresses against live ownership.

**Permanent fix:** health-aware registration leases, expiry, lifecycle alarms, per-answer synthetic probes, and deadline-aware retries. Do not hard-code live IPs or merely flush A caches.

# Verification

Before: first-attempt failure 34%, connect timeout 71/s, attempts/request 1.36, healthy B CPU 51%, p99 3.1 s.

After: all answers are `.18/.19`, first-attempt success 99.96%, timeouts 0/s, attempts/request 1.00, healthy B CPU 37%, p99 174 ms for 30 minutes. Business reconciliation finds no duplicates from prior retries.

# Prevention

* Alert when advertised targets are absent from compute/EndpointSlice inventory.
* Dashboard first-attempt and final success, attempts/request, target and connection reuse.
* Runbook compares answer correctness, target outcome, pool age, and retries.
* Test retirement under watch disconnect and retry deregistration.
* Bound DNS TTL/JVM caching consistent with lifecycle.
* Require idempotency keys before retrying reservation operations.

# Interview Answer

### What I would say in an interview

For intermittency I compare matched successes and failures. The 32-35% rate resembled one of three answers, and target labels proved every failure selected retired IP `.37`. DNS was fast and returned NOERROR, but `.37` timed out at three seconds while `.18/.19` connected under 10 ms. Retries raised attempts from 210 to 286 per second. Failed deregistration had left the old VM in discovery. I removed it, bounded retries, fixed idempotent lifecycle reconciliation, and verified first-attempt success above 99.9%, one attempt per order, normal peer load, and correct reservations.

### Common interviewer traps

Do not average success and failure, assume normal DNS latency means correct DNS, or celebrate retry-assisted final success. A one-third fraction is a clue, not proof.

### Quick memory flow

Denominator -> matched pairs -> target/IP -> new/reused -> attempts -> inventories -> lifecycle -> remove -> first-attempt verification.

# Interview Follow-up Questions

1. **Why did reused connections work?** They were already connected to healthy targets.
2. **Could round-robin DNS guarantee exactly 33%?** No; caches, ordering, and pools alter selection.
3. **Why not retry twice?** More retries amplify load and may exceed the end-to-end deadline.
4. **What if one zone fails?** Compare resolver view, route, policy, and target zone.
5. **What if only old connections fail?** Investigate stale keep-alive and draining resets.
6. **Does NOERROR prove DNS is healthy?** It proves a successful response code, not correct addresses.
7. **How do circuit breakers help?** A per-target breaker can stop selecting `.37`, but discovery still must be repaired.
8. **Why verify first-attempt success?** Final success can conceal latency and load amplification.
