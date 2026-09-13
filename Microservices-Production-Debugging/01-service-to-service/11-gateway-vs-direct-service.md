# Problem

A direct call from Service A to Service B succeeds, but the same operation through the gateway fails, or the reverse. The paths are not equivalent: DNS, target inventory, port, protocol, TLS termination, SNI, client identity, headers, routing, retries, connection pools, health checks, rate limits, and deadlines can differ.

The objective is to compare hop by hop without treating a direct success as proof that the gateway is faulty.

# Production Situation

At `2026-09-13T22:13:22Z`, A receives errors only through the regional load balancer.

* route `POST /v1/inventory/reservations`
* requestId `ord-57d108`, traceId `0ef92f3577b34da6a3ce929d0e0e000e`
* A `order-a-4.18.2-k2m5q`, gateway `gw-5`, zone `eu-west-1a`
* failing backend `inventory-b-6`, `10.42.7.26:8080`, version `7.5`
* current replacement `inventory-b-9`, version `7.6`
* normal p99 180 ms, errors 0.05%
* abnormal gateway path errors 14%; direct B-9 p99 88 ms
* every failure uses a reused connection to B-6 and gets reset
* B-6 absent from DNS/EndpointSlice but remains LB target in `draining` for 27 minutes
* attempts/request 1.14 because retry to B-9 often succeeds

# Architecture

```text
Direct:
Order A -> current Service discovery -> B-9 -> success

Gateway:
Order A -> LB/gateway target inventory
              +-> B-9 new connection -> success
              `-> B-6 stale reused connection -> RESET
```

The direct path and gateway path share a business destination but not discovery or connection state.

# What I Check FIRST

1. **Make paths explicit.** WHAT: exact URL, resolver, proxy, target, SNI, identity, headers, timeout. WHY: "same call" may not be same. LOOK FOR: direct selects B-9; gateway can select B-6.
2. **Identify generator/transport result.** WHAT: HTTP status/flag or reset/timeout. WHY: 502/503/504 differ. LOOK FOR: upstream reset `connection_termination`.
3. **Join outcome to target/connection.** WHAT: backend, connection ID, age, reuse, gateway instance. WHY: stale pooled state can be probabilistic. LOOK FOR: B-6 + reused = failure.
4. **Reconcile target inventories.** WHAT: DNS, EndpointSlice, gateway/LB list. WHY: independent control planes drift. LOOK FOR: B-6 only in LB.
5. **Rollout/drain timeline.** WHAT: readiness false, endpoint removal, preStop, deregistration, connection close. WHY: ordering controls graceful termination. LOOK FOR: watch disconnect before termination.

# Step-by-Step Investigation

### Step 1 - Build a path comparison table

* **What I check:** source, name/IP, route, proxy, target, port, protocol, SNI, trust, client cert, auth headers, timeout.
* **Why:** only controlled differences can explain outcome.
* **Expected result:** paths differ only by intended gateway hop.
* **Bad result:** direct uses current Service target; gateway inventory includes retired B-6.
* **Meaning:** target control-plane state differs.
* **Next branch:** correlate every request with selected backend.

### Step 2 - Classify the failure precisely

* **What I check:** reset versus refusal/timeout/TLS/status and elapsed phase.
* **Why:** an existing-socket reset differs from new-connect failure.
* **Expected result:** B-9 returns 201 in 88 ms.
* **Bad result:** reused B-6 connection resets immediately; retry reaches B-9.
* **Meaning:** stale connection/target lifecycle, not slow B code.
* **Next branch:** split by reuse, connection age, and target.

### Step 3 - Prove target correlation

* **What I check:** error rate and counts by gateway, backend, target weight, connection reuse.
* **Why:** 14% resembles B-6 weight but probability is not proof.
* **Expected result:** all targets have equal low errors.
* **Bad result:** B-6 has 100% reset; all other targets zero.
* **Meaning:** B-6 selection fully predicts failure.
* **Next branch:** check whether B-6 should still be routable.

### Step 4 - Reconcile discovery sources

* **What I check:** A DNS/service answers, Kubernetes EndpointSlice, gateway/LB target inventory, target state.
* **Why:** direct and gateway can observe different registries.
* **Expected result:** both list only B-9/current peers.
* **Bad result:** B-6 absent from Service but LB says `draining`, weight 0.14.
* **Meaning:** LB lifecycle state outlived compute/discovery.
* **Next branch:** inspect controller watch and deregistration event.

### Step 5 - Inspect graceful-drain sequence

* **What I check:** readiness transition, endpoint withdrawal, deregistration confirmation, preStop, termination grace, connection max age.
* **Why:** safe ordering prevents new routing and lets in-flight requests finish.
* **Expected result:** weight reaches zero before process exits.
* **Bad result:** controller watch disconnects, deregistration never acknowledges, B-6 terminates.
* **Meaning:** stale target and pooled sockets remain.
* **Next branch:** preserve controller/target/connection evidence, then remove stale target.

### Step 6 - Distinguish common direct/gateway branches

* **What I check:** gateway route/path rewrite, Host header, auth, rate limit, health, SNI/TLS, protocol, deadline, retries.
* **Why:** other path-only failures can look similar.
* **Expected result:** contracts match.
* **Bad result:** direct success plus Envoy 503 `UF` -> upstream connect/TLS; Envoy 503 `UH` -> health; Envoy 504 `UT` -> response deadline; NGINX 502 -> inspect its upstream error; 404 -> route rewrite; 401/403 -> identity.
* **Meaning:** product subreason chooses the next owner.
* **Next branch:** test only the differing contract.

### Step 7 - Check capacity and retry impact

* **What I check:** remaining targets' rate/CPU/queue/pools, attempts/request, deadline, idempotency.
* **Why:** removing a weighted target shifts load; retry can duplicate writes.
* **Expected result:** peers have safe headroom and attempts 1.00.
* **Bad result:** attempts 1.14 and peers gain load.
* **Meaning:** mitigation must consider capacity and reconciliation.
* **Next branch:** drain stale target via control plane, limit safe retry, verify business state.

### Direct-versus-gateway comparison worksheet

| Property | Direct path | Gateway path | Failure meaning and next check |
|---|---|---|---|
| DNS source | Kubernetes Service | Gateway target controller | Reconcile answers and target inventory if they differ |
| Selected target | Current B-9 | B-6 or B-9 | Join every outcome to backend and weight |
| Connection | New/current pool | Old gateway pool possible | Split by reuse, ID, age, and target state |
| Port/protocol | HTTP 8080 | HTTP 8080 in this incident | If different, test protocol/port mapping before B code |
| TLS/SNI | None on this hop | None on this hop | In HTTPS incidents compare SNI/trust/client identity |
| Host/path | Direct service route | Gateway rewrite rules | A 404/405 branch requires route and rewrite comparison |
| Authentication | Service credential | Gateway may exchange identity | A 401/403 branch requires generator and identity decision |
| Health selection | EndpointSlice readiness | LB target health/drain | Direct success cannot clear gateway eligibility |
| Timeout | A client budget | Gateway upstream plus A budget | A 504 branch requires exact deadline owner |
| Retry | A client policy | Gateway upstream retry may also run | Count all attempts and propagate idempotency key |

### Status and transport branches

If Envoy returns 503 `UF` and direct works, I inspect gateway-to-B TCP/TLS
using the exact gateway target, transport failure reason, and identity.

If NGINX returns 502 and direct works, I inspect its exact error such as
connect refusal, premature close, invalid header, or protocol mismatch.

If gateway returns 503 `UH`, I compare target-health reasons and inventories;
I do not tune B code before an upstream attempt exists.

If gateway returns 504 `UT`, I compare upstream connect and response duration,
then open B's longest child.

If gateway returns 401/403, I identify whether the gateway or B made the
authorization decision and compare propagated credentials.

If gateway returns 404 while direct returns 200, I compare route match, path
rewrite, Host header, method, base path, and deployment revision.

If the gateway socket resets before an HTTP status, I inspect target lifecycle,
connection reuse, protocol close, and gateway reset reason.

### Lifecycle timeline for B-6

At 21:45:58, the LB controller's EndpointSlice watch disconnects.

At 21:46:02, B-6 readiness becomes false and Kubernetes removes it.

At 21:46:08, B-6 begins its 60-second termination grace period.

The controller misses deregistration and leaves LB weight at 0.14.

At 21:47:08, B-6 exits while gateway workers retain old pooled connections.

At 22:13, B-6 still reports `draining` after 27 minutes and resets requests.

This ordering proves routing state outlived compute; it also explains why
direct Service discovery, which no longer lists B-6, succeeds.

### Per-gateway and connection branches

If only `gw-5` selects B-6, its local watch/cache state is stale; compare its
revision and last successful resync with peer gateways.

If every gateway selects B-6, the shared target controller/control plane is
stale.

If only reused connections fail, expire/drain B-6 connections after preserving
IDs and ages; do not flush healthy target pools.

If new connections also select B-6, weight/target inventory remains active and
must be corrected before pool cleanup.

If B-6 is removed but resets continue, map the new failing target rather than
assuming delayed metrics.

### Retry and business-outcome branches

The trace's first attempt resets before B-6 can create a server span.

The second attempt reaches B-9 and returns 201, so final HTTP success hides a
failed transport attempt.

Both attempts carry the same idempotency key; without it, ambiguous resets
could create duplicate reservations.

Attempts/request must fall from 1.14 to 1.00 after target removal.

If final success is restored but attempts stay high, routing or retry behavior
is still unhealthy.

Reconciliation checks committed reservation count, duplicate keys, late
responses, and orphan order state before closure.

### Controlled drain verification

In a canary rollout I first make B-9 readiness false.

I verify EndpointSlice removal and LB weight zero before process termination.

I watch active connections reach zero or the documented drain limit.

New requests must select peers while in-flight requests finish once.

No reset, retry spike, duplicate, or peer saturation may occur.

Only after this sequence passes do I consider the lifecycle fix durable.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| Direct vs gateway success | Divergence proves path difference, not which component is wrong |
| Error by backend/weight | B-6 100% and fleet 14% ties fault to weighted target |
| New/reused connection | Reused-only reset selects stale pool/drain; new-only can select DNS/TLS |
| Reset/refusal/timeout | Reset closes existing socket; refusal rejects new socket; timeout drops handshake |
| LB target state/age | Draining 27m versus 60s normal reveals stuck lifecycle |
| DNS/EndpointSlice target count | Current list excluding B-6 contrasts stale LB inventory |
| Gateway status/flag | 502/503/504 must be split by generator/subreason |
| Retry attempts/original | High 1.14 masks first-attempt failure and loads peers |
| Peer CPU/queue/pools | High after target removal risks cascade; verify headroom |
| p50/p95/p99/max | Retried successes can inflate tails while final error appears low |
| Per-gateway/zone/version | One gateway suggests local cache/watch; all suggest shared controller |
| After target removal | Resets should drop immediately; attempts normalize and peer load stabilize |

# Distributed Trace Investigation

```text
traceId=0ef92f3577b34da6a3ce929d0e0e000e
Order A server                         121ms span=k001
  gateway client                      113ms span=k002
    attempt=1 target=B-6 reused=true    8ms span=k003 ERROR reset
    attempt=2 target=B-9 reused=false  88ms span=k004
      Inventory B-9 server             76ms span=k005
        PostgreSQL                     19ms span=k006
```

The trace captures a customer success that hides a failed first attempt. Inspect target state, connection ID/age/reuse, gateway instance/version/zone, attempt, remaining deadline, and idempotency key.

No B-6 server span is expected because the old process is gone, but sampling/export loss can also remove it. Controller state and socket reset metrics corroborate. Direct B-9 trace is a control, not proof about gateway inventory.

# Distributed Logs

```text
2026-09-13T22:13:22.417Z level=WARN service=regional-gateway
instance=gw-5 version=3.14.2 zone=eu-west-1a
traceId=0ef92f3577b34da6a3ce929d0e0e000e spanId=k003 requestId=ord-57d108
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
target=10.42.7.26:8080 backend=inventory-b-6 target_state=draining
connection_id=cx-8842 connection_age_s=1732 reused=true attempt=1 latency_ms=8
error="upstream_reset: connection_termination"
```

The log shows the gateway selected stale state; it does not prove why deregistration failed. Correlate with target inventory, endpoint watch disconnect, pod termination, and a matched B-9 success.

# Commands / Tools

```powershell
Resolve-DnsName inventory-b.shop.svc.cluster.local -DnsOnly
Test-NetConnection 10.42.7.29 -Port 8080
curl.exe -v --connect-timeout 2 --max-time 5 https://gateway.internal/v1/inventory/health
```

The direct probe and gateway health route may not use the same business route or target. Capture headers/remote target when exposed safely.

```bash
getent ahosts inventory-b.shop.svc.cluster.local
nc -vz -w 2 10.42.7.29 8080
curl -sS -v --connect-timeout 2 --max-time 5 http://10.42.7.29:8080/actuator/health
```

Do not probe terminated B-6 repeatedly. `nc` does not prove HTTP or identity.

```bash
kubectl get endpointslice -n shop -l kubernetes.io/service-name=inventory-b -o wide
kubectl get pods -n shop -l app=inventory -o wide
kubectl get events -n shop --sort-by=.metadata.creationTimestamp
kubectl logs -n gateway gw-5 --since=30m
```

These read state and history. Gateway target inventory may require an approved product-specific read API. Never manually delete arbitrary resources or flush all pools without preserving evidence.

# Root Cause

The load-balancer controller lost its EndpointSlice watch immediately before B-6 termination. Deregistration did not complete, so the LB retained B-6 with weight 0.14 and old pooled connections reset.

```text
watch disconnect -> missed deregistration -> routing state outlives B-6
-> gateway reuses closing socket -> reset -> retry to B-9
-> 14% first attempts fail and load amplifies
```

# Fix

**Immediate mitigation:** preserve controller/target/socket evidence, verify peer capacity, then remove B-6 using the approved LB control plane and age out its pooled connections. Reconcile possible retried writes.

**Root cause correction:** make watch recovery and deregistration idempotent with full resync.

**Permanent fix:** coordinate readiness false -> endpoint withdrawal -> LB weight zero -> connection drain -> process exit; bound connection age; alarm on excessive drain duration and inventory disagreement.

# Verification

Before: gateway first-attempt error 14%, B-6 reset 100%, attempts/request 1.14, drain age 27 minutes, business p99 412 ms.

After: B-6 absent from LB/DNS/EndpointSlice, resets 0/s, attempts/request 1.00, peer p99 176 ms, business success 99.97% for 30 minutes. A controlled rollout shows weight zero before termination and no duplicate reservations.

# Prevention

* Alert on LB targets absent from service discovery and drain age beyond threshold.
* Dashboard target, connection reuse/age, reset reason, retries, peer capacity, and rollout events.
* Chaos/test watch reconnect, missed events, full resync, and idempotent deregistration.
* Runbook creates an explicit direct/gateway comparison table before conclusions.
* Deployment tests route/Host/SNI/auth/health/deadline and graceful drain.
* Use idempotency keys and bounded retries for writes.

# Interview Answer

### What I would say in an interview

I do not treat direct and gateway calls as equivalent. I compare DNS, target inventory, SNI, identity, route, deadline, and connection state. Here direct calls selected B-9 and succeeded, while every gateway failure used a reused connection to B-6. B-6 was absent from DNS and EndpointSlice but remained in the load balancer as draining for 27 minutes. A controller watch disconnect had missed deregistration. I preserved evidence, removed the stale target through the control plane, fixed idempotent resync/drain ordering, and verified zero resets, one attempt per request, safe peer capacity, and correct reservations.

### Common interviewer traps

Do not say direct success proves gateway failure, compare different identities/routes, or restart B when the failing target no longer exists. Do not ignore retries that hide first-attempt failure.

### Quick memory flow

Define both paths -> classify error -> join target/reuse -> reconcile inventories -> inspect drain timeline -> capacity/retries -> remove stale state -> controlled drain verify.

# Interview Follow-up Questions

1. **What differs between paths?** DNS, targets, ports, TLS/SNI, identity, headers, routes, health, pools, retry, and deadlines.
2. **Why reused connections matter?** They can outlive DNS and target changes.
3. **502/503/504 quick distinction?** NGINX can use 502 for an unusable upstream exchange. Envoy `UF` and `UH` normally map to 503, while `UT` maps to 504. Always retain generator and subreason.
4. **Could direct fail while gateway works?** Yes; gateway may have different route, trust, identity, cache, or target.
5. **Why not flush all pools first?** It destroys connection-age evidence and may cause a connection storm.
6. **How should graceful drain order work?** Stop new eligibility, wait for deregistration, drain in-flight/pooled connections, then terminate.
7. **What if one gateway fails?** Compare its watch/cache/config revision and node path with a healthy gateway.
8. **Why reconcile business writes?** A reset/retry can create late or duplicate effects despite final success.
