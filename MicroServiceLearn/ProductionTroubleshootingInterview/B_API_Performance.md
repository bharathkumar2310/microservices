# Production Troubleshooting Study Chapter: API Performance

## Purpose

This chapter teaches a repeatable way to diagnose slow and timing-out APIs without guessing. It starts with the request path and basic latency vocabulary, then builds toward production diagnosis with traces, metrics, logs, profiles, database evidence, queueing theory, containers, and safe incident response.

The examples use Java, Spring Boot, Micrometer, Prometheus, Linux, and Kubernetes where useful, but the reasoning applies to other languages and platforms.

> Safety: Replace placeholders such as `<namespace>`, `<pod>`, `<route>`, and `<trace-id>` with approved values. Run read-only commands first. Do not paste tokens, customer data, heap contents, or sensitive payloads into shared terminals or tickets. Test from the affected network path and identity when possible.

## Learning goals

After studying this chapter, you should be able to:

1. Decompose end-to-end latency instead of treating "the API" as one black box.
2. Explain p50, p95, p99, throughput, concurrency, utilization, queueing, deadlines, and timeouts.
3. Separate a symptom from what the evidence actually proves.
4. Narrow an incident by time, endpoint, instance, tenant, region, dependency, and deployment version.
5. Use traces, RED metrics, saturation metrics, logs, profiles, and dependency telemetry together.
6. Distinguish CPU work from waiting on threads, locks, pools, databases, networks, and downstream services.
7. Mitigate an incident without hiding the cause or creating retry and timeout amplification.
8. Turn a confirmed cause into a durable fix, an alert, and a regression test.
9. Give concise, evidence-based interview answers while retaining enough depth for follow-up questions.

---

# 1. Foundational mental model

## 1.1 Latency is a sum of stages

An observed response time is the elapsed time from the caller's perspective. It can contain work and waiting in many places:

```text
End-to-end latency
  = client-side queue or connection-pool wait
  + DNS lookup
  + TCP connect
  + TLS handshake
  + request upload
  + gateway/load-balancer queue and processing
  + server accept queue
  + server worker/event-loop queue
  + application CPU time
  + lock or synchronization wait
  + database connection-pool wait
  + database execution and lock wait
  + cache/message-broker/downstream waits
  + serialization/compression
  + response transfer
```

The equation is conceptual: some stages overlap, some are skipped because a pooled connection is reused, and an asynchronous system may schedule work differently. Its value is that it prevents the weak conclusion "the service is slow."

### Service time, waiting time, and response time

- **Service time** is time actively spent performing the work at a resource.
- **Waiting time** is time spent queued or blocked before a resource becomes available.
- **Response time** is service time plus all waiting and transfer time visible to the observer.

A method can use only 20 ms of CPU yet take 5 seconds because it waited 4.98 seconds for a connection, lock, thread, disk, or downstream response.

## 1.2 Throughput, concurrency, utilization, and queueing

- **Throughput** is completed work per unit time, such as requests per second (RPS).
- **Concurrency** is work currently in progress, including requests that are waiting.
- **Utilization** is the fraction of a resource's available capacity being used.
- **Saturation** means demand is at or beyond useful capacity, so queues or rejections grow.

Little's Law gives a valuable steady-state relationship:

```text
average concurrency = throughput * average response time
```

At 100 RPS and 0.2 seconds average latency, about 20 requests are in flight. If latency becomes 5 seconds while traffic remains 100 RPS, about 500 requests can be in flight. Those requests consume sockets, buffers, threads, connections, and memory. Slowness can therefore create more slowness.

Queueing is nonlinear. As a constrained resource approaches full utilization, small demand increases can cause large wait-time increases. This is why peak-traffic latency can rise sharply before CPU shows a perfectly flat 100 percent.

## 1.3 Percentiles: p50, p95, and p99

For a set of request durations:

- **p50** is the median: half completed faster and half slower.
- **p95** means 95 percent completed at or below that duration; the slowest 5 percent took longer.
- **p99** means 99 percent completed at or below that duration; the slowest 1 percent took longer.
- **Maximum** is useful for individual events but is too noisy to represent typical behavior.
- **Average** can hide a bad tail.

Example:

```text
999 requests take 100 ms
1 request takes 20,000 ms

average is about 120 ms
p50 is 100 ms
p99 is 100 ms
p99.9 or maximum reveals the 20,000 ms tail, depending on the
percentile definition and histogram bucket boundaries
maximum is 20,000 ms
```

Always state the time window, endpoint, status class, region, instance, and percentile aggregation method. "p99 is 2 seconds" is incomplete without scope.

Do not average precomputed percentiles from instances. Aggregate histogram buckets or analyze raw events. An average of pod p99 values is not the fleet p99.

## 1.4 Deadline, timeout, cancellation, and retry

- A **timeout** limits one operation or phase, such as connect, pool acquisition, socket read, or statement execution.
- A **deadline** is the remaining end-to-end time budget for the request.
- **Cancellation** tells downstream work to stop when the caller no longer needs it.
- A **retry** is a new attempt; it consumes extra capacity and must fit inside the original deadline.

Timeouts do not make work faster. If the caller times out at 3 seconds but the server continues for 30 seconds, capacity is consumed for a response nobody will use. Blindly increasing timeouts retains more in-flight work and can turn slowness into resource exhaustion.

Retries should be limited to operations that are safe to retry and to transient failures. Use a small attempt cap, exponential backoff, jitter, a retry budget, and an overall deadline. Never retry every layer independently.

## 1.5 Scope is evidence

The shape of an incident narrows the fault domain:

| Scope | Strong starting hypotheses |
|---|---|
| One endpoint | Endpoint code, query, payload, route-specific filter, lock, or downstream |
| All endpoints in one service | Shared resource, process, node, gateway route, runtime, or common dependency |
| One instance | Instance-local state, bad node, uneven traffic, cache, leak, version, or noisy neighbor |
| All instances | Shared dependency, global configuration, release, traffic, database, gateway, or network |
| One tenant/data shape | Data volume, skew, authorization path, partition, query plan, or external integration |
| Peak traffic only | Queueing, finite pool, rate limit, downstream capacity, autoscaling lag, or throttling |
| Immediately after deployment | Code/config/schema/route/version/resource change |
| Gradual over hours | Leak, accumulation, fragmentation, queue backlog, cache behavior, connection leak, or data growth |

This table suggests where to look. It does not prove a cause.

## 1.6 Glossary

| Term | Meaning |
|---|---|
| RED | Rate, Errors, Duration: the core request signals |
| USE | Utilization, Saturation, Errors: a resource-oriented method |
| SLI | Measured service-level indicator, such as successful request latency |
| SLO | Target for an SLI, such as 99.9 percent successful requests per month |
| Error budget | Allowed unreliability implied by an SLO |
| Apdex | A coarse satisfied/tolerating/frustrated latency score; useful but less diagnostic than distributions |
| Tail latency | High-percentile latency such as p95 or p99 |
| Queue depth | Work waiting to be served |
| Pool wait | Time waiting to borrow a thread, connection, or other pooled resource |
| Head-of-line blocking | Slow work ahead of fast work delays everything behind it |
| Backpressure | A mechanism that slows or rejects producers when consumers cannot keep up |
| Load shedding | Deliberately rejecting low-priority/excess work to protect useful capacity |
| Circuit breaker | Stops calls to a failing dependency for a bounded period based on measured failures |
| Bulkhead | Isolates resources so one dependency or workload cannot consume all shared capacity |
| Cold start | Startup or first-use cost such as class loading, JIT compilation, cache warming, or new connections |
| Coordinated omission | A load-test error that stops sending requests while the system is stalled, understating latency |
| Cardinality | Number of unique metric label combinations; unbounded labels can overload observability systems |

---

# 2. Essential metrics and what they mean

## 2.1 Request and dependency metrics

| Metric | What it tells you | Important interpretation |
|---|---|---|
| Request rate by route/status | Offered and completed workload | Compare successful and attempted rate; a fall can mean rejection or caller abandonment |
| Duration histogram by route | Latency distribution | Compare p50, p95, p99 and bucket counts; do not rely only on average |
| Error rate by status/exception | Failure mode | Separate caller cancellation, timeout, 4xx, 5xx, and transport errors |
| In-flight requests | Current concurrency | Rising with flat throughput often means work is waiting |
| Request/worker queue depth | Pending work | A sustained rise means arrivals exceed completions |
| Rejected requests/tasks | Hard saturation | Rejection may protect the service but is an incident signal |
| Trace span duration | Time in one stage | Missing spans and sampling policy matter; a long child span identifies waiting downstream |
| Dependency call rate/latency/errors | External contribution | Compare attempt count with inbound count to detect fan-out or retry amplification |

## 2.2 Resource and pool metrics

| Resource | Metrics | Diagnostic pattern |
|---|---|---|
| CPU | utilization, per-core use, run queue, throttled seconds/periods | High run queue or throttling can delay work; low process CPU does not rule out downstream waiting |
| Server threads | busy, max, queued, rejected | Busy=max plus queue growth indicates saturation; many idle threads do not |
| Event loops | loop lag, blocked-loop duration, queue | One blocking call can stall many requests |
| DB connection pool | active, idle, max, pending, acquisition time, timeout count | Pending/acquisition time rising with active=max indicates pool wait |
| HTTP client pool | leased, available, pending, connect/reuse count | Pending callers with no available connection indicates pool saturation or leak |
| Database | query latency, CPU, I/O, locks, rows scanned, connections | Long execution is different from pool acquisition or lock wait |
| Cache | hit ratio, command latency, evictions, errors | Falling hits can shift load to DB and raise latency |
| Network | RTT, retransmits, packet loss, connection errors, throughput | Application latency can be network transfer or loss even after connect |
| Disk | latency, IOPS, queue, utilization | Logging, temp files, paging, or database I/O may stall |
| GC/runtime | allocation rate, pause time, CPU in GC | Pauses or allocation pressure can affect tail latency |
| Kubernetes | pod CPU/memory, CPU throttling, restarts, readiness, replicas | Requested CPU and limit behavior matter, not only node utilization |

## 2.3 Useful PromQL patterns

Metric names differ by instrumentation. Treat these as patterns, not copy-paste guarantees.

```promql
# Request rate over five minutes
sum by (service, route, status) (
  rate(http_server_requests_seconds_count{service="<service>"}[5m])
)

# Fleet p95 from cumulative histogram buckets
histogram_quantile(
  0.95,
  sum by (le, route) (
    rate(http_server_requests_seconds_bucket{service="<service>"}[5m])
  )
)

# In-flight requests
sum by (pod) (http_server_requests_active{service="<service>"})

# Container CPU throttling rate
sum by (pod) (
  rate(container_cpu_cfs_throttled_seconds_total{
    namespace="<namespace>", pod=~"<pod-pattern>"
  }[5m])
)

# HikariCP callers waiting for a DB connection
max by (pod, pool) (hikaricp_connections_pending{service="<service>"})
```

Use a window long enough to avoid noise but short enough to retain the incident. Compare with the same service and traffic pattern from a healthy period.

## 2.4 Java and Spring evidence

Common Micrometer/Spring Boot metrics include:

- `http.server.requests` duration and count.
- `http.client.requests` or client-library-specific metrics.
- `jvm.threads.*`, `jvm.gc.*`, `jvm.memory.*`, and `process.cpu.usage`.
- `hikaricp.connections.active`, `.idle`, `.pending`, `.max`, and acquisition timing.
- Embedded server metrics such as Tomcat current/busy/max threads.

Actuator endpoints can expose valuable data, but production exposure must be authenticated, authorized, network-restricted, and intentionally configured. Never expose heap dumps, environment values, or unrestricted diagnostics publicly.

---

# 3. Generic API performance investigation workflow

## Step 1 - Stabilize and define the incident

Record:

```text
First bad timestamp and timezone:
Last known good timestamp:
Affected service/version:
Affected route/method/status:
Observed p50/p95/p99 and timeout/error rate:
Normal baseline:
Traffic and concurrency:
Affected instances/zones/tenants/payload sizes:
Recent code/config/schema/infra changes:
Example request ID/trace ID:
```

Protect users first with evidence-based actions: stop a harmful rollout, route away from a proven bad instance, shed optional work, reduce retry amplification, or scale a proven horizontally scalable bottleneck. Preserve diagnostics before replacing a process when safe.

## Step 2 - Verify the measurement boundary

Determine whether the 5 seconds was measured by:

- End user or synthetic client.
- CDN, ingress, gateway, or load balancer.
- Service access log.
- Application timer around business logic.
- Database or downstream client.

Clock mismatch, aggregation, sampling, and different start/end points can produce apparently conflicting numbers. Compare one trace/request across boundaries.

## Step 3 - Slice the symptom

Compare:

- One endpoint versus all endpoints.
- One instance versus the fleet.
- One zone/node/region versus all.
- One tenant, payload size, key range, or status.
- New versus old application version.
- Warm versus newly started instances.
- Peak versus quiet traffic.

A dimension that cleanly separates good and bad requests is often more valuable than another dashboard.

## Step 4 - Follow a representative trace waterfall

Choose:

1. A slow successful request.
2. A normal successful request with similar input.
3. A timed-out or failed request.

For each, identify queue spans, application spans, DB calls, downstream calls, retries, and missing time. Confirm trace sampling preserves errors and slow requests. If a 5-second request has only 200 ms of recorded spans, investigate uninstrumented queueing, pool acquisition, locks, request/response transfer, or instrumentation gaps.

## Step 5 - Apply RED to services and USE to resources

For each relevant component:

- Rate: did demand or attempt count change?
- Errors: which status or exception changed?
- Duration: which percentile and operation changed?
- Utilization: how much of the resource is busy?
- Saturation: is work waiting or rejected?
- Errors: is the resource reporting failures?

Queue depth and wait duration are usually stronger saturation evidence than utilization alone.

## Step 6 - Correlate with changes

Overlay deployment, feature-flag, configuration, certificate, autoscaling, node, database plan/statistics, index, schema, and dependency changes. Correlation is a hypothesis. Prove it with a version split, rollback/canary result, configuration diff, or direct technical evidence.

## Step 7 - Capture runtime evidence

Choose the least invasive tool that can answer the current question:

```bash
# Client timing breakdown; omit credentials and sensitive payloads
curl --silent --show-error --output /dev/null \
  --connect-timeout 5 --max-time 15 \
  --write-out 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} first_byte=%{time_starttransfer} total=%{time_total} code=%{http_code}\n' \
  https://<host>/<safe-route>

# Kubernetes status and recent events
kubectl -n <namespace> get pods -l app=<service> -o wide
kubectl -n <namespace> top pod -l app=<service> --containers
kubectl -n <namespace> describe pod <pod>

# Java thread dump; collect multiple snapshots, for example 10 seconds apart
jcmd <pid> Thread.print -l > <approved-secure-path>/threads-1.txt

# Short Java Flight Recorder capture when approved
jcmd <pid> JFR.start name=latency settings=profile duration=60s \
  filename=<approved-secure-path>/latency.jfr
```

Interpretation:

- DNS/connect/TLS high in `curl`: investigate before application logic.
- First-byte high with quick connect: gateway queue, server queue, application, or dependency.
- Total much higher than first-byte: response streaming/size/network/client read.
- Same stack across multiple thread dumps: persistent blocking or hot loop; one dump is only a snapshot.
- JFR: look for execution samples, monitor contention, socket/file I/O, allocation pressure, and GC pauses.

## Step 8 - Mitigate the proven bottleneck

Prefer reversible, bounded changes:

- Pause or roll back a harmful deployment.
- Isolate or drain a proven bad instance while preserving evidence.
- Disable a costly optional feature through an established flag.
- Apply load shedding or rate limiting at an intentional boundary.
- Stop retry storms and align retry budgets.
- Scale the saturated tier only if its dependency can accept the extra load.
- Fail over through an approved, tested path.

Do not blindly restart, increase threads, enlarge pools, raise timeouts, or scale callers. Those actions may erase evidence or push overload downstream.

## Step 9 - Verify and close the loop

Verify at the original measurement boundary and under representative traffic:

- p50, p95, p99, throughput, timeout rate, and error rate recover.
- Queue depths and pool waits return to baseline.
- Dependency metrics recover.
- No hidden rejection or data-integrity problem replaced latency.
- The fix survives a traffic cycle and a new instance start.

Document the causal chain, not just the changed setting.

---

# 4. Original interview questions

## 1. An API that normally takes 200 ms suddenly takes 5 seconds. How would you investigate?

### What the symptom means and does not prove

The observer measured a 4.8-second regression at some boundary. It proves neither that application code consumed 5 seconds of CPU nor that every request is slow. First determine whether this is p50, p95, p99, one request, a timeout threshold, or an average. A 5-second cluster can reveal a fixed timeout, retry delay, DNS fallback, pool-acquisition timeout, lock wait, or scheduled pause.

### Where it can happen

It can occur in the client queue, DNS/TCP/TLS path, gateway, load balancer, server queue, application code, JVM pause, lock, DB/HTTP pool, database, cache, broker, downstream service, serialization, or response transfer.

### Detailed causes and mechanisms

| Cause | Mechanism |
|---|---|
| Traffic or concurrency jump | A finite worker/pool/resource reaches capacity; requests wait before work starts |
| Slow DB query | A changed plan, stale statistics, lock, missing index, or larger data set increases execution time |
| DB pool exhaustion | Connections remain busy or leak; request threads wait to borrow one |
| Downstream regression | A long child call dominates the parent request and may trigger retries |
| CPU saturation/throttling | Runnable work waits for CPU; a container can be throttled while node CPU looks available |
| Lock contention | Requests serialize behind one owner, often with low aggregate CPU |
| GC pause/allocation burst | Application threads pause or lose CPU to collection |
| Connection/DNS/TLS issue | New connections pay resolution, handshake, retransmission, or fallback delay |
| Payload/data change | More rows, larger JSON, compression, validation, or fan-out increases work |
| Deployment/config change | A feature, timeout, proxy, query, logging, instrumentation, or resource limit changed |
| Retry amplification | One logical request creates multiple physical attempts and more load |

### Ordered investigation

1. Confirm the exact start time, observer, route, method, status, percentile, and affected dimensions.
2. Compare a healthy and slow trace with equivalent inputs.
3. Split the trace into pre-server, queue, application, dependency, and response time.
4. Check request rate, in-flight count, queue depth, errors, and instance distribution.
5. Check server threads/event-loop lag, CPU throttling, GC pauses, DB/HTTP pool pending counts, and acquisition time.
6. Inspect long DB/downstream spans, query plans, lock waits, retries, and per-attempt deadlines.
7. Correlate the first bad minute with releases, flags, config, database, infrastructure, and dependency events.
8. If time remains inside the process, take repeated thread dumps or a bounded profile/JFR.
9. Mitigate the confirmed bottleneck and verify the original p50/p95/p99 boundary.

### Commands, tools, metrics, and interpretation

- Use the `curl` timing command in the generic workflow. A high `time_starttransfer` with normal connect isolates post-connect waiting.
- Query traces by route and `duration > 5s`; compare span waterfalls and attempt counts.
- Check `http.server.requests`, active requests, server queue, Hikari pending/acquisition, downstream latency, GC pause, and CPU throttling.
- Use DB slow-query/lock tools and `EXPLAIN` on a safe representative query. Do not run `EXPLAIN ANALYZE` on expensive production writes.
- Repeated identical `BLOCKED`/`WAITING` stacks identify a persistent wait. High execution samples in one method identify CPU work.

### Immediate mitigation

Pause or roll back a correlated harmful rollout, drain a proven bad instance, disable an expensive optional path, reduce unsafe retries, shed excess low-priority load, or scale the proven constrained tier if dependencies have headroom.

### Root-cause fixes

Optimize the measured query/algorithm; correct locking; size and manage pools against downstream capacity; propagate deadlines/cancellation; bound fan-out; reuse connections; correct CPU requests/limits; fix leaks; and redesign long-running synchronous work as an asynchronous job where appropriate.

### Prevention and alerting

Alert on SLO burn rate, p95/p99 plus traffic/error context, queue/pool wait, rejected work, dependency latency, and CPU throttling. Preserve route-level histograms, slow traces, deployment markers, and representative load tests that avoid coordinated omission.

### Common mistakes

- Looking only at average latency or host CPU.
- Treating correlation with a deployment as proof.
- Increasing the timeout before finding where time is spent.
- Restarting before capturing traces, dumps, and pool state.
- Scaling Service A when Service A is waiting on an overloaded database.

### Concise interview-ready answer

> I would first define the scope: measurement boundary, route, percentile, status, start time, and affected instances or tenants. Then I would compare one normal and one 5-second trace to decompose DNS/connect/TLS, queueing, application, database, downstream, and response time. I would correlate RED metrics with thread/event-loop queues, DB and HTTP pool waits, CPU throttling, GC, database locks/query plans, and recent changes. I would mitigate the proven bottleneck, not blindly raise a timeout or restart, and verify recovery at p50, p95, p99, error rate, and queue depth.

---

## 2. Only one API endpoint is slow while all other endpoints are normal. What would you check?

### What the symptom means and does not prove

The service and common request path can handle at least some work normally. This narrows attention to route-specific code, data, dependencies, policies, and resource isolation. It does not prove the host, JVM, gateway, or database is healthy; the endpoint may uniquely stress a shared resource.

### Where it can happen

Check route matching and filters, authorization, controller/service code, endpoint-specific thread pool, cache, query, partition, downstream call, serializer, response size, gateway policy, and client behavior.

### Detailed causes and mechanisms

- A missing index or parameter-sensitive query plan makes only this endpoint's SQL expensive.
- N+1 access turns one request into hundreds of queries.
- Large or skewed tenant data increases rows scanned, objects allocated, and output serialized.
- A route-specific downstream service, cache key, distributed lock, or bulkhead is slow.
- One endpoint performs file I/O, report generation, synchronous messaging, or compression.
- Regex validation, JSON mapping, encryption, or authorization checks scale badly with input size.
- Cache misses or an eviction wave move this route's workload to the database.
- A route-specific rate/concurrency policy queues rather than rejects.
- One HTTP method or content type follows a different gateway or application path.
- Client retries disproportionately amplify this route.

### Ordered investigation

1. Compare route-level p50/p95/p99, rate, errors, in-flight count, payload size, tenant, and status.
2. Compare normal and slow traces for the same endpoint and data class.
3. Compare this endpoint's spans with a healthy endpoint only to identify common versus unique stages.
4. Count queries/downstream calls per request and inspect the longest unique span.
5. Check route-specific queues, semaphores, locks, cache hit ratio, response size, and retry count.
6. Inspect query plans, rows estimated/actual, lock waits, and data skew.
7. Profile the endpoint with representative input if time is in local code.
8. Reproduce in a safe environment with production-shaped data volume and the same configuration.

### Commands, tools, metrics, and interpretation

```promql
histogram_quantile(
  0.99,
  sum by (le, route) (
    rate(http_server_requests_seconds_bucket{service="<service>"}[5m])
  )
)
```

- Group traces by route, tenant class, payload-size bucket, and version. Never put raw customer IDs in metric labels.
- In Spring, inspect controller-to-repository spans, Hibernate statistics in a safe test, Hikari wait, cache metrics, and method profiles.
- In the DB, compare plans and rows for fast and slow parameter classes. A fast test with tiny data does not invalidate production data skew.
- If total latency tracks response bytes, inspect serialization/compression and network transfer.

### Immediate mitigation

Rate-limit or temporarily disable only the expensive optional route, serve a safe degraded response, bypass a broken optional dependency, route heavy report/export work to an asynchronous job, or roll back the route-specific change.

### Root-cause fixes

Fix the query/index and N+1 behavior, paginate/bound responses, isolate workload with a correctly sized bulkhead, cache safely, remove route-specific blocking work, optimize algorithms, and set explicit data-size/concurrency limits.

### Prevention and alerting

Maintain per-route histograms and error metrics for important routes; test worst-case data shapes; alert on route-specific SLO burn, query count, pool wait, response size, and bulkhead rejection. Use bounded labels.

### Common mistakes

- Assuming "other endpoints are fine" rules out a shared database or JVM.
- Testing only with a small development data set.
- Adding an index without validating write cost and the actual plan.
- Caching incorrect or unbounded data as a quick fix.
- Increasing a global pool for one endpoint and harming every other route.

### Concise interview-ready answer

> Since only one endpoint is slow, I would compare its route-level latency, traffic, errors, payload and tenant dimensions, then inspect a fast and slow trace. I would focus on code and dependencies unique to that route: query plan and locks, N+1 calls, cache misses, data skew, downstream fan-out, route-specific pools or locks, and serialization size. I would mitigate at the route boundary and make a targeted fix rather than changing global timeouts or pool sizes.

---

## 3. All APIs in a service suddenly become slow. What could be the possible causes?

### What the symptom means and does not prove

A shared path or resource is likely affected because unrelated routes regressed together. It does not prove the application process is the source; a gateway, node, common database, runtime pause, shared pool, or observability pipeline can affect all routes.

### Where it can happen

At the service ingress, load balancer, every service instance, one overloaded instance receiving disproportionate traffic, JVM/runtime, shared worker pool, shared connection pool, database/cache, node, network zone, or common downstream dependency.

### Detailed causes and mechanisms

| Category | Mechanism affecting all routes |
|---|---|
| Traffic overload | Global queue grows; all requests wait for workers or connections |
| Uneven balancing | A bad/hot instance makes the fleet tail slow |
| Worker/event-loop exhaustion | Shared execution capacity is occupied or blocked |
| Shared DB/HTTP pool exhaustion | Every route waits to acquire the same finite connection resource |
| Database/cache degradation | Common persistence operations slow across routes |
| CPU saturation or cgroup throttling | All runnable application work receives less CPU |
| GC/runtime pause | Stop-the-world pauses or runtime contention delay all request threads |
| Node/network issue | Packet loss, noisy neighbor, disk/logging stall, DNS, or service-mesh problem |
| Gateway/sidecar issue | Queueing, policy, TLS, logging, or capacity regression precedes the app |
| Deployment/config | Common filter, interceptor, feature, logging, tracing, or resource limit changed |
| Retry storm | Failed/slow calls multiply inbound and outbound work |

### Ordered investigation

1. Verify whether all routes, statuses, instances, zones, and versions are affected.
2. Compare gateway latency with service access-log latency to find pre-service delay.
3. Check per-instance request rate and latency; isolate a hot or bad instance pattern.
4. Review shared server queue, active threads/event-loop lag, in-flight requests, and rejections.
5. Review CPU utilization/throttling, GC pauses, disk/logging latency, and network errors.
6. Review DB and HTTP pools, common dependency latency, DB locks/CPU/I/O, and cache health.
7. Correlate with fleet-wide deploy/config/sidecar/node/dependency changes.
8. Capture repeated thread dumps or a profile if in-process evidence remains unexplained.

### Commands, tools, metrics, and interpretation

- `kubectl get pods -o wide` plus per-pod latency/RPS reveals node or instance concentration.
- `kubectl describe pod <pod>` reveals changed limits, restarts, readiness, and events.
- Gateway upstream time minus total time helps separate gateway/client transfer from backend time.
- Busy server threads at max with a growing queue means shared execution saturation; low busy threads with high gateway latency points earlier in the path.
- Hikari pending on all pods plus high DB execution time means DB capacity/query issues; pending with normal DB execution can indicate leaks or undersized pool relative to legitimate concurrency.

### Immediate mitigation

Roll back a fleet-wide regression, restore a failed common dependency, remove a proven unhealthy node/instance through normal procedures, stop retry amplification, shed optional traffic, or scale the proven bottleneck within downstream capacity.

### Root-cause fixes

Remove blocking work from shared pools, isolate workloads, repair dependency/query capacity, correct container CPU configuration, make load balancing readiness-aware, bound retries and queues, and performance-test common filters/interceptors.

### Prevention and alerting

Use service and per-instance dashboards, route and fleet histograms, dependency SLOs, queue/pool alerts, throttling/GC alerts, canaries, automated rollback criteria, and zone/node breakdowns.

### Common mistakes

- Aggregating the fleet and missing one bad pod.
- Checking only application duration and ignoring gateway queueing.
- Scaling every tier at once, destroying causal evidence.
- Increasing shared threads while they are blocked on a fixed-size dependency.
- Calling readiness healthy when only a shallow endpoint works.

### Concise interview-ready answer

> If all APIs slow together, I look for a shared path or resource. I compare gateway and service timing, then slice by pod, node, zone, and version. I check shared worker queues, connection-pool waits, common DB/cache/downstream latency, CPU throttling, GC, network and logging I/O, load distribution, retries, and recent fleet-wide changes. The mitigation and permanent fix target the measured shared bottleneck, with per-instance and queue metrics used to prove recovery.

---

## 4. Service A is slow, but its CPU and memory look normal. How would you investigate?

### What the symptom means and does not prove

Normal CPU and memory only say those two sampled resource views are not obviously high. They do not show whether Service A is waiting on I/O, a lock, a pool, a queue, throttled CPU quota, DNS, a downstream service, or the network. They also may hide short spikes, per-core saturation, pauses, or one bad instance.

### Where it can happen

In server queues, blocked/waiting threads, event loops, DB/HTTP connection pools, locks, databases, caches, downstream services, DNS/TLS/network, disk/logging, gateway queues, and client response transfer.

### Detailed causes and mechanisms

- Threads wait for DB/downstream I/O, so elapsed latency rises while CPU stays low.
- Pool exhaustion queues callers; memory can remain stable.
- A lock serializes requests behind one owner.
- A deadlock or lost signal stops progress without consuming CPU.
- CPU throttling can limit a container despite modest host CPU or coarse utilization charts.
- Disk or synchronous logging blocks writers.
- Network loss causes retransmissions and long reads.
- A remote rate limiter delays or rejects calls.
- An event loop is blocked by one synchronous operation while total CPU remains low.
- A gateway queues before the service, so service resources appear normal.

### Ordered investigation

1. Compare client/gateway/service timestamps to locate whether A receives requests late or completes them late.
2. Inspect traces for uninstrumented gaps and long DB/downstream spans.
3. Check in-flight requests, request queues, thread states, event-loop lag, and lock contention.
4. Check DB/HTTP pool pending counts and acquisition duration.
5. Check dependency latency/errors, DB lock waits, DNS/connect/TLS timings, retransmits, and disk latency.
6. Check per-container CPU throttling, per-core/run-queue data, and short-window metrics.
7. Take three thread dumps several seconds apart; identify repeated stacks and common lock owners.
8. Verify whether cancellation reaches A and its dependencies after caller timeout.

### Commands, tools, metrics, and interpretation

```bash
jcmd <pid> Thread.print -l > <approved-secure-path>/threads-1.txt
sleep 10
jcmd <pid> Thread.print -l > <approved-secure-path>/threads-2.txt
```

- Many request threads waiting in `HikariPool.getConnection`: DB pool acquisition is the bottleneck, not CPU.
- Many threads in socket reads to the same host: inspect that dependency and client deadlines.
- Many `BLOCKED` threads naming one monitor owner: lock contention.
- High gateway upstream queue with normal service time: delay is before A's application handling.
- Rising TCP retransmits/RTT with normal spans: network or transfer path.

### Immediate mitigation

Isolate the slow dependency with a circuit breaker or bulkhead according to existing policy, stop excessive retries, shed optional work, fail over through a tested path, or drain a proven stuck instance after preserving diagnostics.

### Root-cause fixes

Add deadlines and cancellation, eliminate connection leaks, optimize dependency operations, reduce lock scope, move blocking work off event loops, use bounded pools/queues, make logging asynchronous only with safe loss/backpressure behavior, and correct network or quota configuration.

### Prevention and alerting

Alert on wait metrics: queue depth, pool pending/acquisition, event-loop lag, lock contention, dependency p99, network retransmits, and throttling. CPU/memory-only dashboards are insufficient.

### Common mistakes

- Declaring the service healthy because CPU and memory are green.
- Increasing workers when workers are waiting on a dependency.
- Taking one thread dump and treating every `WAITING` thread as broken.
- Ignoring gateway/client-side timing.
- Raising read timeouts, increasing retained work.

### Concise interview-ready answer

> Low CPU and normal memory make waiting more likely, not health certain. I would compare client, gateway, and service timing, then inspect traces for queue, pool, lock, database, downstream, DNS/TLS, and response-transfer time. I would check thread states, event-loop lag, DB and HTTP connection acquisition, dependency latency, DB locks, network retransmits, disk/logging waits, and container throttling. Repeated thread dumps help distinguish normal idle waits from persistent blocking. I would fix the wait source rather than add threads or timeouts.

---

## 5. Service A is slow only during peak traffic. What could be happening?

### What the symptom means and does not prove

Demand-dependent latency strongly suggests a capacity, queueing, contention, or autoscaling problem. It does not prove that adding Service A instances is safe or sufficient; the saturated resource may be a database, connection pool, rate-limited dependency, NAT gateway, or lock that scaling callers will worsen.

### Where it can happen

At gateway limits, load balancing, CPU quota, worker/event-loop queues, pools, locks, database/cache/broker, downstream quotas, network/NAT, autoscaler, or a hot partition/tenant.

### Detailed causes and mechanisms

- Arrival rate approaches service capacity; queue wait grows nonlinearly.
- Burst traffic exceeds queue or token-bucket capacity even if one-minute average looks safe.
- Worker threads or event-loop tasks are all busy.
- DB/HTTP connections are all leased; callers queue.
- CPU throttling enforces a container limit during bursts.
- Autoscaling reacts late, scales on the wrong signal, or waits for startup/readiness/cache warming.
- A database, cache shard, partition, or downstream quota saturates.
- Lock contention increases with concurrency.
- Retry storms and synchronized clients multiply peak load.
- Load balancing creates a hot pod or zone.
- Large peak payloads change service demand per request.

### Ordered investigation

1. Plot offered RPS, completed RPS, concurrency, p50/p95/p99, errors, and timeouts on one timeline.
2. Use short windows to find bursts hidden by averages.
3. Plot queue depth/wait and utilization for workers, CPU quota, DB/HTTP pools, DB, cache, broker, and downstream quotas.
4. Determine the first resource whose queue or rejection rises before latency.
5. Compare pods/nodes/zones and check load distribution.
6. Inspect autoscaler desired/current replicas, trigger metric, stabilization, scheduling, image pull, startup, readiness, and warm-up time.
7. Count attempts per logical request to detect retry amplification.
8. Load-test the confirmed path with open-loop arrivals and production-shaped data.

### Commands, tools, metrics, and interpretation

```bash
kubectl -n <namespace> get hpa <service-hpa> -o yaml
kubectl -n <namespace> describe hpa <service-hpa>
kubectl -n <namespace> get pods -l app=<service> -o wide
```

- RPS flat, latency and in-flight rising, completions falling: work is accumulating.
- CPU below 100 percent but throttled seconds rising: cgroup quota is constraining bursts.
- Hikari active=max, pending rising: DB connection pool saturation; inspect query duration and DB capacity before resizing.
- New replicas appear after the peak: autoscaling lag.
- Attempts/inbound request ratio rising: retries are amplifying demand.

### Immediate mitigation

Apply intentional rate/concurrency limits, shed optional traffic, suppress nonessential fan-out, reduce retry amplification, pre-scale using an approved runbook, or protect a dependency with a bulkhead. Scale only after verifying downstream headroom.

### Root-cause fixes

Reduce service demand per request, optimize queries, remove contention, right-size bounded pools against downstream capacity, use backpressure, improve autoscaling signals and startup time, pre-scale predictable peaks, partition hot keys, cache safely, and establish capacity headroom.

### Prevention and alerting

Run capacity tests with bursts and open-loop load; alert on SLO burn, concurrency, queues, rejections, pool wait, throttling, autoscaler lag, downstream quota, and retry ratio. Review capacity before known events.

### Common mistakes

- Using only average RPS and CPU.
- Blindly increasing thread or connection counts.
- Scaling callers into a saturated database.
- Load testing with closed-loop clients that hide queueing through coordinated omission.
- Setting a very large queue that converts overload into long timeouts.

### Concise interview-ready answer

> Peak-only slowness usually means demand is crossing a finite capacity boundary. I would correlate RPS, concurrency, percentiles, errors, and completions with worker queues, CPU throttling, pool waits, DB/cache/downstream saturation, retries, load distribution, and autoscaler timing. I would identify the first queue that grows, protect the system with bounded load shedding or scaling that respects downstream capacity, then fix service demand, contention, pool sizing, and autoscaling lag.

---

## 6. The API is fast in development but slow in production. How would you investigate?

### What the symptom means and does not prove

The environments differ in one or more workload, data, topology, dependency, runtime, resource, security, or observability dimensions. It does not automatically mean production hardware is inadequate or the code is fine.

### Where it can happen

In data volume/distribution, query plans, network distance, gateway/service mesh, TLS/auth, resource limits, replica/node shape, JVM/runtime settings, dependency tiers, cache state, logging/tracing, environment configuration, and real concurrency.

### Detailed causes and mechanisms

- Development has tiny data and warm local dependencies; production scans or serializes much more.
- Production parameters trigger a different database plan or hot partition.
- Development is single-user; production queueing appears only under concurrency.
- Production has gateway, WAF, mesh, mTLS, proxy, and cross-zone hops.
- Production CPU limits cause throttling; development runs without cgroups.
- Production uses different pool sizes, timeouts, DNS, feature flags, or JVM options.
- Cold caches or frequent eviction increase production DB load.
- Production authentication, auditing, masking, logging, or tracing adds work.
- Production dependency service levels differ.
- JIT warm-up, class loading, or newly scaled instances affect early requests.
- Debug builds, agents, or misconfigured logging can alter either environment.

### Ordered investigation

1. Define equivalent requests, data shape, concurrency, cache state, and measurement boundary.
2. Compare configuration and topology using a sanitized, approved diff.
3. Compare traces stage by stage; identify the first production-only span or longer stage.
4. Compare data volume, distribution, query plan, statistics, indexes, and rows returned.
5. Compare CPU requests/limits/throttling, memory/JVM flags, pool sizes, instance count, and startup state.
6. Compare gateway/WAF/mesh/TLS/network hops and dependency endpoints.
7. Compare logging, tracing, security, feature flags, and retry/timeout policy.
8. Reproduce with production-like data and open-loop load in a safe performance environment.

### Commands, tools, metrics, and interpretation

- Use `kubectl get deployment <service> -o yaml` only through approved access; sanitize secrets before comparison.
- Compare Java flags with `jcmd <pid> VM.flags` and runtime properties through approved diagnostics.
- Compare DB plans, estimates versus actual rows, and buffer/I/O evidence.
- Compare trace waterfalls, not just totals. A production-only 300 ms WAF span and a 2-second query are separate issues.
- Compare container throttling rather than host CPU alone.

### Immediate mitigation

Correct a proven production misconfiguration, roll back a harmful flag/release, warm or pre-scale through a tested process, route around a failed dependency, or temporarily limit the expensive workload.

### Root-cause fixes

Create a production-like performance environment, use representative anonymized data distributions, version configuration, validate resource limits, test gateway/security layers, make plans stable where appropriate, and include cold/warm and concurrency cases in performance gates.

### Prevention and alerting

Automate configuration drift detection, canary performance comparisons, synthetic checks through the real production path, query-plan monitoring, capacity tests, and deployment markers on latency dashboards.

### Common mistakes

- Comparing one local request with production p99.
- Copying production data or secrets unsafely.
- Disabling security controls to make benchmarks look fast.
- Assuming equal code versions mean equal behavior.
- Tuning production based on a development-only profile.

### Concise interview-ready answer

> I would make the comparison equivalent: same operation, data shape, cache state, concurrency, and timing boundary. Then I would compare trace stages and sanitized configuration across data/query plans, gateway and network hops, dependencies, CPU limits and throttling, JVM and pool settings, security/observability features, retries, and warm-up. I would reproduce with production-shaped data and traffic safely, fix the proven environmental or workload difference, and add drift and canary performance checks.

---

## 7. An API suddenly starts timing out after a new deployment. What would you check?

### What the symptom means and does not prove

The timing creates a strong release hypothesis, but the timeout may come from application code, configuration, schema compatibility, dependencies, resource limits, readiness, routing, or load. It does not prove the deployment caused the incident, and "timeout" does not identify whether connect, pool acquisition, read, statement, or overall deadline expired.

### Where it can happen

In the caller or gateway deadline, new service version, mixed-version interaction, database migration/query plan, configuration/secret/certificate, readiness/startup, sidecar, route, resource manifest, or downstream contract.

### Detailed causes and mechanisms

- New code adds a slow query, N+1 pattern, fan-out, blocking call, lock, or large serialization.
- A config value changes URL, proxy, pool, timeout, retry count, feature flag, or cache behavior.
- A schema/index migration is missing, blocking, incompatible, or causes a plan regression.
- New versions and old versions disagree on payload or protocol.
- Readiness becomes true before caches, connections, or compilation are ready.
- CPU/memory requests or limits change and cause throttling or pressure.
- Connection reuse, DNS, TLS trust, or certificates differ in the new image.
- Logging/tracing volume or a Java agent causes synchronous overhead.
- Caller and server timeout budgets become misaligned.
- A coincident dependency incident begins during deployment.

### Ordered investigation

1. Identify the exact timeout layer, exception, configured value, and first timestamp.
2. Split metrics and traces by application version/pod revision.
3. Compare new and old versions under the same traffic and input.
4. Inspect rollout events, readiness, restart state, resource limits, sidecars, and routing weights.
5. Review code, configuration, dependency, image/base runtime, and manifest diffs.
6. Check migrations, indexes, DB locks, query plans, and mixed-version compatibility.
7. Count retries and confirm deadline propagation/cancellation.
8. If evidence links the new revision, pause or roll back through the approved process.
9. Verify old-version recovery and reproduce the regression safely before fixing forward.

### Commands, tools, metrics, and interpretation

```bash
kubectl -n <namespace> rollout status deployment/<service>
kubectl -n <namespace> rollout history deployment/<service>
kubectl -n <namespace> get pods -l app=<service> \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image,NODE:.spec.nodeName,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount'
kubectl -n <namespace> describe deployment <service>
```

- New-version-only timeouts strongly support release causality.
- Both versions fail with the same long downstream span: inspect the dependency or shared change.
- Timeouts cluster only during startup: readiness/warm-up issue.
- Five-second timeout with three attempts visible: retry policy may consume the deadline.
- New Hikari pending or server queue growth identifies a changed demand/resource relationship.

### Immediate mitigation

Pause rollout, shift traffic to the known-good revision, or roll back if the change is proven and rollback is safe for schema/data compatibility. Disable the implicated feature through a tested flag. Preserve slow traces, logs, and runtime evidence first when safe.

### Root-cause fixes

Fix the code/query/config/contract, make migrations backward compatible, align timeout budgets, propagate cancellation, correct readiness, restore resource settings, add bounded retry policy, and test mixed versions during rolling deployments.

### Prevention and alerting

Use canaries, automated latency/error rollback gates, version-tagged telemetry, deployment markers, backward-compatible expand/contract migrations, startup/readiness tests, and performance regression tests.

### Common mistakes

- Immediately raising the timeout.
- Rolling forward every replica before comparing versions.
- Rolling back code that cannot safely run against a changed schema.
- Looking only at application source and missing manifest/config/base-image changes.
- Assuming deployment timing is conclusive proof.

### Concise interview-ready answer

> I would first identify which timeout fired and split latency, errors, traces, and saturation by old versus new revision. I would compare code, config, manifests, resources, dependencies, migrations, readiness, and mixed-version behavior, while checking retries and deadline propagation. If the new revision is clearly responsible and rollback is data/schema-safe, I would pause or roll it back, then fix forward with a canary and version-tagged verification rather than increasing the timeout.

---

## 8. Response time increases gradually over several hours. What could cause this?

### What the symptom means and does not prove

A monotonic or sawtooth degradation suggests state accumulates, a resource leaks, data/backlog grows, or a periodic event partially resets the condition. It does not prove a Java heap leak. The accumulating resource may be connections, threads, file descriptors, native memory, cache entries, queues, locks, temporary files, DB sessions, or work in a dependency.

### Where it can happen

Inside the process, JVM/native runtime, connection pools, caches, queues, local disk, node/container, database, broker, downstream service, gateway connection tables, or autoscaling/load distribution.

### Detailed causes and mechanisms

- Connection leak slowly reduces available DB/HTTP connections; pool wait rises.
- Memory/object retention increases GC frequency and pause time before OOM.
- Unbounded cache/session/map increases lookup cost and memory pressure.
- Thread/task leak increases scheduling overhead or leaves more blocked work.
- Queue/backlog arrivals slightly exceed completions, so wait grows over time.
- File descriptor/socket leak makes connection establishment fail or wait.
- Long transactions or lock chains accumulate.
- Database tables/temp data grow, statistics age, or plan behavior changes.
- A hot instance receives sticky traffic and diverges from peers.
- Log buffers, disk usage, or synchronous I/O worsen.
- A dependency degrades with its own cache, leak, quota, or backlog.
- Scheduled jobs compete progressively or run at a predictable time.

### Ordered investigation

1. Plot latency with heap/RSS, GC, threads, file descriptors, open sockets, pool active/pending, queue depth, disk, and dependency metrics.
2. Determine whether the slope resets after restart, traffic drop, cache expiry, failover, or scheduled event. Do not restart merely to test this.
3. Compare instances by age. If oldest is worst, inspect retained/leased resources.
4. Check pool acquisition time and leak diagnostics, queue arrival/completion rates, and file descriptors.
5. Capture repeated class histograms, thread dumps, and native/runtime metrics at intervals.
6. Inspect DB long transactions, locks, session counts, temp space, table growth, and query plans.
7. Inspect cache size/evictions/hit rate and backlog age, not only item count.
8. Trace the first stage whose duration follows the slope.
9. Reproduce with a soak test and verify the resource reaches a steady state after the fix.

### Commands, tools, metrics, and interpretation

```bash
# Java class histogram; output can contain sensitive class names/data hints
jcmd <pid> GC.class_histogram > <approved-secure-path>/histogram-1.txt

# Linux process descriptors
ls /proc/<pid>/fd | wc -l

# Kubernetes restart and age comparison
kubectl -n <namespace> get pods -l app=<service> \
  -o custom-columns='NAME:.metadata.name,START:.status.startTime,RESTARTS:.status.containerStatuses[0].restartCount'
```

- Pool pending rises while active stays at max and idle stays zero: connection capacity is unavailable; distinguish long use from leak.
- Queue depth and oldest-item age rise while input slightly exceeds output: backlog instability.
- Oldest pods have worst latency and larger RSS/thread/FD counts: process-age accumulation.
- Heap after full GC trends upward: retained heap is plausible, but use a heap dump/dominator analysis to prove ownership.
- Heap stable while RSS rises: inspect direct buffers, thread stacks, native libraries, mapped files, allocator behavior, and page cache.

### Immediate mitigation

Reduce incoming work, disable the leaking/accumulating optional path, route new work away from a proven degraded instance, stop retries, clear a backlog only through a data-safe runbook, or recycle instances through an approved rolling process after preserving evidence. A restart is containment, not root cause.

### Root-cause fixes

Close resources deterministically, bound caches and queues, enforce task cancellation, fix long transactions, stabilize consumers above arrival rate, cap fan-out, correct file/socket handling, tune only after evidence, and add soak-test coverage.

### Prevention and alerting

Alert on trends and derivative signals: pool pending/acquisition, queue age, FD/thread count, heap-after-GC, RSS minus heap, GC CPU/pause, cache size, backlog, long transactions, and per-instance age divergence.

### Common mistakes

- Calling every rising memory graph a heap leak.
- Restarting on a schedule and considering the problem solved.
- Looking at queue count without arrival, completion, and oldest age.
- Taking a heap dump only after the process is too unhealthy to produce one safely.
- Increasing pool or heap size, merely delaying failure.

### Concise interview-ready answer

> Gradual degradation suggests an accumulating state or backlog. I would correlate latency over process age with pool waits, queues and oldest age, heap-after-GC, RSS/native memory, GC, threads, file descriptors, sockets, cache size, disk, long DB transactions, and downstream state. I would compare old and new instances and capture interval evidence such as histograms and thread dumps. After containing impact and preserving evidence, I would fix the leak or unstable queue and prove a steady state with a soak test.

---

# 5. Related interview questions

## 5.1 p50 is stable, but p99 is getting worse. What does that suggest?

Most requests remain healthy, while a minority encounter a slow path. Segment by instance, zone, tenant/data size, cache hit/miss, dependency, retry count, status, and deployment version. Common mechanisms are one bad pod, hot key/partition, occasional GC pause, lock convoy, connection creation, cache miss, large payload, dependency tail latency, or retry.

Compare p50/p95/p99 histograms and exemplar traces. Avoid the claim "only one percent is affected": a p99 regression can affect more or less depending on distribution and SLO threshold, and high-volume services can expose many users. Fix the separating dimension, and alert on tail-latency burn rather than average.

**Interview-ready answer:** I would treat stable p50 with bad p99 as a conditional slow path. I would use high-cardinality traces/logs, not metric labels, to split by pod, zone, data size, cache outcome, dependency, retries, and GC/lock events, then compare normal and tail traces and fix the dimension that predicts the tail.

## 5.2 The gateway reports 504, but the service later logs HTTP 200. What happened?

The gateway deadline expired before it received the upstream response. The service continued after the caller abandoned the request and eventually logged success. Possible causes include a slow queue/dependency, timeout mismatch, missing cancellation propagation, buffered access-log timing, or a response-delivery problem.

Identify the 504 generator, align timestamps with a request ID, and compare gateway timeout with service start/completion and downstream attempts. Determine whether the operation committed data after the timeout; client retries can duplicate non-idempotent work.

Fix the latency source, propagate deadlines/cancellation, make retried operations idempotent, and set nested timeouts so inner work fails early enough for the caller to respond. Do not simply make the gateway wait longer.

## 5.3 How do you prove connection-pool exhaustion rather than guess it?

Show that:

1. Active/leased connections reach the configured maximum.
2. Idle/available connections approach zero.
3. Pending borrowers and acquisition duration rise before request latency.
4. Thread dumps show callers in pool acquisition.
5. Connection hold time or leak evidence identifies why connections do not return.

Then distinguish legitimate long operations from leaks and from an unreachable dependency. A larger pool can overload the database or remote service. Fix query/call duration and resource lifecycle first, then size the pool from downstream capacity, expected concurrency, and measured hold time.

## 5.4 How should an end-to-end timeout budget be designed?

Start from the user-facing deadline and reserve time for network return and error handling. Each nested operation must have a smaller deadline than its caller and receive the remaining budget, not a fresh full timeout. Bound pool acquisition, connect, write, read, and database statement time separately where supported.

Retries must fit within the same overall deadline and be safe, capped, jittered, and budgeted. Cancellation should stop downstream work. Validate the budget against healthy p99 plus known variance; a timeout shorter than normal p99 creates false failures, while a much larger timeout retains overload.

## 5.5 Why can adding threads make a slow API worse?

If work is CPU-bound, more runnable threads increase context switching and cache contention. If work is blocked on a fixed database or downstream capacity, more threads create more waiting requests, memory use, sockets, and timeout/retry pressure. If locks serialize work, more contenders increase convoying.

Increase threads only after proving worker capacity itself is the bottleneck and confirming CPU and downstream headroom. Prefer bounded queues, backpressure, workload isolation, nonblocking I/O where justified, and removal of the real wait.

## 5.6 What makes a production load test trustworthy?

A useful test has production-like request mix, data distribution, payload sizes, cache state, dependency behavior, network path, security layers, resource limits, and concurrency. It uses an open-loop arrival model when measuring capacity, records client-side latency including queueing, and avoids coordinated omission. It tests warm and cold starts, bursts, steady state, failure behavior, and enough duration to reveal leaks.

It has safety limits, isolated test data, clear abort conditions, and telemetry across every tier. Passing one average-RPS test is not proof of production capacity.

---

# 6. Decision trees

## 6.1 Slow API decision tree

```text
API is reported slow
|
+-- Is the measurement boundary and exact request known?
|   +-- No -> obtain timestamp, route, status, percentile, trace/request ID
|   `-- Yes
|
+-- Is delay before the service receives the request?
|   +-- Yes -> client queue, DNS, connect, TLS, gateway/LB queue, request upload
|   `-- No
|
+-- Does a trace show one dominant child span?
|   +-- DB -> pool wait vs query execution vs lock/I/O/plan
|   +-- downstream -> pool/connect vs server latency vs retries
|   +-- cache/broker -> command latency, backlog, partition, quota
|   `-- No dominant span / missing time
|
+-- Is work queued or rejected?
|   +-- Server workers -> CPU-bound vs blocked workers
|   +-- DB/HTTP pool -> long hold vs leak vs capacity
|   +-- Event loop -> blocking operation or loop lag
|   `-- No
|
+-- Is local execution consuming time?
|   +-- CPU/throttling -> profile and inspect quota
|   +-- Locks -> repeated thread dumps/JFR contention
|   +-- GC -> correlate pauses and allocation
|   `-- Serialization/response -> bytes, compression, transfer
|
`-- Compare good/bad dimensions and investigate instrumentation gaps
```

## 6.2 Peak-only latency decision tree

```text
Latency rises with traffic
|
+-- Does offered RPS rise?
|   +-- No -> heavier requests, retries, scheduled work, dependency change
|   `-- Yes
|
+-- Which signal rises first?
|   +-- CPU run queue/throttling -> CPU capacity or service-demand problem
|   +-- Worker queue -> blocked or insufficient execution capacity
|   +-- DB pool pending -> long query/transaction, leak, or DB capacity
|   +-- HTTP pool pending -> slow/leaked downstream connections
|   +-- DB/cache/broker queue -> downstream bottleneck
|   +-- Gateway rejection/queue -> ingress capacity or policy
|   `-- No obvious queue -> inspect locks, hot partitions, network, metrics gaps
|
+-- Does adding a healthy instance reduce queueing without harming dependencies?
|   +-- Yes -> improve autoscaling/capacity and still reduce demand per request
|   `-- No -> bottleneck is shared, serialized, or downstream
|
`-- Add bounded backpressure/load shedding and fix the first constrained resource
```

## 6.3 Post-deployment timeout decision tree

```text
Timeouts began near deployment
|
+-- Which timeout fired? connect / pool / read / statement / overall
|
+-- New revision only?
|   +-- Yes -> code, config, resource, readiness, image, route, certificate
|   `-- No -> shared migration/config/dependency or coincidence
|
+-- Is rollback schema/data compatible?
|   +-- Yes and release is proven harmful -> pause/rollback safely
|   `-- No -> disable feature, shift compatible traffic, or fix forward
|
+-- Where does new trace time appear?
|   +-- Queue/pool -> demand or resource configuration
|   +-- DB -> migration, index, plan, lock, data
|   +-- downstream -> contract, endpoint, timeout, retry
|   +-- local code -> profile/lock/serialization
|   `-- Before app -> readiness, route, gateway, network
|
`-- Verify by revision, then add a regression/canary gate
```

---

# 7. Final cheat sheet

## First five questions to ask

1. Who measured the latency, and what exact interval does it include?
2. Which route, status, percentile, instance, zone, tenant class, and version are affected?
3. What changed at the first bad timestamp?
4. Where does one slow trace spend time compared with a normal trace?
5. Which queue, pool, dependency, or resource changed before latency?

## Fast interpretation table

| Evidence | Likely next step |
|---|---|
| p50 and p99 both rise | Broad/common regression or sustained saturation |
| p50 stable, p99 rises | Conditional slow path, bad instance, data skew, pause, lock, retry |
| In-flight rises, throughput flat | Work accumulates; find first queue |
| Worker busy=max, queue rises | Execution capacity saturated; determine CPU versus blocking |
| DB pool pending rises | Inspect connection hold time, query/transaction duration, leak, DB capacity |
| HTTP pool pending rises | Inspect downstream latency, connection leaks, per-host limit |
| CPU throttling rises | Container quota constrains work even if host CPU is free |
| Service time normal, gateway total high | Pre-service queue/network or response-transfer delay |
| Old pods worse than new pods | Accumulation/leak/backlog tied to process age |
| 5-second clusters | Look for a 5-second timeout, retry delay, lock wait, or pool wait |
| 504 then server 200 | Caller deadline expired; work continued; inspect cancellation/idempotency |

## Safe evidence checklist

```text
[ ] Exact timestamp, timezone, route, method, status
[ ] p50/p95/p99, RPS, errors, in-flight, queue depth
[ ] Request/trace ID for normal, slow, and failed request
[ ] Pod, node, zone, version, tenant/data-size class
[ ] Gateway versus service timing
[ ] Thread/event-loop queue and repeated thread dumps if needed
[ ] DB and HTTP pool active/idle/pending/acquisition
[ ] DB plan/locks/latency and downstream latency/retries
[ ] CPU utilization plus throttling; GC pause/allocation
[ ] Deployment/config/schema/infra events
[ ] Mitigation result at the original user-facing boundary
```

## Actions that require proof

- **Restart:** only as controlled containment after preserving evidence; it does not identify or fix a leak.
- **More heap:** only after proving live-set need and container headroom; it can lengthen pauses or trigger cgroup OOM.
- **More threads:** only when the worker pool itself is limiting and CPU/dependencies have capacity.
- **More connections:** only when the database/downstream can sustain them and no leak/slow operation exists.
- **Longer timeout:** only when the operation's valid SLO requires it and capacity/deadline/cancellation analysis supports it.
- **More replicas:** only when the bottleneck is horizontally scalable and shared dependencies have headroom.

## Interview answer structure

Use this compact sequence:

```text
1. Clarify symptom and measurement boundary.
2. Scope by endpoint, instance, tenant, region, version, and time.
3. Compare RED metrics and representative traces.
4. Find the first queue or long stage.
5. Use USE metrics and targeted runtime/dependency evidence.
6. Correlate and prove changes; do not guess from one graph.
7. Apply a bounded, reversible mitigation.
8. Fix the causal mechanism and verify SLO plus saturation recovery.
9. Add an alert, test, and runbook that would catch recurrence earlier.
```
