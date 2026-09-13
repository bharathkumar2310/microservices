# Microservices Production Troubleshooting Interview Curriculum

## Why this guide exists

This curriculum turns common production-support interview questions into a structured learning path. It is written for someone starting with little production troubleshooting knowledge and builds toward senior-level reasoning.

The goal is not to memorize a hundred isolated answers. Most incidents can be solved by repeatedly answering five questions:

1. **What exactly is failing?**
2. **Where in the end-to-end path is it failing?**
3. **Why is that component failing?**
4. **What evidence proves the root cause?**
5. **How do we restore service and prevent recurrence safely?**

## Chapters

| Order | Topic | What you will learn |
|---:|---|---|
| A | [Service-to-service communication](../../L301_Serv_To_Serv.md) | DNS, TCP, TLS, gateways, 502/503/504, instance-specific failures, and health checks |
| B | [API performance and sudden latency](./B_API_Performance.md) | Latency decomposition, percentiles, queueing, endpoint-specific and system-wide slowness |
| C | [CPU, memory, JVM, and threads](./C_CPU_Memory_JVM.md) | CPU diagnosis, heap/native memory, GC, OOM, thread dumps, deadlocks, and pool exhaustion |
| D | [Database troubleshooting](./D_Database.md) | Query latency, execution plans, locks, deadlocks, connection pools, N+1, and data-size problems |
| E | [Kafka and asynchronous communication](./E_Kafka_Async.md) | Consumer lag, offsets, duplicates, ordering, rebalances, poison records, retries, and idempotency |
| F | [Distributed logging, metrics, and tracing](./F_Observability_Logging_Tracing.md) | Trace context, log correlation, RED/USE metrics, sampling, clock skew, and evidence-driven debugging |
| G | [Resilience](./G_Resilience.md) | Deadlines, timeouts, retries, jitter, circuit breakers, bulkheads, backpressure, and graceful degradation |
| H | [Distributed transactions and Saga](./H_Distributed_Transactions_Saga.md) | Eventual consistency, compensation, orchestration, choreography, idempotency, and stuck-saga recovery |
| I | [Deployment and configuration](./I_Deployment_Configuration.md) | Diff-first diagnosis, rollouts, mixed versions, config/secret drift, migrations, probes, and rollback |
| J | [Scaling and load](./J_Scaling_Load.md) | Throughput, concurrency, Little's Law, saturation, autoscaling, bottlenecks, and load-test interpretation |
| K | [Caching](./K_Caching.md) | Cache strategies, invalidation, stale data, Redis failure, stampedes, hot keys, and hit-ratio analysis |
| L | [Security troubleshooting](./L_Security.md) | 401/403, JWT claims/signatures, JWKS rotation, token propagation, mTLS, workload identity, and safe diagnosis |
| M | [Complex real-production scenarios](./M_Complex_Production_Scenarios.md) | Cross-domain incidents that require combining all earlier chapters |

## Recommended learning order

### Stage 1 - Understand the request path

Study chapters A and F first. You should be able to draw:

```text
client
  -> DNS
  -> network/TCP
  -> TLS
  -> gateway/load balancer
  -> service queue and code
  -> database/cache/Kafka/downstream service
  -> response
```

You should also understand what logs, metrics, and traces can and cannot prove.

### Stage 2 - Understand latency and resource saturation

Study B, C, and D. Learn to separate:

```text
latency != CPU only

latency =
  queue wait
  + connection acquisition
  + execution
  + dependency wait
  + serialization
  + network transfer
```

### Stage 3 - Understand asynchronous processing and correctness

Study E and H. Learn why distributed systems normally provide retries and at-least-once delivery, and why business operations therefore need durable idempotency and reconciliation.

### Stage 4 - Understand protection and capacity

Study G, J, and K. Learn how timeouts, retries, circuit breakers, bulkheads, rate limits, caching, and autoscaling interact. A protection mechanism can become harmful when configured without a total deadline, capacity model, or correctness rules.

### Stage 5 - Understand change and trust boundaries

Study I and L. Many outages start with a deployment, configuration, secret, certificate, policy, or identity change rather than an application-code defect.

### Stage 6 - Practice synthesis

Study M without looking at the answers first. For each scenario:

1. State the impact and immediate safety concern.
2. Identify the exact error and its generator.
3. Draw the request/data path.
4. List the smallest set of evidence needed.
5. Form one testable hypothesis.
6. Explain safe mitigation separately from permanent correction.
7. Explain how to verify and prevent recurrence.

## Universal production investigation framework

### Phase 1 - Confirm and scope

Record:

```text
What is the user-visible impact?
When did it begin?
Is it still happening?
Which services, endpoints, tenants, regions, and versions are affected?
What percentage fails?
What changed near the first failure?
```

Do not begin with a restart. A restart may remove the evidence and may only hide a leak or race temporarily.

### Phase 2 - Protect users and data

Depending on the incident:

- Drain one bad instance.
- Pause an unsafe rollout.
- Roll back a clearly bad version.
- Apply an existing rate limit.
- Disable a noncritical feature with an approved flag.
- Stop unsafe retries that can duplicate writes.
- Preserve idempotency and transaction correctness.

Mitigation and root cause are different. "Restart fixed it" is not a root cause.

### Phase 3 - Classify the failure layer

```text
configuration
DNS/service discovery
network/TCP
TLS/identity
gateway/load balancer
HTTP contract
queue/thread/pool
application code
database/cache/Kafka
downstream service
deployment/infrastructure
```

### Phase 4 - Correlate evidence

Use:

- **Metrics** to find when, how much, and which component changed.
- **Traces** to locate the slow or failing span.
- **Logs** to explain the specific event.
- **Profiles, thread dumps, heap dumps, query plans, broker diagnostics, and packet captures** only when the earlier evidence points to those layers.

### Phase 5 - Prove the root cause

A strong root-cause statement contains a mechanism:

```text
Weak:
The database was slow.

Strong:
A new unbounded search query performed a full scan of 40 million rows.
It saturated database I/O, raised query p99 from 80 ms to 8 seconds,
held all 50 application pool connections, and caused API requests to
time out while waiting for a connection.
```

### Phase 6 - Correct and verify

Verification should include:

- The original failing business path.
- All instances, zones, and versions.
- Error rate and latency percentiles.
- Queue, pool, and resource saturation.
- Dependency health.
- Duplicate or incomplete business operations.
- A representative load period.

### Phase 7 - Prevent recurrence

Ask:

- Why did the defect reach production?
- Why did detection take this long?
- Why did the system not contain the failure?
- What automated test, deployment guard, alert, capacity limit, runbook, or design change closes the gap?

## How to structure an interview answer

Use this six-part format:

1. **Clarify:** exact error, scope, timing, and recent changes.
2. **Localize:** identify the failing layer or slow span.
3. **Investigate:** list ordered checks and explain what each result means.
4. **Mitigate:** protect users and data safely.
5. **Correct:** fix the proven mechanism, not only the symptom.
6. **Verify and prevent:** retest, monitor, and close detection/design gaps.

Example opening:

> First I would confirm the user impact, exact error, affected scope, and first-failure time. I would compare that time with deployments and infrastructure/configuration changes. Then I would use service-level metrics to identify the component and instance where error rate, latency, or saturation changed, follow a failed trace through the request path, and use correlated logs or runtime diagnostics to explain why that span failed. I would mitigate impact without risking duplicate or inconsistent work, fix the measured bottleneck, verify the original business flow across all instances, and add a preventive control.

## Important habits

- Use UTC timestamps and request/trace IDs.
- Compare a failing request or instance with a successful one.
- Inspect p95 and p99, not only averages.
- Separate queue time from execution time.
- Separate connection acquisition from query or downstream execution.
- Understand which component generated an HTTP status.
- Treat retries as extra load and duplicate-delivery risk.
- Never log access tokens, passwords, private keys, or sensitive payloads.
- Never disable TLS/JWT validation as a production fix.
- Never change several unrelated settings at once and then claim one was the cause.
- Do not increase pools, heaps, threads, replicas, or timeouts until you identify the constrained resource and downstream capacity.

