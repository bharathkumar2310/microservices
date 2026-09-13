# Problem

API latency is high in a Java 17/Spring Boot production service.

The affected operation is `POST /orders` in `order-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `order-service` begins.

- Traffic: 2,000 requests/min
- Normal: p50=95ms, p95=280ms, p99=430ms, max=900ms
- Current: p50=180ms, p95=4.8s, p99=5.2s, max=8.9s
- Errors: business success=81%, HTTP 5xx=3%, client timeouts=16%
- Resource snapshot: CPU=29%, heap=56%, GC max=24ms
- Highlighted instance: `order-7f9d`
- Dependency path: inventory-service -> MySQL
- A release completed 14 minutes before the sustained deviation.
- All evidence is queried with the same UTC interval and labels.

These values create hypotheses. A high metric or temporal correlation alone is not root-cause proof.

# Architecture

```text
Client
  | HTTPS
  v
API Gateway / Load Balancer
  | W3C trace context
  v
order-service (order-7f9d)
  |
  +--> inventory-service -> MySQL
  |
  +--> Micrometer -> Prometheus/Grafana
  +--> OpenTelemetry -> Jaeger/Zipkin
  +--> structured logs -> central log store
```

Elapsed time can be CPU execution, GC stop time, lock wait, executor queue, connection-pool wait, downstream I/O, retry, or response transfer. The investigation assigns time rather than guessing.

# What I Check FIRST

1. **Impact and scope**
   - WHAT: Rate, errors, p50/p95/p99/max by route, status, version, pod, zone, tenant, and payload.
   - WHY: Fleet averages hide a bad slice.
   - EXPECT: The smallest population and first timestamp that reproduce the issue.
2. **Deployment and traffic overlay**
   - WHAT: Release/config markers, RPS, payload size, and retry rate.
   - WHY: CPU or memory totals must be normalized by useful work.
   - EXPECT: A controlled comparison, not a conclusion from timing alone.
3. **RED plus USE**
   - WHAT: Rate/errors/duration, then utilization/saturation/errors for resources, pools, and queues.
   - WHY: RED shows impact; USE finds the constrained resource.
   - EXPECT: Saturation or wait that moves with p99.
4. **Matched slow and healthy traces**
   - WHAT: Same endpoint/payload class, different outcome.
   - WHY: The waterfall identifies where time accumulated.
   - EXPECT: Long child, retry, or parent gap.
5. **Correlated logs and JVM evidence**
   - WHAT: traceId/spanId plus bounded JFR, repeated dumps, or memory evidence.
   - WHY: Population and runtime evidence convert a hypothesis into causality.
   - EXPECT: Repeated agreement; never one log line alone.

# Step-by-Step Investigation

### Step 1 - Scope the latency

- **What I check:** Group RED by route, status, version, pod, tenant, and payload class.
- **Why:** Aggregates can hide one route, version, pod, or CPU owner; normalization finds the smallest affected population before deep diagnostics.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** Only POST /orders on .2 is slow.
- **Bad:** Every route is slow.
- **Meaning:** A narrow slice suggests route code; a broad slice suggests shared infrastructure.
- **Next:** Compare matched slow and healthy traces.

### Step 2 - Include business and client outcomes

- **What I check:** Compare gateway 499/504, server status, business success, and client deadlines.
- **Why:** Server 5xx misses abandoned clients and business failures, so transport success alone understates impact.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** Tail latency aligns with client timeout growth.
- **Bad:** Server metrics look normal while clients are slow.
- **Meaning:** Time may be before the gateway, in transfer, or after client abandonment.
- **Next:** Open an end-to-end trace.

### Step 3 - Read the waterfall

- **What I check:** Find the longest child or uninstrumented gap.
- **Why:** A trace assigns elapsed time to queue, local code, retry, connection acquisition, SQL, or downstream work.
- **Example command/query/tool:** Trace search: `service.name=<service> AND http.route=<route> AND duration>2s`, then compare a matched healthy trace.
- **Expected:** Hikari acquisition owns 4.91s; SQL owns 72ms.
- **Bad:** The SQL span itself owns 4.8s.
- **Meaning:** Pool wait and slow SQL require different fixes.
- **Next:** Inspect pool and DB metrics.

### Step 4 - Inspect the connection pool

- **What I check:** Graph active/max, idle, pending, acquisition percentiles, checkout, and return.
- **Why:** Pool utilization without pending/acquisition time cannot distinguish healthy reuse from request starvation.
- **Example command/query/tool:** Query `hikaricp_connections_active`, `idle`, `pending`, `max`, acquisition histograms, DB lock views, and checkout/return counters.
- **Expected:** Active has headroom and pending is zero.
- **Bad:** 40/40 active, zero idle, 186 pending.
- **Meaning:** Requests queue before reaching MySQL.
- **Next:** Compare checkout and return counters.

### Step 5 - Prove the leak

- **What I check:** Overlay checkout-return delta, borrowed age, validation errors, and version.
- **Why:** Growth alone is correlation; ownership, return/disposal behavior, and a triggering path establish retention causality.
- **Example command/query/tool:** Query `hikaricp_connections_active`, `idle`, `pending`, `max`, acquisition histograms, DB lock views, and checkout/return counters.
- **Expected:** Counters converge after requests settle.
- **Bad:** Version .2 checks out 420/min but returns 392/min.
- **Meaning:** A growing 28/min gap shows retained connections.
- **Next:** Inspect the validation error path.

### Step 6 - Reject slow-DB alternatives

- **What I check:** Check query percentiles, locks, transactions, rows examined, DB CPU, and connections.
- **Why:** Full application pools can be caused by slow SQL or locks, so DB evidence must reject those alternatives.
- **Example command/query/tool:** Query `hikaricp_connections_active`, `idle`, `pending`, `max`, acquisition histograms, DB lock views, and checkout/return counters.
- **Expected:** SQL p99=72ms, locks=0, DB CPU=34%.
- **Bad:** Long query/transaction time matches borrow time.
- **Meaning:** Borrowed connections may be legitimately busy.
- **Next:** Follow the long SQL or lock owner.

### Step 7 - Inspect repeated thread dumps

- **What I check:** Group three dumps 10s apart by state and stack.
- **Why:** Repeated states and stacks distinguish CPU execution, monitor blocking, pool parking, and downstream I/O waiting.
- **Example command/query/tool:** Run `jcmd <pid> Thread.print -l` three times 10 seconds apart; pair with JFR or OS per-thread CPU.
- **Expected:** Few threads wait for Hikari.
- **Bad:** 146 threads repeatedly WAITING in ConcurrentBag.borrow.
- **Meaning:** Threads are victims of pool starvation; adding threads worsens it.
- **Next:** Confirm no second pool is saturated.

### Step 8 - Tie it to the rollout

- **What I check:** Compare equal traffic and invalid-request rates by version.
- **Why:** Comparing equal traffic across versions separates changed work per request from legitimate traffic-driven demand.
- **Example command/query/tool:** Compare `kubectl get pods -o wide`, ReplicaSet revision, image digest, config checksum, limits, node, and deployment events.
- **Expected:** Versions behave alike.
- **Bad:** Only .2 leaks under flat traffic after rollout.
- **Meaning:** This supports a code regression, not a traffic increase.
- **Next:** Diff connection ownership on early returns.

### Step 9 - Verify the corrected path

- **What I check:** Load valid and invalid orders for two prior failure windows.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** Borrowed baseline settles and counters converge.
- **Bad:** After-test active baseline rises each cycle.
- **Meaning:** The resource leak remains despite short-term recovery.
- **Next:** Block promotion and recapture ownership evidence.

### Cross-signal decision table

| Observation | Meaning | Next evidence |
|---|---|---|
| p50/p95/p99 all rise | Broad degradation | Shared path, capacity, dependency, or global change |
| p50 normal, p99/max rise | Tail subset or queue | Instance, tenant, payload, retry, dependency grouping |
| CPU low, latency high | Waiting is likely | Pools, queues, locks, I/O stacks, trace gaps |
| CPU high, throttling low | JVM receives CPU and does work | JFR CPU/allocation and hot stacks |
| CPU modest, throttling high | Cgroup denies bursts | Limit-normalized CPU and runnable queue |
| Heap peaks, after-GC flat | Temporary allocation | JFR allocation and pause impact |
| After-GC baseline rises | Retained live set grows | Histograms and dominators |
| RSS rises, heap flat | Native/direct/metaspace/stacks | NMT, buffers, classloaders, threads |
| active=max and queue grows | Arrival exceeds completion | Task time, dependencies, retries |
| one pod differs | Local state or skew | Matched probe, version/config/node comparison |

Little's Law is a consistency check: `concurrency = throughput * time in system`. Increased latency alone can fill a previously healthy pool; it does not prove more useful work.

# Metrics to Check

| Metric | What I inspect | If high | If low or changed |
|---|---|---|---|
| RED rate | request and completion rate by route/status/version/pod | high can mean traffic pressure | low can mean upstream rejection or saturation, not recovery |
| RED errors | HTTP, client timeout, rejection, business failure | high shows impact and boundary | low 5xx does not exclude abandonment or wrong results |
| RED duration | p50/p95/p99/max on the same population | high p50 is broad; high p99 is tail | a deploy step-change suggests changed work/config |
| USE CPU | utilization, run queue, throttling, errors | high plus throttle/queue means saturation | low with latency means waiting elsewhere |
| Memory | heap, after-GC, non-heap, RSS, direct, threads | a rising component selects diagnostics | flat heap does not exclude native pressure |
| GC | count, pause percentiles/max, CPU, allocation, after-GC | high pause/CPU can explain latency | low impact redirects investigation |
| Threads | repeated RUNNABLE/BLOCKED/WAITING stacks | common stacks show CPU/monitor/pool/I/O | state alone is not root cause |
| Executor | active/max, queue/capacity/age, rejected/completed | pinned plus queue growth means saturation | quiet executor redirects to another pool |
| Dependencies | latency/errors, client pools, DB locks, Kafka lag, Redis | long child or pool wait explains parent | healthy dependencies favor local work |
| Per-instance | RPS, percentiles, version, node, config hash | one outlier means skew/local state | fleet-wide movement means shared cause |
| Scenario: DB query rate | Current `falls while requests remain flat` | High suggests requests are blocked before query execution | A deploy-correlated change supports but does not prove causality |
| Scenario: checkout-return delta | Current `+28/min on .2` | High suggests resource ownership leak | A deploy-correlated change supports but does not prove causality |
| Scenario: pool pending | Current `186 and rising` | High suggests queue before DB | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `order-7f9d`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 18ms -> order 5.12s -> validation 35ms -> Hikari acquire 4.91s -> SQL 72ms
```

1. Select a slow trace from the same route, version, pod, and minute.
2. Verify parent/child timing; child spans must fit in the parent.
3. Compare client and downstream server spans. Long client plus short server suggests client queue/pool, network/proxy, or retries.
4. Compare DB acquisition with SQL span; SQL instrumentation often begins only after a connection is borrowed.
5. Count retry spans and determine whether they are sequential or overlapping.
6. Compare a healthy trace with matching payload rather than an unrelated route.
7. Use traceId for the request and spanId for the exact operation.

Interpretation:
- Long local span with short children: CPU, lock, queue, serialization, or uninstrumented local work.
- Long client and server spans: downstream actually handled slowly.
- Long client but short/missing server span: pool wait, connect/TLS/network, rejection, sampling, or broken propagation.
- A missing child does not prove no call occurred. Check sampling, async context propagation, instrumentation coverage, and logs.
- Trace evidence is validated with metrics, logs, or runtime capture.

# Distributed Logs

```text
2026-09-13T16:39:22.481Z level=ERROR service=order-service instance=order-7f9d
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="POST /orders" downstream="inventory-service -> MySQL" latencyMs=5012
error="SQLTransientConnectionException pool=HikariPool-1 active=40 idle=0 pending=186 timeoutMs=5000" appVersion=2026.09.13.2
```

Correlation:
1. Search exact UTC window and traceId, not only error text.
2. Confirm service, instance, version, endpoint, and outcome match the trace.
3. Follow spanId so retry attempts are not combined.
4. Compare the same event in a healthy request.
5. Aggregate event rate by version/pod and align with metrics.

A log line alone is not proof because it may be a downstream symptom, occur after the client deadline, be duplicated by retries, or omit prior waits. Missing logs may mean buffering, crash, sampling, or broken MDC. Causality needs timing, population correlation, and mechanism.

Structured logs must omit tokens, full bodies, personal data, and sensitive SQL values. Bound stack-trace volume so logging does not become another bottleneck.

# Commands / Tools

Use approved access on the correct Java 17 process. Replace `<pid>` only after identifying it. Keep Actuator internal and authenticated.

| Command | What it proves | What it does not prove |
|---|---|---|
| `jcmd <pid> Thread.print -l` | Java stacks, states, locks, synchronizers | not CPU proof; compare repeated dumps |
| `jcmd <pid> JFR.start name=incident settings=profile duration=60s filename=incident.jfr` | bounded CPU/allocation/lock/JVM events | only the capture window; protect the file |
| `jcmd <pid> GC.class_histogram` | class counts and shallow bytes | not full retained ownership |
| `jcmd <pid> GC.heap_dump filename=heap.hprof` | heap objects and references | can pause, fill disk, expose data, and miss native bytes |
| `jcmd <pid> VM.native_memory summary` | native reservation/commit categories when NMT enabled | no useful history if NMT was disabled |
| `curl -fsS --max-time 5 http://127.0.0.1:8080/actuator/health` | local HTTP and reported health | not business success or tail performance |
| `curl -fsS http://127.0.0.1:8080/actuator/prometheus` | authorized metric snapshot | not a trend; prefer monitoring backend |
| `Invoke-WebRequest -TimeoutSec 5 http://localhost:8080/actuator/health` | Windows HTTP response | not representative routing/load |
| `Get-Process -Id <pid> | Select CPU,WorkingSet64,PrivateMemorySize64,Threads` | Windows process totals | not JVM pool attribution |
| `top -H -p <pid>; pidstat -p <pid> 1` | Linux thread/process CPU | map native ID to Java nid; one sample can mislead |

Actuator safety:
- `/actuator/health` proves only that endpoint and configured contributors responded; not business health.
- `/actuator/metrics` and `/actuator/prometheus` provide measurements, while the backend provides trends.
- `/actuator/threaddump` reveals internals and needs strict access.
- `/actuator/heapdump` can pause, exhaust disk, and expose customer data; keep disabled or tightly controlled.
- `Test-NetConnection`/`nc` prove TCP reachability only; `curl` adds HTTP/TLS evidence.
- Ping proves ICMP reachability only, never application connectivity.

# Root Cause

Worked root cause: **release 2026.09.13.2 leaked a JDBC connection when validation returned early**.

```text
release 2026.09.13.2 leaked a JDBC connection when validation returned early
  -> changes work, waiting, or ownership per request
  -> Hikari active=40/40, idle=0, pending=186, acquire p99=4.9s; DB QPS falls from 1,900 to 1,120 while requests stay flat
  -> completion rate falls below arrival rate
  -> concurrency and queueing rise
  -> p95/p99 rise before or more than p50
  -> deadlines, retries, rejection, or termination add errors
  -> retry/load feedback amplifies the defect
```

Each arrow is supported:
- Version/time scope ties the trigger to the affected population.
- The waterfall assigns elapsed time to the predicted stage.
- USE metrics show the matching resource or queue.
- Logs/runtime evidence identify the code path or effective config.
- Removing the trigger reverses leading signals.
- The fixed build survives the reproduction that fails the old build.

# Fix

## Immediate mitigation
- disable retries, drain version .2, and roll back after preserving evidence.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- use Spring transaction ownership/try-with-resources on every path and test resource return on validation failures.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **checkout and return rates converge; pending=0; acquire p99<20ms; API p99<450ms**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | p50=180ms, p95=4.8s, p99=5.2s, max=8.9s | Normal p50/p95/p99 and max below deadline |
| Errors | business success=81%, HTTP 5xx=3%, client timeouts=16% | 5xx/timeouts/rejections<1% and business success restored |
| Resource | CPU=29%, heap=56%, GC max=24ms | Leading resource/queue at measured baseline |
| Per-instance | Affected pod/version present | Every live pod in agreed band |
| Trace | Slow stage dominates | Stage bounded and no unintended retries |

I compare identical route/payload/traffic windows, run success and failure/cancellation paths, and soak for at least two previous failure windows. A restart-only improvement is not accepted.

# Prevention

- Dashboard RED by route/version/pod and USE for CPU, memory, pools, queues, and dependencies.
- Alert on sustained p99 plus impact, not noisy max alone.
- Alert on queue age/rejection, throttle, after-GC slope, direct memory, and pool pending where relevant.
- Emit image digest, config hash, node, zone, and revision with controlled cardinality.
- Propagate trace/MDC through async executors and clear MDC between requests.
- Bound queues and use admission control, end-to-end deadlines, and safe backoff/jitter.
- Test dependency slowdown, cancellation, retry, recovery, and realistic payload distributions.
- Canary and compare normalized metrics before full rollout.
- Require rollout convergence and automated rollback criteria.
- Treat JFR, thread dumps, heap dumps, and Actuator output as sensitive.

# Interview Answer

### What I would say in an interview

For high API latency, I first lock the time window and scope RED metrics by route, version, and instance. Here normal `p50=95ms, p95=280ms, p99=430ms, max=900ms` became `p50=180ms, p95=4.8s, p99=5.2s, max=8.9s`. I compare USE metrics and a matched trace; the key waterfall is `gateway 18ms -> order 5.12s -> validation 35ms -> Hikari acquire 4.91s -> SQL 72ms`. I correlate its traceId with structured logs and capture bounded JVM evidence before restarting. Together they prove release 2026.09.13.2 leaked a JDBC connection when validation returned early. I mitigate by disabling retries, draining version .2, and rolling back after preserving evidence. The permanent correction uses Spring transaction ownership or try-with-resources on every path and tests connection return on validation failures. I verify equal load and failure paths against p50/p95/p99/max, errors, the leading resource signal, every instance, and a multi-window soak. I do not call a restart or resource increase a fix.

### Common interviewer traps

- Treating the highest metric as root cause without a mechanism.
- Using average latency and ignoring p99/max/client deadlines.
- Assuming high CPU means an infinite loop or low CPU means health.
- Reading thread state without repeated stacks.
- Treating health endpoint success as business success.
- Increasing resources, pools, queues, threads, heap, or timeout before evidence.
- Restarting first and destroying causality.
- Ignoring per-version and per-instance comparison.

### Quick memory flow

```text
Symptom -> window -> scope RED -> USE saturation
-> matched trace -> correlated logs -> JVM evidence
-> causal chain -> safe mitigation -> permanent fix
-> equal-load verification -> prevention
```

# Interview Follow-up Questions

## Q1. Why can DB QPS fall during pool exhaustion?

**Answer:** Fewer requests obtain a connection, so fewer queries reach the DB; lower QPS is not proof of recovery.

## Q2. Why not increase Hikari max?

**Answer:** It can hide a leak briefly and overload MySQL; prove ownership and DB capacity first.

## Q3. What proves a connection leak?

**Answer:** Sustained checkout-return divergence tied to one path/version, growing borrowed baseline, and normal SQL time.

## Q4. Why can client timeouts exceed server 5xx?

**Answer:** A client may abandon while the server later records 200 or another non-5xx.

## Q5. What does a missing DB span mean?

**Answer:** Pool wait may precede it, or sampling/context propagation may be missing; it does not prove no DB use.

## Q6. Would you restart first?

**Answer:** No. Preserve pool, trace, and stack evidence, then drain/restart only as mitigation.

## Q7. How do retries affect this incident?

**Answer:** They increase pool demand and concurrency, amplifying the original leak.
