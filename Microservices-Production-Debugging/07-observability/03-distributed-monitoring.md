# Problem

**Incident: Search failures found with RED, USE, and golden signals**

Catalog search errors rise while total traffic and JVM resources look normal. Monitoring must quantify customer impact, locate saturation, and provide a representative request instead of becoming a collection of unrelated product graphs.

Metrics say whether something is wrong and how much. Traces say where the path spent time or failed. Logs say what happened. Root cause requires agreement among signals and verification.

# Production Situation

At 2026-09-13 17:00 UTC the on-call sees:

- Catalog receives 6,500 requests/minute, unchanged from baseline.
- Search 5xx rises from 0.2% to 11.4%.
- p50/p95/p99 changes from 90/240/480 ms to 95/1,900/3,200 ms.
- Catalog CPU is 46%, heap 63%, and GC p99 21 ms.
- Elasticsearch client connections are 100/100 with 740 pending.
- Elasticsearch rejections rise from zero to 420/minute.
- Redis, MySQL, and Kafka signals remain normal.

The initial hypothesis stays broad. Green health or low CPU cannot eliminate waiting, correctness failure, retry amplification, or telemetry loss.

# Architecture

Relevant path:

    Web clients
      v
    Gateway
      v
    Catalog Service
      +-- Redis
      +-- Elasticsearch
      +-- Product MySQL

    Micrometer -> Prometheus -> Grafana
    OTel SDK -> Collector -> Trace backend

Every arrow is a runtime failure boundary and a context-propagation boundary.

# What I Check FIRST

The first checks are read-only, bounded, and designed to protect evidence.

## 1. Confirm business impact

- WHAT: Confirm business impact.
- WHY: Start from route success, not host graphs.
- RESULT: Only catalog search success falls.

## 2. Apply RED

- WHAT: Apply RED.
- WHY: Rate, errors, and duration define scope and onset.
- RESULT: Rate is flat while errors and tail latency rise.

## 3. Apply USE

- WHAT: Apply USE.
- WHY: Utilization, saturation, and errors identify constrained resources.
- RESULT: Elasticsearch pool saturation and rejections rise.

## 4. Compare dependencies

- WHAT: Compare dependencies.
- WHY: Normal peers eliminate broad theories.
- RESULT: Redis, MySQL, CPU, heap, and GC remain normal.

## 5. Open an exemplar

- WHAT: Open an exemplar.
- WHY: A p99 exemplar connects the aggregate to one request.
- RESULT: The trace has a 2.9-second rejected Elasticsearch span.

# Step-by-Step Investigation

I change the next check when evidence changes; I do not jump from one error string to a fix.

### Step 1 - Validate the alert

- What I check: Compare raw counter increases, ratios, targets, and restarts.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare raw counter increases, ratios, targets, and restarts.`
- Expected result: 11.4% persists across replicas and two windows.
- Different result: One restarted target creates the spike.
- Meaning: Counter reset can imitate a rate change.
- Next check: Check process start and increase().

### Step 2 - Slice RED

- What I check: Group by route, status, region, instance, and release.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Group by route, status, region, instance, and release.`
- Expected result: Only /catalog/search in one region fails.
- Different result: Every route fails.
- Meaning: A gateway or shared runtime is more likely.
- Next check: Check gateway RED and app USE.

### Step 3 - Read distributions

- What I check: Compare p50, p95, p99, max, and volume.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Compare p50, p95, p99, max, and volume.`
- Expected result: p50 is flat and tails rise.
- Different result: All percentiles rise.
- Meaning: A common step affects most requests.
- Next check: Split by cache result and query class.

### Step 4 - Inspect service USE

- What I check: Check CPU, GC, threads, queues, and pools.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Check CPU, GC, threads, queues, and pools.`
- Expected result: App resources are normal; search pending is 740.
- Different result: CPU is 98% with runnable threads.
- Meaning: Compute pressure may be causal.
- Next check: Profile hot code.

### Step 5 - Inspect dependency signals

- What I check: Read Elasticsearch traffic, errors, duration, queue, and rejection.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Read Elasticsearch traffic, errors, duration, queue, and rejection.`
- Expected result: Throughput plateaus while queue and rejection rise.
- Different result: Server looks normal but client pending rises.
- Meaning: Client pool or network wait is possible.
- Next check: Compare client/server spans.

### Step 6 - Use a p99 exemplar

- What I check: Open the exact hot histogram observation.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Open the exact hot histogram observation.`
- Expected result: Elasticsearch owns 2.9 of 3.1 seconds.
- Different result: The exemplar is fast.
- Meaning: It is stale or from another label set.
- Next check: Search traces by exact cohort.

### Step 7 - Inspect workload shape

- What I check: Use bounded query class, not raw query or campaign ID.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Use bounded query class, not raw query or campaign ID.`
- Expected result: Broad wildcard work increased fourfold.
- Different result: No mix change exists.
- Meaning: Capacity loss or shard imbalance remains.
- Next check: Inspect node and shard USE.

### Step 8 - Check telemetry health

- What I check: Verify scrape, rule, remote-write, and collector drops.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Verify scrape, rule, remote-write, and collector drops.`
- Expected result: Targets are up and drops are zero.
- Different result: Gaps align with the incident.
- Meaning: Missing data is unknown, not healthy.
- Next check: Restore pipeline trust.

### Step 9 - Mitigate load

- What I check: Disable query expansion and rate-limit the expensive class.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Disable query expansion and rate-limit the expensive class.`
- Expected result: Pending and rejection fall.
- Different result: Retries keep offered load high.
- Meaning: Retry amplification defeats control.
- Next check: Cap attempts and honor Retry-After.

### Step 10 - Verify business recovery

- What I check: Check success, percentiles, saturation, and traces.
- Why: It separates the current hypothesis from its closest look-alike.
- Example query/tool: `Check success, percentiles, saturation, and traces.`
- Expected result: Success exceeds 99.7% and p99 is under 550 ms.
- Different result: Infrastructure is green but conversion stays low.
- Meaning: A correctness issue remains.
- Next check: Inspect business outcomes.

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

    traceId=mon-73ac
    Gateway SERVER                         3.18s
      Catalog SERVER                      3.12s
        Redis GET                            12ms cache.hit=false
        Elasticsearch SEARCH              2.90s ERROR
          queue.wait EVENT                 2.40s
          exception EVENT rejected_execution
        MySQL fallback                      180ms OK
      response.status_code=503

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

    2026-09-13T17:10:20.001Z WARN service=catalog-service instance=catalog-55dc traceId=mon-73ac spanId=es-220 requestId=req-998 businessId=search-session-41 endpoint=/catalog/search downstream=elasticsearch error=rejected_execution latency_ms=2900 queryClass=broad_wildcard
    2026-09-13T17:10:20.009Z INFO service=catalog-service instance=catalog-55dc traceId=mon-73ac spanId=root-100 requestId=req-998 endpoint=/catalog/search status=503 latency_ms=3120
    2026-09-13T17:10:21.000Z WARN service=otel-collector instance=otel-3 pipeline=traces queue_utilization=0.31 dropped_spans=0 export_errors=0

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

    A promotion enables broad wildcard searches.
    |
    v
    Each request consumes more Elasticsearch work although HTTP rate is flat.
    |
    v
    Search queues saturate and reject work.
    |
    v
    Catalog requests wait behind the saturated client pool.
    |
    v
    Tail latency crosses deadlines and produces 503 responses.
    |
    v
    Retries briefly add more dependency load.

Each arrow is supported by timing, a metric or invariant, a trace or durable state, and correlated event evidence. Deployment timing alone is correlation, not proof.

# Fix

## Immediate mitigation

- Disable broad wildcard expansion.
- Rate-limit the expensive query class and degrade safely.

## Permanent correction

- Cap retries with jitter and a shared deadline.
- Optimize query mappings with representative data.
- Alert on queue saturation and rejection before customer errors.

Do not blindly increase timeouts, pools, CPU, or memory. That is valid only when measurements show legitimate capacity demand rather than a leak, blocked dependency, or retry storm.

# Verification

Use the same signals that proved impact plus a targeted failure-path test.

- Search success rises above 99.7%.
- p99 falls from 3.2 s below 550 ms.
- Pending falls from 740 below 20.
- Rejections fall from 420/minute to zero.
- Telemetry drop counters remain zero.

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

Metrics tell me whether something is wrong and how much. I start with RED and business success, then use USE on each finite resource. Here rate was flat, errors and tail latency rose, and JVM signals were normal, while Elasticsearch pending and rejection spiked. A p99 exemplar opened a trace whose critical path was Elasticsearch queue wait. We disabled the expensive wildcard query, bounded retries, and verified success, percentiles, saturation, and telemetry health.

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
