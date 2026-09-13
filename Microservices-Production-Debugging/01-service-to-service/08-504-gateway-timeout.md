# Problem

HTTP 504 Gateway Timeout means a gateway or proxy generated an HTTP response because an upstream operation did not finish within its deadline. It proves the downstream caller reached an HTTP intermediary. It does not prove the upstream was down, the network was slow, or the gateway itself was faulty.

```text
A -> gateway: DNS/TCP/TLS/HTTP succeeds
gateway -> B: connect succeeds, request sent, response exceeds 3s
gateway -> A: 504
B may continue after the gateway cancels
```

# Production Situation

At `2026-09-13T20:07:22Z`, Order A receives 504 on Inventory reservations.

* route `POST /v1/inventory/reservations`
* requestId `ord-35d218`, traceId `0cf92f3577b34da6a3ce929d0e0e000c`
* A `order-a-4.18.2-k2m5q`, gateway `gw-4`
* B `inventory-b-7.6-r8x2p`, target `10.42.7.18:8080`, zone `eu-west-1b`
* scope: SKU family 88 writes
* normal p99 180 ms; abnormal 504 27/s exactly at 3.000 s
* Envoy flag `UT` means upstream request timeout in this deployment
* gateway upstream connect 8 ms; B continues to 4.92 s
* DB child 4.71 s, 8.2 million rows examined, lock wait 3 ms

# Architecture

```text
Order A -> Envoy (3s upstream deadline) -> Inventory B
                 |                          |
                 | 504 UT at 3s             v
                 +-------------------  PostgreSQL
                                            X sequential scan 4.71s
```

A direct call to B returns 200 after 4.9 s. That proves late completion, not acceptable health.

# What I Check FIRST

1. **504 generator and subreason.** WHAT: response headers/flag/gateway log. WHY: B or another proxy could generate it. LOOK FOR: Envoy `UT`.
2. **Timeout value and phase.** WHAT: exact 3 s plateau, connect versus upstream response. WHY: it identifies whose budget fired. LOOK FOR: connect 8 ms, response exceeds 3 s.
3. **Did B receive and continue?** WHAT: B access/span and cancellation. WHY: separates pre-B failure from slow work. LOOK FOR: B span 4.92 s.
4. **Longest B child.** WHAT: queue, pool, query, lock, downstream. WHY: 504 is only an outer symptom. LOOK FOR: DB scan 4.71 s.
5. **Scope/change.** WHAT: SKU/data, query fingerprint, migration. WHY: one missing index can affect a subset. LOOK FOR: migration omitted index.

# Step-by-Step Investigation

### Step 1 - Identify the HTTP responder

* **What I check:** status, `server`, gateway ID, response flag, request ID, elapsed time.
* **Why:** a 504 must be interpreted at its generator.
* **Expected result:** gateway forwards B's 201 under 200 ms.
* **Bad result:** Envoy emits 504 `UT` at 3.000 s.
* **Meaning:** this gateway's upstream request deadline expired.
* **Next branch:** inspect gateway upstream phases and target.

### Step 2 - Clear A-to-gateway transport

* **What I check:** A DNS/TCP/TLS/pool and gateway downstream receive.
* **Why:** 504 already suggests HTTP reached a gateway, but metrics confirm hop.
* **Expected result:** under 20 ms and matched counts.
* **Bad result:** none; A-to-gateway is normal.
* **Meaning:** move to gateway-to-B rather than A networking.
* **Next branch:** compare upstream connect and response time.

### Step 3 - Separate connect from upstream response

* **What I check:** target selection, connect duration/error, TLS, first-byte, total.
* **Why:** mappings are product-specific: Envoy normally uses 503 `UF` for upstream connection failure and 504 `UT` when its upstream request deadline expires; NGINX may use 502 for a connect failure.
* **Expected result:** connect 8 ms and response under 180 ms.
* **Bad result:** connect 8 ms, upstream response beyond 3 s.
* **Meaning:** B or its dependency is slow after accepting.
* **Next branch:** confirm B request ID/server span.

### Step 4 - Compare direct and gateway behavior

* **What I check:** safe equivalent request, target, identity, payload class, and deadline.
* **Why:** direct success after 4.9 s shows bypassed timeout, not correctness.
* **Expected result:** both finish under SLO.
* **Bad result:** direct returns 200 at 4.9 s; gateway 504 at 3 s.
* **Meaning:** gateway exposes a real slow B operation.
* **Next branch:** inspect B waterfall, not increase gateway timeout.

### Step 5 - Inspect B runtime and queues

* **What I check:** p50/p95/p99/max, in-flight, worker queue, threads, CPU/throttle, heap/GC.
* **Why:** resource saturation may own or amplify latency.
* **Expected result:** p99 under 180 ms, queue near zero.
* **Bad result:** SKU-88 p99 4.92 s, queue 18, CPU 46%, GC max 31 ms.
* **Meaning:** queue is secondary; CPU/GC do not explain 4.7 s.
* **Next branch:** inspect pool and dependency children.

### Step 6 - Separate DB pool, lock, and scan

* **What I check:** Hikari acquisition, query fingerprint, rows examined, plan, lock wait.
* **Why:** each requires a different correction.
* **Expected result:** acquire under 15 ms, index scan under 30 ms.
* **Bad result:** acquire 14 ms, lock 3 ms, sequential scan 8.2M rows in 4.71 s.
* **Meaning:** missing/unused index makes execution slow.
* **Next branch:** compare reviewed production plan and migration state.

### Step 7 - Inspect cancellation and retries

* **What I check:** B work after gateway 504, DB query cancellation, attempts/request.
* **Why:** abandoned work and retries amplify load.
* **Expected result:** deadline cancels B/DB and attempts stay 1.00.
* **Bad result:** B completes at 4.92 s and attempts reach 1.18.
* **Meaning:** customers receive failure while obsolete work continues.
* **Next branch:** contain retries and use compatible query path while fixing plan.

### 504 result branches

| Result | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| Upstream connect consumes deadline | Gateway cannot establish B socket in time | B application is slow | Inspect refusal/timeout, retransmits, flow, and listener |
| Connect/TLS low; B span high | Request reached B and B owns observed delay | Database is the cause | Open B queue, pool, DB, cache, and external children |
| B queue high before handler | Executor admission delays work | More threads fix it | Inspect active/max/rejected, blocked state, and dependency capacity |
| DB pool acquire high; no query | Work waits for a connection | Pool size is too small | Compare checkout/return, holders, leaks, and DB limits |
| Query high; rows examined high | DB execution scans substantial data | Index addition alone is safe | Inspect plan, selectivity, lock/IO, and online migration risk |
| Query high; lock child high | Another transaction blocks it | The query plan is healthy | Identify blocker owner, age, scope, and transaction design |
| External child high | B waits on another service | Gateway network is slow | Decompose that child's DNS/TCP/TLS/HTTP and server trace |
| B finishes after 504 | Cancellation did not stop all work | The business mutation committed | Check DB cancellation and reconcile idempotency key |
| B span absent but access log exists | Tracing is incomplete | B did not process the request | Inspect sampling/export and server logs/metrics |
| Direct call returns after 4.9 s | B can eventually complete without gateway deadline | Direct path meets SLO | Locate the 4.9 s owner and optimize it |

### Deadline hierarchy

The end-to-end order deadline must be longer than A's gateway timeout, which
must be longer than B's internal dependency budgets plus cleanup margin.

In this incident Envoy stops at 3 s while B and JDBC continue toward 4.92 s.

If A also has a 2.8 s response timeout, A may report its own read timeout
before receiving Envoy's 504, changing the visible symptom.

If the DB statement timeout is 10 s, it cannot protect a 3 s gateway budget.

The corrected design propagates remaining time and configures the DB statement
to stop before the outer deadline.

A retry cannot fit after a 3 s first attempt inside a 3.5 s order budget.

### Query-plan evidence

The trace selects fingerprint `9ac2`; I do not run broad slow-query searches.

The plan's sequential scan and 8.2 million examined rows explain 4.71 s.

Low lock wait rejects the lock branch for this sample.

Low pool acquisition rejects connection starvation for this sample.

An index existing in schema metadata does not prove the optimizer uses it; I
verify the post-fix plan and rows examined.

An online index deployment still needs DB capacity, lock, replica-lag, and
rollback review before production execution.

### Cancellation and business-state checks

When Envoy cancels, B should stop optional work and propagate cancellation to
JDBC where transaction safety permits.

If a write already committed, cancellation cannot undo business state.

The idempotency key lets a later client retry return the existing result
instead of creating a second reservation.

I query reconciliation by request/idempotency ID, not by unbounded table scan.

Late successes, duplicates, and orphan reservations are counted before closure.

If cancellation reduces DB work but query latency stays 4.7 s for completed
requests, the index defect remains and must still be corrected.

### Layered recovery expectations

After index creation, query p99 should fall first.

B p99 and in-flight should then fall as old scans drain.

Gateway 504 should reach zero once responses fit below 3 s.

Attempts/request should return to 1.00 after retry pressure clears.

Business success and reconciliation must recover last; that sequence makes the
causal prediction testable.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| 504 by generator/flag | High `UT` identifies Envoy upstream timeout; status alone is incomplete |
| 504 duration | Flat 3.000 s reveals configured budget; variable short errors suggest another branch |
| Upstream connect | Low 8 ms clears connect; high/refusal would move to network/listener |
| Upstream response | High beyond deadline points inside/after B |
| B accepted/completed | Accept high, completion late/low means in-flight slow work |
| B p50/p95/p99/max | Normal p50 and high p99 means tail/data scope; split SKU |
| Queue/in-flight | High is accumulation from slow service time; inspect child owner |
| CPU/GC | Moderate/flat rejects primary compute/pause; low CPU can still mean waiting |
| Pool acquire | Low 14 ms rejects pool exhaustion; high pending would precede query |
| Query duration/rows | High scan and rows examined point to plan/index |
| Lock wait | Low rejects blocking; high would select blocker investigation |
| Retry/cancelled work | High means amplification and wasted work after deadline |
| After migration | Sudden scan/504 change suggests migration; plan evidence proves it |

# Distributed Trace Investigation

```text
traceId=0cf92f3577b34da6a3ce929d0e0e000c
Order A server                       3,020ms span=h001 ERROR
  Envoy gateway                     3,000ms span=h002 status=504 flag=UT
    upstream.connect                    8ms span=h003
    Inventory B server              4,920ms span=h004 cancelled
      db.pool.acquire                  14ms span=h005
      SELECT stock                  4,710ms span=h006 fingerprint=9ac2
```

The gateway parent ends before its B child because the downstream deadline fires. Trace tooling may display late children differently. Inspect cancellation attributes and clock skew.

Missing B child could mean pre-B timeout, missing propagation, unsampled B trace, or exporter loss. Confirm gateway upstream request count and B access logs before concluding.

# Distributed Logs

```text
2026-09-13T20:07:25.417Z level=WARN service=envoy-gateway
instance=gw-4 version=1.31.1 zone=eu-west-1b
traceId=0cf92f3577b34da6a3ce929d0e0e000c spanId=h002 requestId=ord-35d218
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
target=10.42.7.18:8080 response_code=504 response_flags=UT
upstream_connect_ms=8 upstream_timeout_ms=3000 latency_ms=3000
```

```text
2026-09-13T20:07:27.329Z level=WARN service=inventory-service
instance=inventory-b-7.6-r8x2p traceId=0cf92f3577b34da6a3ce929d0e0e000c
spanId=h006 requestId=ord-35d218 query_fingerprint=9ac2
latency_ms=4710 rows_examined=8204331 cancelled=false
```

Together they show timing and linkage, not causation alone. The plan, migration state, and reversal after index deployment prove the mechanism.

# Commands / Tools

```powershell
curl.exe -v --connect-timeout 2 --max-time 4 https://gateway.internal/v1/inventory/health
Test-NetConnection 10.42.7.18 -Port 8080
```

Gateway health may not execute the slow query. Never repeatedly invoke mutating production routes.

```bash
curl -sS -v --connect-timeout 2 --max-time 6 -o /dev/null \
  -w 'code=%{http_code} connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' \
  http://10.42.7.18:8080/actuator/health
kubectl top pod -n shop inventory-b-7.6-r8x2p
kubectl logs -n shop inventory-b-7.6-r8x2p --since=10m
```

Direct curl shows one path/sample and may bypass authentication, gateway headers, or data shape.

```sql
EXPLAIN SELECT available
FROM stock
WHERE sku_id = :sku AND warehouse_id = :warehouse;
```

Use authorized read-only `EXPLAIN`, never `EXPLAIN ANALYZE` on a costly production query without approval. A plan estimate is not actual latency but can show a sequential scan.

# Root Cause

Migration `2026.09.13` omitted index `idx_stock_sku_warehouse`; fingerprint `9ac2` scanned 8.2 million rows. Missing cancellation allowed DB work to continue after Envoy's three-second deadline.

```text
missing index -> 4.71s scan -> B exceeds gateway budget
-> Envoy returns 504 -> B continues obsolete work -> retry adds another scan
```

# Fix

**Immediate mitigation:** route affected SKU operations to the prior compatible query path, reduce batch load and unsafe retries, and preserve the plan.

**Root cause correction:** deploy the reviewed online index and confirm optimizer selection.

**Permanent fix:** plan regression tests, migration verification, deadline/cancellation propagation to JDBC, and idempotent bounded retry. Increase timeout only if the business SLO and measured legitimate work require it after optimization.

# Verification

Before: 504 27/s, gateway 3.000 s, B p99 4.92 s, query p99 4.71 s, 8.2M rows, attempts/request 1.18.

After: 504 0/s, gateway p99 166 ms, B p99 151 ms, query p99 23 ms, rows examined 1-4, attempts/request 1.00, business success 99.97% for 30 minutes. Cancellation test stops an intentionally delayed query and reconciliation finds no duplicate reservations.

# Prevention

* Alert on 504 by gateway flag, deadline plateau, and route.
* Dashboard upstream connect beside response, B child, query plan/rows, retries, and late work.
* Verify required indexes after every migration and canary representative data.
* Propagate deadlines through Gateway, WebClient/Feign, Spring MVC, JDBC, and DB statement timeout.
* Runbook records generator-specific mappings: NGINX 502 plus its error log; Envoy 503 `UF`/`UH`; Envoy 504 `UT`; and direct slow success.
* Load tests include worst-cardinality SKU families and cancellation.

# Interview Answer

### What I would say in an interview

I first identify the 504 generator and whose budget expired. Envoy returned `UT` exactly at three seconds. Its connection to B took 8 ms, B received the request, and a direct call returned only after 4.9 seconds. The B trace spent 4.71 seconds in one query; pool acquisition and lock wait were normal, while the plan scanned 8.2 million rows because a migration omitted an index. I used the prior query path to mitigate, deployed the reviewed index, added cancellation, and verified zero 504s, query p99 23 ms, one attempt per order, and correct business results.

### Common interviewer traps

Do not blame the gateway merely because it generated 504, increase timeout first, or accept a late direct 200 as healthy. Always check continued work after cancellation.

### Quick memory flow

504 -> generator/deadline -> upstream connect versus response -> B received -> longest child -> plan/lock/pool -> cancellation -> targeted fix -> business verify.

# Interview Follow-up Questions

1. **Can a network issue produce 504?** A gateway may time out on upstream operations, but its phase/subreason decides.
2. **Why not 502?** TCP/TLS succeeded and the gateway waited for a response rather than failing exchange setup.
3. **Why did B continue?** Cancellation was not propagated to the DB statement.
4. **What if pool pending were 186?** Investigate acquisition/leak/slow holders before query plan.
5. **What if lock wait were 4.7 s?** Identify blocker and transaction design instead of index first.
6. **Can Spring Cloud Gateway set this timeout?** Yes; inspect route and HTTP-client response timeout plus outer deadlines.
7. **When is increasing timeout valid?** Only for legitimate measured work inside business SLO and capacity, after fixing pathological delay.
8. **Why verify rows examined?** It confirms the mechanism reversed, not only symptom masking.
