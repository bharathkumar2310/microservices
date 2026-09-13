# Problem

**Incident: Duplicate shipment reconstructed from correlated logs**

One order creates two shipments. The async trace is incomplete, so structured logs and durable identifiers must reconstruct whether the producer duplicated an event, Kafka replayed it, or the log pipeline duplicated records.

Metrics say whether something is wrong and how much. Traces say where the path spent time or failed. Logs say what happened. Root cause requires agreement among signals and verification.

# Production Situation

At 2026-09-13 17:00 UTC the on-call sees:

- Orders arrive at 8,000 messages/minute.
- Duplicate shipment rate rose from 0.01% to 0.8%.
- Consumer lag peaked at 48,000 during a restart.
- Processing p95 is 240 ms and CPU is 42%.
- Order ord-8842 has message msg-a91 and two shipment rows.
- The same topic-partition-offset ran on two instances around a rebalance.

The initial hypothesis stays broad. Green health or low CPU cannot eliminate waiting, correctness failure, retry amplification, or telemetry loss.

# Architecture

Relevant path:

    Order Service
      | Kafka order-created
      | traceparent plus messageId
      v
    Shipping Consumer Group
      +-- Shipment MySQL
      +-- Notification Service
      v
    Kafka offset commit

Every arrow is a runtime failure boundary and a context-propagation boundary.

# What I Check FIRST

The first checks are read-only, bounded, and designed to protect evidence.

## 1. Protect customers

- WHAT: Protect customers.
- WHY: Enable the existing idempotency guard or pause the affected key.
- RESULT: New duplicate side effects stop.

## 2. Collect stable IDs

- WHAT: Collect stable IDs.
- WHY: Use business ID, message ID, topic, partition, offset, trace and request IDs.
- RESULT: The same delivery can be followed across instances.

## 3. Build one timeline

- WHAT: Build one timeline.
- WHY: A narrow event history is better than searching ERROR globally.
- RESULT: Receive, write, notify, crash, rebalance, replay are ordered.

## 4. Check consumer lifecycle

- WHAT: Check consumer lifecycle.
- WHY: At-least-once replay follows uncommitted work.
- RESULT: The first process died after side effects but before offset commit.

## 5. Check telemetry duplication

- WHAT: Check telemetry duplication.
- WHY: A duplicated index record is not duplicate execution.
- RESULT: Different spans and two durable rows prove two executions.

# Step-by-Step Investigation

I change the next check when evidence changes; I do not jump from one error string to a fix.

### Step 1 - Confirm the duplicate

- What I check: Count shipment rows by order and message ID.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Count shipment rows by order and message ID.`
- Expected result: One message ID produced two rows.
- Different result: Two message IDs exist.
- Meaning: Producer duplication or legitimate second intent is possible.
- Next check: Inspect the producer outbox.

### Step 2 - Build correlation keys

- What I check: Collect messageId, businessId, topic, partition, offset, traceId, spanId.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Collect messageId, businessId, topic, partition, offset, traceId, spanId.`
- Expected result: Every record groups without free-text guesses.
- Different result: Some boundaries lack messageId.
- Meaning: Async correlation is broken.
- Next check: Inspect Kafka header mapping and MDC setup.

### Step 3 - Order the timeline

- What I check: Sort UTC events but preserve offset and elapsed time.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Sort UTC events but preserve offset and elapsed time.`
- Expected result: Write and notification precede crash and replay.
- Different result: Events appear out of order.
- Meaning: Clock skew or shipping delay affects wall time.
- Next check: Use offset, span events, and elapsed_ms.

### Step 4 - Inspect first delivery

- What I check: Find the first instance's receive, insert, notify, and commit events.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Find the first instance's receive, insert, notify, and commit events.`
- Expected result: Side effects finish before commit starts.
- Different result: No insert log exists although the row exists.
- Meaning: The log may be lost or another code path wrote it.
- Next check: Use DB audit and collector counters.

### Step 5 - Inspect rebalance

- What I check: Correlate SIGKILL, partition revoke, and group assignment.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Correlate SIGKILL, partition revoke, and group assignment.`
- Expected result: Partition moves before a commit success.
- Different result: Commit success is present.
- Meaning: The async commit may still fail or another event exists.
- Next check: Check the broker's committed offset.

### Step 6 - Inspect replay

- What I check: Find the same topic-partition-offset on the new owner.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Find the same topic-partition-offset on the new owner.`
- Expected result: Offset 991044 runs on both instances.
- Different result: Offsets differ.
- Meaning: Producer duplication is more likely.
- Next check: Compare payload hash and outbox rows.

### Step 7 - Test idempotency

- What I check: Check for a unique message_id in the side-effect transaction.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Check for a unique message_id in the side-effect transaction.`
- Expected result: No unique key exists; check-then-insert is non-atomic.
- Different result: A unique key rejects the second insert.
- Meaning: Notification may still be outside the guard.
- Next check: Trace notification idempotency.

### Step 8 - Correlate traces

- What I check: Use a span link from consumer work to producer context.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Use a span link from consumer work to producer context.`
- Expected result: Replay trace links to the producer and includes offset.
- Different result: Consumer creates an unrelated root.
- Meaning: Trace discovery is weak, though IDs still work.
- Next check: Propagate traceparent and add an OTel link.

### Step 9 - Exclude log-pipeline duplicates

- What I check: Compare log event ID, instance, span, and durable rows.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare log event ID, instance, span, and durable rows.`
- Expected result: Events have different spans and two rows exist.
- Different result: Identical event IDs appear twice but one row exists.
- Meaning: The pipeline duplicated logs only.
- Next check: Fix telemetry deduplication separately.

### Step 10 - Reproduce the crash window

- What I check: Kill a test consumer after DB commit and before offset commit.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Kill a test consumer after DB commit and before offset commit.`
- Expected result: Replay occurs but idempotency blocks a second side effect.
- Different result: A second row appears.
- Meaning: The guard is not atomic.
- Next check: Move it into the business transaction.

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

    producer traceId=aa11 span=0101
    Order PRODUCE topic=order-created 12ms
      header messageId=msg-a91 traceparent=aa11/0101

    consumer traceId=bb22 span=0201 link=aa11/0101
    Shipping CONSUME partition=12 offset=991044 180ms
      MySQL INSERT shipment 35ms OK
      Notification POST 70ms OK
      offset commit EVENT missing after SIGKILL

    replay traceId=cc33 span=0301 link=aa11/0101
    Shipping CONSUME partition=12 offset=991044 175ms

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

    2026-09-13T17:01:00.100Z INFO service=shipping-service instance=shipping-2a11 traceId=bb22 spanId=0201 messageId=msg-a91 businessId=ord-8842 topic=order-created partition=12 offset=991044 event=shipment_inserted shipmentId=ship-701
    2026-09-13T17:01:00.172Z INFO service=shipping-service instance=shipping-2a11 traceId=bb22 spanId=0201 messageId=msg-a91 businessId=ord-8842 event=notification_sent
    2026-09-13T17:01:00.180Z WARN service=shipping-service instance=shipping-2a11 traceId=bb22 spanId=0201 messageId=msg-a91 event=consumer_revoked reason=pod_terminated
    2026-09-13T17:01:01.004Z INFO service=shipping-service instance=shipping-8c32 traceId=cc33 spanId=0301 messageId=msg-a91 businessId=ord-8842 topic=order-created partition=12 offset=991044 event=message_received delivery=2

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

    The first consumer commits shipment and notification side effects.
    |
    v
    It is killed before Kafka records the offset commit.
    |
    v
    Kafka reassigns the partition and redelivers the same offset.
    |
    v
    No atomic unique message ID guards the business transaction.
    |
    v
    The second consumer repeats the customer-visible work.

Each arrow is supported by timing, a metric or invariant, a trace or durable state, and correlated event evidence. Deployment timing alone is correlation, not proof.

# Fix

## Immediate mitigation

- Enable a temporary duplicate-side-effect guard.
- Add a unique processed_message.message_id constraint in the shipment transaction.

## Permanent correction

- Treat duplicate key as already processed.
- Publish notification through a transactional outbox.
- Retain message ID and topic-partition-offset in structured logs.

Do not blindly increase timeouts, pools, CPU, or memory. That is valid only when measurements show legitimate capacity demand rather than a leak, blocked dependency, or retry storm.

# Verification

Use the same signals that proved impact plus a targeted failure-path test.

- Replay offset 991044 and keep one shipment row.
- Notification count remains one.
- The idempotency-conflict counter increments once.
- Lag returns below 2,000.
- Duplicate rate remains below 0.01% through a controlled restart.

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

I reconstruct async incidents with stable IDs: business ID, message ID, topic-partition-offset, traceId, and spanId. Here logs showed the same offset on two instances around a rebalance. The first completed the database and notification side effects but died before offset commit. Kafka therefore redelivered it, and no atomic idempotency key existed. We added a unique message ID guard in the business transaction and an outbox, then repeated the crash window and verified one shipment and one notification.

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
