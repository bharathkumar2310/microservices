# Problem

**Incident: Health is green while a connection leak breaks business traffic**

Every pod reports health OK and service transport is healthy, yet checkout success collapses. The health route proves only its configured lightweight checks; it does not prove a business request can obtain a connection and finish.

Metrics say whether something is wrong and how much. Traces say where the path spent time or failed. Logs say what happened. Root cause requires agreement among signals and verification.

# Production Situation

At 2026-09-13 17:00 UTC the on-call sees:

- Service A sends 2,000 checkout requests/minute to Service B.
- Business success falls from 99.7% to 72% after B-2026.09.13.5.
- Service B CPU is 29%, heap 56%, and GC max 24 ms.
- DNS, TCP, TLS, gateway, and HTTP transport are healthy.
- Actuator health is UP in 18 ms on every B instance.
- Service B business p99 is 5.1 seconds.
- Worker queue grows from 4 to 612.
- Hikari active is 40/40, idle 0, pending 186, acquisition p99 4.9 seconds.
- DB query rate falls from 1,850/sec to 620/sec under steady requests.
- Checkout and return counts diverge on validation errors.
- Service A retries amplify attempts by 38%.

The initial hypothesis stays broad. Green health or low CPU cannot eliminate waiting, correctness failure, retry amplification, or telemetry loss.

# Architecture

Relevant path:

    Client
      v
    Gateway
      v
    Service A
      | REST transport healthy
      v
    Service B
      | worker queue 612
      | Hikari active 40/40 idle 0 pending 186
      v
    MySQL

    Health path: shallow process check -> UP
    Business path: checkout -> connection -> validation error -> leaked connection

Every arrow is a runtime failure boundary and a context-propagation boundary.

# What I Check FIRST

The first checks are read-only, bounded, and designed to protect evidence.

## 1. Separate health from business success

- WHAT: Separate health from business success.
- WHY: Health proves only configured checks.
- RESULT: Health is UP while success is 72%.

## 2. Read RED and saturation

- WHAT: Read RED and saturation.
- WHY: Low CPU does not exclude waiting.
- RESULT: B p99 is 5.1 s and queues grow.

## 3. Inspect Hikari as one state

- WHAT: Inspect Hikari as one state.
- WHY: Pool values together describe starvation.
- RESULT: 40/40 active, zero idle, 186 pending, 4.9 s acquisition.

## 4. Compare request and query rates

- WHAT: Compare request and query rates.
- WHY: Falling DB traffic can mean work cannot reach DB.
- RESULT: Requests stay flat while queries fall.

## 5. Compare checkout and return

- WHAT: Compare checkout and return.
- WHY: Borrowed resources must return.
- RESULT: Divergence starts on validation errors after deployment.

# Step-by-Step Investigation

I change the next check when evidence changes; I do not jump from one error string to a fix.

### Step 1 - Confirm business impact

- What I check: Use a business outcome ratio by operation.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Use a business outcome ratio by operation.`
- Expected result: Success is 72% versus 99.7% baseline.
- Different result: Health and probe transport pass.
- Meaning: Reachability is not business correctness.
- Next check: Split by release, instance, and outcome.

### Step 2 - Establish timeline

- What I check: Overlay rollout, outcome, queues, pool, and retries.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Overlay rollout, outcome, queues, pool, and retries.`
- Expected result: Divergence starts two minutes after .5 rollout.
- Different result: Leak predates rollout.
- Meaning: Traffic or dependency change remains.
- Next check: Compare old and new instances.

### Step 3 - Eliminate host saturation

- What I check: Check CPU, heap, GC, throttling, and runnable threads.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Check CPU, heap, GC, throttling, and runnable threads.`
- Expected result: CPU 29%, heap 56%, GC max 24 ms.
- Different result: CPU or GC is saturated.
- Meaning: Compute or memory can be causal.
- Next check: Profile or inspect heap.

### Step 4 - Eliminate transport

- What I check: Measure DNS, TCP, TLS, proxy, and HTTP setup.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Measure DNS, TCP, TLS, proxy, and HTTP setup.`
- Expected result: Service B accepts quickly.
- Different result: Connection setup consumes seconds.
- Meaning: Network or gateway remains causal.
- Next check: Inspect client and proxy spans.

### Step 5 - Read critical path

- What I check: Compare A client, B server, acquire, and SQL spans.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare A client, B server, acquire, and SQL spans.`
- Expected result: B is 5.1 s; acquire is 4.9 s; SQL is 38 ms.
- Different result: SQL is 4.9 s.
- Meaning: Query, lock, or DB compute is likely.
- Next check: Inspect plans and locks.

### Step 6 - Interpret Hikari

- What I check: Read max, active, idle, pending, timeout, and histogram together.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Read max, active, idle, pending, timeout, and histogram together.`
- Expected result: 40/40 active, zero idle, 186 pending.
- Different result: Active is low but pending high.
- Meaning: Creation or validation failure may exist.
- Next check: Check connection creation.

### Step 7 - Explain falling query rate

- What I check: Compare requests, acquisitions, and completed queries.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare requests, acquisitions, and completed queries.`
- Expected result: Requests stay 2,000/min while queries fall to 620/sec.
- Different result: Queries remain high and slow.
- Meaning: DB saturation or locks are possible.
- Next check: Inspect DB server metrics.

### Step 8 - Account checkout and return

- What I check: Use increases within one process lifetime.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Use increases within one process lifetime.`
- Expected result: 3,200 checkouts minus 3,160 returns equals 40 active.
- Different result: Divergence exceeds active by thousands.
- Meaning: Counters may reset or double count.
- Next check: Segment by instance and process start.

### Step 9 - Slice by outcome

- What I check: Use bounded outcome_class, never raw validation text.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Use bounded outcome_class, never raw validation text.`
- Expected result: Only validation_error leaks.
- Different result: All outcomes diverge.
- Meaning: Common transaction ownership is suspect.
- Next check: Inspect the common wrapper.

### Step 10 - Read thread dumps

- What I check: Look for WAITING in Hikari getConnection.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Look for WAITING in Hikari getConnection.`
- Expected result: 186 workers wait; few run SQL.
- Different result: Threads are BLOCKED on a Java monitor.
- Meaning: Application locking controls throughput.
- Next check: Find the lock owner.

### Step 11 - Correlate logs

- What I check: Search traceId, businessId, release, and instance.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Search traceId, businessId, release, and instance.`
- Expected result: checkout and validation_failed exist; return is absent.
- Different result: Return log absent but active falls.
- Meaning: The log may be lost or misplaced.
- Next check: Trust pool state and add close outcome.

### Step 12 - Review deployment diff

- What I check: Inspect connection ownership on changed error paths.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Inspect connection ownership on changed error paths.`
- Expected result: getConnection occurs before validation; early return bypasses close.
- Different result: try-with-resources covers all paths.
- Meaning: Leak may be framework or async ownership.
- Next check: Use leak detection and a test.

### Step 13 - Use leak detection

- What I check: Set a bounded threshold above legitimate holds.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Set a bounded threshold above legitimate holds.`
- Expected result: Stack points to ValidationLookup.loadRules.
- Different result: Normal long transactions trigger warnings.
- Meaning: Threshold is noisy.
- Next check: Compare with normal hold SLO.

### Step 14 - Measure retry amplification

- What I check: Compare original requests with attempts.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare original requests with attempts.`
- Expected result: 2,000 originals create 2,760 attempts.
- Different result: Retries remain flat.
- Meaning: Leak alone sustains saturation.
- Next check: Still fix ownership first.

### Step 15 - Mitigate safely

- What I check: Rollback, mark bad pods unready, and drain with a deadline.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Rollback, mark bad pods unready, and drain with a deadline.`
- Expected result: Old code stops new leaks and capacity recovers.
- Different result: Restart alone gives brief relief.
- Meaning: Restart resets, not fixes, the leak.
- Next check: Continue rollback and code fix.

### Step 16 - Failure-path verification

- What I check: Run 10,000 mixed requests with a pool of four.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Run 10,000 mixed requests with a pool of four.`
- Expected result: Active returns to baseline and deltas converge.
- Different result: Active ratchets after each error.
- Meaning: Cleanup remains incomplete.
- Next check: Inspect every ownership transfer.

## Detailed connection accounting

For one stable process lifetime, the useful invariant is:

    connection checkouts - connection returns = currently borrowed connections

Adjust for restarts, failed creation, eviction, and instrumentation placement. Do not subtract counters from different process lifetimes.

On instance b-6f91 over five minutes:

- Checkouts increase by 3,200.
- Returns increase by 3,160.
- Difference is 40.
- Hikari active is 40.
- Hikari idle is zero.

The equality is strong accounting evidence. The bounded outcome slice is stronger: all 40 unmatched borrows are validation_error after release .5.

## Correct resource ownership

The corrected Java shape validates before borrowing when possible and always closes after borrowing:

    ValidationResult validate(Request request) {
        if (!syntaxValid(request)) return ValidationResult.invalid();
        try (Connection connection = dataSource.getConnection()) {
            return validateAgainstRules(connection, request);
        }
    }

Spring-managed transactions are also valid when ownership is explicit. Every normal return, early return, and exception must release the resource.

## Why DB query rate falls

Incoming requests remain constant, but they stop before SQL because no connection is available. Fewer submitted queries therefore indicate starvation in front of the database, not a faster or healthier database. A slow DB would normally show long SQL spans, locks, or server saturation; here SQL is 38 ms and acquisition is 4.9 s.

## Why easy mitigations fail

- Increasing the pool from 40 to 80 only doubles time before the leak exhausts it and risks the DB.
- Increasing acquisition timeout makes customers and worker threads wait longer.
- Restarting reclaims connections briefly, but validation errors leak the new pool again.
- Adding retries creates 38% more attempts against the same finite pool.
- Marking health DOWN without fixing ownership may reduce capacity further; readiness is for safe traffic admission, not diagnosis.

## Rollback and drain sequence

1. Halt release .5 and restore the last known good artifact.
2. Mark a bad instance unready before termination so it receives no new work.
3. Let in-flight idempotent work drain for a bounded interval.
4. Suppress attempts that cannot fit inside the remaining caller deadline.
5. Restart only after removal from traffic to reclaim leaked resources.
6. Confirm good-version capacity before draining the next instance.
7. Canary the cleanup fix under sustained validation errors.
8. Gate rollout on business success, acquisition latency, pending count, and counter convergence.

## Verification matrix

| Evidence | Before | After target | Why it matters |
|---|---:|---:|---|
| Business success | 72% | over 99.7% | customer outcome recovered |
| Service B p99 | 5.1 s | below 420 ms | end-to-end delay removed |
| Acquisition p99 | 4.9 s | below 20 ms | causal wait removed |
| Pool active | 40/40 stuck | follows load | connections return |
| Pool pending | 186 | 0 | no borrower queue |
| Worker queue | 612 | below 10 | upstream saturation cleared |
| DB query rate | 620/sec | near 1,850/sec | work reaches DB again |
| Retry amplification | 38% | below 2% | feedback loop stopped |
| Checkout-return delta | grows to 40 | converges after drain | leak invariant restored |

Health still remains useful: liveness answers whether restart may help, readiness answers whether traffic should be sent, and startup answers whether initialization finished. None replaces a business SLI or synthetic transaction.

# Metrics to Check

Metrics answer: Is something wrong, and how much? Traces answer: Where in the request path did time accumulate or failure occur? Logs answer: What happened at a specific step?

Telemetry pipeline health is part of every conclusion: verify scrape, receive, queue, retry, export, reject, drop, ingestion delay, and backend query visibility.

## RED, USE, and golden signals

- RED means Rate, Errors, and Duration for request-driven services.
- USE means Utilization, Saturation, and Errors for each finite resource.
- Golden signals are Latency, Traffic, Errors, and Saturation.
- RED is the service view; USE is the resource view; golden signals connect customer impact to capacity.
- Read successful and failed latency separately because fast failures can make combined latency look better.

## Metric types

- A counter only increases until process restart: requests, errors, retries, checkouts, and returns. Query it with rate() or increase().
- A gauge moves both ways: active connections, queue depth, heap used, or consumer lag.
- A histogram counts observations in buckets and supports aggregatable p50, p95, and p99.
- p50 is the middle observation; p95 leaves 5% slower; p99 leaves 1% slower. No percentile is the maximum.
- Averages hide small severe cohorts. Maximums are noisy, so read both with volume and the histogram.

## Incident metric table

| Metric | High means | Low means | Sudden post-deploy change |
|---|---|---|---|
| Request rate | load or retries | traffic loss or blocking | caller/routing changed |
| Error ratio | failed outcome budget | not proof of business correctness | regression or incompatibility |
| p95/p99 | tail queueing, retry, dependency | observed cohort is faster | new slow path |
| CPU | compute pressure if sustained | work may be waiting | expensive code or traffic shape |
| Heap/GC | allocation or retention pressure | memory unlikely critical path | allocation regression |
| Worker queue | completion below demand | no executor backlog | workers blocked |
| Pool pending | callers cannot borrow | resource is available | leak, slow hold, or creation failure |
| Retry rate | original load amplified | retry not masking issue | policy/dependency regression |
| Telemetry drops | evidence is biased | pipeline healthier | collector/export regression |

## Prometheus queries

- Error ratio: `sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))`.
- p99: `histogram_quantile(0.99, sum by (le,uri) (rate(http_server_requests_seconds_bucket[5m])))`.
- Hikari utilization: `hikaricp_connections_active / hikaricp_connections_max` plus `hikaricp_connections_pending`.
- Counter divergence: `sum(increase(connection_checkouts_total[10m])) - sum(increase(connection_returns_total[10m]))`.
- Collector loss: inspect OTel received, exported, failed, retried, rejected, and dropped spans.
- Never average per-instance p99 values. Aggregate histogram buckets first.

## Spring Boot sources

- Micrometer instruments Spring MVC, WebClient, JVM, executors, HikariCP, and bounded business outcomes.
- `/actuator/prometheus` exposes current scrape data; Prometheus stores history and Grafana visualizes it.
- `/actuator/metrics` inspects meter names and tags locally.
- `/actuator/health` proves only configured contributors, not a real business transaction.
- `/actuator/threaddump` distinguishes RUNNABLE compute, WAITING pool acquisition, and BLOCKED monitor contention.
- `/actuator/heapdump` is sensitive and large; capture only with authorization and secure storage.

A safe Micrometer counter uses bounded outcome classes:

    Counter.builder("checkout.business.outcomes")
        .tag("outcome", outcomeClass)
        .register(meterRegistry)
        .increment();

Do not use traceId, user ID, order ID, message ID, raw URL, exception text, or SQL as metric labels. Those values create unbounded cardinality.
Grafana should annotate deployments and attach exemplars so a latency bucket opens the exact trace.

# Distributed Trace Investigation

OpenTelemetry models one distributed operation as a trace and each timed operation as a span.

## Incident trace

    traceId=health-bad-91
    Gateway SERVER                              5.18s ERROR
      Service A SERVER                          5.15s
        Service B CLIENT                        5.10s ERROR
          Service B SERVER                      5.10s ERROR
            validation.start EVENT                 2ms
            db.connection.acquire              4.90s ERROR
              pool.pending=186 active=40
            exception SQLTransientConnectionException

    Earlier leaking trace health-leak-44:
    Service B SERVER                              42ms ERROR
      db.connection.acquire                       3ms OK
      validation.failed EVENT ADDRESS_INVALID
      connection.return EVENT missing

## Reading the trace

- traceId identifies the distributed operation.
- spanId identifies one operation and correlates a precise log event.
- Parent-child means causal nesting; do not add durations of parallel children.
- A link relates async, replayed, batched, fan-in, or fan-out work without false nesting.
- Span kind is SERVER, CLIENT, PRODUCER, CONSUMER, or INTERNAL.
- Status records success or ERROR; a business failure can occur with HTTP 200.
- Attributes are indexed facts such as http.route, peer.service, db.system, release, and bounded outcome class.
- Events are timestamped facts inside a span, such as retry scheduled, exception, or queue wait.

## W3C propagation

- traceparent is `version-trace-id-parent-id-flags`, for example `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`.
- REST clients inject it into HTTP headers; servers extract it before creating SERVER spans.
- Kafka producers inject it into message headers; consumers extract it or create a span link.
- Thread-local context does not automatically cross executors, Reactor callbacks, or Kafka boundaries.
- Baggage propagates farther than logs. Allow-list only small non-sensitive values; never put PII or JWTs in it.

## Timing interpretation

- Client latency can include client queueing, pool acquisition, DNS, TCP, TLS, network, server work, and response transfer.
- Server latency starts after server instrumentation and may exclude caller pool and proxy wait.
- DB execution and DB connection acquisition are different operations and should not share one ambiguous span.
- External API spans need peer, operation, attempt, status, and remaining deadline without sensitive URLs.
- Retries need separate attempt spans or events, otherwise amplification is hidden.

## Missing child decision

A missing child can mean:

1. Propagation failed and another root was created.
2. Instrumentation did not create a span.
3. A span started but never ended because of a path or crash.
4. Head sampling rejected it before outcome.
5. Tail sampling did not retain it.
6. SDK, collector, network, or backend dropped it.
7. The operation never executed.

Use business state, logs, offsets, context headers, and started/ended/exported counters to distinguish them. Absence proves none by itself.

## Sampling and clock traps

- Head sampling is cheap but biased against rare slow outcomes not known at trace start.
- Tail sampling can retain errors and slow traces but consumes collector memory and wait time.
- Tail sampling cannot recover data already lost by head sampling or SDK drops.
- Sampled traces do not provide exact request rates.
- Clock skew can place a child before its parent. Check NTP offset and prefer durations plus causal IDs.
- Service maps show observed edges; a missing edge can be sampling or instrumentation loss.
- Exemplars join histogram observations to trace IDs.
- Trust trace completeness only after collector receive, queue, retry, export, reject, and drop metrics are healthy.

# Distributed Logs

Structured logs explain events at the location selected by metrics and traces.

## Incident records

    2026-09-13T17:40:00.100Z INFO service=service-b instance=b-6f91 release=B-2026.09.13.5 traceId=health-leak-44 spanId=b100 requestId=req-440 businessId=ord-440 endpoint=/reserve event=connection_checked_out pool_active=17 pool_idle=23
    2026-09-13T17:40:00.112Z WARN service=service-b instance=b-6f91 release=B-2026.09.13.5 traceId=health-leak-44 spanId=b100 requestId=req-440 businessId=ord-440 endpoint=/reserve event=validation_failed error=ADDRESS_INVALID latency_ms=12
    2026-09-13T17:44:59.100Z WARN service=service-b instance=b-6f91 release=B-2026.09.13.5 traceId=health-bad-91 spanId=acq91 requestId=req-991 businessId=ord-991 endpoint=/reserve event=connection_acquire_timeout error=SQLTransientConnectionException latency_ms=4900 pool_active=40 pool_idle=0 pool_pending=186
    2026-09-13T17:44:59.120Z WARN service=service-a instance=a-17ce traceId=health-bad-91 spanId=a200 requestId=req-991 businessId=ord-991 downstream=service-b endpoint=/reserve error=ReadTimeout latency_ms=5100 attempt=1
    2026-09-13T17:45:00.000Z INFO service=service-b instance=b-6f91 endpoint=/actuator/health status=200 health=UP latency_ms=18

## Correlation procedure

1. Start from the exact traceId in an exemplar, response, or failed span.
2. Restrict the UTC window and service, then group by spanId and instance.
3. Add requestId to distinguish edge attempts.
4. Add businessId to follow the workflow, but never use it as a metric label.
5. For Kafka add messageId, topic, partition, offset, consumer group, and attempt.
6. Account for clock skew and ingestion delay before ordering cross-host events.
7. Compare the event sequence with spans, metrics, code, and durable state.

## MDC and JSON

Spring MVC can put traceId, spanId, requestId, and a safe business ID into MDC, then clear them in finally:

    try (MDC.MDCCloseable ignored = MDC.putCloseable("requestId", requestId)) {
        filterChain.doFilter(request, response);
    }

Executor and Reactor boundaries need supported context propagation; ordinary ThreadLocal MDC can leak one request identity into another.
Use stable JSON fields: timestamp, level, service, instance, release, traceId, spanId, requestId, endpoint, event, error class, and latency_ms.

## Why one log is not proof

- A timeout log proves the caller observation, not whether network, queue, server, or deadline caused it.
- An exception may be fallout after the real resource exhausted.
- Retries generate multiple errors for one business operation.
- Clock skew and asynchronous shipping can reorder events.
- A present record can belong to another attempt; a missing one can have been filtered or dropped.
- Corroborate logs with metrics, trace structure, durable state, deployment timing, and repeatable tests.

## Missing-log path

Check code statement, effective level, filters, async appender queue, discard count, crash/flush, runtime file, collector input, parser, route, buffer/backpressure, exporter response, index rejection, time range, and retention.
A missing log does not prove no execution. A present log does not prove causal interpretation.

## Safety

- Never log authorization headers, JWTs, passwords, full bodies, card data, or raw PII.
- Hash or tokenize identifiers only when policy permits.
- Apply access and retention controls because correlation IDs can still be sensitive.

# Commands / Tools

Commands test one layer. No single result proves the root cause.

## Windows

- `Resolve-DnsName service-b` proves DNS returned records now; it does not prove TCP, TLS, HTTP, or business success.
- `Test-NetConnection service-b -Port 8080` tests TCP establishment only.
- `curl.exe -v --max-time 5 http://service-b:8080/actuator/health` tests one HTTP path, not the business path.
- `curl.exe -s http://localhost:8080/actuator/prometheus` reads current local metrics when authorized.
- `curl.exe -s http://localhost:8080/actuator/threaddump` captures thread states; compare repeated dumps.

## Linux

- `dig service-b` or `getent hosts service-b` tests name resolution, not a listener.
- `nc -vz service-b 8080` tests TCP, not TLS or HTTP correctness.
- `curl -v --max-time 5 http://service-b:8080/actuator/health` tests one HTTP exchange.
- `ss -lntp` shows local listeners when permitted; it does not prove remote firewall policy.
- `curl -s http://localhost:8080/actuator/prometheus | grep hikaricp_connections` shows current pool meters.

## Observability tools

- Prometheus: use rate for counters and histogram_quantile for buckets.
- Grafana: start at RED, retain the alert labels/time, and open an exemplar.
- Jaeger or Zipkin: search by service, operation, duration, status, and traceId.
- Logs: query exact structured fields such as `traceId=... AND service=...`, not broad text.
- Actuator endpoints must be authenticated and network-restricted.

## Pipeline proof

- Compare application emitted with collector received.
- Compare received with exported, failed, retried, and dropped.
- Compare backend accepted with query-visible data after ingestion delay.
- The first count gap locates loss; it does not by itself explain why.

# Root Cause

Evidence-supported causal chain:

    Release .5 moves a validation lookup before the managed transaction.
    |
    v
    Code checks out JDBC before validating.
    |
    v
    The validation-error early return skips close().
    |
    v
    Each error permanently removes one of 40 pool connections.
    |
    v
    At 40 leaks, idle is zero and 186 callers wait.
    |
    v
    Acquisition consumes 4.9 of B's 5.1 seconds.
    |
    v
    Fewer callers reach SQL, so DB query rate falls.
    |
    v
    Service A retries add 38% more attempts.
    |
    v
    The shallow health path never borrows this business resource.
    |
    v
    Health stays green while business traffic fails.

Each arrow is supported by timing, a metric or invariant, a trace or durable state, and correlated event evidence. Deployment timing alone is correlation, not proof.

# Fix

## Immediate mitigation

- Halt and roll back release .5.
- Mark bad instances unready, then drain before restart.

## Permanent correction

- Suppress retries that do not fit the remaining deadline.
- Use try-with-resources or framework-managed ownership.
- Validate before acquisition when DB state is unnecessary.
- Test validation failures against pool-accounting invariants.
- Do not increase pool size or timeout; that only delays exhaustion.

Do not blindly increase timeouts, pools, CPU, or memory. That is valid only when measurements show legitimate capacity demand rather than a leak, blocked dependency, or retry storm.

# Verification

Use the same signals that proved impact plus a targeted failure-path test.

- Business success returns above 99.7%.
- B p99 falls from 5.1 s below 420 ms.
- Worker queue falls from 612 below 10.
- Hikari active returns to workload baseline.
- Idle remains available and pending falls from 186 to zero.
- Acquisition p99 falls from 4.9 s below 20 ms.
- DB query rate recovers near 1,850/sec.
- Checkout and return deltas converge after drain.
- Validation load does not ratchet active upward.
- Retry amplification falls from 38% below 2%.

Compare canary with control for a full traffic cycle. Confirm recovery is not traffic loss, moved failures, changed sampling, or dropped logs.

# Prevention

- Define business SLIs in addition to liveness, readiness, and transport checks.
- Alert on RED symptoms and the nearest saturation signal with a runbook.
- Instrument bounded outcomes, retries, queues, pools, and telemetry drops with Micrometer.
- Propagate W3C traceparent through REST and Kafka and test it in CI.
- Use MDC JSON logs with trace, request, business, and message IDs where relevant.
- Keep IDs out of metric labels and PII out of logs, attributes, and baggage.
- Attach exemplars to latency histograms.
- Monitor trace and log collectors: receive, queue, retry, export, reject, and drop.
- Annotate Grafana with deployments and compare releases and instances.
- Use an end-to-end deadline, jitter, retry budget, and one retry owner.
- Test exceptions, rebalances, exporter outage, backpressure, and graceful shutdown.
- Gate canaries on business success, tail latency, saturation, and telemetry coverage.
- Synchronize clocks and alert on material offset.
- Use service maps as observed evidence, not proof of complete topology.
- Review cardinality before every metric label or indexed trace attribute.
- Alert when checkout-return divergence persists beyond in-flight work.
- Load-test validation errors and assert pool active returns to baseline.
- Keep readiness useful for traffic admission, but gate releases on business success.

# Interview Answer

### What I would say in an interview

I never treat health UP as proof that business functionality works. Here CPU was 29%, heap 56%, and GC 24 milliseconds, so I looked for waiting. The trace put 4.9 of Service B's 5.1 seconds in Hikari acquisition; the pool was 40 of 40 active, zero idle, and 186 pending. Falling DB query rate meant work was not reaching SQL. Checkout and return counters diverged only on validation errors after the deploy, and code review found an early return without close. We rolled back, drained bad instances, fixed ownership, bounded retries, and verified pool, queue, latency, success, query rate, and counter convergence.

### Common interviewer traps

- Saying health UP proves business success.
- Treating one log line as root-cause proof.
- Summing parallel span durations instead of reading the critical path.
- Assuming missing telemetry means no execution.
- Averaging p99 values across instances.
- Increasing resources or timeouts before finding the constrained resource.
- Ignoring sampling bias, clock skew, pipeline drops, cardinality, and PII.

### Quick memory flow

    Symptom
      |
      v
    Scope and business impact
      |
      v
    RED and golden signals
      |
      v
    USE and saturation
      |
      v
    Exemplar and trace critical path
      |
      v
    Correlated logs and durable state
      |
      v
    Root cause, fix, verify, prevent

# Interview Follow-up Questions

## Q1. What is the shortest signal distinction?

**Answer:** Metrics tell whether and how much; traces tell where; logs tell what happened.

## Q2. Why compare p50, p95, and p99?

**Answer:** Their separation reveals whether most requests or only a tail cohort is slow.

## Q3. What does a missing child span prove?

**Answer:** Only that the expected relation is absent; check execution, propagation, instrumentation, sampling, export, and drops.

## Q4. When is an OTel span link appropriate?

**Answer:** For async, replay, batch, fan-in, or fan-out work where strict nesting is misleading.

## Q5. How do you avoid cardinality incidents?

**Answer:** Use bounded labels; keep trace, user, order, message, raw URL, and exception text out of metric labels.

## Q6. How does head sampling bias evidence?

**Answer:** It decides before the outcome and can discard rare slow or failed traces.

## Q7. Why check clock skew?

**Answer:** Cross-host timestamps can misorder spans and logs, so duration and causal IDs need healthy clocks.

## Q8. How do you prove a fix?

**Answer:** Repeat the failure path and verify business outcome plus the causal metric while telemetry remains healthy.

## Q9. Why did DB query rate fall?

**Answer:** Requests waited before connection acquisition, so fewer could submit SQL; that was starvation, not DB recovery.

## Q10. How does checkout-return divergence show a leak?

**Answer:** Within one process lifetime, the difference should match active borrows; persistent outcome-specific unmatched borrows plus checkout stacks are strong evidence.
