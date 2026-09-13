# Production Troubleshooting Study Chapter: Observability, Logging, and Tracing

## Purpose and safety

Observability is the ability to infer a system's internal state from evidence it emits. It is not a product, a dashboard, or "having logs." This chapter starts with the evidence model and builds toward investigating incomplete, sampled, high-volume production telemetry.

Use read-only queries first. Restrict access by role, follow retention policy, and redact tokens, credentials, payloads, personal data, and regulated identifiers. Never enable unrestricted debug logging or put sensitive data into trace baggage during an incident.

## Learning goals

After studying this chapter, you should be able to:

1. Distinguish system signals from logs, metrics, and traces used to observe them.
2. Explain traces, spans, parentage, W3C Trace Context, propagation, sampling, and baggage.
3. Correlate synchronous and asynchronous work without treating a user ID as a trace ID.
4. use RED and USE metrics to scope impact, then pivot from metrics to traces, logs, and code.
5. Diagnose clock skew, missing spans, broken context, high cardinality, and misleading service maps.
6. Search large log volumes safely and efficiently.
7. State what each piece of evidence proves, what it does not prove, and how to proceed when telemetry is incomplete.

---

# 1. Mental model

## 1.1 Signals are not telemetry

A **signal** is a property of system behavior: latency, errors, traffic, saturation, correctness, availability, or a business outcome. **Telemetry** is recorded evidence used to estimate a signal.

```text
real request and system state
        |
        +-- measurements aggregated over time --------> metrics
        +-- discrete timestamped records -------------> logs
        +-- causally linked operation records --------> traces/spans
        +-- snapshots/samples ------------------------> profiles, dumps
        +-- external observation ---------------------> probes, user reports
```

Telemetry is lossy. An absent log may mean no event, a filtered logger, a full buffer, exporter failure, retention expiry, or a query against the wrong tenant. A sampled-out trace does not mean the request did not occur. A healthy average can hide a bad tail. Treat every source as evidence with a collection boundary, not as truth.

## 1.2 Logs, metrics, and traces

| Type | Best at answering | Strength | Important limit |
|---|---|---|---|
| Metrics | When, how often, how bad, which dimension? | Cheap aggregation, trends, alerts | Usually no single-request detail; labels must be bounded |
| Logs | What happened at a specific code point? | Rich event and error detail | Expensive volume; inconsistent schemas; may be dropped |
| Traces | Where did time/error flow across calls? | Causal request structure and timing | Sampling and broken propagation create gaps |
| Profiles/dumps | What code/resource consumed time or memory? | Runtime causality below a span | Usually targeted and time-bounded |
| Synthetic/RUM | What did an external user/path observe? | Independent of server self-report | Limited path/coverage; client conditions vary |

Use metrics to find the bad interval and scope, traces to localize a path, logs to explain an event, and code/config/runtime evidence to prove the mechanism. This is a workflow, not a rigid rule.

## 1.3 Trace anatomy

```text
trace_id = one causal operation

span A: checkout HTTP server             span_id=a, parent=none
  |
  +-- span B: reserve inventory          span_id=b, parent=a
  |     |
  |     +-- span C: inventory SQL        span_id=c, parent=b
  |
  +-- span D: charge payment             span_id=d, parent=a
```

- A **trace** is a set of spans sharing a trace ID.
- A **span** represents one timed operation and has a span ID, name, start/end, status, attributes, events, and resource identity.
- A **parent span ID** expresses causal nesting. Siblings may overlap.
- A **link** relates work without making it a child, useful for batched or async processing.
- A span's duration includes time inside the operation, including child work; summing all span durations double-counts nested time.
- Span status is instrumentation output. HTTP `500` may be recorded as error, while a timeout may produce an exception event and an incomplete child.

## 1.4 W3C context and async propagation

W3C Trace Context standardizes:

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             vv trace-id                         parent-id        flags
tracestate: vendor-specific-list
```

`traceparent` carries version, 16-byte trace ID, current parent span ID, and flags such as sampled. Validate format and generate new context for malformed/untrusted input. Do not use trace IDs as authentication or authorization.

Propagation requires both **inject** and **extract**:

```text
HTTP producer injects headers -> HTTP consumer extracts -> starts child span
message producer injects properties -> broker -> consumer extracts -> starts/links span
thread/task submission captures context -> worker restores it -> finally clears it
```

Async boundaries are common breakpoints: thread pools, callbacks, reactive operators, scheduled jobs, message headers stripped by middleware, and serialization that omits context. For one message, a consumer span can be a child of the producer. For delayed, fan-out, fan-in, or batched processing, links often represent causality more honestly than a single parent. Context must not leak from one pooled-thread task to the next.

## 1.5 Correlation and business identity

- **Trace ID:** technical causal operation, usually short lived.
- **Request/correlation ID:** application identifier; may equal the trace ID by convention, but its lifecycle and uniqueness must be documented.
- **Business ID:** order, payment, shipment, or workflow identifier; can span many traces and days.
- **Message ID:** identifies a delivery; retry/redelivery needs an attempt field and often an original-message ID.
- **User ID:** sensitive, high-cardinality identity; not an appropriate primary correlation key or metric label.

Log structured fields such as `trace_id`, `span_id`, `service.name`, `service.instance.id`, `deployment.environment`, `request_id`, and approved hashed/tokenized business IDs. Do not infer causality merely because two events share a user.

## 1.6 Sampling, exemplars, and cardinality

**Head sampling** decides near trace start and is cheap, but cannot know the eventual latency/error. **Tail sampling** buffers trace data and can preferentially retain errors/slow traces, but costs memory, adds delay, and still loses traces during collector failure. Parent-based sampling maintains a coherent decision when propagation works. Sampling probability must be available for statistically valid rate estimates; traces are usually poor counters.

An **exemplar** attaches a representative trace ID to a metric observation, allowing a jump from a latency histogram bucket to a trace. The trace can still be expired, inaccessible, or sampled out.

Cardinality is the number of distinct label combinations:

```text
series ~= methods * routes * statuses * regions * instances * other labels
```

Never put trace ID, raw URL, user ID, order ID, exception message, or unbounded SQL text in metric labels. Use normalized routes (`/orders/{id}`), coarse error classes, bounded status codes, and service/region/version. High-cardinality values belong in indexed logs or trace attributes under policy.

## 1.7 Time is evidence, not causality

Services may use different wall clocks, time zones, timestamp precision, buffering, and ingestion delays. NTP keeps clocks close, not perfectly identical. Use UTC with timezone and nanosecond/millisecond precision, retain both event time and ingestion time, and compare monotonic span durations when available.

A parent can appear to start after a child because of skew. Do not reorder causal events solely by wall-clock timestamp. Trace parentage, message offsets, sequence numbers, database transaction records, and business versions can be stronger evidence.

## 1.8 Service maps

A service map is inferred from observed spans. An absent edge can mean no traffic, sampling, failed propagation, unsupported instrumentation, or exporter loss. A displayed edge proves only that the backend observed matching telemetry during that window. Maps are useful orientation, not an authoritative inventory or proof of health.

---

# 2. Glossary

| Term | Meaning |
|---|---|
| Attribute/tag | Key-value metadata on a span, log, or metric point |
| Baggage | W3C key-value context propagated across process boundaries; not automatically safe |
| Collector | Agent/gateway receiving, processing, and exporting telemetry |
| Context propagation | Transferring trace identity across process/thread/message boundaries |
| Exemplar | Metric sample associated with a trace/span ID |
| Histogram | Distribution represented by buckets or native histogram data |
| MDC/log context | Per-execution context injected into logs; must propagate and be cleared |
| RED | Rate, Errors, Duration for request-driven services |
| Resource attributes | Identity of emitter: service, version, environment, instance, region |
| Span event | Timestamped event within a span, such as an exception |
| Tail latency | High percentile such as p95 or p99, not the maximum |
| Telemetry gap | Evidence expected but absent because of instrumentation, transport, sampling, or query boundaries |
| USE | Utilization, Saturation, Errors for resources |

---

# 3. Essential metrics and evidence

## 3.1 RED and business outcomes

For every ingress and dependency, collect:

- **Rate:** requests/second, preferably split by normalized route and outcome.
- **Errors:** explicit errors and policy-defined failures; do not count all cancellations alike.
- **Duration:** histogram and p50/p95/p99; avoid averages alone.
- **Business outcomes:** orders accepted, payments settled, messages completed, duplicates, reconciliation gaps.

Example PromQL:

```promql
sum(rate(http_server_requests_seconds_count{service="checkout"}[5m])) by (route, status)

sum(rate(http_server_requests_seconds_count{service="checkout",status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count{service="checkout"}[5m]))

histogram_quantile(
  0.99,
  sum(rate(http_server_requests_seconds_bucket{service="checkout"}[5m])) by (le, route)
)
```

Interpretation: `rate` uses counter change over the window. A ratio needs the same scope in numerator and denominator. `histogram_quantile` is an estimate and is only valid when buckets and aggregation are compatible.

## 3.2 USE and telemetry-pipeline health

For CPU, memory, disks, network, thread/connection pools, queues, and collectors:

- **Utilization:** fraction busy/in use.
- **Saturation:** queued work, wait time, rejected work.
- **Errors:** allocation, I/O, export, scrape, and processing failures.

Also monitor telemetry itself:

```text
SDK dropped logs/spans/metrics
export queue size and enqueue failures
collector accepted, refused, dropped, and exported items
collector memory/CPU and restarts
backend ingestion errors/throttling
scrape failures and stale series
clock synchronization offset
sampling policy/version
```

If production errors rise while recorded error traces fall and collector drops rise, "fewer traces" is a telemetry failure, not recovery.

## 3.3 Minimum structured event

```json
{
  "timestamp": "2026-09-13T14:52:31.418Z",
  "severity": "ERROR",
  "service.name": "checkout",
  "service.version": "2026.09.13.2",
  "service.instance.id": "checkout-7d9f6b8f4f-k2m8x",
  "deployment.environment": "production",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "request_id": "req-81d2",
  "route": "/orders/{id}",
  "error.type": "InventoryTimeout",
  "message": "inventory reservation exceeded remaining deadline"
}
```

Prefer stable fields and error types over parsing prose. Record exception type, safe message, stack trace, operation, timeout/deadline, attempt, destination logical name, and response class. Avoid duplicate stack traces at every layer; add context once and preserve the cause.

---

# 4. Generic investigation workflow

1. **Define impact.** Record user-visible symptom, first/last time, environment, region, route, version, tenant cohort, and business correctness risk.
2. **Identify the observer.** Determine which component generated the status, timeout, or alert.
3. **Fix the time window.** Use UTC, include a pre-failure baseline, and account for ingestion delay/skew.
4. **Start with coarse signals.** Compare RED and business metrics by route, version, zone, and dependency; check telemetry health.
5. **Select representative evidence.** Use an exemplar, returned trace/request ID, or a narrow error/latency slice.
6. **Read the trace critically.** Find the first error and longest critical-path span; distinguish child duration from parent self-time and note missing edges.
7. **Pivot to logs.** Query exact IDs first, then service/time/error class. Include instance/version and use event time.
8. **Inspect the indicated layer.** Code/config/deploy diff, pool metrics, profile, database plan, broker state, or network evidence.
9. **Form and test one mechanism.** Explain cause -> resource/behavior -> telemetry -> user impact. Compare healthy versus failing cohorts.
10. **Mitigate safely.** Reduce harm without deleting evidence or creating duplicate writes.
11. **Verify.** Repeat the original business path and confirm RED, saturation, business outcomes, and telemetry pipeline recovery.
12. **Prevent.** Correct instrumentation, alerts, runbooks, tests, capacity, and ownership.

If an expected signal is absent, branch:

```text
Is there an external/user symptom?
  no  -> validate alert/query and stop claiming an outage
  yes
   |
   +-> metrics present? scope time/service/version
   |     no -> check scrape/export/backend and external probes
   |
   +-> trace present? inspect critical path and gaps
   |     no -> use request/business ID, logs, gateway records, sampling health
   |
   +-> logs present? explain exact event
         no -> inspect runtime/platform/audit evidence and instrumentation path
```

---

# 5. Original interview questions

## 1. A request passes through five microservices and eventually fails. How would you find where it failed?

### Exact meaning and limits of the symptom

The only fact is that the end-to-end operation failed after entering a multi-service path. "Five services" may describe an expected architecture, not the actual route. The error seen by the caller may have been generated by a gateway, an early service, or a compensation step. A red span marks recorded status, not necessarily the root cause; later failures can be consequences.

### Possible failure locations and mechanisms

The fault can occur at ingress, context propagation, routing, DNS/TCP/TLS, authentication, a service queue/thread pool, application validation, database/cache, a synchronous dependency, an asynchronous publish/consume step, serialization, response transfer, or telemetry export. One slow dependency can consume the deadline; an upstream then reports timeout while the downstream later commits successfully.

### Ordered debugging reasoning

1. Capture UTC time, route, request/trace/business ID, caller result, and whether a write may have occurred.
2. Scope error rate and latency by service, route, region, version, and instance.
3. Open a representative trace from an exemplar or known trace ID.
4. Follow the causal tree in request order; locate the first error event/status and the last successful boundary.
5. Check the critical path, not the largest sum of child durations.
6. For a missing expected span, inspect the parent log and client metrics; absence is not proof the call was skipped.
7. Query logs for trace ID at the suspected service and instance, including nested exception and deadline.
8. Inspect the indicated dependency/runtime evidence and compare a successful trace from the same cohort.
9. State a causal mechanism and reproduce safely or validate with a cohort/change comparison.

### Evidence, tools, queries, and interpretation

```text
Trace search:
  trace_id = "4bf92f3577b34da6a3ce929d0e0e4736"

Log search:
  trace_id:"4bf92f3577b34da6a3ce929d0e0e4736"
  | sort timestamp asc
  | fields timestamp, service.name, span_id, severity, error.type, message
```

Check gateway access logs, span status/events, client/server request counters, deployment markers, pool queue/wait, and telemetry drop counters. If A's client span ends `DEADLINE_EXCEEDED` at 2 seconds but B's server span continues to 2.4 seconds and writes, the problem includes deadline/cancellation behavior and possible ambiguous completion.

### Immediate mitigation

Drain a proven bad instance, roll back a correlated release, disable a noncritical failing path, shed excess load, or route to a safe dependency. Preserve write idempotency; do not blindly replay uncertain operations.

### Permanent correction/design

Fix the proven code/config/dependency issue. Instrument every ingress/egress with consistent resource identity, propagate context, record remaining deadlines and safe error classes, and design idempotent write/status lookup where outcomes can be ambiguous.

### Prevention and alerts

Alert on SLO burn rate and business failure, plus dependency RED, queue saturation, and telemetry drops. Test propagation and error recording across the complete route in staging and canaries.

### Common mistakes

- Starting with unbounded log browsing rather than a time/ID.
- Assuming the last red span caused the incident.
- Summing nested span durations.
- Treating a missing span as proof no call happened.
- Restarting all five services and destroying the comparison.

### Interview-ready answer

I scope the failure with RED and business metrics, select a representative trace, find the first failing boundary and critical-path delay, and pivot by trace ID to structured logs and then the relevant runtime or dependency evidence. I compare a successful request, account for sampling and missing spans, mitigate only the proven fault domain, and verify the original business outcome.

## 2. You see an error in Service A, but the actual failure occurred in Service D. How would you trace it?

### Exact meaning and limits of the symptom

A's error is an observation at A's boundary. It may wrap D's failure, report a timeout caused by D, or merely coincide with it. "Actual failure occurred in D" must be established by causal context, not timestamp proximity.

### Possible failure locations and mechanisms

A may call B, B call C, and C call D. D can reject input, exhaust a pool, fail a dependency, or exceed a deadline. B/C may replace the error, lose stack/cause information, retry D, or fail to propagate trace context. A may time out first even though D records success later.

### Ordered debugging reasoning

1. Extract A's trace ID, span ID, safe error type, route, deadline, and request ID.
2. Open the trace and walk A -> B -> C -> D using parent IDs and links.
3. Find the earliest causally relevant error event; distinguish propagated errors from original exceptions.
4. Compare client and server spans at every hop for status, duration, peer, and attempt.
5. Query D's logs by trace ID; if absent, use C's outbound request ID, approved business ID, destination instance, and narrow time.
6. Confirm D's mechanism with its dependency, pool, and deployment evidence.
7. Verify that A's error mapping preserves a safe stable cause and that cancellation propagated.

### Evidence, tools, queries, and interpretation

```text
trace_id:"4bf92f3577b34da6a3ce929d0e0e4736"
AND service.name:("service-a" OR "service-b" OR "service-c" OR "service-d")
```

Inspect W3C headers in controlled non-sensitive debug capture, not routine payload logs. Matching trace ID plus correct parent chain is stronger than matching time. If D uses a different trace ID, C's client span and D's access log may still correlate through request ID, peer address, route, and precise bounded time, but label the conclusion with its confidence.

### Immediate mitigation

Protect D by shedding load or disabling the failing feature; drain a bad D instance or roll back its correlated change. Adjust traffic only within existing safe policy.

### Permanent correction/design

Correct D's fault and preserve error causes across layers without exposing internals to clients. Standardize W3C propagation and structured error fields; add async links where parentage is not appropriate.

### Prevention and alerts

Alert on D's dependency errors/saturation and on A's SLO. Add tests that assert the same trace crosses all hops and that error spans preserve causal attributes.

### Common mistakes

- Searching D only for A's human-readable message.
- Claiming timestamp adjacency proves causation.
- Returning D's raw exception or secret-bearing payload to clients.
- Marking every upstream wrapper as a separate root cause.

### Interview-ready answer

I start from A's trace context, follow parent-child spans to D, find the earliest error rather than the outer wrapper, and corroborate D's span with structured logs and resource/dependency metrics. If propagation broke, I bridge only with bounded request/business evidence and state uncertainty. Then I fix D and the propagation/error-mapping gap.

## 3. How would you trace one user's request across multiple microservices?

### Exact meaning and limits of the symptom

The goal is one causal request, not every action by a user. A user may issue concurrent requests, and one business workflow may outlive an HTTP trace. User identity is sensitive and does not establish parentage.

### Possible failure locations and mechanisms

Context can be lost at gateways, HTTP clients, thread pools, reactive callbacks, queues, scheduled jobs, protocol bridges, and retries. Duplicate IDs can arise from trusting malformed inbound context or reusing mutable thread-local state.

### Ordered design and debugging reasoning

1. At the trusted ingress, validate incoming `traceparent` or start a new trace.
2. Start a server span and return an approved request/correlation ID for support.
3. Instrument framework clients/servers; inject and extract W3C `traceparent` and `tracestate`.
4. Carry context through executor/reactive APIs and clear it after work.
5. Put trace context in message headers; use child spans or links according to async semantics.
6. Add trace/span IDs automatically to structured logs through logging context.
7. Add approved business IDs for long workflows; query them across multiple traces.
8. Test fan-out, retries, batch consumers, errors, and unsampled requests.

### Evidence, tools, queries, and interpretation

Check that all spans share a 32-hex trace ID, each non-root parent resolves or is intentionally remote/missing, service identity changes correctly, and retry attempts are separate spans. A user-facing request ID lookup should resolve to the server trace through an indexed log or mapping, not by exposing backend access.

### Immediate mitigation

During a propagation gap, use the gateway request ID and a narrow service/time/route query. Temporarily raise targeted sampling only through approved controls; never enable payload or identity logging broadly.

### Permanent correction/design

Adopt OpenTelemetry-compatible instrumentation and W3C propagation, define ownership at each boundary, use business IDs for durable workflows, and enforce baggage allowlists and size limits.

### Prevention and alerts

Run synthetic trace-continuity tests and alert on unexpected root-span rate, orphan spans, or abrupt drops in spans per request by service/version.

### Common mistakes

- Using raw user email as correlation.
- Creating a new trace at every service.
- Copying headers but failing to restore context in worker threads.
- Making a multi-day workflow one enormous trace.
- Treating baggage as secure storage.

### Interview-ready answer

I use a validated W3C trace at ingress, propagate it through HTTP, task, and messaging boundaries, inject trace/span IDs into structured logs, and use an approved business ID for workflows spanning traces. I do not use the user ID as causality, and I test context continuity, retries, async links, and cleanup.

## 4. You have millions of logs. How would you find the logs belonging to one request?

### Exact meaning and limits of the symptom

High volume makes free-text browsing slow, expensive, and noisy. "Belonging" may mean causally within the trace, or merely sharing a business entity; those are different result sets.

### Possible failure locations and mechanisms

The ID may be absent, parsed into the wrong field, truncated, sampled, redacted, changed at a proxy, delayed in ingestion, or stored in another index/tenant. Multiline stack traces may be split. High-cardinality indexes may be throttled.

### Ordered debugging reasoning

1. Obtain the exact trace/request ID from the client response, gateway, exemplar, or incident record.
2. Select environment/index and a tight UTC interval around event time.
3. Query the exact structured field, not a wildcard full-text scan.
4. Add service, route, severity, version, and instance only as needed.
5. Sort by event timestamp but retain ingestion timestamp and parent/span IDs.
6. Expand the interval for buffering/skew; check aliases, retention, parse failures, and telemetry drops if absent.
7. For async continuation, pivot from trace ID to approved business/message ID and clearly mark the new trace boundary.
8. Save a bounded query or export with redaction; do not download the entire corpus.

### Evidence, tools, queries, and interpretation

```text
OpenSearch/Lucene:
deployment.environment:"production"
AND trace_id:"4bf92f3577b34da6a3ce929d0e0e4736"

Loki:
{environment="production",service_name=~"checkout|inventory|payment"}
| json
| trace_id="4bf92f3577b34da6a3ce929d0e0e4736"

Cloud-style SQL:
fields @timestamp, service_name, instance_id, span_id, level, error_type, message
| filter trace_id = "4bf92f3577b34da6a3ce929d0e0e4736"
| sort @timestamp asc
| limit 500
```

Exact indexed fields sharply reduce scanned data. `limit 500` means results may be incomplete; check result count/truncation. No matches requires telemetry-path checks before concluding no event occurred.

### Immediate mitigation

Use a known exemplar or gateway ID and narrow scope. Increase only targeted logging/sampling for a short approved interval when necessary, with rollback and volume/privacy guardrails.

### Permanent correction/design

Standardize structured schemas, index trace/request/business IDs according to use, normalize routes, enforce retention, and make request IDs available to support without exposing secrets.

### Prevention and alerts

Monitor parse failure, ingestion delay, dropped records, index rejection, and query latency. Test that emitted IDs remain searchable end to end.

### Common mistakes

- Searching raw message text across all time.
- Using user ID and collecting unrelated concurrent requests.
- Assuming search limits return a complete request.
- Ignoring event versus ingestion time.
- Logging every payload to make search "easy."

### Interview-ready answer

I start with an exact trace or request ID, constrain environment and UTC time, query its structured indexed field, then sort with service, instance, span, and ingestion metadata. I pivot to a business/message ID only across explicit async boundaries. If results are missing, I investigate parsing, sampling, export, retention, and clock skew rather than declaring the event absent.

## 5. The API returns 500, but Service A's logs don't show the root cause. What would you do?

### Exact meaning and limits of the symptom

HTTP 500 says an HTTP component reported an internal failure. It does not prove Service A generated it, nor that a log must exist. A gateway can rewrite status; A can catch and replace an exception; the log can be elsewhere or lost.

### Possible failure locations and mechanisms

Gateway/sidecar filters, A's framework before application code, exception handlers, downstream B, database, serialization after successful business work, process crash, OOM, or exporter failure can all produce this pattern.

### Ordered debugging reasoning

1. Identify the response generator from headers and gateway/access records.
2. Capture route, UTC time, request/trace ID, A instance/version, and response size.
3. Compare gateway request counts/status with A server metrics. A missing A request suggests failure before A or telemetry loss.
4. Inspect the trace for first error, exception event, missing server span, or abrupt end.
5. Query A by trace ID, then request ID and instance/time; inspect stdout/platform logs and previous container logs after restart.
6. Check log levels, exception-handler behavior, asynchronous appenders, rate limits, disk/backpressure, collector drops, and backend ingestion.
7. Inspect downstream client spans/logs and runtime events such as OOM/restart.
8. Reproduce with safe synthetic input and improve instrumentation before guessing.

### Evidence, tools, queries, and interpretation

```bash
kubectl logs -n production checkout-7d9f6b8f4f-k2m8x --since=30m
kubectl logs -n production checkout-7d9f6b8f4f-k2m8x --previous
kubectl describe pod -n production checkout-7d9f6b8f4f-k2m8x
```

Check ingress access status/upstream status, A request/error counters, restart reason, exception-handler logs, and collector rejected counts. A gateway 500 with no upstream target means A may never have received it. A restart with `OOMKilled` near the request supports abrupt termination but still requires memory-cause analysis.

### Immediate mitigation

Roll back a correlated release, drain the affected instance, or disable the failing noncritical route. Preserve crash artifacts and avoid turning on global debug logging.

### Permanent correction/design

Fix the generating defect; add top-level exception recording with safe error type and trace ID, reliable stdout/export, gateway/upstream fields, and tests for serialization and error middleware.

### Prevention and alerts

Reconcile gateway and service request/error counts, alert on process exits and telemetry drops, and use synthetic checks for real business routes.

### Common mistakes

- Assuming no A log means A succeeded.
- Exposing stack traces in the API response.
- Logging and rethrowing at every layer.
- Restarting before collecting previous-container/platform evidence.

### Interview-ready answer

I first identify who generated the 500, correlate gateway, trace, service, platform, and downstream evidence, and verify A received the request. I check exception mapping, process death, buffering/export loss, and prior-container logs. I mitigate the proven instance/release and add reliable top-level error instrumentation; I never infer success from missing logs.

## 6. Distributed tracing shows one downstream service taking 4 seconds. How would you investigate further?

### Exact meaning and limits of the symptom

The span duration is four seconds from the instrumented boundary's clock. It may include connection acquisition, DNS/TLS, queueing, server processing, streaming, retries, or instrumentation error. It does not prove four seconds of CPU or identify the root cause.

### Possible failure locations and mechanisms

Client pool wait, network retransmission, load-balancer queue, server concurrency queue, GC pause, lock contention, slow code, database/cache/API wait, payload transfer, retry, or clock skew can inflate the span.

### Ordered debugging reasoning

1. Determine whether 4 seconds is isolated or a percentile shift; compare baseline, route, version, region, and instance.
2. Check whether it lies on the critical path and consumes the caller's deadline.
3. Compare client and server span timings and attributes. A large client-only gap points before server observation or after response.
4. Expand server children and calculate parent self-time without summing nested durations.
5. Check USE: request queue, thread/concurrency permits, connection pools, CPU throttling, GC, memory, disk/network.
6. Follow the slowest child into database plans, dependency metrics, or profile.
7. Compare a fast trace with identical route and payload-size class.
8. Correlate deployment/config/load changes and prove the mechanism under representative load.

### Evidence, tools, queries, and interpretation

```promql
histogram_quantile(
  0.99,
  sum(rate(http_client_request_duration_seconds_bucket{
    service="checkout",server_address="inventory"
  }[5m])) by (le)
)
```

Inspect `server.address`, normalized route, status, attempt, request/response size, queue time, pool acquisition time, and deadline remaining. A 3.7-second database child explains location but not why; its execution plan, lock wait, rows scanned, and I/O establish mechanism.

### Immediate mitigation

Reduce nonessential traffic, shed overload, drain a bad instance, roll back a regression, or use an explicitly correct degraded path. Do not simply raise every timeout: that can increase concurrency and collapse.

### Permanent correction/design

Remove the proven bottleneck, bound queues/concurrency, optimize queries/code, propagate deadlines/cancellation, and instrument queue/pool phases separately.

### Prevention and alerts

Alert on tail latency and deadline-exhaustion burn, plus saturation leading indicators. Load-test at realistic concurrency and payload distributions.

### Common mistakes

- Calling the four-second service "slow" without population context.
- Treating span duration as CPU time.
- Increasing timeout before analyzing capacity.
- Ignoring client/server timing mismatch and retries.

### Interview-ready answer

I confirm the four seconds is a real tail shift on the critical path, compare client and server spans, decompose queue/pool/network/server/child time, and use USE metrics plus a fast-trace comparison to prove the bottleneck. I mitigate load or the bad cohort safely; I do not merely increase the timeout.

## 7. Logs from different services have different timestamps. How would you correlate them?

### Exact meaning and limits of the symptom

Different timestamps can be valid concurrency, different zones/formats, clock skew, buffering, or ingestion delay. Timestamp order alone cannot establish causality.

### Possible failure locations and mechanisms

NTP failure, suspended VM, bad host clock, local-time formatting, missing timezone, low precision, application-created timestamps, collector batching, retry, and backend parsing can reorder records.

### Ordered debugging reasoning

1. Normalize displayed timestamps to UTC while retaining originals.
2. Separate event time from ingestion time and inspect timezone/precision.
3. Correlate by trace ID, parent/span IDs, request/message/business ID, and sequence/offset.
4. Use monotonic span duration and causal parentage rather than wall-clock ordering.
5. Estimate per-host offset from clock monitoring or trusted gateway/server boundaries.
6. Compare instance/node identity; correct synchronization and parser issues.
7. Widen query windows by measured skew and ingestion lag, not an arbitrary day.

### Evidence, tools, queries, and interpretation

```bash
timedatectl status
chronyc tracking
chronyc sources -v
```

On Windows, inspect `w32tm /query /status`. Alert on time offset and sync source. If D's child has parent ID from C but appears 300 ms earlier, parentage establishes causality and measured host offset explains ordering. Do not rewrite source records silently; preserve auditability.

### Immediate mitigation

Use IDs and causal metadata, widen the incident window by known skew, and annotate the timeline. Restore approved time synchronization if broken.

### Permanent correction/design

Emit ISO 8601 UTC timestamps with offsets, record event and ingestion time, use monotonic clocks for durations, maintain NTP/chrony, and include instance/node identity.

### Prevention and alerts

Alert on clock offset, unsynchronized nodes, ingestion delay, and timestamp parse failures. Test cross-service timeline rendering.

### Common mistakes

- Sorting logs and declaring the first timestamp the cause.
- Manually adding a guessed offset to evidence.
- Ignoring time zones or DST.
- Using wall clock to calculate in-process durations.

### Interview-ready answer

I normalize to UTC but correlate primarily through trace parentage, request/message IDs, offsets, and sequence numbers. I distinguish event from ingestion time, measure host clock offsets, and use monotonic span durations. I correct synchronization and parsing while preserving original timestamps.

## 8. A request succeeds in Service A but fails somewhere downstream. How would you identify the exact service?

### Exact meaning and limits of the symptom

"Succeeds in A" may mean A accepted or queued work, not that the end-to-end business operation completed. The downstream path may be synchronous or asynchronous. A `2xx` can therefore be contractually correct, premature, or success-shaped masking.

### Possible failure locations and mechanisms

After A, failures can occur in routing, B/C/D code, queue publish, broker delivery, consumer processing, database commit, callback/webhook, or compensation. A fallback may hide failure; async context or business correlation may be missing.

### Ordered debugging reasoning

1. Define success contract: accepted, persisted, completed, or externally confirmed.
2. Capture trace/request/business/message IDs and expected state transition.
3. For synchronous flow, follow trace spans to the first failing boundary.
4. For async flow, verify A's durable publish/outbox, broker record, consumer receipt, attempt, side effect, and checkpoint/DLT in order.
5. Pivot across traces with approved business/message IDs and span links.
6. Reconcile authoritative state; do not rely only on logs.
7. Compare expected service map with observed path but investigate absent telemetry separately.
8. Identify the last durable successful transition and first failed transition.

### Evidence, tools, queries, and interpretation

Use gateway and service RED, producer acknowledgements, outbox state, message topic/partition/offset, consumer lag/attempt, DLT record, database audit/version, and trace links. A producer "send invoked" log does not prove broker acknowledgement. A consumer success log before transaction commit does not prove the effect persisted.

### Immediate mitigation

Pause unsafe processing, stop duplicate-producing retries, replay only with idempotency and owner approval, or route to a correct degraded mode. Communicate accepted-versus-completed status accurately.

### Permanent correction/design

Define API semantics, use transactional outbox/inbox where needed, durable idempotency, status APIs, reconciliation, trace links, and explicit terminal business metrics.

### Prevention and alerts

Alert on accepted-to-completed divergence, age of pending workflows, DLT growth, missing callbacks, and reconciliation gaps.

### Common mistakes

- Treating A's `200` as end-to-end success.
- Trusting inferred service maps as delivery proof.
- Replaying messages without idempotency.
- Correlating async work only by timestamp.

### Interview-ready answer

I first define what A's success means. I follow synchronous spans or, for async work, verify each durable transition from outbox/publish through broker, consumer, side effect, and completion using message/business IDs and links. The exact fault is after the last proven durable success and at the first failed transition, confirmed against authoritative state.

## 9. How would you investigate a production issue when you have logs, metrics and traces available?

### Exact meaning and limits of the symptom

Having all three does not guarantee completeness, consistency, retention, or correct instrumentation. The objective is a proven mechanism and safe recovery, not forcing the sources to agree.

### Possible failure locations and mechanisms

The product path, runtime, dependency, platform, deployment, or telemetry pipeline can fail. Metrics can aggregate away a cohort, traces can sample it out, and logs can be dropped or misparsed.

### Ordered debugging reasoning

1. Confirm user/business impact and data-safety risk.
2. Establish UTC window and changes; identify error generator.
3. Use RED/business metrics to scope route, version, region, tenant class, and time.
4. Check USE and telemetry-pipeline health.
5. Pivot from a bad histogram exemplar or known request ID to traces.
6. Read critical path, first error, attempts, deadlines, attributes, and missing spans; compare a healthy trace.
7. Pivot by trace/span ID to structured logs for exception and state transitions.
8. Inspect code/config/deploy and targeted runtime/dependency evidence.
9. Build a causal statement and test a discriminating prediction.
10. Mitigate, verify the original path and business state, then correct and prevent.

### Evidence, tools, queries, and interpretation

```text
metric: checkout p99 rose only on version 2026.09.13.2
trace: slow requests wait 1.8 s before acquiring a DB connection
log: PoolTimeout with active=50, idle=0, pending=420
DB: query latency unchanged
code diff: new path holds connection during remote call

mechanism:
release held scarce DB connections across a slow remote call -> pool saturated
-> requests queued -> deadline expired -> 5xx increased
```

This cross-source chain is stronger than "database slow." Still verify telemetry drops and business outcomes.

### Immediate mitigation

Roll back the implicated version or reduce traffic to it; shed noncritical work and preserve ambiguous-write safety. Keep a small evidence window before changing state.

### Permanent correction/design

Fix the connection lifetime, add bounded concurrency/deadlines, improve phase-level instrumentation, and make dashboards link metrics exemplars to traces and logs.

### Prevention and alerts

Use SLO burn alerts plus pool saturation, queue wait, telemetry loss, and business reconciliation. Canary by version and verify observability during deployments.

### Common mistakes

- Opening logs first with no scope.
- Assuming dashboards are complete.
- Confusing correlation with mechanism.
- Fixing an alert while users still fail.
- Verifying only technical `2xx`, not business correctness.

### Interview-ready answer

I use metrics to scope when and which cohort, traces to locate the failing critical-path boundary, logs to explain the exact event, and code/runtime/dependency evidence to prove mechanism. At every pivot I check sampling and pipeline health. I mitigate safely, verify both SLO and business state, and improve the system and its telemetry.

---

# 6. Additional important interview questions

## 10. Why are spans missing from a trace, and how do you investigate?

### Meaning, locations, and mechanisms

A missing span can result from head/tail sampling, parent not sampled, absent instrumentation, propagation failure, exporter queue overflow, collector/backend rejection, process crash before flush, retention, query permissions, clock-skew windowing, or a code path that never made the call. It cannot by itself prove which.

### Step-by-step investigation and evidence

1. Draw the expected edge from code/config, not the service map alone.
2. Check the parent span for an outbound call and propagation attributes.
3. Compare client request metrics with downstream server metrics.
4. Search both sides by trace/request/business ID and a skew-aware window.
5. Inspect SDK sampled flag, sampler policy/version, exporter/collector/backend drop counters, and process restarts.
6. Run an approved synthetic request with known sampling and verify header injection/extraction.
7. If the call occurred without a span, repair instrumentation; if no call evidence exists, investigate control flow.

### Mitigation, correction, prevention, and mistakes

Temporarily use logs/metrics/platform evidence and narrowly increase sampling if capacity permits. Permanently test propagation, size exporter queues, monitor drops, flush on graceful shutdown, and define retention. Do not label every missing edge a network failure or force 100% sampling indefinitely.

### Interview-ready answer

I treat a missing span as an observability gap until corroborated. I compare parent-client and child-server metrics/logs, check propagation, sampling, exporter/collector/backend loss, restart, retention, and query scope, then run a known synthetic trace. I repair the proven collection boundary rather than inventing a service failure.

## 11. How do sampling and tail sampling affect incident conclusions?

### Meaning, mechanisms, and reasoning

Head sampling is decided before outcome, so rare errors may be missed by chance. Parent-based decisions preserve traces but can inherit an upstream drop. Tail sampling can retain errors/slow requests after observing them but requires buffering and can bias the dataset. Neither makes sampled traces a reliable request counter.

1. Read the active policy, probability, rules, and deployment time.
2. Determine whether decisions are consistent across collectors and tenants.
3. Check late-span, memory, queue, and drop counters.
4. Compare trace-derived rates with unsampled request/business counters.
5. Use weighted estimates only when inclusion probability is known and selection is appropriate.
6. Preserve representative error/slow traces while maintaining privacy/capacity budgets.

### Mitigation, correction, prevention, and mistakes

During an incident, adjust a bounded route/service rule rather than global 100% capture. Use deterministic parent-based head sampling or capacity-tested tail policies, always retain a controlled error/latency subset, and alert on effective sample rate and dropped traces. Never claim "10% of traces failed, therefore 10% of requests failed" under outcome-biased tail sampling.

### Interview-ready answer

Sampling controls recorded evidence, not real traffic. I inspect the exact policy and loss counters, derive rates from metrics/business counters, and use traces for representative causality. Tail sampling improves retention of interesting traces but biases populations and adds failure modes, so I capacity-test it and monitor effective sampling.

## 12. What is trace baggage, and what are its operational and security risks?

### Meaning, locations, and mechanisms

W3C baggage is application metadata propagated downstream, separately from `traceparent`. It can aid routing or correlation, but every hop may forward it. Risks include PII leakage across trust boundaries, forged values, oversized headers causing 431/rejection, high propagation cost, metric-cardinality explosion, and accidental authorization based on untrusted data.

### Step-by-step design

1. Default to no baggage; document a business need.
2. Allowlist bounded keys and value sizes at ingress.
3. Treat incoming baggage as untrusted; never authorize from it.
4. Remove or transform values at trust boundaries.
5. Do not automatically copy baggage into logs or metric labels.
6. Monitor header size and rejected propagation.
7. Test deletion, fan-out cost, and privacy retention.

### Mitigation, correction, prevention, and mistakes

Strip an offending key at the nearest approved boundary, revert its producer, and assess exposure. Permanently use opaque, non-sensitive, bounded identifiers and governance. A trace attribute that stays local may be safer than baggage. Never put email, tokens, account details, or unrestricted tenant IDs in baggage.

### Interview-ready answer

Baggage is propagated application context, not secure identity. I use it only for allowlisted, bounded, non-sensitive values, validate and strip it at trust boundaries, never authorize from it, and keep it out of metric labels and automatic logs.

---

# 7. Decision trees

## 7.1 Metric to trace to log to code

```text
User or SLO symptom
 |
 +-- Is the metric query valid and telemetry healthy?
 |     no -> repair/query around telemetry; use external/business evidence
 |     yes
 |
 +-- Which route/version/region/instance/dependency changed?
 |
 +-- Exemplar or trace for bad cohort available?
 |     yes -> first error + critical path + attempts + gaps
 |     no  -> known request ID or targeted trace search
 |             |
 |             +-- still none -> gateway/log/business state/platform evidence
 |
 +-- Logs for trace/span available?
 |     yes -> exception/state/deadline/instance
 |     no  -> parse/export/retention/restart checks; runtime evidence
 |
 +-- Does code/config/runtime evidence explain mechanism?
       no -> compare healthy cohort and test next hypothesis
       yes -> mitigate -> verify -> correct -> prevent
```

## 7.2 Correlation decision

```text
Same synchronous causal request?
  use trace_id + parent/span IDs

Support needs one HTTP attempt?
  use request_id mapped to trace_id

Workflow crosses queues, retries, or days?
  use approved business/message ID + trace links + attempt/sequence

Only user ID available?
  narrow by consent, time, route, and privacy policy;
  do not claim causality until a request/business identifier confirms it
```

## 7.3 Missing evidence decision

```text
Expected record absent
 |
 +-- wrong environment/index/time/permission/retention?
 +-- timestamp parse, skew, or ingestion delay?
 +-- event filtered by level or sampler?
 +-- SDK buffer/export failure?
 +-- collector/back-end rejection or throttling?
 +-- process crash before flush?
 +-- propagation/instrumentation gap?
 +-- code path genuinely not executed?

Conclude only after positive evidence distinguishes these branches.
```

---

# 8. Cheat sheets

## 8.1 What each signal can prove

| Evidence | Reasonable conclusion | Unsafe conclusion |
|---|---|---|
| Error-rate counter rose | Recorded errors increased in that label scope | Every user failed |
| p99 histogram rose | Estimated tail latency rose for observed population | One shown trace is typical |
| Red server span | Instrumentation marked that operation failed | That service is root cause |
| No child span | Child was not observed | Call never happened |
| Same trace and parent chain | Strong causal relationship | Payload/business outcome is correct |
| Same user and timestamp | Events may be related | Events are the same request |
| Service map edge absent | Backend did not display observed edge | Dependency does not exist |
| No log result | Query found no retained matching record | Event did not happen |

## 8.2 Production query checklist

```text
[ ] Correct environment, tenant, and access scope
[ ] UTC start/end plus baseline and known skew/ingestion lag
[ ] Exact indexed ID before free text
[ ] Normalized route, service, version, region, instance
[ ] Event time and ingestion time retained
[ ] Result limit/truncation understood
[ ] Sampling, parse, drop, and retention state checked
[ ] Secrets and personal data excluded
```

## 8.3 Instrumentation checklist

```text
[ ] Stable service.name, version, environment, instance, region
[ ] RED at ingress and egress; USE for constrained resources
[ ] W3C inject/extract on HTTP and messaging
[ ] Context capture/restore/clear on async execution
[ ] Structured trace_id/span_id/request_id fields
[ ] Business IDs approved, tokenized where required
[ ] Baggage allowlist and size/privacy controls
[ ] Normalized routes and bounded metric labels
[ ] Errors, deadlines, attempts, queue/pool time recorded safely
[ ] Exemplars where supported
[ ] SDK/collector/backend drops and ingestion delay monitored
[ ] Synthetic propagation and failure-path tests
```

## 8.4 Concise interview framework

```text
Scope -> identify observer -> inspect telemetry health
      -> metrics/business signals for when/how much/which cohort
      -> trace for first error and critical path
      -> logs for exact event and state
      -> code/config/runtime/dependency proof
      -> safe mitigation
      -> original-path and business-state verification
      -> permanent correction and prevention
```

The senior-level habit is to say both **what the evidence supports** and **what it cannot establish**. Observability reduces uncertainty; disciplined reasoning closes the remaining gap.
