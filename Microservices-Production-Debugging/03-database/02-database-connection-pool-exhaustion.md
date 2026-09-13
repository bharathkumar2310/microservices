# Problem

Checkout requests time out while MySQL CPU and executed statement latency look healthy.

The affected operation is `POST /checkout/validate`.
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

At 10:31, twelve minutes after checkout release 2026.09.13, `POST /checkout/validate` degraded.

* Application request rate: 2,000/min
* Normal endpoint p95: 310 ms
* Current endpoint p95: 5.3 s
* Business/error signal: 18% acquisition timeouts
* CPU: MySQL 29%, application 31%
* HikariCP: active 40/40, idle 0, pending 186, acquire p99 4.9 s
* Database evidence: query rate fell from 1,700/s to 620/s; executed SELECT p95 stayed 31 ms
* Heap: 56%; maximum GC pause: 24 ms
* Service-to-service transport: normal

Representative SQL shape: `SELECT id,status FROM inventory WHERE sku=?`

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
* WHAT RESULT AM I LOOKING FOR: A 4.9 s datasource acquire span occurs before SQL; many failures have no MySQL child span.

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

* What I check: RED and business metrics for POST /checkout/validate
* Why I check it: I need severity, start time, and affected cohorts before changing production
* Example command/query/tool: `Grafana with deployment annotations`
* Expected result: rate near 2,000/min and p95 near 310 ms
* Bad result: p95 5.3 s and 18% acquisition timeouts
* What the bad result means: the incident is customer-facing but the database phase is not yet known
* What I check next: split by instance, version, tenant, parameter, and outcome

### Step 2 - Locate elapsed time

* What I check: healthy and failed distributed traces
* Why I check it: parent/child duration shows where time accumulates
* Example command/query/tool: `Tempo or Jaeger comparison`
* Expected result: A 4.9 s datasource acquire span occurs before SQL; many failures have no MySQL child span.
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
* Expected result: query rate fell from 1,700/s to 620/s; executed SELECT p95 stayed 31 ms
* Bad result: work or latency changes sharply, or execution count falls despite demand
* What the bad result means: execution is costly or upstream admission is blocked
* What I check next: capture a safe plan and compare parameter classes

### Step 7 - Read EXPLAIN

* What I check: access type, key, estimated rows, join order, and Extra
* Why I check it: the plan describes how MySQL intends to access and combine rows
* Example command/query/tool: `EXPLAIN FORMAT=TREE SELECT id,status FROM inventory WHERE sku='SKU-441'`
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

* Active means checked out, idle means immediately reusable, pending means threads waiting, and acquisition is the checkout delay.
* Persistent checkout/return divergence is leak evidence; slow holders eventually return connections but have a high usage-time distribution.
* A falling DB query rate is bad when application rate is constant because pool wait prevents statements from reaching MySQL.
* Budget every replica: instance count times maximumPoolSize plus jobs and operations reserve must remain below the DB safe limit.

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

Observed evidence: A 4.9 s datasource acquire span occurs before SQL; many failures have no MySQL child span.

```text
traceId=pool-a81c
Gateway                         18 ms
  | parent spanId=gw-42
  v
POST /checkout/validate service      5.3 s
  | spanId=checkout-19
  +-- hikari.acquire             4,902 ms ERROR
  +-- transaction.begin         not reached
  +-- mysql.execute             no child span
  +-- jdbc.fetch/map            not reached
  +-- transaction.commit        not reached
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
2026-09-13T10:31:22.481Z ERROR service=checkout-service instance=checkout-7f9d endpoint="/checkout/validate"
traceId=pool-a81c spanId=checkout-19 requestId=checkout-882 version=2026.09.13
pool=HikariPool-checkout active=40 idle=0 pending=186 acquireMs=4902 timeoutMs=5000
error=SQLTransientConnectionException detail="Connection is not available"
2026-09-13T10:31:22.486Z INFO service=checkout-service instance=checkout-7f9d
traceId=pool-ok-17 spanId=mysql-44 queryDigest=inventory-by-sku executeMs=31 rowsReturned=1
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
WHERE SCHEMA_NAME='checkout_db'
ORDER BY SUM_TIMER_WAIT DESC LIMIT 20;
EXPLAIN FORMAT=TREE SELECT id,status FROM inventory WHERE sku='SKU-441';
SHOW ENGINE INNODB STATUS;
```

These are observations within retention and configuration limits. Performance Schema consumers can be disabled, and latest-deadlock evidence can be overwritten.

EXPLAIN is the safer first plan check for SELECT. EXPLAIN ANALYZE executes the statement. Never use it for a write or unbounded expensive production read.

Do not issue KILL, change globals, run ANALYZE TABLE, or create/drop indexes without owner confirmation, approval, impact analysis, and rollback.

# Root Cause

A deployment added a validation-error path that manually borrowed a JDBC connection but did not close it; checkout count rose while return count stopped.

Causal chain:

```text
validation-error code borrows a JDBC connection
  -> exception return skips close()
  -> checkouts rise while returns stop
  -> all 40 connections remain checked out
  -> idle reaches 0 and pending reaches 186
  -> acquisition p99 reaches 4.9 s before SQL starts
  -> DB query rate falls although request rate stays at 2,000/min
  -> acquisition timeouts and retries amplify the queue
```

Validate every arrow with time-aligned deployment, phase span, pool lifecycle, digest/lock evidence, queue, retry, and business signals. An unsupported arrow remains a hypothesis.

# Fix

### Immediate mitigation

* Rollback the release and drain affected instances so new traffic stops while in-flight requests finish.
* Bound retries with an overall deadline, exponential backoff, jitter, and a low attempt cap.
* Preserve traces, pool metrics, plans, locks, and deadlock evidence before it disappears.
* Prefer draining a proven bad instance to abruptly terminating fleet work.
* Do not blindly increase timeout, pool size, CPU, or memory; this can move saturation.

### Permanent fix

* Use Spring-managed transactions or try-with-resources on every ownership path, remove manual borrowing, and test validation, exception, timeout, and cancellation paths.
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
| Endpoint p95 | 5.3 s | near 310 ms | customer latency recovered |
| Business/error signal | 18% acquisition timeouts | below 1% and baseline | business success recovered |
| Hikari pending | up to 186 | 0 steady state | no acquire queue |
| Acquire p99 | up to 4.9 s | below 20 ms | pool admission recovered |
| Active/idle | 40/0 in saturation | active falls; idle returns | capacity recycles |
| Checkout-return delta | growing if leak | near zero | ownership closes |
| Inventory SELECT p95 | 31 ms | at or below 31 ms | executed SQL remained healthy |
| DB query rate | 620/s | returns toward 1,700/s | admitted throughput recovered |
| MySQL CPU | 29% | proportional to restored query rate | recovery did not overload MySQL |

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

For database connection pool exhaustion, I first confirm business impact and scope by endpoint, instance, version, tenant, and parameter. Then I use a trace to separate Hikari acquisition, physical connect/login, transaction begin, lock wait, SQL execution, result mapping, and commit. I correlate traceId with logs and overlay app rate, pool active/idle/pending/acquire, DB query rate, statement digests, rows examined, locks, CPU, and I/O. Here, A deployment added a validation-error path that manually borrowed a JDBC connection but did not close it; checkout count rose while return count stopped. I use the smallest reversible approved mitigation, implement the permanent fix, and verify business success plus causal metrics at peak load.

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

### 1. Why can query rate fall during an outage?

Threads cannot acquire connections, so fewer statements reach MySQL while incoming application demand remains constant.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 2. How do you distinguish a leak from slow SQL?

A leak shows persistent checkout/return divergence; slow SQL shows long DB spans and eventual returns.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 3. Should you increase maximumPoolSize?

Not first. It can move saturation into MySQL and violate the fleet-wide connection budget.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 4. What does pending mean?

Application threads are queued for a pool connection; it is not the number of MySQL queries running.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 5. Why can a trace have no DB span?

The request may fail during acquisition before a statement span is created.

I would still require same-window trace, metric, and server evidence before declaring root cause.

### 6. How do you verify cleanup?

Returns catch checkouts, active falls after traffic, pending remains zero, and acquire p99 returns to milliseconds.

I would still require same-window trace, metric, and server evidence before declaring root cause.
