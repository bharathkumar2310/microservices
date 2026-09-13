# Problem

**Incident: A trace disappears at a Kafka boundary**

An HTTP request publishes a Kafka event, and fulfillment completes, but the original trace ends at the producer. A missing child is an observability symptom until execution, propagation, instrumentation, sampling, export, and drop paths are tested.

Metrics say whether something is wrong and how much. Traces say where the path spent time or failed. Logs say what happened. Root cause requires agreement among signals and verification.

# Production Situation

At 2026-09-13 17:00 UTC the on-call sees:

- Order accepts 3,200 requests/minute.
- REST traces are 98% complete, but only 34% connect to fulfillment.
- Consumer throughput proves workers are active.
- Producer spans export at 3,180/minute; linked consumer spans fell to 1,090/minute.
- Collector rejected spans are zero and queue utilization is 22%.
- Messages retain messageId but lost traceparent after release 4.7.0.
- Consumers create unrelated root traces.

The initial hypothesis stays broad. Green health or low CPU cannot eliminate waiting, correctness failure, retry amplification, or telemetry loss.

# Architecture

Relevant path:

    Client
      v
    Order Service
      | PRODUCE traceparent plus messageId
      v
    Kafka order-accepted
      | CONSUME with context or link
      v
    Fulfillment Service
      v
    Warehouse API

Every arrow is a runtime failure boundary and a context-propagation boundary.

# What I Check FIRST

The first checks are read-only, bounded, and designed to protect evidence.

## 1. Prove execution separately

- WHAT: Prove execution separately.
- WHY: Missing telemetry does not prove missing business work.
- RESULT: Message ID, offset, and fulfillment row exist.

## 2. Find the last known span

- WHAT: Find the last known span.
- WHY: The first absent edge narrows the break.
- RESULT: Producer exists; consumer relation is absent.

## 3. Inspect Kafka headers

- WHAT: Inspect Kafka headers.
- WHY: Async context travels in message headers.
- RESULT: traceparent is missing while messageId remains.

## 4. Check instrumentation and sampling

- WHAT: Check instrumentation and sampling.
- WHY: A consumer can run without recording a span.
- RESULT: Listener instrumentation and sampler counters are inspected.

## 5. Reconcile export stages

- WHAT: Reconcile export stages.
- WHY: A created span can be lost later.
- RESULT: Started, ended, queued, exported, and dropped counts reconcile.

# Step-by-Step Investigation

I change the next check when evidence changes; I do not jump from one error string to a fix.

### Step 1 - Choose a known request

- What I check: Capture traceId, order ID, message ID, partition, and offset.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Capture traceId, order ID, message ID, partition, and offset.`
- Expected result: Producer span and log agree.
- Different result: No message ID exists.
- Meaning: Async recovery is harder.
- Next check: Use broker and business audit data.

### Step 2 - Prove publication

- What I check: Check producer acknowledgment and broker offset.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Check producer acknowledgment and broker offset.`
- Expected result: Partition 8 offset 44120 is acknowledged.
- Different result: Only send_start exists.
- Meaning: Publication may have failed.
- Next check: Check callback and producer errors.

### Step 3 - Prove consumption

- What I check: Find durable state and logs by message ID.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Find durable state and logs by message ID.`
- Expected result: Fulfillment row f-991 exists.
- Different result: Lag rises and no evidence exists.
- Meaning: This may be a processing outage.
- Next check: Investigate assignment and consumer errors.

### Step 4 - Inspect W3C headers

- What I check: Compare sanitized producer output and consumer input.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare sanitized producer output and consumer input.`
- Expected result: traceparent is absent after serialization.
- Different result: traceparent arrives valid.
- Meaning: Extraction or later stages are suspect.
- Next check: Inspect consumer propagator.

### Step 5 - Validate syntax

- What I check: Check version, hexadecimal lengths, nonzero IDs, and flags.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Check version, hexadecimal lengths, nonzero IDs, and flags.`
- Expected result: A valid header extracts.
- Different result: Header is malformed.
- Meaning: The propagator correctly starts a root.
- Next check: Fix injection and count invalid context.

### Step 6 - Check instrumentation

- What I check: Confirm Spring Kafka listener creates and ends CONSUMER spans.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Confirm Spring Kafka listener creates and ends CONSUMER spans.`
- Expected result: One span ends per delivery.
- Different result: Custom listener bypasses instrumentation.
- Meaning: No span exists to export.
- Next check: Add supported interceptor or manual span.

### Step 7 - Check links

- What I check: Search consumer roots for links to producer context.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Search consumer roots for links to producer context.`
- Expected result: Async trace has the correct link.
- Different result: Analyst searches only direct children.
- Meaning: Data exists under link semantics.
- Next check: Use link-aware search.

### Step 8 - Check sampling

- What I check: Read trace flags and SDK sampling counters.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Read trace flags and SDK sampling counters.`
- Expected result: Consumer honors sampled context.
- Different result: Head sampler rejects it.
- Meaning: Rare failures can disappear early.
- Next check: Align policy or use bounded tail sampling.

### Step 9 - Check export loss

- What I check: Reconcile started, ended, queued, exported, failed, and dropped.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Reconcile started, ended, queued, exported, failed, and dropped.`
- Expected result: Counts reconcile.
- Different result: Started exceeds ended or queue drops rise.
- Meaning: Span leak, crash, or exporter pressure exists.
- Next check: Fix the first divergent boundary.

### Step 10 - Canary propagation

- What I check: Publish a synthetic message through 4.7.1.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Publish a synthetic message through 4.7.1.`
- Expected result: Consumer link and Warehouse child appear.
- Different result: Header arrives but child is absent.
- Meaning: Extraction or instrumentation remains broken.
- Next check: Continue at the next boundary.

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

    Expected:
    traceId=miss-100 Order SERVER
      Kafka PRODUCE span=p100
      Fulfillment CONSUME span=c100 or link=p100
        Warehouse CLIENT span=w100

    Observed:
    traceId=miss-100 Order SERVER
      Kafka PRODUCE span=p100

    traceId=new-900 Fulfillment CONSUME span=c900 parent=none link=none
      Warehouse CLIENT span=w900

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

    2026-09-13T17:20:00.100Z INFO service=order-service instance=order-21aa traceId=miss-100 spanId=p100 requestId=req-700 businessId=ord-700 messageId=msg-700 topic=order-accepted partition=8 offset=44120 event=publish_ack
    2026-09-13T17:20:00.240Z WARN service=fulfillment-service instance=fulfill-61bd traceId=new-900 spanId=c900 businessId=ord-700 messageId=msg-700 topic=order-accepted partition=8 offset=44120 event=context_extract propagation=missing_traceparent
    2026-09-13T17:20:00.410Z INFO service=fulfillment-service instance=fulfill-61bd traceId=new-900 spanId=c900 businessId=ord-700 messageId=msg-700 event=fulfilled latency_ms=170

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

    Release 4.7.0 replaces the Kafka header mapper.
    |
    v
    Its allow-list keeps business headers but omits traceparent and tracestate.
    |
    v
    Producer telemetry remains healthy through Kafka publication.
    |
    v
    Consumer context extraction has no remote parent.
    |
    v
    Consumer creates an unrelated root with no link.
    |
    v
    Searching the original traceId hides valid downstream work.

Each arrow is supported by timing, a metric or invariant, a trace or durable state, and correlated event evidence. Deployment timing alone is correlation, not proof.

# Fix

## Immediate mitigation

- Restore safe W3C headers in the mapper.
- Inject and extract with the same OTel propagator.

## Permanent correction

- Create a consumer span or link appropriate to async work.
- Count missing and invalid context by bounded boundary.
- Retain messageId and businessId for fallback correlation.

Do not blindly increase timeouts, pools, CPU, or memory. That is valid only when measurements show legitimate capacity demand rather than a leak, blocked dependency, or retry storm.

# Verification

Use the same signals that proved impact plus a targeted failure-path test.

- Synthetic messages retain valid context.
- Linked coverage rises from 34% above 98%.
- Sampled producer and consumer rates reconcile.
- Collector drop and reject counters remain zero.
- TraceId, messageId, and link searches find Warehouse.

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

A missing span does not prove code did not run. I prove business execution with message IDs, offsets, logs, and state, then walk propagation, extraction, instrumentation, span end, sampling, SDK queue, collector, and backend. Here a Kafka header-mapper release removed W3C traceparent, so the consumer ran under an unrelated root. We restored safe propagation and links, then verified coverage and export/drop counters with a synthetic message.

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
