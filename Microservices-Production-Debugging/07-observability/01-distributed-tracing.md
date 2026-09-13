# Problem

**Incident: Intermittent checkout latency isolated by a distributed trace**

Checkout p99 rises while median latency and host resources stay normal. The incident needs request-level causality because aggregate graphs cannot locate time inside a multi-service path.

Metrics say whether something is wrong and how much. Traces say where the path spent time or failed. Logs say what happened. Root cause requires agreement among signals and verification.

# Production Situation

At 2026-09-13 17:00 UTC the on-call sees:

- Traffic is 2,400 checkout requests/minute.
- Normal p50/p95/p99 is 180/420/850 ms; current is 190/1,800/6,400 ms.
- Error rate is 2.1%, mainly caller timeouts.
- Order CPU is 37%, heap is 61%, and GC p99 is 18 ms.
- Only the 7% fraud-review cohort is slow.
- Slow traces contain two Payment attempts and a 3.5-second fraud call.

The initial hypothesis stays broad. Green health or low CPU cannot eliminate waiting, correctness failure, retry amplification, or telemetry loss.

# Architecture

Relevant path:

    Client
      | HTTPS
      v
    Gateway
      | REST plus traceparent
      v
    Order Service
      +-- Inventory Service
      +-- Payment Service
            +-- Fraud Provider
            +-- MySQL

Every arrow is a runtime failure boundary and a context-propagation boundary.

# What I Check FIRST

The first checks are read-only, bounded, and designed to protect evidence.

## 1. Scope the tail

- WHAT: Scope the tail.
- WHY: Split route, status, instance, release, and bounded payment class.
- RESULT: p50 is flat while p99 rises only for fraud review.

## 2. Open a p99 exemplar

- WHAT: Open a p99 exemplar.
- WHY: A histogram exemplar selects a real slow request.
- RESULT: The trace matches the alert window and label set.

## 3. Read the critical path

- WHAT: Read the critical path.
- WHY: Serial spans explain elapsed time; parallel spans must not be added.
- RESULT: Two serial Payment attempts own nearly all 6.4 seconds.

## 4. Compare client and server spans

- WHAT: Compare client and server spans.
- WHY: The gap separates caller-side wait from callee work.
- RESULT: Payment client and server durations nearly agree.

## 5. Check retries and release

- WHAT: Check retries and release.
- WHY: A retry policy can amplify only the tail.
- RESULT: Release 2026.09.13.2 added a second retry owner.

# Step-by-Step Investigation

I change the next check when evidence changes; I do not jump from one error string to a fix.

### Step 1 - Confirm RED scope

- What I check: Query route rate, error ratio, and histogram percentiles.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Query route rate, error ratio, and histogram percentiles.`
- Expected result: p99 is 6.4 s while p50 is 190 ms.
- Different result: All percentiles rise.
- Meaning: Broad saturation is more likely; inspect USE metrics.
- Next check: Split by payment class and release.

### Step 2 - Choose a representative trace

- What I check: Open the p99 exemplar, not a convenient success.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Open the p99 exemplar, not a convenient success.`
- Expected result: The trace is from the affected route and minute.
- Different result: The exemplar is fast or stale.
- Meaning: The sample does not represent the alert cohort.
- Next check: Search by service, route, duration, status, and release.

### Step 3 - Read the critical path

- What I check: Follow serial parent-child spans and retry events.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Follow serial parent-child spans and retry events.`
- Expected result: Payment attempt 1 is 2.0 s, backoff 200 ms, attempt 2 is 3.8 s.
- Different result: Visible spans total only 700 ms.
- Meaning: Queueing, missing spans, or clock skew may fill the gap.
- Next check: Inspect span events, queues, and clocks.

### Step 4 - Compare client and server

- What I check: Match Order CLIENT span to Payment SERVER span.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Match Order CLIENT span to Payment SERVER span.`
- Expected result: 3.8 s client contains 3.6 s server.
- Different result: Client is 3.8 s but server is 300 ms.
- Meaning: Connection acquisition, proxy wait, or network time is outside server work.
- Next check: Inspect HTTP client and proxy telemetry.

### Step 5 - Inspect retry spans

- What I check: Require one child span per attempt with attempt.number.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Require one child span per attempt with attempt.number.`
- Expected result: Attempt 1 times out; attempt 2 succeeds.
- Different result: Only one broad client span exists.
- Meaning: Retry instrumentation is incomplete.
- Next check: Enable client instrumentation and bounded attempt attributes.

### Step 6 - Follow the slow child

- What I check: Open Payment and Fraud spans.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Open Payment and Fraud spans.`
- Expected result: Fraud owns 3.5 of Payment's 3.6 seconds.
- Different result: Payment is slow with no child.
- Meaning: The external call is uninstrumented or local queueing dominates.
- Next check: Check Payment logs, pools, and exporter health.

### Step 7 - Correlate logs

- What I check: Search exact traceId, requestId, and payment business ID.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Search exact traceId, requestId, and payment business ID.`
- Expected result: Logs confirm timeout, backoff, and second attempt.
- Different result: A log claims a different attempt count.
- Meaning: The message may count the initial call or another request.
- Next check: Verify spanId and requestId before concluding.

### Step 8 - Check sampling

- What I check: Inspect head decision, tail rules, and retained counts.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Inspect head decision, tail rules, and retained counts.`
- Expected result: Errors and traces over 2 s are retained.
- Different result: Only 1% head sampling is active.
- Meaning: Rare failures may be absent before outcome is known.
- Next check: Use a bounded tail rule and watch collector capacity.

### Step 9 - Check clocks and export

- What I check: Compare time offset and span receive/export/drop counters.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare time offset and span receive/export/drop counters.`
- Expected result: Clock offset is under 20 ms and drops are zero.
- Different result: Children precede parents or queue drops rise.
- Meaning: The waterfall or trace set is incomplete.
- Next check: Fix time or pipeline health before interpretation.

### Step 10 - Canary the retry fix

- What I check: Disable duplicate retry ownership on one instance.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Disable duplicate retry ownership on one instance.`
- Expected result: Canary p99 falls below 900 ms and attempts become one.
- Different result: Latency stays high.
- Meaning: Fraud or another branch remains causal.
- Next check: Compare fresh canary critical paths.

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

    traceId=4bf92f3577b34da6a3ce929d0e0e4736
    Gateway SERVER                         6.40s span=1001
      Order SERVER                        6.10s span=2001 parent=1001
        Inventory CLIENT                    90ms span=2101
        Payment CLIENT attempt=1          2.00s span=2201 ERROR
        retry.backoff EVENT                200ms
        Payment CLIENT attempt=2          3.80s span=2202 OK
          Payment SERVER                  3.60s span=3001 parent=2202
            Fraud CLIENT                  3.50s span=3101
            MySQL SELECT                    40ms span=3201

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

    2026-09-13T16:55:21.100Z INFO service=order-service instance=order-7f9d traceId=4bf92f3577b34da6a3ce929d0e0e4736 spanId=2001 requestId=req-781 businessId=ord-5501 endpoint=/checkout event=payment_start
    2026-09-13T16:55:23.102Z WARN service=order-service instance=order-7f9d traceId=4bf92f3577b34da6a3ce929d0e0e4736 spanId=2201 requestId=req-781 downstream=payment-service error=ReadTimeout latency_ms=2000 attempt=1
    2026-09-13T16:55:26.902Z INFO service=payment-service instance=payment-64bc traceId=4bf92f3577b34da6a3ce929d0e0e4736 spanId=3001 requestId=req-781 downstream=fraud-provider event=score_complete latency_ms=3500
    2026-09-13T16:55:27.500Z INFO service=order-service instance=order-7f9d traceId=4bf92f3577b34da6a3ce929d0e0e4736 spanId=2001 requestId=req-781 endpoint=/checkout status=201 latency_ms=6400

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

    Payment release enabled a retry around a client that already retried.
    |
    v
    The first fraud call reaches a two-second timeout.
    |
    v
    Order waits through backoff and starts another Payment call.
    |
    v
    Serial attempts extend the fraud cohort to 6.4 seconds.
    |
    v
    Retries increase offered load and caller timeouts.

Each arrow is supported by timing, a metric or invariant, a trace or durable state, and correlated event evidence. Deployment timing alone is correlation, not proof.

# Fix

## Immediate mitigation

- Roll back duplicate retry ownership.
- Keep one jittered retry only when it fits the remaining deadline.

## Permanent correction

- Instrument each attempt and retry reason.
- Use an idempotency key so retries cannot double charge.
- Set the Fraud timeout from the end-to-end budget.

Do not blindly increase timeouts, pools, CPU, or memory. That is valid only when measurements show legitimate capacity demand rather than a leak, blocked dependency, or retry storm.

# Verification

Use the same signals that proved impact plus a targeted failure-path test.

- Checkout p99 falls from 6.4 s below 900 ms.
- Fraud cohort p99 falls below 1.1 s.
- Retries fall from 1,900/min below 30/min.
- Fresh traces show one normal Payment attempt.
- Error rate stays below 0.3% for thirty minutes.

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

# Interview Answer

### What I would say in an interview

I start with RED metrics, then open a real p99 exemplar. I read the trace as a critical path, compare caller client spans with callee server spans, and look for retries and gaps. Here two serial Payment attempts plus a 3.5-second fraud call explained the tail while CPU and GC were normal. Logs with the same trace and request IDs confirmed the timeout sequence but were not proof alone. We removed duplicate retry ownership and verified p99, errors, retry count, and fresh traces returned to baseline.

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
