# Problem

A DNS failure occurs before Service A has a destination IP. Common outcomes are:

* `NXDOMAIN`: the name does not exist in this resolver view.
* `SERVFAIL`: the resolver could not complete resolution.
* timeout: no usable DNS response before the lookup budget.
* wrong/stale answer: lookup succeeds but points to the wrong target.

No IP means no TCP, TLS, or HTTP attempt.

# Production Situation

At `2026-09-13T16:22:22Z`, Order A cannot resolve Inventory.

* route `POST /v1/inventory/reservations`
* requestId `ord-01a849`, traceId `08f92f3577b34da6a3ce929d0e0e0008`
* A `order-a-4.18.2-k2m5q`, zone `eu-west-1b`
* name `inventory-b.corp:443`
* normal lookup p99 7 ms, success 99.96%
* abnormal `SERVFAIL` 100%, lookup p99 2.001 s
* TCP attempts to B fall from 218/s to zero
* `.svc` names resolve in 3 ms; public names in 11 ms
* conditional forwarder `10.40.0.53` is unreachable

# Architecture

```text
Order A -> CoreDNS 10.96.0.10
              |
              +-- cluster.local -> OK
              +-- public -> OK
              `-- inventory.corp -> forwarder 10.40.0.53 X route missing

No answer -> no B IP -> no TCP/TLS/HTTP
```

Laptop resolution through corporate DNS is not equivalent to the pod's resolver view.

# What I Check FIRST

1. **Exact resolver outcome.** WHAT: name, search suffix, nameserver, rcode, duration. WHY: NXDOMAIN, SERVFAIL, and timeout differ. LOOK FOR: SERVFAIL after 2 s.
2. **Scope by zone/name/source.** WHAT: same FQDN from A pods plus control names. WHY: isolate conditional zone. LOOK FOR: only `.corp` fails.
3. **Later-layer counts.** WHAT: resolved IP presence and TCP attempts. WHY: DNS failure must stop before TCP. LOOK FOR: zero.
4. **Resolver chain.** WHAT: CoreDNS health/cache/forward metrics and upstream. WHY: resolver can be alive while forwarding fails. LOOK FOR: upstream `.53` timeout.
5. **Recent DNS/route change.** WHAT: CoreDNS config, private-zone record, route/firewall. WHY: correlate then prove. LOOK FOR: route revision `rt-corp-17`.

# Step-by-Step Investigation

### Step 1 - Capture the exact name-resolution failure

* **What I check:** full requested name, exception, rcode, lookup duration, nameserver, attempts.
* **Why:** `UnknownHostException` can hide NXDOMAIN or resolver failure.
* **Expected result:** NOERROR with approved private IP and TTL under 10 ms.
* **Bad result:** SERVFAIL after two one-second attempts.
* **Meaning:** the resolver cannot complete the lookup; it is not claiming the name is absent.
* **Next branch:** compare other names through the same resolver.

### Step 2 - Test controls from A's context

* **What I check:** affected FQDN, `kubernetes.default.svc`, and an approved public control.
* **Why:** controls separate total resolver outage from zone forwarding.
* **Expected result:** all resolve.
* **Bad result:** only `inventory-b.corp` fails.
* **Meaning:** CoreDNS receives queries; conditional `.corp` path is defective.
* **Next branch:** inspect conditional forwarding metrics/config.

### Step 3 - Distinguish NXDOMAIN, SERVFAIL, timeout, stale answer

* **What I check:** DNS response code, authority, answer set, TTL, and elapsed time.
* **Why:** each has a different owner.
* **Expected result:** NOERROR with current answers.
* **Bad result:** NXDOMAIN -> record/view; SERVFAIL -> resolver chain; timeout -> reachability/load; stale answer -> lifecycle/cache.
* **Meaning:** this SERVFAIL selects forwarding, not B listener.
* **Next branch:** follow the configured forwarder.

### Step 4 - Inspect CoreDNS and upstream

* **What I check:** query rate, cache, errors by zone/rcode, forward duration/upstream, CPU/memory/queue.
* **Why:** healthy process metrics do not prove upstream completion.
* **Expected result:** forward response under 20 ms.
* **Bad result:** only upstream `10.40.0.53` reaches 2 s and errors.
* **Meaning:** conditional forwarder is unreachable or failing.
* **Next branch:** verify bounded route/flow/forwarder health.

### Step 5 - Confirm later stages are absent

* **What I check:** A HTTP-pool creation, TCP/TLS, gateway upstream, B accepts, B logs.
* **Why:** a true DNS failure has no selected IP.
* **Expected result:** zero failed-request samples later.
* **Bad result:** TCP attempt exists to a cached IP.
* **Meaning:** some clients use cache; scope by cache age/process.
* **Next branch:** compare JVM DNS cache and new/reused connections.

### Step 6 - Compare laptop and production correctly

* **What I check:** resolver IP, DNS view, search domains, proxy/route, and trust.
* **Why:** laptop success only proves its own context.
* **Expected result:** intended split-view answers documented.
* **Bad result:** laptop resolves through `10.1.0.53`; pod's production view lacks/fails record.
* **Meaning:** source context explains disagreement.
* **Next branch:** repair the production view/forwarder; never hard-code IP.

### Step 7 - Prove the route-triggered mechanism

* **What I check:** route table, authorized flow telemetry, change time, redundant forwarder.
* **Why:** CoreDNS timeout alone does not identify network owner.
* **Expected result:** route to `10.40.0.53` exists and secondary works.
* **Bad result:** route removed by `rt-corp-17`; no secondary configured.
* **Meaning:** one conditional upstream became unreachable.
* **Next branch:** restore approved route or designed secondary and verify every layer.

### DNS result branch matrix

| Result from A context | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| NXDOMAIN in 5 ms | A's resolver view says name absent | Other private views lack the record | Query authoritative zone/view and compare resolver contexts |
| SERVFAIL in 2 s | Resolver chain could not complete | Record is missing | Test control zones, then inspect forwarder/upstream route |
| No response/timeout | Query did not receive timely DNS response | CoreDNS process is down | Test service IP, pod health, UDP/TCP 53 path, queue, and upstream |
| NOERROR with no answer | DNS response lacks requested record type | Host does not exist | Check CNAME chain, A/AAAA type, and authoritative data |
| NOERROR with stale IP | Resolution works but target correctness is wrong | Stale IP is unreachable | Reconcile TTL/cache/registry and test selected target |
| Laptop resolves; pod NXDOMAIN | Resolver views differ | Pod view is misconfigured | Confirm intended split-horizon ownership and production record |
| `dig` succeeds; Java fails | DNS protocol query can succeed | JVM uses same cache/resolution path | Inspect JVM cache, search suffix, proxy, and application resolver |
| A record works; AAAA fails | IPv4 resolution succeeds | Client will select IPv4 | Inspect address preference, IPv6 route, and Happy Eyeballs behavior |
| Existing calls work; new fail | Existing sockets may bypass DNS | B is healthy for fresh connections | Inspect pool age, DNS cache, and new-connection outcomes |
| One node fails | Node-local resolver/cache/path differs | Shared CoreDNS is healthy | Compare node DNS config, local cache, route, and another pod |

### Resolver-chain evidence

The application asks the resolver configured in its container namespace.

CoreDNS may answer from cache, authoritative plugin, Kubernetes plugin, or a
conditional forwarder; each is a separate branch.

The `.svc` control proves the Kubernetes plugin path works for that sample.

The public control proves general forwarding works for that sample.

Only `.corp` failing isolates the conditional forwarding rule more strongly
than CoreDNS CPU or pod status.

CoreDNS `Running` and low CPU do not prove upstream `10.40.0.53` is reachable.

A resolver retry can double the observed 1 s upstream timeout into A's 2 s
lookup duration.

### Java resolution caveats

Java 17 can cache positive and negative results according to security/network
properties and resolver implementation.

A corrected authoritative record may not immediately clear a negative JVM
cache; I verify supported cache behavior rather than restart blindly.

Feign, WebClient, and RestTemplate may use different underlying resolution and
connection-pool implementations.

An existing pooled connection does not perform DNS again.

For every failed attempt I retain hostname, resolved address if any, cache
source if observable, and connection reuse.

I avoid logging full internal zone dumps or sensitive service names beyond the
incident scope.

### Command outcome branches

`getent` success plus `dig` success but application failure sends me to JVM or
client cache/config.

`dig @10.96.0.10` SERVFAIL plus `.svc` success sends me to conditional
forwarding.

`dig` NXDOMAIN with an authoritative SOA sends me to record/view ownership.

UDP lookup failure with TCP DNS success suggests fragmentation, MTU, or UDP 53
policy; I then inspect response size and network rules.

Both UDP and TCP DNS failures to CoreDNS Service IP send me to service routing,
policy, or resolver availability.

These bounded commands prove individual observations and do not measure fleet
availability; the time series and per-zone synthetics provide scope.

### Layered recovery checks

First, every A zone must receive the approved answer and TTL.

Second, TCP must connect to each advertised address.

Third, TLS identity and HTTP route must succeed.

Fourth, B must record the request and complete the reservation.

If any later layer fails, DNS recovery is real but the overall incident has a
second cause and remains open.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| DNS rcode rate | High NXDOMAIN differs from SERVFAIL; split name/zone/resolver |
| Lookup p50/p95/p99/max | High 2 s plateau shows retry/timeout; low NXDOMAIN can fail fast |
| CoreDNS request rate | High may overload; normal rate plus one-zone failure points upstream |
| Forward duration/errors | High only for `.53` identifies conditional forward path |
| Cache hit/miss | Low hit can increase upstream dependency; stale hit can return dead IP |
| A TCP attempts | Falling to zero while A requests stay flat proves stop before TCP |
| A business success | Low confirms impact; health on cached/local route may remain high |
| CPU/memory/queue | High can make resolver slow; low/flat here does not clear upstream |
| Per-zone/node | One node suggests node-local DNS; cluster-wide one zone suggests shared view |
| Retry count | High DNS retries amplify resolver load and consume deadline |
| After route change | SERVFAIL starts immediately after revision; response recovery proves mechanism |
| B metrics | Flat/normal expected because no calls arrive; not standalone proof |

# Distributed Trace Investigation

```text
traceId=08f92f3577b34da6a3ce929d0e0e0008
Order A server                     2,010ms span=i001 ERROR
  Inventory client                2,004ms span=i002
    dns.lookup                    2,001ms span=i003 ERROR SERVFAIL
  [no resolved address]
  [no TCP/TLS/gateway/B spans]
```

The failed child occurs before a target/instance can be assigned. Search trace attributes for resolver and attempt count.

Missing later spans are expected, but trace sampling/instrumentation can omit them. Confirm A socket counters, gateway rates, and B access logs. DNS spans themselves may be absent in some Java clients; client exception and resolver metrics then provide evidence.

# Distributed Logs

```text
2026-09-13T16:22:22.417Z level=ERROR service=order-service
instance=order-a-4.18.2-k2m5q version=4.18.2 zone=eu-west-1b
traceId=08f92f3577b34da6a3ce929d0e0e0008 spanId=i002 requestId=ord-01a849
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
hostname=inventory-b.corp resolver=10.96.0.10 latency_ms=2001 attempts=2
error="UnknownHostException: server failure"
```

This shows A's resolver observation, not the broken component. Correlate CoreDNS forward logs, route/flow evidence, and control-name success. Avoid logging sensitive query data or huge resolver output.

# Commands / Tools

```powershell
Resolve-DnsName inventory-b.corp -DnsOnly
Resolve-DnsName kubernetes.default.svc.cluster.local -DnsOnly
nslookup inventory-b.corp
```

`Resolve-DnsName`/`nslookup` expose one machine's configured resolver, not necessarily the JVM cache.

```bash
getent ahosts inventory-b.corp
dig +time=2 +tries=1 inventory-b.corp
dig +time=2 +tries=1 @10.96.0.10 inventory-b.corp
```

`getent` follows OS name-service configuration; `dig` queries DNS explicitly. Neither proves TCP after an answer.

```bash
kubectl exec -n shop order-a-4.18.2-k2m5q -- getent hosts inventory-b.corp
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl logs -n kube-system -l k8s-app=kube-dns --since=10m
```

Use bounded authorized logs. A debug pod may have different DNS policy. Do not restart CoreDNS before preserving zone/upstream evidence.

# Root Cause

Route revision `rt-corp-17` removed reachability from CoreDNS to sole conditional forwarder `10.40.0.53`.

```text
route removal -> CoreDNS cannot reach .corp forwarder
-> SERVFAIL after 2s -> A obtains no IP -> no TCP/TLS/HTTP
-> reservations fail
```

# Fix

**Immediate mitigation:** restore the approved route or activate the designed redundant reachable forwarder. Do not hard-code B's IP.

**Root cause correction:** configure at least two reachable conditional forwarders across failure domains and correct route ownership.

**Permanent fix:** probe each delegated zone from consumer contexts, manage views/routes as code, validate changes, and keep bounded negative caching/retries.

# Verification

Before: SERVFAIL 100%, lookup p99 2.001 s, B TCP attempts 0/s, business success 0%.

After: NOERROR with approved answers/TTL, lookup p99 8 ms from every A zone, SERVFAIL 0, TCP/TLS/API succeed, B accepts 218/s, business success 99.96% for 30 minutes.

# Prevention

* Alert by DNS zone, rcode, resolver, and forwarder latency.
* Synthetic checks run from each consumer zone and validate answer ownership, not only response.
* Runbook branches NXDOMAIN/SERVFAIL/timeout/stale answer and compares controls.
* Redundant authoritative/forwarding paths and route-change tests.
* Dashboard A requests versus resolved/TCP attempts catches pre-connect failures.
* Review JVM DNS caching and TTL behavior during failover tests.

# Interview Answer

### What I would say in an interview

I determine the resolver result before checking B. Here A got SERVFAIL after two seconds, had no resolved IP, and TCP attempts fell to zero. From the same pod, cluster and public names resolved quickly, so only the conditional corporate zone failed. CoreDNS forwarding metrics identified `10.40.0.53`, and route evidence showed a recent route removal. I restored the approved path, added a redundant forwarder and zone-specific probes, then verified correct answers and TTL under 10 ms followed by TCP, TLS, API, and business success from every A zone.

### Common interviewer traps

Do not use laptop DNS as production proof, confuse NXDOMAIN with SERVFAIL, inspect B's database before an IP exists, or hard-code an address.

### Quick memory flow

Name/rcode -> A resolver -> control names -> answer/TTL -> forwarder/authority -> no later attempts -> route/view fix -> full-stack verify.

# Interview Follow-up Questions

1. **NXDOMAIN versus SERVFAIL?** NXDOMAIN says absent in that view; SERVFAIL says resolution could not complete.
2. **Can DNS be fast and wrong?** Yes, a stale or split-view answer can return quickly.
3. **Why use getent and dig?** `getent` follows application-like OS resolution; `dig` exposes DNS protocol details.
4. **What about JVM caching?** Positive/negative caches can preserve old outcomes beyond a control query.
5. **Could existing traffic continue?** Yes, pooled connections may avoid fresh lookups temporarily.
6. **Does an IP answer prove reachability?** No; next test TCP on the selected IP/port.
7. **Why avoid application DNS retries?** They amplify resolver load and consume the deadline.
8. **What proves recovery?** Correct answer plus successful later layers and business completion.
