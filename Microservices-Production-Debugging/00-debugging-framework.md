# Microservices Production Debugging Framework

## Purpose

This is the starting point for the entire handbook.

It is not a list of technologies to memorize. It is a repeatable production investigation method:

```text
PROBLEM
  -> WHAT I CHECK FIRST
  -> WHY
  -> WHAT THE RESULT MEANS
  -> NEXT CHECK
  -> METRICS
  -> TRACE
  -> LOGS OR RUNTIME EVIDENCE
  -> ROOT CAUSE
  -> MITIGATE
  -> FIX
  -> VERIFY
  -> PREVENT
```

The same method works for:

- Service-to-service failures.
- High latency.
- CPU, memory, GC, and thread incidents.
- Database and connection-pool incidents.
- Kafka lag and message-processing incidents.
- Cache failures.
- Retry storms and cascading failures.
- Missing or misleading telemetry.

## Guide map

### Service-to-service

- [Service cannot connect](./01-service-to-service/01-service-cannot-connect.md)
- [Connection refused](./01-service-to-service/02-connection-refused.md)
- [Connection timeout](./01-service-to-service/03-connection-timeout.md)
- [Read timeout](./01-service-to-service/04-read-timeout.md)
- [Intermittent timeout](./01-service-to-service/05-intermittent-timeout.md)
- [502 Bad Gateway](./01-service-to-service/06-502-bad-gateway.md)
- [503 Service Unavailable](./01-service-to-service/07-503-service-unavailable.md)
- [504 Gateway Timeout](./01-service-to-service/08-504-gateway-timeout.md)
- [DNS failure](./01-service-to-service/09-dns-failure.md)
- [TLS/SSL failure](./01-service-to-service/10-tls-ssl-failure.md)
- [Gateway versus direct service](./01-service-to-service/11-gateway-vs-direct-service.md)

### Performance and JVM

- [API latency high](./02-performance/01-api-latency-high.md)
- [High CPU](./02-performance/02-high-cpu.md)
- [High memory](./02-performance/03-high-memory.md)
- [Out of memory](./02-performance/04-out-of-memory.md)
- [High GC](./02-performance/05-high-gc.md)
- [Thread-pool exhaustion](./02-performance/06-thread-pool-exhaustion.md)
- [One instance slow](./02-performance/07-one-instance-slow.md)

### Database

- [Slow database query](./03-database/01-slow-database-query.md)
- [Database connection-pool exhaustion](./03-database/02-database-connection-pool-exhaustion.md)
- [Database lock](./03-database/03-database-lock.md)
- [Deadlock](./03-database/04-deadlock.md)
- [N+1 queries](./03-database/05-n-plus-one.md)
- [Database CPU high](./03-database/06-db-cpu-high.md)
- [Database query rate drops](./03-database/07-db-query-rate-drops.md)

### Kafka

- [Consumer lag](./04-kafka/01-consumer-lag.md)
- [Consumer not consuming](./04-kafka/02-consumer-not-consuming.md)
- [Producer failure](./04-kafka/03-producer-failure.md)
- [Duplicate messages](./04-kafka/04-duplicate-messages.md)
- [Slow consumer](./04-kafka/05-slow-consumer.md)
- [Message-processing failure](./04-kafka/06-message-processing-failure.md)
- [Retry and replay](./04-kafka/07-retry-and-replay.md)

### Cache

- [Low cache hit ratio](./05-cache/01-low-cache-hit-ratio.md)
- [Redis slow](./05-cache/02-redis-slow.md)
- [Redis unavailable](./05-cache/03-redis-unavailable.md)
- [Cache stampede](./05-cache/04-cache-stampede.md)

### Distributed failures

- [Retry storm](./06-distributed-failures/01-retry-storm.md)
- [Cascading failure](./06-distributed-failures/02-cascading-failure.md)
- [Circuit breaker open](./06-distributed-failures/03-circuit-breaker-open.md)
- [One bad instance](./06-distributed-failures/04-one-bad-instance.md)
- [Dependency failure](./06-distributed-failures/05-dependency-failure.md)

### Observability

- [Distributed tracing](./07-observability/01-distributed-tracing.md)
- [Distributed logging](./07-observability/02-distributed-logging.md)
- [Distributed monitoring](./07-observability/03-distributed-monitoring.md)
- [Missing trace](./07-observability/04-missing-trace.md)
- [Missing logs](./07-observability/05-missing-logs.md)
- [Health check OK but business failing](./07-observability/06-health-check-ok-business-failing.md)

---

# 1. The first principle: describe evidence, not conclusions

Do not begin with:

> "Service B is down."

That sentence hides the exact failure.

Begin with:

```text
At 10:31:22 UTC, order-service version 4.8.1 on pod order-7f9d
called https://inventory-service:8443/inventory/reserve.

The call failed after 3.000 seconds with a TCP connect timeout.
DNS resolved 10.42.8.17 in 2 ms.
No inventory-service server span or access log exists.
Failures affect only order-service pods in zone b.
```

This statement already narrows the problem:

- DNS completed.
- TCP did not complete.
- HTTP was never reached.
- The failure has a zone-specific scope.
- Service B's SQL queries are not the first place to investigate.

## Symptom, trigger, root cause, and contributing factor

Use these terms precisely.

```text
Symptom:
  Users receive HTTP 504.

Trigger:
  A deployment activated a new database query.

Root cause:
  The query scans 18 million rows because the deployment omitted
  the required composite index.

Contributing factors:
  The gateway and application both retry.
  The canary did not receive the affected data shape.
  No alert exists for rows examined per query.

Mitigation:
  Roll back the route to the previous compatible query.

Permanent fix:
  Add and validate the index, propagate cancellation, remove duplicate
  retry layers, and test the production data distribution.
```

A restart, rollback, traffic shift, or scale-out may restore service. It is not automatically the root cause.

---

# 2. Build the request and dependency map

Before opening random dashboards, draw the real path.

```text
Client
  |
  v
API Gateway / Ingress
  |
  v
Order Service
  |
  +----> Inventory Service ----> MySQL
  |                 |
  |                 +---------> Redis
  |
  +----> Payment Service -----> External Provider
  |
  +----> Kafka producer ------> Topic
```

For one failing request, identify:

```text
source service and instance
destination hostname, IP, and port
protocol: HTTP/HTTPS/Kafka/MySQL/Redis
gateway/load balancer/service-mesh hops
authentication identity
timeout and retry policy at every hop
destination instance/version/zone
database/cache/broker/downstream calls
business operation and expected invariant
```

## The synchronous request progression

```text
configuration
  -> DNS
  -> destination IP
  -> route/firewall/network policy
  -> TCP connection
  -> TLS and identity
  -> HTTP request
  -> gateway route and target selection
  -> server queue and worker
  -> application code
  -> connection pool
  -> database/cache/downstream
  -> response serialization and transfer
```

Each error occurs at a different point:

| Symptom | Last stage that probably worked | First investigation area |
|---|---|---|
| `UnknownHostException` / NXDOMAIN | Configuration produced a hostname | DNS, name, namespace, resolver |
| Connection refused | DNS and route may work; destination actively rejected TCP | Listener, bind address, host, port, target |
| Connection timeout | DNS may work; TCP handshake did not finish | Routing, drops, firewall, policy, return path |
| TLS error | TCP generally completed | SAN/SNI, CA trust, certificate, mTLS, protocol |
| HTTP 401/403/404/500 | HTTP responder was reached | Identity, authorization, route, application |
| Read timeout | Connection completed; response was late | Queue, server, pools, DB, downstream, transfer |
| 502 | Gateway could not obtain/accept a valid upstream response | Gateway-to-upstream DNS/TCP/TLS/protocol/reset |
| 503 | Responder considers service unavailable | Generator, healthy targets, readiness, overload |
| 504 | Gateway's upstream deadline expired | Backend queue/processing/dependency/timeout budget |

These are starting points. Identify the component that generated the status and use its documented semantics.

---

# 3. What I check first in every incident

## Check 1: user and business impact

### What

Measure:

- Which user flow is affected?
- How many requests or messages are affected?
- Is data correctness at risk?
- Are writes completing after callers time out?
- Is the issue active now?

### Why

Technical health is not the goal. The goal is a correct business outcome.

Examples:

```text
checkout requests:        2,000/min
successful checkouts:     1,420/min
business success:         71%
duplicate payment count:  12
orders waiting > 10 min:  3,800
```

### Meaning

- High technical 200 rate with low business success means the measured endpoint may not represent the full business operation.
- Timeouts on writes create an unknown outcome; retries may duplicate work.
- A large Kafka backlog may preserve data but violate processing-time SLOs.

### Next

Protect data and users first. Pause an unsafe rollout, stop harmful retries, drain a proven bad instance, or apply an existing rate limit through approved controls.

## Check 2: exact error and generator

Capture:

```text
exception class and nested cause
HTTP status and response headers
gateway/vendor response flag
timeout phase
timestamp in UTC
request/trace/message/business ID
source and destination instance
target IP and port
attempt number
elapsed time and remaining deadline
```

Why:

```text
Connect timeout != read timeout
HTTP 503 from gateway != HTTP 503 from Service B
pool acquisition timeout != database TCP timeout
Kafka send() called != broker acknowledged
```

## Check 3: scope

Split successes and failures by:

```text
route / operation
status / exception reason
source instance and version
destination instance and version
zone / region / node
tenant or safe data category
payload / response size bucket
new versus reused connection
retry attempt
Kafka partition
database query fingerprint
cache key class
```

Why:

- All instances failing suggests a shared dependency, common configuration, or fleet-wide capacity problem.
- One instance failing suggests drift, local state, node, zone, resource, or lifecycle.
- One endpoint failing suggests endpoint code, query, payload, identity, or dependency.
- Failure rate near `1 / backend_count` suggests one equally weighted bad target.

## Check 4: first-failure timeline and changes

Overlay:

```text
deployments
configuration and feature-flag changes
secret and certificate rotations
schema/index migrations
network policy and route changes
autoscaling and node events
traffic and payload changes
scheduled jobs and maintenance
dependency incidents
```

A change near the first failure is a hypothesis, not proof. Compare old/new or healthy/failing populations and prove the mechanism.

## Check 5: preserve evidence

Before restart or replacement, preserve what is safe and useful:

- Logs around the exact UTC window.
- Failed and successful trace IDs.
- Pod/instance/node/version identity.
- Effective configuration fingerprints.
- Thread dumps or pool state where justified.
- Query plan/fingerprint and lock evidence.
- Kafka group/partition/offset evidence.
- Cache server and client-pool metrics.

Do not expose tokens, private keys, passwords, customer payloads, heap contents, or unrestricted environment dumps.

---

# 4. Metrics: how I know where to look

Metrics answer:

> Is something wrong, when did it start, how severe is it, and where is it concentrated?

Metrics do not normally explain the exact code line or request by themselves.

## 4.1 Metric types

### Counter

A counter increases until process restart.

Examples:

```text
http_server_requests_total
http_client_errors_total
kafka_records_consumed_total
db_connection_timeouts_total
```

Use a rate:

```text
rate(counter[5m])
```

Do not compare raw counter values across restarted instances.

### Gauge

A gauge rises and falls.

Examples:

```text
queue depth
active threads
Hikari active/idle/pending
Kafka lag
healthy target count
heap used
```

### Histogram

A histogram groups observations into duration or size buckets.

Use it for:

```text
p50
p95
p99
request latency
pool acquisition
DB query duration
GC pause
message processing
```

Do not average pod p99 values. Aggregate histogram buckets or raw events at the required scope.

## 4.2 Percentiles

```text
p50: half of observations are at or below this value
p95: slowest 5% are above this value
p99: slowest 1% are above this value
max: slowest observed event
```

Example:

```text
999 requests = 100 ms
1 request    = 20,000 ms

p50          = 100 ms
p99          = 100 ms
p99.9/max    exposes the rare tail, depending on quantile method
```

A normal average can hide a damaging tail.

## 4.3 RED method

Use RED at every request boundary:

- **Rate:** how much traffic?
- **Errors:** how many failed and why?
- **Duration:** how long, including percentiles?

Apply RED separately to:

```text
gateway downstream
gateway upstream
Service A server
Service A HTTP client
Service B server
Service B database client
Service B Redis client
Kafka producer/consumer processing
```

## 4.4 USE method

Use USE for finite resources:

- **Utilization:** how busy is the resource?
- **Saturation:** is work waiting?
- **Errors:** is work rejected or failing?

Examples:

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | CPU busy % | run queue / throttling | throttled time |
| Worker pool | active/max | queue depth | rejected tasks |
| Hikari pool | active/max | pending/acquire time | acquire timeout |
| Disk | throughput/IOPS | queue/latency | I/O errors |
| Kafka consumer | processing capacity | lag/oldest age | handler/commit errors |

## 4.5 Golden signals

Golden signals are:

- Latency.
- Traffic.
- Errors.
- Saturation.

RED focuses on service requests. USE focuses on resources. Golden signals connect user symptoms to capacity and failure.

## 4.6 Relationship, not one red chart

Example:

```text
request rate:           flat at 200/s
API p99:                300 ms -> 5.1 s
CPU:                    29%
Hikari active:          40/40
Hikari idle:            0
Hikari pending:         186
pool acquisition p99:   4.9 s
DB query rate:          falling
```

Interpretation:

1. Traffic did not rise, so a simple demand spike is unsupported.
2. Low CPU does not mean healthy; threads may be waiting.
3. All DB connections are checked out.
4. Requests wait 4.9 seconds before obtaining a connection.
5. DB query rate falls because fewer requests reach query execution.
6. This is not evidence that the database itself became slow.
7. Next inspect connection hold time, transaction duration, checkout/return counters, thread stacks, and slow traces.

## 4.7 Little's Law

At steady state:

```text
in_flight ~= throughput x average time
```

Example:

```text
200 requests/s x 0.2 s = about 40 in flight
200 requests/s x 5.0 s = about 1,000 in flight
```

Flat request rate plus slower service time creates more concurrency. That fills queues, threads, and pools and can cause a cascading failure.

---

# 5. Master production decision tree

```text
Production issue reported
        |
        v
What is the user-visible symptom?
        |
        +----------------------+----------------------+----------------------+
        |                      |                      |
        v                      v                      v
Errors/unavailable?       High latency?        Async/data issue?
        |                      |                      |
        v                      v                      v
Exact error/status        Which percentile?     Kafka lag/loss/duplicate?
and generator?            Which route/scope?     Cache stale?
        |                      |                      |
        v                      v                      v
DNS/TCP/TLS/HTTP?         Request rate changed? Producer or consumer?
        |                      |                      |
  +-----+-----+          +-----+------+         +----+-----+
  |     |     |          |            |         |          |
 DNS   TCP   HTTP       yes           no       rate       correctness
  |     |     |          |            |         |          |
  v     v     v          v            v         v          v
resolver, listener,   capacity,    waiting,    lag,       offsets,
record,  route,       autoscale,   pool, DB,   skew,      idempotency,
TTL      policy       rate limit   lock, C     rebalance  replay
```

## 5.1 Error/unavailable branch

```text
Exact failure
  |
  +-- DNS error
  |     -> exact name, resolver, rcode, answer, TTL, source environment
  |
  +-- Connection refused
  |     -> target IP/port, listener, bind address, Service targetPort,
  |        stale backend, active firewall reject
  |
  +-- Connect timeout
  |     -> forward/return route, drop policy, security group, NetworkPolicy,
  |        NAT/conntrack, packet evidence
  |
  +-- TLS failure
  |     -> SNI/SAN, CA chain, expiry, mTLS identity, TLS version
  |
  +-- 401/403
  |     -> token/identity/issuer/audience/scope/role/policy/clock
  |
  +-- 502
  |     -> generator and upstream subreason: DNS/connect/TLS/reset/protocol
  |
  +-- 503
  |     -> generator, healthy-target count, readiness, overload/breaker
  |
  +-- 504/read timeout
        -> timeout owner, server receipt, queue, pool, DB/downstream waterfall
```

## 5.2 Latency branch

```text
Latency rose
  |
  +-- Only one endpoint
  |     -> endpoint trace, query count, payload/data, auth, external call
  |
  +-- All endpoints
  |     -> shared pool, runtime, node, DB, gateway, network, deployment
  |
  +-- Only p99
  |     -> one instance/partition/data shape, GC pause, lock, cache miss
  |
  +-- All percentiles
  |     -> shared dependency, common config, broad saturation
  |
  +-- Peak traffic only
  |     -> concurrency, queues, threads, pools, DB/downstream capacity
  |
  +-- Gradually worsens
        -> leak, growing cache/queue, connection/resource retention,
           fragmentation, data growth, periodic maintenance
```

## 5.3 Resource branch

```text
Resource alert
  |
  +-- High CPU
  |     -> traffic? throughput? throttling? process or node?
  |        hot thread/profile, GC, serialization, crypto, query loop
  |
  +-- High memory/RSS
  |     -> heap after GC? RSS only? direct buffers? threads? metaspace?
  |
  +-- High GC
  |     -> allocation rate, live set, after-GC baseline, pause type,
  |        heap/container headroom
  |
  +-- Thread pool full
  |     -> active/max, queue, rejection, thread states, blocked dependency
  |
  +-- DB pool full
        -> active/idle/pending, acquisition, hold time, leaks, slow query/lock
```

## 5.4 Data correctness branch

```text
Wrong/duplicate/missing business result
  |
  +-- Caller timed out on write
  |     -> outcome unknown; query by stable operation/idempotency ID
  |
  +-- Kafka duplicate
  |     -> side effect succeeded before offset commit; durable dedupe
  |
  +-- Cache stale
  |     -> DB commit, invalidation order/event, key/version/TTL/replica lag
  |
  +-- Saga stuck
        -> durable state, messages, attempts, external truth, compensation,
           reconciliation/manual review
```

---

# 6. Metrics to trace to logs to code

Use the signals in this order:

```text
Metrics
  -> identify time, service, route, instance, dependency, and severity

Trace
  -> identify the exact request and slow/failing span

Logs
  -> explain the event around that span

Runtime/query/config/code evidence
  -> prove the mechanism
```

## Example

Metrics:

```text
order API p99:             300 ms -> 5.2 s
inventory p99:             280 ms -> 5.1 s
inventory DB child p99:     40 ms
Hikari acquire p99:          3 ms -> 4.9 s
Hikari pending:              0 -> 186
```

Trace:

```text
traceId=abc123

Gateway                         5.04 s
  |
  +-- Order server              5.02 s
        |
        +-- Inventory client    5.01 s
              |
              +-- Inventory     5.00 s
                    |
                    +-- DB pool acquire 4.90 s
                    +-- SQL query       0.04 s
```

The SQL query is not the main delay. Pool acquisition is.

Logs:

```text
2026-09-13T10:31:22.417Z level=ERROR
service=inventory-service
instance=inventory-7f9d
trace_id=abc123
span_id=def456
route=/inventory/reserve
event=db_pool_acquire_timeout
pool_active=40
pool_idle=0
pool_pending=186
acquire_ms=4902
```

Next evidence:

```text
connection checkout count
connection return count
hold-time histogram
transaction duration
thread stacks
error/cancellation code paths
deployment diff
```

Mechanism:

```text
new validation-error path
  -> obtains connection
  -> returns before close/finally
  -> checked-out connections accumulate
  -> pool reaches 40/40
  -> new requests wait
  -> DB query rate falls
  -> API p99 reaches timeout
  -> retries increase pending work
```

---

# 7. Distributed trace investigation

## 7.1 Terms

- **Trace ID:** identifies the distributed operation.
- **Span ID:** identifies one operation inside the trace.
- **Parent span:** operation that invoked or contains another operation.
- **Child span:** invoked database, HTTP, Kafka, or code operation.
- **Span status:** success/error/unset according to instrumentation.
- **Attributes:** route, method, target, instance, status, query fingerprint, retry attempt.
- **Span event:** exception or important timed event.
- **Link:** relationship that is not a strict parent-child call, useful for async/fan-out.

## 7.2 How I navigate a trace

1. Start at the gateway or entry server span.
2. Compare total duration with the user's duration.
3. Expand the critical path.
4. Find the longest child and error status.
5. Compare with a successful trace from the same scope.
6. Check target instance, zone, version, retry, and connection reuse.
7. Use the span to select logs or runtime evidence.

## 7.3 Missing child span

A missing span can mean:

- The call was never attempted.
- DNS/TCP/TLS failed before server instrumentation.
- Trace context was not propagated.
- The destination is not instrumented.
- Sampling removed the span.
- Collector/exporter dropped telemetry.
- The process crashed before export.

Do not conclude "Service B was never called" from a missing span alone. Corroborate with client metrics, gateway logs, B access logs, and telemetry-pipeline health.

## 7.4 Retry spans

Each attempt should be distinguishable:

```text
Inventory client attempt=1 target=B3 timeout 3.0 s
Inventory client attempt=2 target=B1 success 0.1 s
```

A top-level success can hide a failed first attempt and doubled load.

---

# 8. Distributed logs

Use structured logs:

```text
timestamp
level
service
instance
version
zone
trace_id
span_id
request_id or message_id
route/operation
target/dependency
attempt
duration
error class/reason
```

Example:

```text
2026-09-13T10:31:22.417Z level=ERROR
service=order-service
instance=order-7f9d
version=4.8.1
trace_id=abc123
span_id=xyz456
request_id=ord-901
downstream=inventory-service
endpoint=/inventory/reserve
attempt=1
error=SocketTimeoutException
phase=read
timeout_ms=3000
elapsed_ms=3001
```

## Why this log is not root-cause proof

It proves:

- Order service observed a read timeout.
- It waited for the configured interval.
- It targeted Inventory.

It does not prove:

- Inventory never received the request.
- Inventory is the root cause.
- The database was slow.
- Increasing the timeout is correct.

Next:

Use the trace and Inventory access log to determine whether the request arrived and where time was spent.

## MDC in Spring

MDC can attach trace/request fields to logs on the current thread.

Be careful:

- Clear MDC when work completes.
- Propagate context across executors/reactive boundaries.
- Do not put tokens, full user identity, or sensitive payloads in MDC.
- For Kafka, propagate approved trace/message context in headers.

---

# 9. Production-safe command framework

Commands answer one bounded question. They do not prove the entire root cause.

## DNS

Windows:

```powershell
Resolve-DnsName service-b
nslookup service-b
```

Linux:

```bash
getent hosts service-b
dig service-b
nslookup service-b
```

Interpret:

| Result | Meaning | Next |
|---|---|---|
| Expected IPs quickly | Resolver returned an answer | Test every IP and port |
| NXDOMAIN | Name reported nonexistent | Name, namespace, zone, registration |
| SERVFAIL | Resolver could not complete query | Resolver/upstream/DNSSEC/health |
| Timeout | Resolver unreachable/blocked/overloaded | Resolver route, UDP/TCP 53, policy |
| Wrong/stale IP | Resolution works but data is wrong | TTL/cache/record/discovery lifecycle |

`getent` follows the host's normal name-service configuration. `dig` directly inspects DNS. They can differ because hosts files or other name services may be involved.

## TCP

Windows:

```powershell
Test-NetConnection service-b -Port 8080 -InformationLevel Detailed
```

Linux:

```bash
nc -vz -w 5 service-b 8080
```

Interpret:

- Success: this test completed TCP from this source.
- Immediate refusal: destination/rejecting device actively rejected the port.
- Timeout: handshake did not complete; investigate route/drop/return path/capacity.

It does not prove TLS, authentication, HTTP route, or business functionality.

## HTTP

Windows:

```powershell
curl.exe -v --connect-timeout 3 --max-time 5 http://service-b:8080/actuator/health
```

Linux:

```bash
curl -v --connect-timeout 3 --max-time 5 http://service-b:8080/actuator/health
```

Explain the boundary:

- `Connected to` proves TCP for this request.
- TLS certificate output proves negotiation details.
- HTTP status proves an HTTP responder answered.
- `/actuator/health` proves only its configured health contract.

## Listener

Linux:

```bash
ss -lntp
```

Windows:

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen
netstat -ano | findstr :8080
```

Check:

```text
port
owning process
127.0.0.1 versus 0.0.0.0 / intended interface
IPv4/IPv6
```

## Ping

```text
ping tests ICMP reachability
API normally uses TCP + TLS + HTTP

ping success != port success != API success
```

## Kubernetes

```bash
kubectl get pods -n <namespace> -l app=<service> -o wide
kubectl describe pod -n <namespace> <pod>
kubectl logs -n <namespace> <pod> --since=30m
kubectl get service -n <namespace> <service> -o yaml
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service> -o wide
kubectl get networkpolicy -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

These are read-only inspections. A debug pod may have different labels, policies, sidecar, service account, or node, so it is not automatically equivalent to the failing application.

---

# 10. Java and Spring Boot evidence ladder

Use the least intrusive evidence that can answer the question.

## Actuator

Common endpoints when securely enabled:

```text
/actuator/health
/actuator/metrics
/actuator/prometheus
/actuator/threaddump
/actuator/heapdump
```

### `/actuator/health`

Useful for:

- Liveness/readiness components.
- Dependency health as configured.

Does not automatically prove a business endpoint works.

### `/actuator/metrics` and `/actuator/prometheus`

Useful for:

- HTTP server/client timers.
- Hikari pool metrics.
- JVM CPU/memory/GC/threads.
- Kafka client metrics.
- Custom business metrics.

Metric names vary with Spring Boot/Micrometer versions and tags.

### `/actuator/threaddump`

Useful for:

- Repeated BLOCKED stacks.
- Threads waiting on DB/HTTP pools.
- Deadlock clues.
- Executor starvation.

Collect several snapshots when possible. One dump is a moment, not a trend.

### `/actuator/heapdump`

High impact and sensitive.

Only use with authorization:

- It can pause or pressure the process.
- It can be very large.
- It can contain credentials and customer data.

Prefer memory/GC metrics and class histograms first when they answer the question.

## JVM commands

Examples:

```bash
jcmd <pid> Thread.print
jcmd <pid> GC.class_histogram
jcmd <pid> GC.class_histogram -all
jcmd <pid> VM.native_memory summary
jcmd <pid> JFR.start name=incident settings=profile duration=60s filename=<secure-path>
```

Caveats:

- `GC.class_histogram` normally requests a stop-the-world Full GC before
  producing the histogram. `GC.class_histogram -all` includes all objects
  without first forcing that collection. Confirm the exact behavior for the
  deployed JDK and use the lower-impact option when it answers the question.
- Native Memory Tracking must be enabled at JVM startup.
- JFR/profiling overhead depends on settings and workload.
- Store diagnostic artifacts securely.
- Do not repeatedly capture high-impact artifacts during an overload.

## Client libraries

When `RestTemplate`, `WebClient`, or OpenFeign calls fail, separate:

```text
pool acquisition
DNS
TCP connect
TLS
request write
time to first byte/read
response body
overall deadline
retry attempt
```

A single "timeout" label is not enough.

## HikariCP

Key concepts:

```text
active: checked out
idle: reusable
pending: waiting to acquire
acquisition duration: time waiting for pool
usage/hold duration: time connection remains checked out
```

Do not increase pool size until:

- Database capacity is known.
- Leak/slow-holder evidence is checked.
- Total fleet pool capacity is calculated.

## Hibernate and JPA

Investigate:

- Normalized SQL fingerprints.
- Query count per request.
- N+1 behavior.
- Lazy-load timing.
- Rows returned versus used.
- Transaction boundaries.
- Entity graph/fetch join/batch strategy.
- Pagination and result size.

Avoid logging sensitive SQL literals.

---

# 11. Root cause proof

A root cause statement needs four parts:

1. The change or condition.
2. The mechanism.
3. Evidence at each arrow.
4. A fix that reverses or removes the mechanism.

Example:

```text
Deployment 7.7 added an early validation return
  -> connection checkout counter continued
  -> connection return counter stopped on validation errors
  -> active reached 40/40 and idle reached 0
  -> pending reached 186
  -> acquisition p99 reached 4.9 s
  -> fewer requests reached SQL, so DB query rate fell
  -> API p99 reached 5.1 s
  -> retries increased pending work
```

Evidence:

- Change begins with version 7.7.
- Failure path trace ends after checkout without return.
- Thread stacks wait in Hikari acquisition.
- DB queries themselves remain fast.
- Rollback makes checkout/return counts converge and pending drain.

---

# 12. Mitigation versus permanent fix

## Mitigation

Restores or protects service now:

- Pause or roll back a bad deployment.
- Drain one bad instance.
- Stop unsafe retries.
- Apply existing rate/concurrency limits.
- Disable a noncritical expensive feature.
- Pause an offending batch.
- Route to healthy capacity when consistency permits.

## Permanent fix

Removes the mechanism:

- Correct listener/route/DNS/TLS configuration.
- Close leaked connection in every exit path.
- Add the justified index and test query plans.
- Make message effects idempotent.
- Bound retry and add jitter.
- Correct readiness/draining lifecycle.
- Add backpressure and dependency-aware capacity.

Do not call "restart" the permanent fix unless the root cause is explicitly a one-time transient state and recurrence controls exist.

---

# 13. Verification

Verify the same path that failed.

## Technical verification

```text
error/timeout rate
p50/p95/p99/max
source and destination instances
queue depth
pool active/idle/pending/acquire
CPU/throttling
heap/RSS/GC
DB query/lock/connection metrics
Kafka lag and oldest age
Redis latency/hit ratio
retry attempts/request
circuit-breaker state
```

## Business verification

```text
order/payment/reservation success
no duplicate writes
no orphaned or late operations
backlog drains within SLO
synthetic business transaction succeeds
```

Example:

```text
Before:
  reservation success       72%
  p99                       5.1 s
  Hikari pending            186
  acquire p99               4.9 s
  attempts/request          1.8

After:
  reservation success       99.95%
  p99                       310 ms
  Hikari pending            0
  acquire p99               4 ms
  attempts/request          1.01
```

Observe for a representative traffic period. A single successful request does not prove recovery.

---

# 14. Prevention

For each incident, close gaps in:

## Detection

- Business SLO and error-budget alerts.
- Route and dependency RED metrics.
- Per-instance outlier alerts.
- Pool pending/acquisition.
- Queue/rejection and throttling.
- Kafka lag growth and oldest age.
- Cache hit ratio/latency/eviction.
- Certificate expiry/JWKS refresh.

## Containment

- Deadlines and cancellation.
- Bounded retries with exponential backoff and jitter.
- Circuit breakers with minimum volume and measured thresholds.
- Bulkheads and concurrency limits.
- Rate limiting and load shedding.
- Graceful degradation that remains truthful.
- Readiness and graceful draining.

## Prevention before deployment

- Canary comparison by version.
- Production-shaped load tests.
- Query-count and execution-plan regression tests.
- Failure-path resource lifecycle tests.
- Network/DNS/TLS route tests.
- Schema and message compatibility.
- Secret/certificate rotation tests.
- Idempotency and crash-boundary tests.

## Operations

- Runbook with exact dashboards and safe commands.
- Ownership and escalation.
- Evidence preservation.
- Reconciliation procedures.
- Audited manual intervention for high-risk data.

---

# 15. Interview answer framework

A natural 60-90 second answer:

> First, I would make the symptom exact: the error or status, which component generated it, when it began, and which routes, instances, versions, zones, or data are affected. I would check business success and then follow RED metrics across the request path to find the last healthy boundary and the first failing or saturated one. I would compare failed and successful requests instead of relying on a fleet average. Then I would inspect a representative trace to find the failing or longest span, and use its trace ID, target, instance, query fingerprint, or message ID to select the exact logs and runtime evidence. I would form a causal hypothesis and look for evidence that can disprove it. I would mitigate impact without risking duplicate work or overloading dependencies, fix the measured mechanism, and verify the original business path, tail latency, errors, queues, pools, retries, and every instance. Finally, I would add the alert, test, runbook, or design control that prevents recurrence.

## Common interviewer traps

- Saying only "I will check logs."
- Listing every possible cause without ordering tests.
- Treating correlation as root cause.
- Treating timeout as proof the operation failed.
- Treating ping or health as proof the API works.
- Increasing timeout, threads, pools, memory, or replicas without capacity evidence.
- Ignoring retries and duplicate effects.
- Skipping verification and prevention.

## Quick memory flow

```text
Symptom
  -> Scope
  -> Timeline/change
  -> Business impact
  -> RED/USE metrics
  -> Last good boundary
  -> Failed/slow trace span
  -> Logs/runtime/query/config
  -> Causal mechanism
  -> Mitigate
  -> Fix
  -> Verify
  -> Prevent
```

---

# 16. Final cheat sheet

| Symptom | First check | Key metric | Trace clue | Likely areas | Next step |
|---|---|---|---|---|---|
| Connection refused | Exact target IP/port | refusals by target | client span fails immediately; no B span | listener, bind, wrong port, stale target | inspect socket and Service/targetPort |
| Connection timeout | Timeout phase and source scope | connect p99, retransmits | connect consumes full deadline | route, drop, firewall, NetworkPolicy, return path | compare source/zone and packet/flow evidence |
| Read timeout | Did B receive request? | first-byte/read p99 | B span or dependency is long | queue, pool, DB, downstream, response | follow longest child and compare success |
| DNS failure | Exact name and resolver | rcode, lookup latency | no connect/server span | name, namespace, resolver, record, TTL | query from failing runtime |
| TLS failure | Reason and TLS peer | handshake failures by reason/SNI | TCP child succeeds, TLS fails | SAN/SNI, CA, expiry, mTLS, version | inspect chain/trust/effective SNI |
| 502 | Identify generator/subreason | 502 by upstream target/reason | gateway upstream fails before valid response | DNS/TCP/TLS/reset/protocol | inspect gateway upstream logs and direct comparison |
| 503 | Identify generator | healthy targets, overflow/breaker | no upstream or service rejects | readiness, empty pool, overload, maintenance | inspect health reason/capacity |
| 504 | Identify gateway deadline | upstream response p99 | gateway ends while B/dependency continues | backend queue, DB/downstream, timeout budget | follow B critical path and cancellation |
| High latency | Scope and percentile | route p95/p99, in-flight | longest child span | queue, pool, DB, downstream, runtime | split waiting from execution |
| High CPU | Traffic/throughput/change | process CPU, throttling | long CPU/self time or sparse I/O waits | code, GC, serialization, crypto, load | profile/hot threads and compare version |
| High memory | Heap after GC vs RSS | heap/RSS/direct/threads | large allocations may appear | retention, cache, buffers, threads, metaspace | class histogram/heap/native evidence |
| OOM | Exact OOM/termination reason | heap/RSS/container limit | request may end abruptly | heap, direct, native thread, metaspace, cgroup | preserve dump/logs and classify memory area |
| High GC | After-GC baseline and allocation | pause/count/allocation/live set | request gaps align with pauses | high allocation, retention, small heap/headroom | inspect allocation/retained classes |
| Thread exhaustion | Pool and queue | active/max/queued/rejected | queue gap or blocked spans | slow DB/downstream, lock, nested executor | collect multiple thread dumps |
| DB slow | Separate phase | pool, lock, query, IO/CPU | DB child duration/attributes | query plan, locks, capacity, storage | inspect fingerprint/waits/plan safely |
| DB pool exhausted | Active/idle/pending | acquire p99, hold time | acquire span long; query absent | leak, long transaction/query, too much concurrency | compare checkout/return and holders |
| DB lock | Blocking chain | lock wait and blocker age | SQL child mostly lock wait | long transaction, hot row, access order | reconstruct blocker and transaction |
| Deadlock | Capture deadlock graph | deadlocks/s by operation | one transaction chosen as victim | cyclic lock order | compare all participants and standardize order |
| N+1 | Query count/request | calls per route, total DB time | repeated similar DB children | ORM lazy loading/loop | batch/fetch join/entity graph and test count |
| Kafka lag | Produce vs consume | lag slope and oldest age | handler/downstream spans slow | demand, slow handler, skew, rebalance | inspect per partition and processing stages |
| Kafka duplicate | Commit/side-effect timeline | dedupe hits, redelivery | same message ID repeated | crash after effect before commit, producer retry | make effect durably idempotent |
| Redis slow | Client/server/network split | command p99, pool pending | Redis child long | server CPU, hot key, network, pool | inspect command/key class and server |
| Retry storm | Attempts/original request | attempts/request, retry rate | repeated attempt spans | broad retry, no jitter, slow dependency | reduce retries/admission and protect dependency |
| Circuit breaker open | Triggering failures | state, rejected calls, failure rate | no downstream span while open | dependency failure or wrong thresholds/classification | inspect pre-open window and root failure |
| One bad instance | Correlate by target | per-target error/p99 | failures target one instance | version/config/node/zone/local state | drain safely, preserve and compare |
