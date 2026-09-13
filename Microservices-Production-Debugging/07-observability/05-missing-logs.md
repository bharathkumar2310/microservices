# Problem

**Incident: Business errors occur but central logs are missing**

The API error alert fires and traces contain exceptions, but central log search is nearly empty. The engineer must distinguish no statement, log level, filter, backpressure, crash, collector parse, export, index, and query failures.

Metrics say whether something is wrong and how much. Traces say where the path spent time or failed. Logs say what happened. Root cause requires agreement among signals and verification.

# Production Situation

At 2026-09-13 17:00 UTC the on-call sees:

- Refund handles 1,100 requests/minute.
- HTTP 500 rises from 0.1% to 7.6%.
- Traces contain 84 failed spans in five minutes.
- Central search shows 11 matching ERROR records.
- The app reports 89 ERROR events emitted.
- Fluent Bit receives 91 records but parser_drop_total rises by 78.
- Release 2.9.0 changed timestamp from ISO text to an epoch number.

The initial hypothesis stays broad. Green health or low CPU cannot eliminate waiting, correctness failure, retry amplification, or telemetry loss.

# Architecture

Relevant path:

    Refund Service
      | JSON stdout
      v
    Container runtime log
      v
    Fluent Bit
      v
    Buffer and log gateway
      v
    Central index
      v
    Search and alert

    OTel traces use a separate pipeline

Every arrow is a runtime failure boundary and a context-propagation boundary.

# What I Check FIRST

The first checks are read-only, bounded, and designed to protect evidence.

## 1. Confirm errors independently

- WHAT: Confirm errors independently.
- WHY: Metrics and traces prevent missing logs from hiding impact.
- RESULT: Failed spans and HTTP counters agree.

## 2. Prove app emission

- WHAT: Prove app emission.
- WHY: A logger counter or safe local sample separates code from pipeline.
- RESULT: The app emitted 89 ERROR events.

## 3. Walk pipeline boundaries

- WHAT: Walk pipeline boundaries.
- WHY: Received, parsed, buffered, exported, indexed, and dropped counts locate loss.
- RESULT: Parser drops explain most missing records.

## 4. Check query time and fields

- WHAT: Check query time and fields.
- WHY: Timezone, index, service, or spelling can hide present data.
- RESULT: UTC window and service.name are correct.

## 5. Protect data

- WHAT: Protect data.
- WHY: Debugging is not permission to dump bodies or tokens.
- RESULT: Only safe IDs and error classes are emitted.

# Step-by-Step Investigation

I change the next check when evidence changes; I do not jump from one error string to a fix.

### Step 1 - Estimate expected volume

- What I check: Compare failed spans with terminal-log policy.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare failed spans with terminal-log policy.`
- Expected result: About 84 terminal errors are expected.
- Different result: Policy samples repeated errors.
- Meaning: A lower count may be designed.
- Next check: Read logger policy.

### Step 2 - Check the code path

- What I check: Verify ControllerAdvice reaches a terminal log.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Verify ControllerAdvice reaches a terminal log.`
- Expected result: The failed path calls logger.error once.
- Different result: 500 is returned silently.
- Meaning: No pipeline can deliver an uncreated event.
- Next check: Add a safe terminal event.

### Step 3 - Check effective level

- What I check: Read runtime package logger levels.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Read runtime package logger levels.`
- Expected result: ERROR is enabled.
- Different result: Package is OFF.
- Meaning: Suppression happens before appenders.
- Next check: Correct the bounded override.

### Step 4 - Check filters

- What I check: Inspect TurboFilter and appender decisions.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Inspect TurboFilter and appender decisions.`
- Expected result: Accepted=89 and rejected=0.
- Different result: A marker rule rejects events.
- Meaning: The app filter drops them.
- Next check: Fix rule and safe defaults.

### Step 5 - Check async appender

- What I check: Read queue utilization, drops, block time, and flush.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Read queue utilization, drops, block time, and flush.`
- Expected result: Queue is 18% and ERROR drops are zero.
- Different result: Queue is full.
- Meaning: Backpressure discards or blocks events.
- Next check: Protect ERROR and size from bursts.

### Step 6 - Check crash loss

- What I check: Align pod termination with nonempty buffers.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Align pod termination with nonempty buffers.`
- Expected result: No restarts occurred.
- Different result: SIGKILL occurs with queued events.
- Meaning: logger.error does not guarantee durable output.
- Next check: Use graceful flush and durable audit storage.

### Step 7 - Check collector input

- What I check: Compare runtime bytes with collector received records.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare runtime bytes with collector received records.`
- Expected result: Collector receives 91 records.
- Different result: Runtime has data but input is zero.
- Meaning: Tail state, rotation, or permissions failed.
- Next check: Inspect node collector state.

### Step 8 - Check parser and route

- What I check: Inspect parse success/drop and route counts.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Inspect parse success/drop and route counts.`
- Expected result: Seventy-eight numeric timestamps are rejected.
- Different result: Parsing succeeds but output falls.
- Meaning: Routing or buffering is next.
- Next check: Inspect output queue.

### Step 9 - Check backend ingestion

- What I check: Compare output acknowledgments with indexed documents and lag.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare output acknowledgments with indexed documents and lag.`
- Expected result: Accepted records appear within 12 s.
- Different result: Index rejects or retries rise.
- Meaning: Schema, quota, auth, or backend failed.
- Next check: Inspect rejection reason and dead letter.

### Step 10 - Canary the fix

- What I check: Send safe known INFO and ERROR records.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Send safe known INFO and ERROR records.`
- Expected result: Both appear once with traceId within SLO.
- Different result: Only one severity appears.
- Meaning: Routing remains wrong.
- Next check: Validate severity and required fields.

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

    traceId=log-884 Refund SERVER 420ms ERROR
      Auth INTERNAL 8ms OK
      MySQL UPDATE 35ms OK
      Ledger CLIENT 310ms ERROR
        exception EVENT type=LedgerRejected
      exception EVENT type=RefundFailed

    The trace proves the request failed.
    It does not prove logger.error reached the index.

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

    Application stdout before parsing:
    {"timestamp":1789319400100,"level":"ERROR","service":"refund-service","instance":"refund-7dd","traceId":"log-884","spanId":"rf-100","requestId":"req-884","businessId":"refund-771","endpoint":"/refunds","downstream":"ledger-service","error":"LedgerRejected","latency_ms":420}
    Collector diagnostic:
    2026-09-13T17:30:00.200Z ERROR service=fluent-bit instance=node-22 input=containers parser=app-json error=timestamp_type_mismatch expected=string actual=number source_service=refund-service dropped=1
    After fix:
    2026-09-13T17:31:00.100Z ERROR service=refund-service instance=refund-7dd traceId=log-885 spanId=rf-101 requestId=req-885 businessId=refund-772 endpoint=/refunds downstream=ledger-service error=LedgerRejected latency_ms=405

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

    Release 2.9.0 changes timestamp from ISO text to epoch milliseconds.
    |
    v
    The collector parser contract still requires text.
    |
    v
    The application emits ERROR and the separate trace pipeline works.
    |
    v
    Fluent Bit rejects records before routing.
    |
    v
    Only old-version pod records reach the index.
    |
    v
    Search cannot recover records dropped before ingestion.

Each arrow is supported by timing, a metric or invariant, a trace or durable state, and correlated event evidence. Deployment timing alone is correlation, not proof.

# Fix

## Immediate mitigation

- Restore the agreed ISO-8601 UTC schema.
- Route invalid records to a bounded dead-letter sink.

## Permanent correction

- Contract-test and version the log schema.
- Alert on parse drops, retries, buffers, and ingestion delay.
- Keep audit truth in durable business storage, not operational logs.

Do not blindly increase timeouts, pools, CPU, or memory. That is valid only when measurements show legitimate capacity demand rather than a leak, blocked dependency, or retry storm.

# Verification

Use the same signals that proved impact plus a targeted failure-path test.

- Emitted and received ERROR counts reconcile.
- Parser drops stay zero.
- Known trace IDs appear within 30 seconds.
- Terminal log count matches failed spans under policy.
- No PII, JWT, or payload was added.

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

I first prove the incident with independent metrics and traces. Then I follow each log event through code, effective level, filters, async queue, stdout, node collector, parser, buffer, exporter, and index. Here the app emitted 89 errors and Fluent Bit received them, but parser drops rose because a deployment changed timestamp type. We restored the schema, added dead-letter handling and pipeline alerts, and verified known trace IDs appeared within the ingestion SLO.

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
