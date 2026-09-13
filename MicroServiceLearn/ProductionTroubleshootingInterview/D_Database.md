# Production Troubleshooting Study Chapter: Databases

## Purpose and safety

This chapter builds a database troubleshooting method from first principles to production diagnosis. Examples use PostgreSQL-style views and SQL, with notes for MySQL and SQL Server. Translate names to the engine and managed service in use.

> Start read-only. Capture timestamps, scope, query fingerprints, wait events, plans, and configuration before changing anything. Never run `EXPLAIN ANALYZE` on an unknown write or expensive production query: it executes the statement. Prefer an existing sampled plan, `EXPLAIN` without `ANALYZE`, or a safe replica. Do not kill sessions, clear caches, rebuild indexes, fail over, or change global settings without approval and impact analysis. Redact literals, credentials, and customer data.

## Learning goals

After studying this chapter, you should be able to:

1. Separate local pool acquisition, DNS/TCP/TLS/login, transaction, lock, execution, and result-transfer time.
2. Use workload, saturation, wait, query, plan, lock, and host evidence together.
3. Diagnose leaks, blocking, deadlocks, bad plans, stale statistics, parameter-sensitive plans, and index problems.
4. Explain isolation, transactions, pagination, large results, N+1 queries, failover, and per-instance failures.
5. Mitigate safely, prove a root cause, and build prevention rather than merely increasing timeouts.

---

# 1. Architecture and mental models

## 1.1 The request-to-row path

```text
HTTP request
  -> application worker
  -> wait for local connection-pool slot
  -> DNS -> TCP connect -> TLS -> database login/session setup
  -> begin transaction
  -> parse/bind/plan (or reuse prepared plan)
  -> wait for locks / CPU / memory / I/O / worker
  -> execute operators: scan, join, sort, aggregate
  -> stream rows over network
  -> deserialize/map rows
  -> commit/rollback (log flush and replication may be involved)
  -> close logical connection (return physical connection to pool)
```

The word "database timeout" is incomplete. Record the timer and its owner:

| Timer/stage | What expiration usually means |
|---|---|
| Pool acquisition | No local pooled connection became available |
| DNS/TCP connect | Name resolution or network handshake did not finish |
| TLS/login | Secure negotiation, authentication, or session establishment stalled/failed |
| Statement/query | Server execution plus server-side waits exceeded a limit |
| Lock timeout | A requested lock was not granted in time |
| Socket/read | Connection exists, but result bytes did not arrive in time |
| Transaction | The whole unit of work exceeded an application/server policy |
| Commit | WAL/log flush, storage, or synchronous replica acknowledgement may be slow |

Measure these separately. A 5-second pool wait followed by a 20-ms query is not a slow query.

## 1.2 Latency equation and queueing

```text
observed DB dependency time
  = pool_acquire
  + connect_and_login (usually zero on a reused connection)
  + transaction_setup
  + lock_wait
  + server_queue_wait
  + parse_plan
  + execute_CPU_and_IO
  + result_transfer_and_mapping
  + commit
```

At steady state, Little's Law gives:

```text
concurrent DB work ~= completed transactions/second * average DB time in seconds
```

If traffic remains 200 transactions/s and duration rises from 0.05 s to 2 s, concurrency rises from about 10 to 400. Pools fill, callers queue, retries add load, and a query regression becomes an outage.

## 1.3 Database architecture

```text
application instances
  | each has a bounded pool
  v
proxy/pooler/load balancer (optional)
  v
primary database ---- WAL/binlog/log ----> replicas
  | buffer/cache memory
  | query workers + lock manager
  v
data/index pages on storage
```

Important limits exist at every layer: app pool size, proxy sessions, server connections, CPU, memory grants/work memory, IOPS, throughput, replication bandwidth, and lock concurrency. `20 pods * pool max 50 = 1,000` possible sessions before administration, migrations, and failover headroom.

## 1.4 Transactions, locks, and isolation

A transaction is an atomic unit: all changes commit or none do. Keep it short. Waiting on an API, user input, or expensive computation while a transaction is open retains locks and often a connection.

Common isolation ideas:

| Isolation | Typical property and trade-off |
|---|---|
| Read uncommitted | Dirty reads may be allowed; many engines treat it as read committed |
| Read committed | Each statement sees committed data; values can change between statements |
| Repeatable read/snapshot | Stable transaction snapshot; version retention/conflict behavior matters |
| Serializable | Strongest illusion; may block more or abort transactions for retry |

Locks protect correctness. A blocker is not automatically "bad"; the question is why it is long-lived, why so many requests need the same resource, and whether access order is consistent. MVCC reduces reader/writer blocking but does not eliminate schema, metadata, uniqueness, or writer/writer conflicts.

## 1.5 Query plans, statistics, parameters, and indexes

The optimizer estimates row counts and costs, then chooses scans, joins, order, parallelism, and memory. A good plan depends on accurate statistics and representative parameter values.

- **Stale statistics** produce bad cardinality estimates.
- **Data skew/correlation** can fool simple histograms.
- **Parameter-sensitive plans** occur when one cached plan is good for a selective value but terrible for a common value, or vice versa ("parameter sniffing" in SQL Server terminology).
- **Index usefulness** depends on leading columns, predicates, sort order, selectivity, included/covered columns, and write cost.
- **Missing index** is only one hypothesis. An unused or redundant index consumes memory and slows writes.
- Functions/casts on indexed columns, leading-wildcard searches, mismatched types/collations, and non-sargable predicates can prevent efficient seeks.

Compare estimated versus actual rows at each plan node in a controlled execution. A large divergence is a cardinality clue; a large actual-time node is a work clue. The node with the largest displayed cost is not automatically the runtime cause.

## 1.6 Glossary

| Term | Meaning |
|---|---|
| Connection pool | Reusable physical sessions managed locally by an application |
| Active/idle connection | Executing versus awaiting work; "idle in transaction" still owns a transaction |
| Query fingerprint | Normalized SQL shape with literals removed |
| QPS/TPS | Queries or transactions completed per second |
| Wait event | Resource or condition preventing a session from progressing |
| Cardinality | Number of rows, or an optimizer row-count estimate |
| Selectivity | Fraction of rows matched by a predicate |
| Sargable | Predicate shape usable as an index search argument |
| Covering index | Index contains columns needed to filter and return a query |
| Full/table scan | Reads a table broadly; appropriate for some large-result queries |
| Nested loop/hash/merge join | Join algorithms suited to different sizes/orderings |
| Spill | Sort/hash exceeds memory grant and uses temporary storage |
| MVCC | Multi-version concurrency control |
| Blocking | One session waits for an incompatible lock held by another |
| Deadlock | Cycle of waits; the engine chooses a victim |
| WAL/binlog | Durable change log used for recovery and replication |
| Replica lag | Replica replay position/time behind the primary |
| RTO/RPO | Recovery-time and recovery-point objectives |
| N+1 | One query for a list followed by one query per item |
| Keyset pagination | Seek from the last stable key instead of skipping an offset |

## 1.7 Beginner guide to reading database evidence

### A connection pool snapshot

Assume one application instance has a pool maximum of 20:

```text
active=20
idle=0
pending=35
acquisition_p99=4.8s
usage_p99=6.2s
```

This says all slots are borrowed and 35 requests wait. It does not yet say *why*. The 6.2-second usage time suggests connections are held for a long time. Next inspect the borrowers:

```text
15 executing the same report query
4 waiting for a lock
1 idle in a transaction while calling another service
```

Now the response can target actual causes. Opening 20 more sessions would increase report concurrency and may make the database slower.

Contrast that with:

```text
active=5
idle=15
pending=0
physical_connect_errors=200/s
```

The pool has capacity, but replacement connections fail. Investigate TCP/TLS/login or validation churn rather than query duration.

### A query-statistics row

Suppose an incident-window aggregate shows:

```text
fingerprint A: calls=10,000 mean=5ms   total=50s
fingerprint B: calls=20     mean=800ms total=16s
fingerprint C: calls=2      mean=6s    total=12s
```

- A has the largest fleet cost because it is frequent. Removing unnecessary calls may have the biggest impact.
- B is individually slow and may hurt a user-facing endpoint.
- C is slowest per call but might be a low-priority report.

Choose the ranking dimension from the incident question. "Which query burns the most capacity?" differs from "Which query violates this request's deadline?"

### A simplified execution plan

```text
Nested Loop                         actual rows=500,000
  Index Seek customers              actual rows=1
  Index Lookup orders               actual rows=500,000 loops=1
```

For a small customer, repeated lookups may be excellent. For a customer with 500,000 orders, a different access path may be cheaper. Read plans with these questions:

1. How many rows entered and left each operator?
2. How many times did it loop?
3. How different were estimated and actual rows?
4. How many logical/physical reads occurred?
5. Did sort/hash work spill to temporary storage?
6. Was elapsed time work, a lock wait, or another wait?
7. Does this parameter and result size represent production?

An index seek is not automatically good, and a scan is not automatically bad. A query returning most of a table may be best served by a scan. Hundreds of thousands of random lookups can be worse.

## 1.8 Worked end-to-end diagnosis

An API usually completes in 300 ms but now times out at 5 seconds.

```text
API trace
  request queue                 10 ms
  pool acquisition          4,500 ms
  SQL execution               350 ms
  mapping/serialization        80 ms
```

The first conclusion is precise: most observed time is local pool acquisition. Next, pool usage shows every connection held for about 12 seconds. Database sessions reveal they are blocked behind transaction 912. That transaction updated one account row and then called a remote fraud service before committing.

The causal chain is:

```text
slow fraud service
 -> transaction remains open
 -> row lock remains held
 -> other SQL waits
 -> pooled connections remain borrowed
 -> pool pending grows
 -> APIs time out before obtaining a connection
```

Notice how several symptoms are true at once:

- callers have a pool timeout;
- database statements are blocked;
- the database itself is available;
- the original trigger is a remote service call inside a transaction.

The incident mitigation can disable or bound the affected operation and let the blocker finish under approved procedures. The root correction moves the remote call outside the database transaction or uses a workflow/state transition that does not retain locks. Prevention includes transaction-age, blocker-age, pool-pending, and remote-dependency alerts plus a concurrency test.

---

# 2. Core metrics and safe evidence

## 2.1 Core metrics

Correlate every graph to the same clock and incident window:

- **Workload:** transactions/s, query calls by fingerprint, rows read/returned/changed, connections opened/s.
- **Latency:** pool-acquire p50/p95/p99; connect/login; statement; lock wait; commit; result mapping.
- **Errors:** acquisition/connect/login/statement/lock timeouts, serialization failures, deadlocks, disconnects.
- **Saturation:** active versus max connections, pool pending, CPU run queue, IOPS/latency/queue depth, memory pressure, temp bytes/spills.
- **Waits:** lock, data-file read, log flush, network/client, worker/thread, buffer/latch, replication.
- **Query:** calls, total and mean time, p95 if available, rows/call, logical/physical reads, temp bytes.
- **Availability:** primary role, restarts, failovers, replica lag, connection resets, DNS endpoint changes.

Rate and duration need each other. High CPU at twice normal throughput may be healthy; high CPU with falling throughput and growing queues is saturation. High connections can be harmless if idle outside transactions; fewer sessions can still overload CPU.

## 2.2 Safe PostgreSQL-style queries

Check privileges and engine version. These are observational, but query text may contain sensitive literals.

```sql
-- Session states and wait classes
SELECT state, wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state, wait_event_type, wait_event
ORDER BY count(*) DESC;

-- Long-running and idle-in-transaction sessions
SELECT pid, application_name, client_addr, state, wait_event_type, wait_event,
       now() - xact_start AS xact_age,
       now() - query_start AS query_age,
       left(query, 200) AS query_sample
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- Blocking relationships
SELECT blocked.pid AS blocked_pid,
       blocker.pid AS blocker_pid,
       now() - blocker.xact_start AS blocker_xact_age,
       left(blocked.query, 160) AS blocked_query,
       left(blocker.query, 160) AS blocker_query
FROM pg_stat_activity blocked
CROSS JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid)
JOIN pg_stat_activity blocker ON blocker.pid = b.pid;

-- Requires pg_stat_statements
SELECT queryid, calls, total_exec_time, mean_exec_time, rows,
       shared_blks_hit, shared_blks_read, temp_blks_written,
       left(query, 200) AS query_sample
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Index/table activity; interpret over a meaningful period
SELECT relname, seq_scan, seq_tup_read, idx_scan, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC
LIMIT 20;
```

MySQL equivalents include Performance Schema, `sys.statement_analysis`, `SHOW PROCESSLIST`, `performance_schema.data_lock_waits`, and `EXPLAIN FORMAT=JSON`. SQL Server equivalents include Query Store, `sys.dm_exec_requests`, `sys.dm_os_wait_stats`, `sys.dm_tran_locks`, `sys.dm_exec_query_stats`, actual execution plans, and Extended Events deadlock reports.

Interpretation cautions:

- Cumulative counters need a baseline/delta; a top query since restart may not be top during the incident.
- High cache-hit ratio can coexist with a bad query reading millions of cached pages.
- `ClientRead`/`ClientWrite` can point to an application not consuming results, not server storage.
- Session snapshots miss short queries; use aggregated query telemetry and traces.
- Plans and query text can expose literals. Store them in approved, access-controlled systems.

## 2.3 Generic production workflow

1. **Stabilize and timestamp.** Declare the incident, preserve evidence, pause risky releases, and define user impact.
2. **Classify the stage.** Exact exception, timer, pool metric, SQLState/vendor code, and whether any server session/query exists.
3. **Scope.** One query, endpoint, tenant, parameter, app instance, database node, region, or all traffic?
4. **Compare.** Healthy versus affected time, parameter, plan ID, app version, instance, database role, and host.
5. **Check demand and saturation.** Traffic, pool pending, sessions, CPU, memory, storage, waits, and retry rate.
6. **Find dominant work/wait.** Query fingerprints, traces, wait events, blockers, plans, rows, reads, and spills.
7. **Form one testable hypothesis.** Example: "plan B underestimates tenant X and spills after statistics changed."
8. **Mitigate safely.** Reduce load, bound concurrency, pause a batch, route away, roll back, or use an approved known-good plan.
9. **Correct root cause.** Query/schema/index/transaction/pool/retry/configuration fix, tested with production-like data.
10. **Verify and prevent.** Same user path recovers; queues drain; no hidden errors; add alerts, tests, runbook, and ownership.

---

# 3. Original interview questions

## 1. The database suddenly becomes slow and all APIs depending on it become slow. How would you investigate?

**Precise meaning.** This is a shared-dependency latency event, not proof that the database engine is the cause. "Suddenly" suggests a workload, plan, lock, host, storage, maintenance, failover, or configuration transition.

**Where it can happen.** Application pools, proxy, primary/replica, lock manager, CPU/memory, storage, network, or a common deployment/retry path.

**Causal mechanisms.** A blocking transaction can queue many queries; a plan regression can consume CPU/I/O; traffic or retries can exceed capacity; checkpoint/log flush/storage latency can delay writes; backups/statistics/index maintenance can compete; memory pressure can evict cache or create spills; failover can create cold caches and reconnect storms.

**Ordered debugging.**

1. Fix the interval and quantify affected APIs, regions, operations, errors, and percentiles.
2. Separate pool wait, connect, lock, query, transfer, and commit time from traces/metrics.
3. Compare traffic and retries with the last healthy period.
4. Check connection/pool saturation, database CPU, memory, I/O latency/queue, log flush, waits, and replica state.
5. Rank incident-window query fingerprints by total time, calls, reads, and mean/p95 change.
6. Inspect active sessions, long transactions, blocking chains, and maintenance jobs.
7. Compare plan IDs, estimates, parameters, statistics freshness, schema/config, releases, and failover events.
8. Test the leading explanation on a safe replica or controlled execution; verify recovery with the same workload.

### Beginner expansion: how the causes produce this symptom

- **Blocking transaction:** imagine an order update takes a row lock and then waits 40 seconds for an HTTP call. Every API that needs that row queues behind it. The database is reachable and may use little CPU, but callers still see long durations.
- **Bad plan:** a statistics or data change can make the optimizer scan 50 million rows instead of seeking 50 rows. Many unrelated APIs then compete for the same CPU and storage.
- **Demand or retry surge:** if 200 calls/s become 600 calls/s, or each timeout causes three retries, the database receives more work than it can finish. Queues grow even if no single query changed.
- **Storage/log delay:** writes wait for durable log flush. Slow storage or a synchronous replica can delay commit, retain locks longer, and indirectly slow reads.
- **Failover/cold cache:** after promotion, clients reconnect together and useful pages are not yet in memory. Connection setup, physical reads, and retries can peak at once.

### Why the diagnostic order matters

1. Establishing the interval prevents comparing today's incident with lifetime counters.
2. Splitting timers avoids tuning SQL when callers actually wait for a pool connection.
3. Comparing demand answers whether the system changed or the workload changed.
4. Resource and wait metrics identify the constrained layer before a query is blamed.
5. Query ranking attributes fleet impact: total time combines cost per call and call count.
6. Lock and maintenance checks catch causes that a plan alone cannot explain.
7. Change comparison converts a broad incident into testable hypotheses.
8. Safe validation proves causation; correlation with a deployment is not enough.

### Evidence interpretation

| Evidence | Likely direction | Next proof |
|---|---|---|
| Pool pending rises, DB active sessions do not | Application pool/leak or connect path | Oldest checkout and connect errors |
| Lock-wait time dominates, one old transaction blocks many | Blocking | Reconstruct the blocker's full transaction |
| CPU and logical reads jump for one fingerprint | Plan/query regression | Compare old/new plan and row estimates |
| Commit/log-flush wait and storage latency rise | Durability path | Storage and synchronous-replica health |
| Calls/s and retries jump before saturation | Demand amplification | Caller retry and traffic timeline |
| All metrics change immediately after promotion | Failover/cold start | Role event, reconnect rate, cache reads |

### Worked mini-example

At 10:02, checkout p99 remains 5 ms but statement p99 rises to 8 seconds. TPS falls, lock waits stay low, CPU reaches 95%, and one normalized query's logical reads rise from 2,000 to 4 million per call after a plan ID change. This evidence points to a plan regression, not pool exhaustion. Pausing the new report reduces CPU and latency, confirming workload attribution. The durable fix is tested query/statistics/index work, followed by plan-regression monitoring.

**Tools, queries, metrics, interpretation.** Use APM dependency spans, pool pending/acquire histograms, managed-database metrics, wait-event breakdown, `pg_stat_activity`, `pg_stat_statements`, Query Store/Performance Schema, and storage telemetry. Falling TPS plus rising active sessions and lock waits means queueing; high CPU plus one fingerprint's increased reads suggests inefficient work; storage latency with data-file waits suggests I/O, not merely "CPU."

**Immediate mitigation.** Stop or throttle an offending batch, shed noncritical requests, remove unbounded retries, roll back a suspect release, or route reads only when consistency allows. Use an approved plan correction or failover only with evidence and owner approval.

**Root-cause correction.** Fix the bad query/index/statistics/transaction, capacity bottleneck, retry policy, maintenance overlap, or infrastructure fault.

**Prevention and alerts.** Baseline query fingerprints/plans, alert on pool pending and latency plus saturation, cap concurrency, load-test peak and skew, schedule maintenance, preserve failover headroom, and monitor blocking age.

**Common traps.** Restarting erases evidence; adding connections may worsen overload; clearing cache makes cold-cache pressure; an average hides tail latency.

**Interview-ready answer.** "I first decompose database time and scope the common impact. I correlate incident-window traffic, pool pressure, waits, CPU/I/O, blockers, query fingerprints, plans, and changes against a healthy baseline. I mitigate the proven bottleneck safely, then fix and regression-test the query, transaction, capacity, or infrastructure cause."

## 2. An API is slow because of a database query. How would you identify the problematic query?

**Precise meaning.** Prove which SQL fingerprint and execution contributes to the API's critical path; do not infer it from a slow endpoint alone.

**Where it can happen.** ORM-generated SQL, stored procedure, trigger, view, repeated query, primary or replica, or result mapping after the server finishes.

**Causal mechanisms.** One long query, many moderate N+1 calls, lock wait, plan regression, large row transfer, slow commit, implicit conversion, spill, or client-side row processing can dominate.

**Ordered debugging.**

1. Trace one slow request with endpoint, instance, tenant category, and correlation ID.
2. Expand database spans; distinguish acquisition from execution and count calls.
3. Map spans to normalized query ID/fingerprint without logging sensitive literals.
4. Compare slow and normal traces, parameter shape, returned rows, plan ID, and database waits.
5. Rank the fingerprint in the incident window by total time, calls, reads, and rows.
6. Obtain a safe plan and inspect actual-versus-estimated rows, scans, joins, sorts/spills, lookups, and lock time.
7. Reproduce with production-like volume/skew and verify a candidate fix.

### Beginner expansion: what "the query" can mean

- **One expensive execution:** `/search` runs one statement for 12 seconds because it scans and sorts a large table.
- **Many small executions:** `/orders` runs 501 queries at 8 ms each. No query appears in a 1-second slow log, but round trips consume about four seconds.
- **Waiting rather than working:** SQL appears active for five seconds, but four seconds are a lock wait. Rewriting the query may not fix the long transaction holding the lock.
- **Too much result data:** the server finishes in 80 ms, then sends 100 MB and the ORM allocates thousands of objects. The API span is slow although engine execution is fast.
- **Slow commit:** all statements finish quickly, but transaction commit waits for log flush or replication.

### Why the diagnostic order matters

1. A request trace ties SQL to the exact user symptom instead of choosing a globally expensive but unrelated query.
2. Phase and call count distinguish one bad statement from N+1 and pool waiting.
3. A fingerprint aggregates the same query safely across different literal values.
4. A healthy comparison exposes parameter, result-size, plan, and wait differences.
5. Fleet ranking shows whether the query matters at production scale.
6. Plan inspection explains *how* rows are obtained, but only after the correct execution is identified.
7. Production-like validation prevents a fix that works only for tiny developer data.

### Evidence interpretation

| Result | Interpretation | Follow-up |
|---|---|---|
| One span dominates and server time matches | Engine query is slow | Waits, reads, plan, parameters |
| Hundreds of repeated child fingerprints | N+1/chattiness | Find loop or lazy relation |
| Pool-acquire span dominates | SQL has not started | Pool usage and oldest checkout |
| Server duration is low, client DB span high | Transfer/mapping/network | Rows, bytes, allocations |
| Actual rows far exceed estimate | Cardinality error | Statistics, skew, parameter class |
| High total time but low mean | Frequency problem | Batch/cache/remove unnecessary calls |

### Worked mini-example

A trace for `GET /customers/42/orders` lasts 3.2 seconds. It contains one 20-ms parent query and 300 repeated 9-ms item queries. The database's "top slow query" list does not flag them because each is fast. The call-count evidence proves N+1. A two-query batched fetch reduces the trace to 65 ms and an integration test now asserts at most three SQL calls.

**Tools, queries, metrics, interpretation.** OpenTelemetry/APM SQL spans, ORM statistics, slow-query log with bounded threshold/sampling, `pg_stat_statements`, Query Store, Performance Schema, `EXPLAIN` and controlled `EXPLAIN (ANALYZE, BUFFERS)`. High total time may mean frequency; high mean means individual latency; rows sent versus rows used exposes over-fetch.

**Immediate mitigation.** Cache or disable a noncritical path, limit page size/concurrency, roll back the query change, or use an approved known-good plan/index path.

**Root-cause correction.** Rewrite SQL, remove N+1, add/adjust a justified index, correct parameter typing/statistics, reduce selected columns/rows, or shorten blocking transactions.

**Prevention and alerts.** Trace DB spans, retain plan history, performance-test realistic data, set statement budgets, and alert on per-fingerprint latency/read regressions.

**Common traps.** Logging full SQL literals leaks data; highest average is not necessarily largest fleet impact; `EXPLAIN ANALYZE` executes writes; ORM method time can include pool wait and mapping.

**Interview-ready answer.** "I use a slow request trace to identify the exact normalized SQL, call count, and phase. I correlate it with server query statistics, waits, parameters, rows, and plan history, then validate the suspected operator with safe plan evidence and production-like data."

## 3. Database CPU suddenly reaches 100%. What would you check?

**Precise meaning.** CPU capacity is saturated, but utilization alone does not identify useful versus wasteful work or the host versus container scope.

**Where it can happen.** Database node, proxy, noisy-neighbor VM, query workers, compilation, compression/encryption, or OS kernel.

**Causal mechanisms.** Traffic spike, inefficient/new plan, missing usable index, excessive parallelism, compile storm, connection churn/TLS, vacuum/maintenance, large sort/hash, polling query, or retry amplification.

**Ordered debugging.**

1. Confirm metric scope, duration, core count/throttling, and whether throughput rose or fell.
2. Compare QPS/TPS, active sessions, queues, errors, and retries to baseline.
3. Rank fingerprints by incident-window CPU if available; otherwise total execution time, calls, and logical reads.
4. Check plan changes, parameter/data skew, scans, join explosions, compilation rate, and parallel workers.
5. Separate engine process CPU from kernel steal/throttle and other processes.
6. Correlate deployments, jobs, statistics/index maintenance, failover, and connection creation.
7. Validate after reducing the suspected workload; CPU, queues, and latency should recover together.

### Beginner expansion: how CPU reaches saturation

- A scan touches millions of pages and evaluates a predicate on every row. Even cached pages require CPU to inspect.
- A poor join can execute an inner lookup millions of times. The plan may look simple while its loop count makes it expensive.
- A polling bug changes one query from 10 calls/s to 10,000 calls/s. Each call is cheap, but aggregate CPU is not.
- Frequent new sessions spend CPU on TLS, authentication, and session setup instead of business queries.
- Excessive parallel plans can let a few reports occupy every core, delaying short transactional work.
- Compilation storms occur when SQL text constantly changes or plans are repeatedly invalidated.

### Why the diagnostic order matters

1. Confirming scope prevents confusing a VM metric, CPU throttle, or one core with whole-database saturation.
2. Throughput tells whether CPU is serving more useful work or producing less work at higher cost.
3. Fingerprint attribution finds who consumes the cores.
4. Plan and call-rate checks separate expensive calls from excessive frequency.
5. OS evidence catches steal time, throttling, and non-database processes.
6. Change correlation explains why the transition was sudden.
7. Coupled recovery of CPU, queue, and latency is a stronger causal test than CPU alone.

### Evidence interpretation

| Evidence | Meaning |
|---|---|
| CPU up, TPS up proportionally, latency stable | Legitimate demand growth, but headroom is low |
| CPU up, TPS down, logical reads/call up | Inefficient query or plan |
| CPU up, calls/s up, cost/call stable | Traffic, polling, or retry amplification |
| Database process low but host CPU high | Neighbor/agent/kernel issue |
| Compile rate and distinct SQL text spike | Plan-cache/SQL-text problem |
| A few parallel queries occupy all workers | Parallel report interference |

### Worked mini-example

After a release, CPU rises from 45% to 100%, TPS falls 30%, and a lookup fingerprint goes from 100 to 12,000 calls/s while reads/call stay constant. The issue is not a missing index; a loop now calls the repository once per item. Rolling back drops call rate and CPU. The root fix batches identifiers into one bounded set query.

**Tools, queries, metrics, interpretation.** Query Store CPU, Performance Schema statement digest, PostgreSQL query/OS telemetry, managed service enhanced monitoring, APM call rate. High logical reads per call points to CPU spent walking pages; high calls with cheap SQL points to chattiness; 100% with stable latency may simply be no safety margin.

**Immediate mitigation.** Throttle/pause the dominant batch, cap expensive endpoint concurrency, stop retry amplification, roll back, or scale up/read-scale if safe and consistency-compatible.

**Root-cause correction.** Improve query/index/statistics, batch work, cache appropriately, reuse connections, tune justified parallelism, or add sustainable capacity.

**Prevention and alerts.** Alert on CPU plus runnable queue/latency and error-budget burn; track plan/read/call regressions; capacity-test at peak with headroom.

**Common traps.** Adding an index blindly; rebooting before capturing top work; treating CPU as the cause rather than a constrained resource; scaling without fixing superlinear work.

**Interview-ready answer.** "I correlate CPU with throughput, latency, sessions, and retries, then attribute incident-window work to fingerprints, plans, logical reads, compilation, parallelism, and jobs. I reduce the dominant load safely and fix its query, workload, or capacity cause."

## 4. The application cannot obtain a database connection. What could be the reasons?

**Precise meaning.** Identify whether it cannot borrow a local pooled connection or cannot establish/use a physical server connection.

**Where it can happen.** Pool, DNS, route/firewall, TCP listener, TLS/trust, authentication/authorization, proxy, server connection limit, failover, or session initialization.

**Causal mechanisms.** Pool fully borrowed; leak or long transactions; server max reached; incorrect endpoint/port/database; DNS stale; dropped packets; certificate expiry/SAN/trust mismatch; password/token expiry; account lock; proxy limit; database startup/failover; ephemeral-port/file-descriptor exhaustion.

**Ordered debugging.**

1. Capture exact exception chain, SQLState/vendor code, timer, source instance, endpoint, and timestamp.
2. Check whether pool acquisition timed out and whether a server connection attempt appears.
3. If physical connect fails, resolve DNS and test approved TCP/TLS/login from the affected runtime identity.
4. Compare one failing app instance with a healthy one: config, secret version, DNS answer, route, clock, trust store, and node.
5. Check server/proxy connection count, rejected/auth failures, listener/role, and failover events.
6. Inspect OS socket/file-descriptor/ephemeral-port pressure and connection creation rate.

### Beginner expansion: follow the connection handshake

Think of connection creation as a sequence. The application first needs a free local pool slot. For a new physical connection it resolves the hostname, opens a TCP socket, negotiates TLS, authenticates, selects a database, and runs session initialization. A failure at an early stage means later stages never happened.

- **Pool exhaustion example:** all 20 connections are executing slow queries; request 21 waits locally and times out. The database never sees request 21.
- **Network example:** DNS resolves, but a firewall silently drops TCP packets. Connect time reaches the configured timeout and no database login is logged.
- **TLS example:** TCP succeeds, but the certificate name does not match the endpoint after migration.
- **Login example:** a rotated password is present on nine pods but one pod still uses the old secret.
- **Server-limit example:** 100 application pods each open 30 sessions, exceeding the database/proxy budget.

### Why the diagnostic order matters

1. The nested exception and elapsed time name the failed stage.
2. Pool metrics determine whether any network diagnosis is necessary.
3. Layer-by-layer testing avoids saying "network" when TLS or login failed.
4. Healthy-instance comparison controls for global database state.
5. Server/proxy counters show rejection and capacity from the receiving side.
6. OS limits explain failures visible only on one busy application host.

### Evidence interpretation

| Observation | What it proves or suggests |
|---|---|
| Acquisition timeout; no physical-open attempt | Local pool path |
| DNS error | TCP was not attempted |
| TCP timeout; no server event | Route/firewall/listener path |
| TLS alert/certificate error | TCP worked; trust/protocol failed |
| Authentication SQLState | Network and usually TLS worked; identity failed |
| "Too many connections" | Server/proxy budget exhausted |
| One pod fails with high socket count | Pod/node resource or stale config |

### Worked mini-example

Only pod 7 reports a 30-second "connection timeout." Pool pending is zero, DNS matches healthy pods, and TCP succeeds. TLS reports that the certificate is not yet valid because pod 7's clock is 18 minutes behind. The correct fix is node time synchronization, not increasing the timeout or changing the certificate.

**Tools, queries, metrics, interpretation.** Pool active/idle/pending/acquire-time; `Resolve-DnsName`, `Test-NetConnection`; TLS client diagnostics; server auth logs; `pg_stat_activity` by `application_name`/client; managed-service events. No SYN/server log points before login; authentication rejection proves network/TLS got farther.

**Immediate mitigation.** Route from bad instance/node, refresh an approved rotated secret/trust bundle, reduce connection churn, free leaked/long work through application controls, or restore service endpoint. Do not increase pool/server limits reflexively.

**Root-cause correction.** Correct config/network/certificate/identity, repair pool lifecycle, coordinate rotation, tune connection budgets, and implement failover-aware reconnect with jitter.

**Prevention and alerts.** Phase-specific metrics, certificate/credential expiry alerts, connection budget calculation, startup validation, synthetic login, and failover tests.

**Common traps.** Testing only from a laptop; calling pool exhaustion a network timeout; placing passwords on command lines; increasing pools beyond server capacity.

**Interview-ready answer.** "I classify local pool acquisition versus DNS/TCP/TLS/login first using the exact exception and phase metrics. Then I test from the failing runtime, compare a healthy instance, and inspect pool, proxy, server limits, identity, and failover evidence."

## 5. Database connection pool is exhausted. How would you troubleshoot it?

**Precise meaning.** All usable pool slots are borrowed, creating waiters or acquisition timeouts. It can be cause or downstream symptom.

**Where it can happen.** One process pool, every replica's pool, a proxy pool, or the server connection budget.

**Causal mechanisms.** Traffic/concurrency growth, slow queries, lock waits, long/idle transactions, leak, undersized pool, oversized request concurrency, external calls inside transactions, database outage, or validation/reconnect storm.

**Ordered debugging.**

1. Graph active, idle, pending, acquisition p95/p99, acquisition timeout, and connection creation by app instance.
2. Compute demand with Little's Law and total fleet maximum against server/proxy budget.
3. Inspect oldest checkouts with leak detection/borrow stack if safely enabled.
4. Map server sessions to application and state; find long queries, transactions, blockers, and client waits.
5. Compare affected instances/endpoints and release/config changes.
6. Determine whether connections return slowly or never return.
7. Load-test the corrected lifecycle/query and verify pending remains bounded.

### Beginner expansion: pool behavior

A pool is a small set of reusable database sessions. Borrowing a connection is like borrowing one of 20 checkout scanners. If every scanner is held, new work waits even when the database could theoretically accept more sessions.

- **Slow return:** a query takes 10 seconds, so each borrower legitimately holds a slot for 10 seconds.
- **Long transaction:** code performs SQL, calls another service, and then commits. The connection is idle during the remote call but cannot return.
- **Leak:** an exception bypasses `close()`. The logical slot remains borrowed forever.
- **Demand mismatch:** 200 request workers can all reach the database, but the pool has 20 slots and requests arrive faster than those 20 finish.
- **Database stall:** all borrowers wait on one blocked table; pool exhaustion is downstream evidence of the lock.

### Why the diagnostic order matters

1. Per-instance pool graphs establish that exhaustion is real and show its onset.
2. Capacity math prevents "fixing" 20 waiters by opening 500 sessions against a 300-session server.
3. Old checkout evidence distinguishes retained connections from merely busy ones.
4. Server state explains what borrowed connections are doing.
5. Scope and release comparison locate a leaking code path or overloaded endpoint.
6. Return behavior is the decisive leak-versus-latency distinction.
7. Load validation proves both safety and sustainable throughput.

### Evidence interpretation

| Pattern | Likely cause | Proof |
|---|---|---|
| Active=max, pending rising, usage p99 rising | Slow/blocking work | Query/transaction waits |
| Active=max, oldest checkout age grows indefinitely | Leak/abandoned transaction | Borrow stack and code path |
| Local active=max, server sees fewer sessions | Broken/stale pool or connect failure | Pool logs and physical-open errors |
| Exhaustion only during traffic peak | Capacity/concurrency mismatch | Arrival rate and Little's Law |
| All sessions blocked by one PID | Lock amplification | Blocking graph |

### Worked mini-example

Pool max is 30. A new export starts 30 transactions and calls object storage before commit. Pool pending reaches 120 although database CPU is 15%. Server sessions show `idle in transaction` for the export. Increasing the pool would create more open transactions. The mitigation pauses exports; the fix moves object upload outside the transaction and limits export concurrency.

**Tools, queries, metrics, interpretation.** HikariCP/Micrometer metrics (`active`, `idle`, `pending`, acquisition/usage), thread dumps, APM spans, `pg_stat_activity`. Active=max plus rising usage duration indicates slow held work; active=max with checkout stacks never closing suggests leak; database has few app sessions while local active=max may indicate broken/stale pool state.

**Immediate mitigation.** Shed/throttle work, pause batches, shorten/rollback slow transactions, roll back a leaking release, and cautiously recycle only affected instances to restore service while retaining evidence.

**Root-cause correction.** Always close via structured lifecycle (`try-with-resources`/context manager/finally), remove remote calls from transactions, fix slow/blocking SQL, right-size bounded pool and request concurrency together.

**Prevention and alerts.** Alert on pending/acquisition tail and long checkout age; connection leak tests; pool-budget review per replica/autoscaling maximum; chaos/failover tests.

**Common traps.** Pool max increase moves the queue into the database; an idle server connection is not necessarily leaked; restarting hides lifecycle evidence; acquisition timeout is not query timeout.

**Interview-ready answer.** "I inspect pool active/idle/pending, acquisition and usage duration, then correlate the oldest borrowed connections to threads, transactions, queries, and blockers. I calculate fleet connection budget and distinguish slow returns from leaks before fixing lifecycle or downstream latency."

## 6. Connections are increasing continuously and never being released. What could be wrong?

**Precise meaning.** Clarify whether logical checkouts, physical pooled sessions, or server sessions increase. Stable idle physical sessions up to pool minimum/max can be normal.

**Where it can happen.** Application code/ORM, transaction manager, pool reconfiguration, autoscaling fleet, proxy, health job, or failed close/reset path.

**Causal mechanisms.** Missing close on exception/early return, unclosed result/statement, abandoned transaction, thread cancellation, streaming result retained, per-request pool creation, redeploy classloader leak, pool max/min change, new pods, connection validation failure, or server sessions lingering after network partitions.

**Ordered debugging.**

1. Define the counted object and label by app instance, application name, user, state, and age.
2. Compare pool checkout count with server session count and deployment/autoscaling timeline.
3. Inspect oldest usage and transaction ages; capture sampled borrow stack traces.
4. Trace code paths including exceptions, cancellation, streaming, and async handoff.
5. Verify one shared pool is created per intended lifecycle and shutdown hooks run.
6. Reproduce under failure injection and assert connection count returns to baseline.

### Beginner expansion: what is actually growing

Three counts are often confused. A **logical checkout** is application code borrowing a pool slot. A **physical pooled connection** is a reusable TCP/database session. A **server session** is what the database sees. A pool may intentionally grow physical sessions from 5 to 30 and keep them idle; that is not a leak.

- Missing `close()` after an exception leaks a logical checkout.
- Creating a new pool for every request leaks whole groups of physical sessions.
- An open streaming result retains its connection until the stream is consumed or closed.
- An abandoned transaction can appear `idle in transaction`: it is doing no work but still owns state and locks.
- Autoscaling from 5 to 50 pods legitimately multiplies idle physical sessions and may still exceed the global budget.

### Why the diagnostic order matters

1. Naming the counted object prevents a false leak diagnosis.
2. Pool-versus-server comparison locates the retention layer.
3. Age exposes abnormal lifetime better than a raw count.
4. Exception, cancellation, and streaming paths are where cleanup is most often missed.
5. Pool construction/shutdown review catches lifecycle rather than query bugs.
6. Failure injection proves cleanup when normal control flow does not run.

### Evidence interpretation

| Evidence | Interpretation |
|---|---|
| Logical active grows; physical total stays capped | Borrowed slots not returning or long work |
| Physical/server sessions rise in pool-sized steps | Repeated pool creation or autoscaling |
| Many old `idle in transaction` sessions | Unclosed transaction/stream |
| Idle sessions plateau at configured minimum/max | Usually expected pooling |
| Sessions remain after pod termination | Shutdown/network detection issue; inspect timeouts |
| Growth begins only on one exception code path | Cleanup failure |

### Worked mini-example

Each scheduled job constructs a new data-source object and never closes it. Every run adds ten server sessions, visible as stair steps exactly ten minutes apart. No individual connection checkout is old, so ordinary leak detection misses it. Making the pool application-scoped and closing it on shutdown stops the growth; a lifecycle test runs 100 jobs and verifies session count returns to baseline.

**Tools, queries, metrics, interpretation.** Pool usage histogram/leak detection, heap/object inspection in approved environments, thread dumps, `pg_stat_activity`, proxy metrics. Growing `idle in transaction` indicates transaction lifecycle; growing ordinary idle sessions may reflect legitimate pool growth or pool recreation.

**Immediate mitigation.** Roll back the leaking version, reduce traffic, recycle affected instances gradually, or cap replicas/pools within the server budget.

**Root-cause correction.** Structured close semantics, transaction boundaries, consuming/closing streams, singleton pool lifecycle, cancellation cleanup, and tested shutdown.

**Prevention and alerts.** Long-checkout alerts, session count slope by application, failure-path tests, code review rules, and fleet-wide connection-budget dashboards.

**Common traps.** Treating expected idle pool sessions as a leak; killing server sessions without fixing clients; enabling noisy leak thresholds below normal transaction duration.

**Interview-ready answer.** "I first identify logical versus physical connection growth and label it by instance/state/age. Old checkout stacks and transaction ages reveal missed close or long work; deployment and pool counts reveal recreation or scaling. I fix lifecycle and validate under exceptions and cancellation."

## 7. A query that normally takes 100 ms suddenly takes 10 seconds. What would you investigate?

**Precise meaning.** Determine whether the same fingerprint now waits longer, performs more work, returns more data, or includes a different parameter/plan.

**Where it can happen.** Pool/lock/server queue, optimizer plan, cache/storage, network transfer, replica, or client mapping.

**Causal mechanisms.** Blocking; parameter-sensitive cached plan; statistics change/staleness; data growth/skew; index dropped/unusable/bloated; cold cache/failover; memory spill; storage latency; changed parameter type; maintenance contention; much larger result.

**Ordered debugging.**

1. Compare one 100-ms and one 10-s trace by phase, parameter shape, rows, database node, and plan ID.
2. Check wait events and blocking during the slow execution.
3. Compare actual/estimated rows and plan operators safely.
4. Check statistics timestamp/distribution, index health/definition, schema/config, and recent deployment.
5. Compare logical reads, physical reads, temp/spill bytes, CPU, and bytes returned.
6. Test representative selective and nonselective parameters with production-like data.
7. Verify the fix across parameter classes, warm/cold states, and concurrent load.

### Beginner expansion: why the same SQL can change

The SQL text may be identical while its environment is not.

- **Lock wait:** query work still needs 100 ms, but it waits 9.9 seconds for another transaction.
- **Parameter-sensitive plan:** customer `A` owns 5 rows and customer `B` owns 5 million. A plan optimized for `A` may perform millions of lookups for `B`.
- **Statistics/cardinality error:** the optimizer expects 10 rows, chooses nested loops, and receives 500,000.
- **Cold data:** after failover, pages previously served from memory must be read from storage.
- **Spill:** a sort expected to handle 1,000 rows handles 1 million, exceeds its memory grant, and writes temporary data.
- **Larger result:** execution is unchanged, but returning a large document or millions of rows dominates transfer.

### Why the diagnostic order matters

1. Fast-versus-slow comparison holds the query shape constant and exposes changed dimensions.
2. Waits separate elapsed time from actual engine work.
3. Plans and row estimates explain changed algorithms.
4. Statistics/schema history explains why the optimizer changed its decision.
5. Reads, spills, CPU, and bytes quantify the expensive operation.
6. Parameter classes prevent tuning only the value used during diagnosis.
7. Concurrency and cache tests avoid a laboratory-only fix.

### Evidence interpretation

| Fast/slow difference | Likely explanation |
|---|---|
| Same reads/CPU, slow has 9.9-s lock wait | Blocking |
| Different plan ID and 1,000x logical reads | Plan regression |
| Same plan, actual rows 1,000x larger | Data/parameter growth |
| Physical reads only after failover | Cold cache/storage |
| Temp bytes appear with underestimated rows | Sort/hash spill |
| Server time same, client bytes/time larger | Result transfer/mapping |

### Worked mini-example

`WHERE tenant_id = ? AND status = 'OPEN'` takes 80 ms for most tenants but 12 seconds for the largest. The cached plan expects 20 rows and performs key lookups; the large tenant returns 800,000. This is not random slowness. Tests must include both tenant classes, and the durable plan/query/index strategy must perform acceptably for both.

**Tools, queries, metrics, interpretation.** Plan history/Query Store, `pg_stat_statements`, safe `EXPLAIN`, lock graph, table/index stats, APM. Same plan plus lock wait means contention; new plan plus huge logical reads suggests regression; same server duration but slow client span suggests transfer/mapping.

**Immediate mitigation.** Remove blocker through approved application action, roll back, pause competing work, or use an approved known-good plan/statistics remedy. Bound large requests.

**Root-cause correction.** Make query/index/statistics robust; address skew with query variants or engine-supported parameter-sensitive optimization; shorten transactions; fix storage/capacity.

**Prevention and alerts.** Plan-change and per-fingerprint regression detection, representative parameter tests, stats maintenance, lock-age alerts, and data-growth performance tests.

**Common traps.** Assuming "cache"; updating statistics blindly during peak; forcing one plan that hurts another parameter class; comparing SQL text while missing different bind values.

**Interview-ready answer.** "I compare fast and slow executions by phase, waits, parameters, rows, node, plan ID, reads, spills, and result size. That distinguishes blocking, parameter/plan regression, data growth, cold I/O, and transfer before I apply a tested durable fix."

## 8. An API works for small data but becomes extremely slow for large data. Why?

**Precise meaning.** Runtime or memory grows poorly with input/result size, or a threshold changes the chosen plan/resource behavior.

**Where it can happen.** SQL operators, database memory/temp storage, network, ORM mapping/serialization, application heap, or pagination.

**Causal mechanisms.** Full scans, quadratic nested loops, large joins/sorts/aggregates, spills, nonselective predicates, huge `IN` lists, offset pagination, over-fetching columns/LOBs, materializing all rows, GC, response serialization, or downstream timeouts.

**Ordered debugging.**

1. Define "large": input count, matched rows, returned rows, bytes, offset, tenant, and concurrency.
2. Plot latency, CPU, reads, temp bytes, and response bytes versus size.
3. Separate server execution, result transfer, mapping, and serialization.
4. Compare plans around the threshold and actual versus estimated cardinality.
5. Inspect scans, join algorithms, sort/hash spills, memory, and application heap/GC.
6. Test bounded pages and representative skew/concurrency.

### Beginner expansion: how size changes cost

Suppose a page of 50 orders is fast. Requesting 500,000 orders changes several stages:

- The database may scan more pages and join far more child rows.
- Sorting 500,000 rows needs more memory; if memory is insufficient, temporary storage is used.
- Deep `OFFSET 500000` may still find and discard the first 500,000 rows.
- The network transfers more bytes, and the ORM creates more objects.
- JSON serialization and application garbage collection can dominate after SQL has finished.
- Concurrent large requests multiply memory, temp storage, sockets, and pool hold time.

### Why the diagnostic order matters

1. Quantifying size turns "large" into an independent variable that can be graphed.
2. Scaling curves reveal whether cost is linear, superlinear, or has a threshold.
3. Stage separation avoids adding a database index for a serializer bottleneck.
4. Plan comparison explains threshold changes in join/sort strategy.
5. Resource evidence locates scans, spills, transfer, or application allocation.
6. Bounded concurrency testing captures production queueing that a single run misses.

### Evidence interpretation

| Evidence as N grows | Interpretation |
|---|---|
| Rows returned and duration grow roughly linearly | Volume cost; contract may need bounds |
| Reads grow near N squared | Repeated/nested work or poor join |
| Temp bytes jump after a threshold | Sort/hash spill |
| DB completes quickly; API heap/GC/serialization rises | Application materialization |
| Deep pages get slower while page size is fixed | Offset pagination |
| One large request is fine; ten collapse | Shared capacity/concurrency problem |

### Worked mini-example

Page size 100 remains 120 ms, but page number 50,000 takes 14 seconds. The plan reads and discards about five million ordered rows for `OFFSET`. A cursor using `(created_at, id)` seeks directly after the previous page and remains near 120 ms. The API caps page size and returns an opaque cursor rather than allowing arbitrary deep offsets.

**Tools, queries, metrics, interpretation.** Plan with buffers/I/O timing in safe test, temp/spill metrics, rows examined versus returned, network bytes, allocation/profile, GC, APM spans. Linear row growth with superlinear reads suggests algorithm/plan; fast DB but slow mapping means application-side volume.

**Immediate mitigation.** Enforce page/row/payload limits, stream bounded exports asynchronously, reject pathological filters, reduce concurrency, or precompute/cached summaries.

**Root-cause correction.** Keyset pagination, selective/indexable predicates, better join/index design, batch processing, projections, partitioning only when justified, and asynchronous export architecture.

**Prevention and alerts.** Contract limits, scale tests using realistic largest tenants, rows/bytes/spill telemetry, query budgets, and performance complexity tests.

**Common traps.** Merely raising memory/timeouts; returning unbounded datasets; assuming an index helps a query returning most rows; using deep `OFFSET`.

**Interview-ready answer.** "I quantify data size and split database execution from transfer and mapping. I inspect cardinality, plan transitions, reads and spills, then bound the contract and use selective queries, projections, keyset pagination, streaming, or asynchronous exports."

## 9. An N+1 query problem appears in production. How would you identify and fix it?

**Precise meaning.** One query retrieves N parent objects and code issues approximately one additional query per parent, causing round trips and load proportional to N.

**Where it can happen.** ORM lazy loading, serializers walking relations, templates, authorization/enrichment loops, GraphQL resolvers, or per-item repository calls.

**Causal mechanisms.** Lazy association access outside the intended fetch plan, missing batching/data loader, looped lookup, new serializer field, or production N much larger than test data.

**Ordered debugging.**

1. Trace a single affected request and count DB spans grouped by fingerprint.
2. Look for one parent query followed by a repeated child fingerprint with different IDs.
3. Correlate query count and duration with returned parent count.
4. Locate the loop/lazy accessor/serializer that triggers SQL.
5. Choose join fetch, batch `IN`, data loader, projection, or prefetch based on cardinality.
6. Verify correct rows, no Cartesian explosion, bounded parameters, and fewer round trips under realistic N.

### Beginner expansion: see the data flow

```text
SELECT orders ... LIMIT 100        -- 1 parent query
for each order:
    SELECT customer WHERE id = ?   -- up to 100 child queries
```

The total is `1 + N`. At 5 ms network/database time per child, 100 children add about 500 ms even though every query is "fast." Lazy loading often hides the SQL behind a property access such as `order.customer.name`, and a serializer can trigger it after business code finishes.

Possible fixes have trade-offs:

- A join fetch uses one round trip but can duplicate parent rows when several collections are joined.
- A batched `IN` query uses two or a few round trips and is often predictable.
- A projection returns only fields needed by the endpoint.
- A data loader groups resolver requests during one execution.
- A cache is suitable only for stable, correctly invalidated lookup data.

### Why the diagnostic order matters

1. One trace preserves request boundaries; global call rate alone cannot prove N+1.
2. Repeated fingerprints reveal the mathematical `N` relationship.
3. Correlation with parent count distinguishes N+1 from constant background queries.
4. Locating the trigger prevents a superficial SQL-only change.
5. Fetch strategy must match cardinality and response needs.
6. Realistic validation catches duplicate rows, parameter limits, and memory growth.

### Evidence interpretation

| Observation | Conclusion |
|---|---|
| Query count is approximately `N+1` | Strong N+1 evidence |
| Same child fingerprint with changing ID | Per-parent lookup |
| Queries begin during JSON serialization | Lazy serializer traversal |
| One join reduces calls but rows explode | Cartesian fetch problem |
| Batch query count stays constant as N grows | N+1 removed |

### Worked mini-example

A developer test has three orders, so four SQL calls seem harmless. Production returns 400 orders and issues 401 calls. A projection query returning order and customer summary uses one call, but a one-to-many item collection would multiply rows. The final design uses one order query plus one bounded item query and maps by order ID.

**Tools, queries, metrics, interpretation.** APM span waterfall, ORM SQL/statistics in controlled sampling, per-request query counter in tests, query digest call rate. Many individually fast calls can dominate through network latency and database CPU.

**Immediate mitigation.** Limit page size, disable expensive expansion, cache stable lookup data, or roll back the triggering field.

**Root-cause correction.** Explicit fetch plan/projection, batched relation load, set-based query, or resolver batching. Sometimes two bounded queries are safer than one enormous multi-join.

**Prevention and alerts.** Integration tests assert query-count upper bounds; trace sampled query count; review lazy loads and serializers; test realistic collections.

**Common traps.** Fixing with a join that duplicates parents explosively; enabling eager loading globally; focusing only on slow-query logs, where each child query looks fast.

**Interview-ready answer.** "I prove N+1 from one trace: a parent query followed by a repeated fingerprint whose count scales with N. I find the lazy/loop trigger, replace it with a bounded set-based or batched fetch, and assert query count under realistic data."

## 10. Two requests are waiting for each other and database operations are stuck. What could be happening?

**Precise meaning.** It may be ordinary blocking, a deadlock cycle, or a non-database application/resource cycle. A database deadlock detector normally aborts a victim rather than leaving it forever.

**Where it can happen.** Row/key/range/table/schema locks, advisory locks, application mutexes, or a transaction holding a DB lock while waiting on another service.

**Causal mechanisms.** Request A locks row 1 then requests row 2 while B locks row 2 then requests row 1; inconsistent table order; range locks; lock upgrade; long transaction plus competing work.

**Ordered debugging.**

1. Capture exact session IDs, transaction ages, wait types, SQL, and application trace IDs.
2. Build blocker/wait graph; determine chain versus cycle.
3. Identify resources and lock modes, transaction start, and first statements, not only current SQL.
4. Correlate code paths and access order.
5. Preserve deadlock report if the engine resolved a cycle.
6. Mitigate through the owning application/DBA procedure and validate consistent order.

### Beginner expansion: blocking versus a cycle

```text
Ordinary blocking:
A holds row 1 -> B waits for row 1
A can finish, then B proceeds

Deadlock:
A holds row 1 and waits for row 2
B holds row 2 and waits for row 1
Neither can proceed without intervention
```

A third possibility is outside the database: A holds a row lock while waiting for service B, while service B waits for work that needs A's row. Database views show only part of that distributed cycle.

### Why the diagnostic order matters

1. Session and trace identities connect database waiting to the two user requests.
2. A graph, not a list, distinguishes a queue from a cycle.
3. Lock resource and mode explain exactly what conflicts.
4. Earlier transaction statements reveal the lock that current SQL is holding.
5. The engine report is the most precise evidence after it chooses a victim.
6. Consistent-order validation proves the cycle cannot recur under the tested paths.

### Evidence interpretation

| Graph/result | Meaning |
|---|---|
| One root blocker, edges only away from it | Blocking chain |
| Path returns to its starting session | Deadlock cycle |
| Engine aborts one transaction with deadlock code | Deadlock detector resolved cycle |
| Wait disappears when blocker commits | Ordinary blocking |
| DB shows a holder waiting on network/client | Investigate application/distributed dependency |
| Large scan locks many keys/ranges | Index/query may broaden contention |

### Worked mini-example

Checkout code updates `account` then `invoice`; refund code updates `invoice` then `account`. Under concurrency each holds one row and requests the other. The deadlock graph proves opposite order. Both paths are changed to lock `account` first, then `invoice`, and a concurrency test repeatedly runs checkout/refund without a cycle.

**Tools, queries, metrics, interpretation.** `pg_blocking_pids`, lock views, SQL Server Extended Events deadlock XML, MySQL InnoDB deadlock/Performance Schema, transaction logs/traces. One blocker with many waiters is blocking; a cycle in the deadlock graph is a deadlock.

**Immediate mitigation.** Stop new conflicting work; let a short transaction complete; with approval cancel/terminate the safest blocker/victim after impact review. Never kill blindly; rollback can be expensive.

**Root-cause correction.** Consistent resource order, shorter transactions, suitable indexes to narrow locked ranges, remove remote calls, atomic statement, or optimistic concurrency.

**Prevention and alerts.** Deadlock event capture, blocker age/waiter alerts, bounded transaction time, retry only engine-selected deadlock/serialization victims with backoff and idempotency.

**Common traps.** Calling every lock wait a deadlock; only inspecting victim SQL; retrying indefinitely; lowering isolation without correctness analysis.

**Interview-ready answer.** "I build the wait graph. A chain is blocking; a cycle is a deadlock and the engine usually aborts a victim. I preserve the graph, map both transactions to code and lock order, mitigate safely, then enforce short, consistently ordered transactions."

## 11. A database deadlock occurs in production. How would you investigate it?

**Precise meaning.** Transactions formed a cyclic dependency on incompatible resources; the engine selected a victim and returned a deadlock/serialization-style error.

**Where it can happen.** Row/key/gap/range, index, metadata, or application/advisory lock interactions, including triggers and cascades.

**Causal mechanisms.** Opposite update order, lookup then update through different indexes, range locking, lock conversion, foreign-key/cascade checks, large scans locking more resources, or transactions expanded by slow external work.

**Ordered debugging.**

1. Capture the engine's deadlock graph/report at event time.
2. List every participant, resource, lock mode, statement, transaction age, host, and application operation.
3. Reconstruct earlier statements in each transaction using traces/logs because current SQL is insufficient.
4. Compare access order and plans/indexes; determine why each lock was retained/requested.
5. Quantify frequency, affected operations, retry outcomes, and user impact.
6. Reproduce concurrency in a safe test and verify the selected fix.

### Beginner expansion: reading a deadlock story

A deadlock report is a snapshot of a cycle. Each participant shows a transaction, the resource it owns, and the resource it requests. The statement selected as victim is not necessarily where the transaction acquired its first lock. Triggers, cascades, and foreign-key validation can add locks not obvious in application SQL.

- **Opposite order:** two code paths update the same tables in reverse order.
- **Lock conversion:** two sessions both hold a shared lock and both request an exclusive lock.
- **Range contention:** an index range is protected to preserve isolation, so apparently different rows conflict.
- **Broad scan:** a missing usable predicate touches and locks more keys/pages than intended.
- **Long transaction:** an HTTP call between SQL statements lengthens the window in which another transaction can form a cycle.

### Why the diagnostic order matters

1. Deadlock reports are ephemeral evidence; capture comes before restarts or retries erase context.
2. Listing all participants prevents blaming only the victim.
3. Earlier statements reconstruct lock acquisition order.
4. Plans and indexes explain unexpectedly broad resources.
5. Frequency tells whether bounded retry is enough for resilience or impact demands urgent mitigation.
6. Only concurrent reproduction can validate an ordering/contention fix.

### Evidence interpretation

| Report clue | Interpretation |
|---|---|
| A owns account 1, requests invoice 9; B is reverse | Opposite access order |
| Same statement, different indexes/resources | Plan/index lock path matters |
| Victim changes between occurrences | Victim choice is cost/policy, not blame |
| Transactions are seconds old with remote spans | Scope is too long |
| Range/gap locks rather than exact keys | Isolation and access path need review |
| Retry succeeds once with no duplicate effect | Resilience works, root cycle still exists |

### Worked mini-example

The engine aborts an `UPDATE invoice` statement, so the team initially blames invoice code. The full report shows that transaction earlier locked `account`; the other transaction earlier locked `invoice`. The victim SQL is merely where the cycle closed. Standardizing order removes the cause; a three-attempt jittered whole-transaction retry handles rare races during rollout.

**Tools, queries, metrics, interpretation.** PostgreSQL error logs/deadlock settings with cautious logging, SQL Server `xml_deadlock_report`, MySQL deadlock logs/Performance Schema, distributed traces, transaction IDs. The victim is chosen for recovery cost/policy; it is not necessarily the buggy transaction.

**Immediate mitigation.** Bounded retry of the entire transaction only if idempotent and error-classified, jittered; reduce conflicting concurrency or roll back the triggering release.

**Root-cause correction.** Same row/table order, shorter units, narrower access through correct indexes, atomic upsert/update, partition hot keys, or optimistic version checks.

**Prevention and alerts.** Always-on bounded deadlock capture, rate alert, concurrency regression tests, transaction-duration limits, and retry metrics.

**Common traps.** Retrying a partial transaction; blaming the victim; logging only one query; adding `NOLOCK`/read-uncommitted and sacrificing correctness; confusing lock timeout with deadlock.

**Interview-ready answer.** "I preserve the deadlock graph, reconstruct all participating transactions and their earlier statements, then identify the resource-order cycle and why locks were broad or long-lived. I add a bounded idempotent retry for resilience and fix ordering, transaction scope, or indexing."

## 12. The database is available, but the application still gets connection timeout errors. Why?

**Precise meaning.** "Available" from one probe proves only that probe's path and moment. The application may time out before, during, or after connection establishment.

**Where it can happen.** Local pool, instance DNS/network, proxy, TLS/auth, database connection queue/limit, failover endpoint, or stale pooled socket.

**Causal mechanisms.** Pool exhaustion; one subnet/security policy; DNS cache with old primary; proxy max; server accepts slowly under overload; login trigger/session init delay; TLS revocation/trust issue; ephemeral-port exhaustion; connection storm; health check uses different user/database/route.

**Ordered debugging.**

1. Identify exact timeout class and elapsed duration; inspect nested cause.
2. Check pool pending/acquisition and whether physical opens increased.
3. Test DNS, TCP, TLS, and authenticated login from the failing process/pod identity.
4. Compare endpoint IP, secret, certificate, route, proxy, and clock with a healthy instance.
5. Check server/proxy limits, login latency/errors, backlog, failover/DNS changes, and connection rate.
6. Check stale pooled connections and keepalive/validation behavior without creating a validation storm.

### Beginner expansion: why "available" is not enough

A monitoring probe may run from a different network, use a different database user, and execute once per minute. The application may run from a blocked subnet, wait behind its own pool, or open hundreds of sessions at once. Availability is therefore a property of a path and operation, not a single green light.

- A bastion login proves the database accepts *that bastion*, not the application pod.
- A health probe may reuse one warm connection while new TLS handshakes fail.
- A failover endpoint may resolve to a new IP, while one JVM retains the old DNS answer.
- The listener may be healthy but its login queue or proxy connection cap is saturated.
- A stale pooled socket can look valid locally until its first write/read times out.

### Why the diagnostic order matters

1. Timeout class identifies which availability claim is relevant.
2. Pool evidence determines whether the request left the application.
3. Same-runtime tests reproduce identity, DNS, route, and trust.
4. Healthy comparison reveals path-specific drift.
5. Receiver-side metrics show whether connections arrive and where they wait.
6. Validation behavior explains stale sockets without flooding an already stressed server.

### Evidence interpretation

| Result | Next direction |
|---|---|
| Probe green, pool acquisition times out | Application concurrency/leak |
| Laptop works, pod TCP times out | Pod/node route, policy, NAT, firewall |
| Existing pooled calls work, new opens fail | TLS/login/server/proxy connection path |
| One resolver returns old primary | DNS cache/failover handling |
| Server logs login slowly under connection burst | Reconnect storm/overload |
| First use of idle sockets fails | Idle timeout/keepalive/validation mismatch |

### Worked mini-example

The managed database console says "available." All 40 pods restart after a deployment and each opens 30 connections, creating 1,200 simultaneous logins through a proxy capped at 500. Warm synthetic monitoring remains green. Jittered startup, a smaller pool, and fleet connection budgeting solve the reconnect storm; increasing connect timeout only makes callers wait longer.

**Tools, queries, metrics, interpretation.** Pool and connect-phase histograms, distributed trace, DNS/TCP tools, TLS diagnostics, auth/server logs, `pg_stat_activity`, network flow logs. Successful database console access from a bastion does not test application path.

**Immediate mitigation.** Drain/reroute affected instances, refresh endpoint/secret/trust safely, stagger reconnects with jitter, and restore pool health. Avoid a fleet-wide restart storm.

**Root-cause correction.** Phase-specific timeout/config fix, failover-aware DNS/driver settings, network policy, credential rotation design, pool validation, server/proxy capacity.

**Prevention and alerts.** Synthetic connection from application networks, phase metrics, DNS/failover drills, certificate alerts, and reconnect rate limits.

**Common traps.** "Port open" as proof login works; raising connect timeout; confusing acquisition with connect; repeatedly retrying and exhausting ports.

**Interview-ready answer.** "Availability is path-specific. I identify whether the timeout is pool acquisition, TCP, TLS, login, or read; test from the failing runtime; compare healthy instances; and inspect pool, proxy, server limits, failover, DNS, and identity evidence."

## 13. Only one application instance is unable to connect to the database. What would you check?

**Precise meaning.** The narrow scope strongly favors instance, pod, node, zone, identity, or cached state over a global database outage.

**Where it can happen.** Instance config/secret/trust store, DNS cache, node route/NAT/firewall, service mesh, pool state, clock, resource limits, version, or zone-specific endpoint.

**Causal mechanisms.** Stale secret/config; different deployment revision; expired local trust bundle; wrong DNS answer; node network policy; exhausted ephemeral ports/file descriptors; corrupted/stale pool; clock skew breaks token/certificate; bad sidecar; instance-specific database allow-list.

**Ordered debugging.**

1. Capture exact phase/error and compare failing versus healthy instance at the same timestamp.
2. Diff image/version, environment/config hashes, secret version, service account, trust store, clock, and JVM/driver flags without exposing secrets.
3. Compare DNS answers, route, source IP/NAT, TCP/TLS/login from each runtime.
4. Inspect pool active/idle/pending, connection creation, sockets, file descriptors, ports, CPU/memory, and sidecar logs.
5. Check node/zone/network policy and server logs filtered by source/application name.
6. Cordon/reroute the single instance as mitigation; preserve evidence and reproduce before replacement if possible.

### Beginner expansion: use the healthy instance as a control

When 19 instances connect and one does not, a global password outage or database shutdown is unlikely. The healthy instance is an experiment control: compare one variable at a time.

- **Configuration drift:** the bad pod has yesterday's secret or a different endpoint.
- **Node path:** only its node uses a broken NAT gateway or restrictive policy.
- **Local resource exhaustion:** thousands of outbound sockets consume ephemeral ports or file descriptors.
- **Clock/trust:** token validation and certificate dates fail only on a host with clock skew or old CA bundle.
- **Sidecar/proxy:** the application connects through a local mesh proxy that is unhealthy.
- **Pool state:** sockets created before failover point to an old server.

### Why the diagnostic order matters

1. Exact phase keeps the comparison focused.
2. Immutable version/config/identity comparison tests the most common drift.
3. Path comparison checks what deployment metadata cannot show.
4. Local metrics catch exhaustion hidden by fleet averages.
5. Node/server evidence confirms source-specific rejection or route.
6. Routing out protects users while evidence is captured.

### Evidence interpretation

| Difference | Likely fault domain |
|---|---|
| Bad pod has different secret/config hash | Deployment/rotation drift |
| Same config, different DNS answer | Resolver/cache |
| Same DNS, TCP fails only on one node | Node route/NAT/policy |
| TCP/TLS works, auth fails for one service identity | Identity/token/clock |
| High file descriptors/ports on bad pod | Local resource leak |
| Moving pod to healthy node fixes it | Node-local path; still investigate root cause |

### Worked mini-example

One pod cannot connect after certificate rotation. Its image and endpoint match peers, but its mounted trust bundle hash differs because secret reload failed. Replacing the pod restores service, while the durable fix makes reload atomic and readiness fail when the expected trust-bundle version is absent. A per-instance config-hash dashboard would expose the drift sooner.

**Tools, queries, metrics, interpretation.** Kubernetes pod/node/config metadata, `Resolve-DnsName`/`Test-NetConnection` or approved equivalents inside pods, pool metrics, OS socket counts, server auth logs. Different source IP or DNS result is a powerful discriminator.

**Immediate mitigation.** Remove the instance from service and replace/restart only it when safe, or move it from a bad node; this restores availability but is not the root cause.

**Root-cause correction.** Eliminate configuration drift, fix node networking/trust/clock/resource leak, make secret reload atomic, and repair health checks so connectivity failures remove the instance.

**Prevention and alerts.** Per-instance labels on metrics, immutable config hashes, startup/readiness DB checks with care, node canaries, credential rotation tests, and alerts on outliers rather than fleet averages.

**Common traps.** Restarting before comparing state; assuming same deployment means same config; testing from a debug pod on another node; hiding one-instance errors in aggregated success rate.

**Interview-ready answer.** "I perform a side-by-side diff of the failing and healthy instances: error phase, version/config/secret/trust, DNS, route/source IP, clock, pool and OS resources, node and sidecar. I route it out, preserve evidence, and fix the specific drift or node-local fault."

---

# 4. Important additional interview questions

## A1. How do you read an execution plan safely and decide whether an index is needed?

**Answer.** First obtain the plan from plan history or use non-executing `EXPLAIN`; run actual plans only with approved representative reads in a safe environment. Start at actual elapsed work and row flow, not the visually highest estimated cost. Compare estimated versus actual rows, loops, rows removed, logical/physical reads, spills, join choice, sort, and lookup count. Check the predicate and ordering against existing composite index leading columns and included columns. A scan is reasonable when much of a table is needed. Propose the smallest index that serves important workload, then measure read improvement, write/storage cost, redundancy, and alternative queries. Never create an index solely because an automatic suggestion says so.

## A2. How do parameter-sensitive plans cause intermittent slowness?

**Answer.** Values may have radically different selectivity: tenant A has 10 rows while tenant B has 10 million. A plan compiled/cached for A may use nested loops and seeks; reused for B it performs millions of lookups. The reverse plan may scan needlessly for A. Prove it by correlating latency with parameter class and plan ID, comparing estimates/actual rows, and checking plan history. Durable choices include engine-supported parameter-sensitive optimization, query variants, better statistics, selective recompilation, or carefully chosen plan policy. Never disable plan caching globally or force one plan without testing all important value classes.

## A3. How do you design safe pagination?

**Answer.** Always use a deterministic unique order. `OFFSET n LIMIT k` must often find/skip `n` rows and becomes slower and less stable at deep pages. Keyset pagination uses the last key:

```sql
SELECT id, created_at, status
FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT :page_size;
```

Support the filter/order with an appropriate index and use an opaque cursor. Define behavior under concurrent inserts/deletes and cap page size. Random page jumps may require a different product design or precomputed search.

## A4. What changes during database failover?

**Answer.** In-flight transactions and sockets can fail; a replica is promoted; endpoint/DNS/proxy routing changes; caches are cold; replicas may lag; session state and temporary objects disappear; reconnecting fleets can stampede. Classify errors as transient only when documented, reconnect with exponential backoff and jitter, retry whole idempotent transactions, validate the new role, and prevent writes to an old primary through fencing. Measure RTO/RPO, promotion time, reconnect rate, error/duplicate rate, replica lag, and data correctness. Regularly exercise failover under realistic pool counts.

## A5. How should application and database timeouts be layered?

**Answer.** Start with an end-to-end deadline. Reserve time for response handling and at most a small bounded retry. Pool acquisition, connect, lock, and statement limits must fit inside the remaining budget and cancellation should propagate. A server statement timeout helps stop orphaned work after callers leave. Values follow SLOs and measured distributions, not guesswork. Alert on which timer expires. A longer timeout is not a capacity fix.

---

# 5. Decision trees

## 5.1 "Database timeout"

```text
Exact exception/timer?
|
+-- Pool acquisition timeout
|   +-- active=max, pending rising?
|       +-- long usage/transactions -> slow query, lock, remote call in transaction
|       +-- checkouts never return -> connection leak/lifecycle
|       +-- demand exceeds design -> bound concurrency and recalculate pool budget
|
+-- DNS/TCP timeout
|   +-- one instance/node -> DNS, route, NAT, firewall, ports, sidecar
|   +-- all instances -> endpoint, network policy, listener, failover
|
+-- TLS/login timeout/error
|   +-- certificate/trust/SAN/clock or credential/account/proxy limit
|
+-- Statement/lock timeout
|   +-- lock wait -> blocker graph and transaction history
|   +-- CPU/I/O/spill -> fingerprint and plan evidence
|
+-- Read/socket timeout
    +-- server still executing -> waits/plan/work
    +-- server finished -> result transfer/client consumption/network
```

## 5.2 Slow query

```text
Slow server execution?
|
+-- Mostly waiting
|   +-- lock -> blocker/deadlock/transaction scope
|   +-- I/O -> reads, cache state, storage latency
|   +-- worker/CPU queue -> workload and top CPU/read fingerprints
|   +-- log/replication -> commit path
|
+-- Mostly working
|   +-- plan changed -> stats, schema, parameters, configuration
|   +-- same plan, more rows -> growth/skew/result contract
|   +-- spills -> estimates, memory, sort/hash volume
|
+-- Server is fast
    +-- pool acquisition, network/result bytes, ORM mapping, serialization
```

---

# 6. Cheat sheets

## 6.1 Evidence card

```text
UTC/local timestamp and duration:
Affected endpoint/tenant class/region:
Application instance/version/config hash:
Database endpoint/node/role:
Exact exception, SQLState, timer:
Pool active/idle/pending/acquire/usage:
Query fingerprint and plan ID:
Calls, rows, logical/physical reads, temp/spill:
Wait event and blocker/transaction age:
DB CPU, memory, I/O latency/queue, log flush:
Traffic/retry/deploy/maintenance/failover change:
Healthy comparison:
```

## 6.2 Symptom-to-first-evidence

| Symptom | First evidence |
|---|---|
| Pool timeout | Per-instance active/idle/pending and oldest checkout |
| Connect timeout | DNS/TCP test from affected runtime and server connection logs |
| Login failure | SQLState/vendor code, identity/secret version, auth logs |
| One query slow | Trace fingerprint, phase, wait, parameters, plan ID |
| DB CPU 100% | Throughput/retries and incident-window top CPU/read fingerprints |
| Many blocked sessions | Blocker graph and oldest transaction |
| Deadlock | Engine deadlock graph plus full transaction histories |
| Large-data slowdown | Rows/bytes versus reads/spills/mapping time |
| One instance fails | Side-by-side config, network, pool, node, and source identity |

## 6.3 Durable rules

1. Name the timing phase.
2. Compare incident deltas, not lifetime totals.
3. A pool is a concurrency limit, not free capacity.
4. Keep transactions short and resource order consistent.
5. Plans depend on estimates, statistics, parameters, and data shape.
6. Index for a measured workload and include write cost.
7. Bound results and prefer stable keyset pagination for deep traversal.
8. Retry only classified transient, idempotent whole operations with backoff and a deadline.
9. Preserve evidence before restart/failover/cache clearing.
10. Verify the same user path and add a regression test plus alert.
