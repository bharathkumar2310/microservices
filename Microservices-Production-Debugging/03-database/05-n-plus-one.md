# Problem

Order-history latency grows almost linearly with page size although each SQL statement is fast.

The affected operation is `GET /customers/{id}/orders?page=0&size=100`.
The objective is to locate elapsed time before naming a cause. Database latency is a pipeline:

1. HikariCP pool acquisition.
2. DNS, TCP, TLS, and MySQL login when a physical connection is created.
3. Transaction begin and snapshot setup.
4. Lock wait before useful execution.
5. SQL execution in MySQL.
6. Result transfer, JDBC decoding, Hibernate hydration, and mapping.
7. Flush and commit, including durable log work.

A timeout reports where a deadline expired, not automatically which phase caused it.

# Production Situation

At 11:40, twenty minutes after an order-history mapper release, `GET /customers/{id}/orders?page=0&size=100` degraded.

* Application request rate: 420/min
* Normal endpoint p95: 190 ms
* Current endpoint p95: 2.8 s
* Business/error signal: 3% gateway timeouts
* CPU: MySQL 46%, application 38%
* HikariCP: active 32/40, idle 8, pending 4, usage p99 2.6 s
* Database evidence: 201 SELECT statements/request; each statement takes 3-12 ms
* Heap: 63%; maximum GC pause: 31 ms
* Service-to-service transport: normal

Actual failure shape:

```sql
-- Once: fetch 100 parent orders.
SELECT id,created_at,total FROM orders
WHERE customer_id=? ORDER BY created_at DESC LIMIT 100;
-- Repeated once for each order when mapper calls order.getItems().
SELECT id,order_id,product_id,quantity FROM order_item WHERE order_id=?;
-- Repeated for lazy products when mapper calls item.getProduct().getName().
SELECT id,name FROM product WHERE id=?;
```

The trace records 1 parent SELECT plus 100 item SELECTs and 100 product SELECTs.

I keep one incident timeline containing deploys, traffic, pool state, DB state, and mitigations. A correlated series is stronger than a snapshot.

# Architecture

```text
Client
  |
  v
API Gateway
  | traceId/requestId
  v
Spring Boot Service
  |
  +--> Controller -> Service -> @Transactional
                              |
                              v
                        Spring Data JPA
                              |
                         Hibernate/JDBC
                              |
                           HikariCP
                              |
                              v
                            MySQL 8
                              |
                    optimizer / locks / redo
```

```text
request -> acquire -> connect/login if needed -> begin -> lock wait
        -> execute -> fetch/map -> flush/commit -> return -> response
```

A green health endpoint proves only its configured checks. It does not prove this business query, transaction, data distribution, or pool path.

# What I Check FIRST

### 1. Scope and change

* WHAT: Endpoint, outcome, instance, version, tenant/SKU/parameter class, and first bad minute.
* WHY: This separates one cohort from shared saturation.
* WHAT RESULT AM I LOOKING FOR: Compare healthy and affected cohorts; do not average them together.

### 2. Trace phase

* WHAT: Healthy and slow traces split into acquire, begin, lock, execute, fetch/map, and commit.
* WHY: Endpoint duration alone cannot identify the slow phase.
* WHAT RESULT AM I LOOKING FOR: The service span contains 201 short MySQL child spans with three repeated normalized digests.

### 3. HikariCP

* WHAT: Active, idle, pending, maximum, acquisition, usage, timeouts, checkouts, and returns.
* WHY: Requests can wait before MySQL sees a statement.
* WHAT RESULT AM I LOOKING FOR: Pending plus high acquire means contention; active alone does not prove a leak.

### 4. MySQL

* WHAT: Digest work, sessions, transactions, lock waits, CPU, I/O, temp tables, and connection errors.
* WHY: These separate admission, blocking, execution, and server saturation.
* WHAT RESULT AM I LOOKING FOR: Correlate the same minute and normalized query with application evidence.

### 5. Safety and change point

* WHAT: Deployment, batch, failover, statistics, schema, pool, replica count, and traffic mix.
* WHY: Timing narrows hypotheses but does not prove them.
* WHAT RESULT AM I LOOKING FOR: Read-only evidence first; risky actions require ownership, approval, and rollback.

# Step-by-Step Investigation

### Step 1 - Confirm impact

* What I check: RED and business metrics for GET /customers/{id}/orders?page=0&size=100
* Why I check it: I need severity, start time, and affected cohorts before changing production
* Example command/query/tool: `Grafana with deployment annotations`
* Expected result: rate near 420/min and p95 near 190 ms
* Bad result: p95 2.8 s and 3% gateway timeouts
* What the bad result means: the incident is customer-facing but the database phase is not yet known
* What I check next: split by instance, version, tenant, parameter, and outcome

### Step 2 - Locate elapsed time

* What I check: healthy and failed distributed traces
* Why I check it: parent/child duration shows where time accumulates
* Example command/query/tool: `Tempo or Jaeger comparison`
* Expected result: The service span contains 201 short MySQL child spans with three repeated normalized digests.
* Bad result: a long parent gap or missing child
* What the bad result means: pool, queue, local code, or uninstrumented work may precede SQL
* What I check next: correlate traceId with logs and pool timers

### Step 3 - Measure Hikari

* What I check: active, idle, pending, acquire, usage, timeout, checkout, and return
* Why I check it: pool wait happens before execution and can hide DB throughput
* Example command/query/tool: `/actuator/prometheus`
* Expected result: idle capacity or low acquire with pending zero
* Bad result: active=max, idle=0, pending rising, or lifecycle divergence
* What the bad result means: capacity is unavailable; usage suggests slow holders while divergence suggests a leak
* What I check next: inspect transaction/holder traces and creation errors

### Step 4 - Separate connection creation

* What I check: DNS, TCP 3306, TLS, login, and Hikari creation logs
* Why I check it: new physical connection failures differ from pool checkout waits
* Example command/query/tool: `Test-NetConnection mysql.prod -Port 3306`
* Expected result: TCP succeeds and no TLS/login errors occur
* Bad result: NXDOMAIN, refusal, timeout, certificate error, access denied, or creation timeout
* What the bad result means: failure occurs before transaction and SQL
* What I check next: investigate only the failing network/authentication stage

### Step 5 - Inspect transactions and locks

* What I check: old transactions, waiters, blockers, isolation, and owner
* Why I check it: open transactions can retain locks and snapshots with low CPU
* Example command/query/tool: `performance_schema.data_lock_waits`
* Expected result: no old blocker chain for the affected table
* Bad result: waiters point to a blocker or error 1213 identifies a deadlock victim
* What the bad result means: contention rather than execution may consume elapsed time
* What I check next: map server thread to application and follow the incident runbook

### Step 6 - Measure statement workload

* What I check: digest count/time, rows examined/sent, temp tables, sort rows, and errors
* Why I check it: digest aggregation links endpoint demand to DB work without literals
* Example command/query/tool: `events_statements_summary_by_digest`
* Expected result: 201 SELECT statements/request; each statement takes 3-12 ms
* Bad result: work or latency changes sharply, or execution count falls despite demand
* What the bad result means: execution is costly or upstream admission is blocked
* What I check next: capture a safe plan and compare parameter classes

### Step 7 - Read EXPLAIN

* What I check: access type, key, estimated rows, join order, and Extra
* Why I check it: the plan describes how MySQL intends to access and combine rows
* Example command/query/tool: `EXPLAIN FORMAT=TREE SELECT id,created_at,total FROM orders WHERE customer_id=772 ORDER BY created_at DESC LIMIT 100`
* Expected result: selective ref/range access and bounded row flow
* Bad result: ALL, huge rows, poor order, filesort, or temp table for this workload
* What the bad result means: query shape, index, statistics, or skew may matter, but one label is not proof
* What I check next: compare estimates with actual evidence

### Step 8 - Compare actual work safely

* What I check: actual/estimated rows and time on representative common and rare values
* Why I check it: large errors reveal skew, correlation, stale stats, or parameter sensitivity
* Example command/query/tool: `EXPLAIN ANALYZE on an approved bounded replica read`
* Expected result: bounded execution confirms expected row flow
* Bad result: analysis itself reads huge data, locks, writes, or adds production load
* What the bad result means: the diagnostic is unsafe or value-sensitive
* What I check next: stop and use a replica or sanitized production-like dataset

### Step 9 - Measure fetch, mapping, and commit

* What I check: bytes, rows, Hibernate entities, query count, flush SQL, and commit time
* Why I check it: server execution can be followed by slow transfer, hydration, dirty checking, or redo
* Example command/query/tool: `trace plus controlled Hibernate statistics`
* Expected result: bounded result and quick flush/commit
* Bad result: large results, N+1, mapping CPU, unexpected flush, or slow commit
* What the bad result means: the server execution phase is not the only cost
* What I check next: fix fetch/page/transaction scope or inspect redo/storage

### Step 10 - Test the hypothesis

* What I check: one reversible targeted mitigation with before/after signals
* Why I check it: causal response is stronger than timing coincidence
* Example command/query/tool: `canary, rollback, or feature flag`
* Expected result: only expected causal metrics and business outcome improve
* Bad result: no improvement or unrelated signals move first
* What the bad result means: the hypothesis is incomplete
* What I check next: revert the experiment and return to phase evidence

### Scenario-specific interpretation

* N plus one is excessive round trips, not one slow statement; query count scales with parent or child count.
* JOIN FETCH can multiply rows and break collection pagination; DTO projection or parent-ID paging can be safer.
* Measure query count, repeated digests, rows, transfer bytes, Hibernate hydration, and serialization separately.
* Large pages amplify SQL calls, transfer, persistence-context memory, dirty checking, and JSON mapping.

Never run KILL, force failover, lower isolation, or create/drop an index from a checklist alone. Capture owner, evidence, impact, rollback, and approval.

# Metrics to Check

| Metric | If HIGH | If LOW | If it changes after deployment |
|---|---|---|---|
| HTTP request rate | demand or retries | traffic loss or upstream blocking | routing/client behavior changed |
| Business failure/error rate | customer impact; classify the error | healthy only if volume is stable | new path may be involved |
| p50/p95/p99/max | broad or tail latency | healthy only with stable success | compare old/new versions |
| Hikari active | many connections checked out | capacity or low demand | holders/config changed |
| Hikari idle | spare capacity when high | no spare when active=max | pool may be starved |
| Hikari pending | threads wait before SQL | no acquire queue | leak, slow holder, or budget issue |
| Hikari acquisition | checkout delay | pool is not bottleneck | inspect pending/creation |
| Hikari usage | long hold duration | short transactions | SQL, locks, remote work, or mapping |
| Checkout minus return | possible leak if persistent | balanced lifecycle | new error path |
| DB query rate | more work/retries | low demand or blocked admission | compare app rate |
| Statement latency | execution/lock/fetch cost | executed SQL is fast | query/data/plan changed |
| Rows examined/sent | inefficient access when ratio is high | selective work | plan or skew changed |
| DB CPU | compute pressure | idle, blocked, or no work | link to digests |
| DB I/O latency | reads, spill, or flush pressure | cached or blocked work | plan/temp/redo changed |
| Lock wait time | blocking | no observed blockers | transaction scope/order changed |
| Deadlocks | cyclic acquisition | no cycle observed | order/concurrency changed |
| Disk temp tables | sort/join spill | fits or avoids temp work | row width/plan changed |

Connection-budget example:

```text
12 replicas * maximumPoolSize 40 = 480
+ batch/report connections         = 35
+ admin/failover reserve            = 25
planned total                      = 540
safe application budget            = 500
```

This configuration is unsafe before a traffic spike. Coordinate pool size and autoscaling rather than silently exceeding MySQL capacity.

# Distributed Trace Investigation

Observed evidence: The service span contains 201 short MySQL child spans with three repeated normalized digests.

```text
traceId=n1-56e8
Gateway                         17 ms
  | parent spanId=gw-71
  v
GET /customers/{id}/orders?page=0&size=100 service      2.8 s
  | spanId=history-35
  +-- hikari.acquire                 42 ms
  +-- hikari.connection.usage     2,603 ms
      +-- orders-parent SELECT       11 ms
      +-- order-items SELECT x100   890 ms total
      +-- product-by-id SELECT x100 970 ms total
      +-- hibernate.map/json        720 ms
```

* traceId joins the request across gateway, service, and DB instrumentation.
* spanId identifies one operation; parentSpanId shows nesting and ownership.
* Client latency includes network and queueing before the server span.
* Server latency includes local queueing, code, DB work, and serialization.
* A DB span starting late suggests acquire, queue, or local work before execution.
* A long DB span can include lock, execute, or fetch depending on instrumentation; server evidence separates them.
* Repeated identical child spans indicate N+1 or retry behavior.
* A long commit span suggests flush SQL, redo durability, or storage rather than SELECT planning.
* A missing child span can mean pre-span failure, sampling, lost context, or missing instrumentation; it is not proof of no DB intent.
* Compare success and failure from the same instance, version, and parameter class.
* Correlate timestamps with metric scrape intervals and clock skew.

# Distributed Logs

```text
2026-09-13T11:40:09.611Z WARN service=order-history-service instance=history-3d6a endpoint="/customers/772/orders"
traceId=n1-56e8 spanId=history-35 requestId=history-77 version=2026.09.13 pageSize=100
pool=HikariPool-history active=32 idle=8 pending=4 acquireMs=42 usageMs=2603
2026-09-13T11:40:12.359Z INFO service=order-history-service instance=history-3d6a
traceId=n1-56e8 spanId=hibernate-91 statement=history-map queryCount=201 uniqueDigests=3 rowsMapped=486 durationMs=2790
queryDigestCounts="orders-parent:1,order-items-by-order:100,product-by-id:100"
```

Log UTC timestamp, level, service, instance, version, traceId, spanId, requestId, endpoint, normalized query ID, phase duration, row count, SQLState, vendor code, and pool state.

1. Start with a failed business request and its traceId.
2. Order all matching logs by timestamp.
3. Confirm instance and version.
4. Preserve the exception cause chain and SQLState.
5. Compare one successful trace and same-minute metrics.
6. Search by normalized digest, never sensitive bind values.

One log is not root-cause proof. A timeout tells where the caller stopped waiting; pool queueing, a blocker, result mapping, commit, or retry amplification can be underneath it.

# Commands / Tools

Use read-only commands first through approved production access.

```powershell
Resolve-DnsName mysql.prod
Test-NetConnection mysql.prod -Port 3306
curl.exe --max-time 5 http://localhost:8080/actuator/health
curl.exe --max-time 5 http://localhost:8080/actuator/prometheus
```

```bash
getent hosts mysql.prod
nc -vz -w 3 mysql.prod 3306
curl --max-time 5 http://localhost:8080/actuator/health
curl --max-time 5 http://localhost:8080/actuator/prometheus
```

DNS proves name resolution at that moment. TCP success proves reachability to that host/port from that location. Neither proves TLS, login, authorization, transactions, or SQL performance.

```sql
SELECT NOW(), @@transaction_isolation, @@max_connections;
SELECT * FROM performance_schema.data_lock_waits;
SELECT THREAD_ID, PROCESSLIST_ID, PROCESSLIST_USER, PROCESSLIST_HOST,
       PROCESSLIST_DB, PROCESSLIST_COMMAND, PROCESSLIST_TIME, PROCESSLIST_STATE
FROM performance_schema.threads
WHERE TYPE='FOREGROUND' AND PROCESSLIST_ID IS NOT NULL;
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT, SUM_ROWS_EXAMINED,
       SUM_ROWS_SENT, SUM_CREATED_TMP_DISK_TABLES, SUM_SORT_ROWS
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME='orders_db'
ORDER BY SUM_TIMER_WAIT DESC LIMIT 20;
EXPLAIN FORMAT=TREE SELECT id,created_at,total FROM orders
WHERE customer_id=772 ORDER BY created_at DESC LIMIT 100;
SHOW ENGINE INNODB STATUS;
```

These are observations within retention and configuration limits. Performance Schema consumers can be disabled, and latest-deadlock evidence can be overwritten.

EXPLAIN is the safer first plan check for SELECT. EXPLAIN ANALYZE executes the statement. Never use it for a write or unbounded expensive production read.

Do not issue KILL, change globals, run ANALYZE TABLE, or create/drop indexes without owner confirmation, approval, impact analysis, and rollback.

# Root Cause

A mapper touched lazy order.items and each item.product while Open Session in View kept the persistence context open, producing one parent query plus 200 child queries.

Causal chain:

```text
new mapper traverses two lazy relationships
  -> one request executes 1 + 100 + 100 SELECTs
  -> 3-12 ms round trips accumulate to almost 2 seconds
  -> Hibernate hydration and JSON mapping add 720 ms
  -> connection usage p99 reaches 2.6 s
  -> active reaches 32 and pending briefly reaches 4, without exhaustion
  -> endpoint p95 reaches 2.8 s and gateway timeouts begin
```

Validate every arrow with time-aligned deployment, phase span, pool lifecycle, digest/lock evidence, queue, retry, and business signals. An unsupported arrow remains a hypothesis.

# Fix

### Immediate mitigation

* Cap the page size and switch the endpoint to an existing bounded summary projection under a feature flag.
* Bound retries with an overall deadline, exponential backoff, jitter, and a low attempt cap.
* Preserve traces, pool metrics, plans, locks, and deadlock evidence before it disappears.
* Prefer draining a proven bad instance to abruptly terminating fleet work.
* Do not blindly increase timeout, pool size, CPU, or memory; this can move saturation.

### Permanent fix

* Use an EntityGraph or DTO projection for the exact read model and two-step ID pagination; use batch fetching where a collection join would multiply rows.
* For one bounded to-many relationship, test a JOIN FETCH or `@EntityGraph(attributePaths={"items"})` and inspect duplicate parent rows.
* For this two-level graph, page stable parent IDs first, then fetch a DTO for those IDs so database pagination remains correct.
* If changing the query is not immediate, use a measured batch size such as `hibernate.default_batch_fetch_size=32`; verify the resulting `IN` queries and memory.
* Never join-fetch multiple large collections blindly because Cartesian multiplication can be worse than N plus one.
* Keep transactions explicit and short; move remote work and response serialization outside them.
* Align Hikari maximum and replica autoscaling with the fleet-wide MySQL budget.
* Treat isolation changes as correctness changes requiring business tests.
* Treat indexes as reviewed schema changes with read benefit and write/storage/DDL cost.
* Bound page and result size; prefer stable keyset pagination for deep pages.
* Test validation, exception, cancellation, timeout, retry, and rollback paths.

# Verification

Use the same endpoint, tenant/parameter mix, and traffic shape so before and after are comparable.

| Signal | Before | After target | Meaning |
|---|---:|---:|---|
| Endpoint p95 | 2.8 s | near 190 ms | customer latency recovered |
| Business/error signal | 3% gateway timeouts | below 1% and baseline | business success recovered |
| Queries/request | 201 | at most 3-4 | N plus one removed |
| Repeated child digests | 100 + 100 | 0 or bounded batches | round trips no longer scale per row |
| Hikari usage p99 | 2.6 s | below 250 ms | connection hold time recovered |
| Hikari pending | 4 | 0 | secondary contention cleared |
| Active/idle | 32/8 | about 16/24 at same rate | fewer long holders |
| Endpoint DB traffic | about 1,407 statements/s | below 30 statements/s | endpoint query amplification removed |
| Rows mapped | 486 with 201 calls | same business result, bounded transfer | no data was lost to gain speed |

Canary first, then observe peak windows. Check no regression in write p95, redo, CPU, total connections, correctness, and result completeness. Exercise failure paths, not only the happy path.

# Prevention

* Alert on business success, endpoint p95/p99, and classified errors, not health status alone.
* Alert when Hikari pending persists and acquisition consumes a meaningful request-deadline fraction.
* Combine active=max and idle=0 with pending and usage; one gauge is not a conclusion.
* Record checkout/return lifecycle counters or equivalent evidence and bounded leak diagnostics.
* Plot application request rate beside DB query rate so throughput collapse is not called recovery.
* Dashboard digest count/latency, rows examined/sent, lock waits, deadlocks, temp spills, CPU, I/O, and connection errors.
* Propagate traceId/spanId in MDC and instrument acquisition, SQL, fetch, and commit where supported.
* Log normalized query names and SQLState without credentials or sensitive bind values.
* Integration-test every error path and assert resources return.
* Concurrency-test deterministic lock ordering and idempotent retries.
* Load-test realistic cardinality, skew, hot keys, page sizes, and common/rare parameters.
* Review plans and actual evidence in a safe environment before schema releases.
* Gate indexes, statistics, pool, and isolation changes with approval and rollback.
* Align request, statement, lock, transaction, and retry deadlines.
* Use readiness for admission and synthetic business checks for critical operations.

# Interview Answer

### What I would say in an interview

For hibernate n plus one queries, I first confirm business impact and scope by endpoint, instance, version, tenant, and parameter. Then I use a trace to separate Hikari acquisition, physical connect/login, transaction begin, lock wait, SQL execution, result mapping, and commit. I correlate traceId with logs and overlay app rate, pool active/idle/pending/acquire, DB query rate, statement digests, rows examined, locks, CPU, and I/O. Here, A mapper touched lazy order.items and each item.product while Open Session in View kept the persistence context open, producing one parent query plus 200 child queries. I use the smallest reversible approved mitigation, implement the permanent fix, and verify business success plus causal metrics at peak load.

### Common interviewer traps

* Calling every database timeout a slow query.
* Treating active connections as a leak without lifecycle or hold-time evidence.
* Calling falling query rate healthy while application demand is constant.
* Treating one log or EXPLAIN label as root-cause proof.
* Recommending KILL, lower isolation, a larger pool, or a new index without safety evidence.
* Ignoring transfer, Hibernate mapping, flush, and commit.
* Verifying only p50 or a shallow health endpoint.

### Quick memory flow

```text
Symptom -> scope/change -> RED/business -> trace phase
-> pool/connect -> transaction/locks -> plan/execute -> fetch/map/commit
-> logs -> causal chain -> safe mitigation -> permanent fix
-> before/after -> alerts and tests
```

# Interview Follow-up Questions

### 1. How do you prove N plus one?

One request trace shows a parent query followed by repeated child digests whose count scales with entities.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 2. Why is every query fast but the API slow?

Driver round-trip and mapping costs repeat hundreds of times and accumulate.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 3. Why not always JOIN FETCH?

Multiple collections can multiply rows, consume memory, and make pagination incorrect.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 4. What is safe pagination?

Page parent IDs with stable ordering, then fetch the required graph, or return a purpose-built DTO.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 5. Should Open Session in View stay enabled?

For APIs I prefer explicit transactions and fetch plans; changing it requires tests for hidden lazy access.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 6. What verifies the fix?

Query count stays bounded as page size changes and endpoint p99, hold time, rows, and bytes fall.

I would still require same-window trace, metric, and server evidence before declaring root cause.
