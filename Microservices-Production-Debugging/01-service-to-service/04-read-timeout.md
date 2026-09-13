# Problem

A read timeout means Service A established the connection, sent the HTTP request, and did not receive the required response data before its response/read deadline. The wait can be inside Service B, a dependency, a gateway queue, or response transfer.

```text
DNS OK -> TCP OK -> TLS OK -> HTTP request sent -> waiting for response X
                                                   read timeout
```

It is not a connect timeout. It may surface as a client exception directly or as a gateway 504 when the gateway's upstream response deadline expires first.

# Production Situation

At `2026-09-13T12:41:22Z`, Order A calls Inventory B route `POST /v1/inventory/reservations`.

* requestId `ord-c82ee1`, traceId `04f92f3577b34da6a3ce929d0e0e0004`
* A `order-a-4.18.2-k2m5q`, B `inventory-b-7.5-r8x2p`
* target `10.42.7.18:8080`, zone `eu-west-1b`
* scope: warehouse 17 writes; reads and other warehouses normal
* normal p99 178 ms; abnormal A read timeout 31/s at 5.000 s
* B p99 5.2 s; DB lock-wait p99 4.7 s
* B CPU 38%, heap 59%, GC max 27 ms
* DB pool active 38/40, idle 2, pending 0, acquisition p99 11 ms

# Architecture

```text
Order A --established HTTP--> Envoy --> Inventory B
                                         |
                                         v
                                  HikariCP 11ms
                                         |
                                         v
                                  PostgreSQL UPDATE
                                  X row lock 4.7s
```

Direct A -> B and A -> gateway -> B both reach B. The gateway adds a separate deadline; direct calls can return late while the gateway returns 504 earlier.

# What I Check FIRST

1. **Timeout phase.** WHAT: connect, TLS, pool, response/TTFB, total. WHY: "timeout" is ambiguous. LOOK FOR: connect 8 ms, TTFB at 5 s.
2. **Receiver evidence.** WHAT: B access log/server span for the request ID. WHY: it separates sent/received work from pre-B loss. LOOK FOR: B accepted `ord-c82ee1`.
3. **Scope and percentiles.** WHAT: route, warehouse, instance, p50/p95/p99/max. WHY: p50 can remain healthy. LOOK FOR: warehouse-17 p99 only.
4. **Longest trace child.** WHAT: queue, pool acquisition, DB/cache/external spans. WHY: it locates observed wait. LOOK FOR: DB lock wait 4.63 s.
5. **Retries/deadline.** WHAT: attempts and continued B work after A cancellation. WHY: retries can amplify locked work. LOOK FOR: late completion after A timeout.

# Step-by-Step Investigation

### Step 1 - Prove setup completed

* **What I check:** DNS, TCP, TLS, HTTP-pool acquisition and client TTFB.
* **Why:** a read timeout begins only after connection/request progress.
* **Expected result:** DNS 3 ms, connect 8 ms, TLS 14 ms, pool 2 ms.
* **Bad result:** TTFB reaches exactly 5000 ms.
* **Meaning:** A waits after sending, not while connecting.
* **Next branch:** confirm gateway and B receive the request.

### Step 2 - Count across boundaries

* **What I check:** A outbound, gateway receive/upstream, B server accepts, and completed reservations.
* **Why:** matched counts show how far work traveled.
* **Expected result:** B accepts every attempt and logs request ID.
* **Bad result:** B completion falls while accepts stay high.
* **Meaning:** work is delayed or fails inside/after B.
* **Next branch:** open B server span and child waterfall.

### Step 3 - Scope slow requests

* **What I check:** route, warehouse, operation, B instance/version/zone, payload class.
* **Why:** a data/operation-specific lock can hide in fleet averages.
* **Expected result:** all dimensions near baseline.
* **Bad result:** only warehouse 17 writes exceed 5 s.
* **Meaning:** shared network and general runtime are unlikely.
* **Next branch:** compare a warehouse-17 failure to warehouse-12 success.

### Step 4 - Inspect B saturation

* **What I check:** CPU/throttle, heap/GC, active threads, queue, rejected tasks, HTTP pool, DB pool.
* **Why:** low CPU can mean blocked workers; high queue is effect, not cause.
* **Expected result:** no saturation and low queue.
* **Bad result:** in-flight rises 28 to 173 and queue reaches 41 while CPU is 38%.
* **Meaning:** workers wait rather than compute.
* **Next branch:** find the child span or blocked thread state that owns the wait.

### Step 5 - Separate pool acquisition from query execution

* **What I check:** Hikari active/idle/pending/acquisition and DB query/lock spans.
* **Why:** a full pool, slow query, and lock need different fixes.
* **Expected result:** acquisition 11 ms and query under 30 ms.
* **Bad result:** query 4.70 s, lock child 4.63 s, with idle connections available.
* **Meaning:** the pool is not exhausted; an acquired connection waits on a DB lock.
* **Next branch:** identify blocker safely using query fingerprint and timestamp.

### Step 6 - Find the blocker

* **What I check:** read-only DB activity, blocked PID, blocker application/job, transaction age.
* **Why:** a lock-wait log alone does not identify ownership.
* **Expected result:** no long blocker.
* **Bad result:** `inventory-reconcile job-882` holds warehouse-17 rows for 4.7-6.1 s.
* **Meaning:** one large batch transaction blocks online reservations.
* **Next branch:** pause the owned job through its scheduler and let the transaction end.

### Step 7 - Keep alternate branches explicit

* **What I check:** rows examined/plan, external latency, GC pause, thread dump, response body transfer, and cancellation.
* **Why:** similar TTFB can come from scan, downstream, pause, or queue.
* **Expected result:** each is below 50 ms in this trace.
* **Bad result:** a different child dominates or no child explains a trace gap.
* **Meaning:** follow that branch; a gap may be GC or uninstrumented code.
* **Next branch:** corroborate with runtime metrics/logs rather than guessing.

### Read-timeout result branches

| Result | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| Pool acquire high; no query child | Request is waiting before DB use | The pool is too small | Compare checkout/return, holders, transaction age, and DB capacity |
| Pool acquire low; query child high | Connection was available and execution is slow | Missing index is the cause | Split lock, IO, CPU, rows examined, and plan |
| Lock child high | Query is blocked by another transaction | Blocking transaction is malicious or unnecessary | Identify blocker owner, age, SQL fingerprint, and business job |
| External API child high | B waits on that dependency | Network is responsible | Decompose dependency DNS/connect/TLS/TTFB and its server trace |
| Worker queue high before B handler | Work waits for an executor | More threads are safe | Inspect active/max, blocked states, service time, and downstream capacity |
| GC pause aligns with trace gap | JVM stopped application progress | Heap leak is the cause | Inspect allocation/heap trend, collector reason, and per-instance config |
| TTFB normal; total response high | Headers arrived and body transfer/consumer is slow | B handler is fast overall | Compare payload size, network throughput, compression, and A consumption |
| B finishes after A timeout | Cancellation is missing or late | The late operation committed | Inspect cancellation logs and reconcile business state/idempotency key |
| Only one B instance is slow | Instance state predicts the tail | Shared DB is healthy | Compare that instance's runtime/config/node and its dependency children |
| Only one warehouse is slow | Data/lock/plan scope predicts failure | Every request uses the same query plan | Compare matched warehouse traces, parameters, fingerprints, and blockers |

### HikariCP evidence interpretation

`active=38`, `idle=2`, and `pending=0` means two connections remain available
at the sample time. It does not prove every acquisition was instantaneous.

The acquisition histogram p99 of 11 ms confirms pool wait is not the
4.7-second owner for this incident.

If `active=40`, `idle=0`, `pending=186`, and acquisition p99 is 4.9 s, I stop
before the query and inspect checked-out holders and leaks.

A larger pool is unsafe without checking PostgreSQL connection capacity,
query concurrency, lock contention, and whether connections are returned.

Checkout and return counters should track over a sufficiently long window
after accounting for currently active connections and process restarts.

### Thread and cancellation evidence

A thread dump showing many workers `WAITING` in JDBC supports blocked work;
one dump is a sample, so I compare at least two bounded captures.

`BLOCKED` on a Java monitor points to in-process lock contention, not a
PostgreSQL row lock.

`RUNNABLE` does not always mean consuming CPU; native socket calls may appear
runnable, so CPU and trace children remain necessary.

When A's deadline fires, its client span should record cancellation and B
should observe the propagated signal before committing obsolete work.

If B cannot cancel safely, the operation requires an idempotency key and
post-incident reconciliation for late success.

### Exact verification branches

If lock p99 falls but B p99 remains high, I reopen the trace and inspect queue,
pool, serialization, cache, and external dependencies.

If B p99 recovers but A still times out, I inspect gateway deadlines, A pool,
response transfer, and stale retries.

If HTTP success recovers but reservation counts do not, the incident remains
open because transport recovery did not restore the business outcome.

If attempts/request remains above 1.00, retry amplification is still present
even when customer-visible errors are zero.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| Read-timeout rate | High at exactly 5 s exposes client deadline; low means tail below threshold |
| DNS/connect/TLS | Low/flat proves setup for samples; high redirects to earlier layer |
| TTFB vs total | High TTFB means server/upstream wait; normal TTFB with high total suggests transfer |
| B p50/p95/p99/max | Normal p50 plus high p99 means tail; split warehouse/operation |
| B in-flight/queue | High means service time exceeds completion capacity; does not name cause |
| CPU/GC | Low CPU plus high latency suggests waiting; high GC with trace gaps suggests pause |
| Hikari pending/acquire | High means wait before query; low 11 ms rejects pool acquisition as owner |
| Query/lock duration | High lock child names mechanism; high scan without locks suggests plan/index |
| DB query rate | Flat/high with lock waits differs from falling due to pool starvation |
| Retries | High adds blocked transactions; compare attempts/original request |
| Per-instance | One high instance suggests local runtime; all instances/data-specific suggests shared DB |
| After-deploy/job | Change just before lock spike suggests trigger, then prove transaction mechanism |

# Distributed Trace Investigation

```text
traceId=04f92f3577b34da6a3ce929d0e0e0004
Order A server                       5,008ms span=d001 ERROR
  Inventory client                  5,001ms span=d002 read_timeout
    gateway                         4,997ms span=d003
      Inventory B server            5,190ms span=d004 cancelled
        worker.queue                   52ms span=d005
        db.pool.acquire                11ms span=d006
        UPDATE inventory             4,700ms span=d007
          db.lock.wait               4,630ms span=d008
```

The child waterfall places almost all B time in a lock, not network or pool acquisition. A's span ends before B because its deadline fires; cancellation propagation must stop late work.

If the B child is missing, check sampling and B access logs. Missing does not automatically mean pre-B. If a DB child is missing despite B delay, inspect queueing, instrumentation, GC, and external calls.

# Distributed Logs

```text
2026-09-13T12:41:22.417Z level=WARN service=inventory-service
instance=inventory-b-7.5-r8x2p version=7.5 zone=eu-west-1b
traceId=04f92f3577b34da6a3ce929d0e0e0004 spanId=d007 requestId=ord-c82ee1
endpoint=POST_/v1/inventory/reservations downstream=postgresql
target=pg-inventory-2 query_fingerprint=reserve-stock latency_ms=4700
lock_wait_ms=4630 blocked_by_app=inventory-reconcile job_id=job-882
```

The log links trace to a fingerprint and blocker label. It is evidence, not proof alone: labels can be stale, clocks can differ, and a warning may report a victim rather than cause. The read-only DB wait graph and job transaction confirm ownership.

# Commands / Tools

```powershell
Test-NetConnection inventory-b -Port 8080
curl.exe -v --connect-timeout 2 --max-time 6 http://inventory-b:8080/actuator/health
```

TCP true and fast health prove only that a listener responds; the business route may still block.

```bash
curl -sS -v --connect-timeout 2 --max-time 6 -o /dev/null \
  -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' \
  http://inventory-b:8080/actuator/health
kubectl top pod -n shop -l app=inventory
kubectl logs -n shop inventory-b-7.5-r8x2p --since=10m
```

Do not repeatedly call a mutating reservation route as a probe. Use an approved synthetic/idempotency key.

```sql
SELECT pid, wait_event_type, wait_event, state, query_start
FROM pg_stat_activity
WHERE datname = 'inventory'
ORDER BY query_start;
```

This is read-only and bounded, but it does not link a row to a request without timestamp/fingerprint evidence. Spring `/actuator/threaddump` can show WAITING workers; it must be authorized and captured briefly.

# Root Cause

Reconciliation job `job-882` changed from small commits to one transaction for warehouse 17 and held stock-row locks beyond A's five-second deadline.

```text
large transaction -> row locks -> B UPDATE waits -> workers accumulate
-> A read deadline expires -> retries add waiters -> customer reservations fail
```

# Fix

**Immediate mitigation:** preserve the wait graph and trace, pause `job-882` via its scheduler, let its transaction finish, and reduce safe retry amplification.

**Root cause correction:** restore bounded batches and consistent row ordering; index the batch predicate; set a lock budget below the request deadline.

**Permanent fix:** propagate cancellation/deadlines, test online writes concurrently with reconciliation, and keep idempotency. Increasing read timeout would retain more blocked work and is not the first fix.

# Verification

Before: read timeouts 31/s, B p99 5.2 s, lock p99 4.7 s, queue 41, business success 85.8%.

After: timeouts 0/s, B p99 164 ms, lock p99 14 ms, queue 0-2, attempts/request 1.00, business success 99.97% for 30 minutes. Reconciliation confirms no duplicate or missing reservations.

# Prevention

* Alert on read-timeout plateaus, B p99, lock wait, queue, and retry amplification.
* Dashboard client phases beside B child durations and DB wait types.
* Runbook differentiates pool acquisition, query execution, lock, scan, and external wait.
* Concurrency tests run batch and online reservations on the same warehouse.
* Deploy batch changes as canaries with transaction-age and max-lock guardrails.
* Enforce request deadline propagation and cancellation.

# Interview Answer

### What I would say in an interview

I first prove connection setup succeeded and that B received the request. Here DNS, TCP, TLS, and pool acquisition were normal, but time to first byte hit A's five-second read deadline. B p99 was 5.2 seconds only for warehouse-17 writes. The trace showed a 4.70-second update containing 4.63 seconds of lock wait, while CPU and Hikari acquisition were normal. A reconciliation job held those rows in one transaction. I paused the job, restored small commits, and verified zero timeouts, lock p99 below 20 ms, normal B latency, and correct reservations.

### Common interviewer traps

Do not call it a network timeout, enlarge Hikari because active is high, or increase read timeout. Low CPU can mean blocked work. A health 200 does not exercise the locked business transaction.

### Quick memory flow

Read phase -> B received? -> scope tail -> runtime saturation -> pool versus query -> longest child -> blocker -> mitigate -> concurrency proof.

# Interview Follow-up Questions

1. **Read timeout versus 504?** Client read timeout is local; 504 is an HTTP response from a gateway.
2. **Why is low CPU useful?** It suggests workers wait instead of compute, but is not proof.
3. **Why not increase threads?** More workers create more blocked DB transactions.
4. **How do you distinguish pool exhaustion?** Pool acquisition dominates and pending rises; here acquisition is 11 ms.
5. **What if DB query rate falls?** With steady requests and high pool pending, work may be blocked before queries.
6. **Can B finish after A times out?** Yes; without cancellation it can commit late, requiring reconciliation.
7. **What would RestTemplate show?** A socket read-timeout wrapper; inspect its deepest cause and trace phase.
8. **What validates the fix?** Technical latency plus exactly-once business reconciliation.
