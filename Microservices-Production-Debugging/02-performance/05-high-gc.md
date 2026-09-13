# Problem

Garbage collection is high in a Java 17/Spring Boot production service.

The affected operation is `GET /products/export` in `product-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `product-service` begins.

- Traffic: 70 exports/s
- Normal: allocation=420MiB/s, GC CPU=4%, p95=320ms
- Current: allocation=2.8GiB/s, young GC=38/min, GC CPU=29%, p95=2.7s, max pause=410ms
- Errors: 504=7%, application 5xx=0.8%
- Resource snapshot: heap oscillates 45-78%; after-GC baseline stable at 1.3GiB
- Highlighted instance: `product-5a62`
- Dependency path: MySQL
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
product-service (product-5a62)
  |
  +--> MySQL
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

### Step 1 - Confirm GC impact

- **What I check:** Overlay latency with pause time, GC CPU, count, allocation, and process CPU.
- **Why:** RED quantifies user impact while USE tests whether a specific resource is utilized, saturated, or failing.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** Short collections do not align with slow requests.
- **Bad:** GC CPU=29% and 410ms pauses align with tails.
- **Meaning:** GC contributes, but allocation code may initiate it.
- **Next:** Inspect live baseline and allocation.

### Step 2 - Read count and pause together

- **What I check:** Break down young/old count, total pause, p95/p99/max pause, and concurrent cycles.
- **Why:** GC symptoms require separating temporary allocation, retained live data, collector behavior, and work per request.
- **Example command/query/tool:** Parse Java 17 unified GC logs and graph young/old collection count, total pause, and pause p95/p99/max over the same request-latency window.
- **Expected:** Young pauses<30ms; old rare.
- **Bad:** 38/min with p99=330ms, max=410ms.
- **Meaning:** Frequent disruptive collection exists.
- **Next:** Check allocation and promotion.

### Step 3 - Separate churn from leak

- **What I check:** Graph old/live heap immediately after GC.
- **Why:** Growth alone is correlation; ownership, return/disposal behavior, and a triggering path establish retention causality.
- **Example command/query/tool:** Graph old-generation used immediately after each GC and compare its slope with allocation rate, promotion rate, and request rate.
- **Expected:** Baseline stays near 1.3GiB.
- **Bad:** Baseline climbs each cycle.
- **Meaning:** Rising live set implies retention; stable baseline implies temporary churn.
- **Next:** Use JFR allocation or heap ownership accordingly.

### Step 4 - Profile allocations

- **What I check:** Record JFR allocation samples and TLAB events.
- **Why:** GC symptoms require separating temporary allocation, retained live data, collector behavior, and work per request.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** Types follow baseline.
- **Bad:** byte[], char[], String and JSON buffers dominate export stacks.
- **Meaning:** Serialization creates excessive garbage.
- **Next:** Compare implementation versions.

### Step 5 - Normalize by request

- **What I check:** Compute allocated bytes per export by version and payload size.
- **Why:** GC symptoms require separating temporary allocation, retained live data, collector behavior, and work per request.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** About 6MiB/request on both.
- **Bad:** 4.6 uses 39MiB/request at flat traffic.
- **Meaning:** Per-request work regressed.
- **Next:** Inspect serialization passes.

### Step 6 - Read trace and payload

- **What I check:** Compare rows, response bytes, encoding, mapping, and DB spans.
- **Why:** A trace assigns elapsed time to queue, local code, retry, connection acquisition, SQL, or downstream work.
- **Example command/query/tool:** Trace search: `service.name=<service> AND http.route=<route> AND duration>2s`, then compare a matched healthy trace.
- **Expected:** Stages scale within budgets.
- **Bad:** SQL=145ms but encoding=2.18s for 14.7MiB.
- **Meaning:** Application encoding, not DB, owns time.
- **Next:** Find the second pass.

### Step 7 - Check collector evidence

- **What I check:** Record Java 17 collector, Xms/Xmx, container awareness, humongous allocations, evacuation failures, and GC logs.
- **Why:** GC symptoms require separating temporary allocation, retained live data, collector behavior, and work per request.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** Settings match tested baseline.
- **Bad:** To-space exhausted or repeated full GC appears.
- **Meaning:** Tuning/capacity may contribute.
- **Next:** Reproduce before changing flags.

### Step 8 - Mitigate with admission

- **What I check:** Rollback and reject excess exports with explicit retry guidance.
- **Why:** Mitigation must reduce admitted work while preserving evidence and in-flight requests, not merely move waiting into a larger queue.
- **Example command/query/tool:** Query executor active/max, queue size/capacity/age, rejected/completed rates, task duration, and retry spans.
- **Expected:** Allocation and queues fall.
- **Bad:** Clients retry immediately and load grows.
- **Meaning:** Admission policy is amplifying load.
- **Next:** Coordinate backoff and limits.

### Step 9 - Verify equal workloads

- **What I check:** A/B test 70/s and 18k rows for 45min.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** Allocation and pause targets pass.
- **Bad:** Baseline or bytes/request remains high.
- **Meaning:** The double pass or another churn path remains.
- **Next:** Diff JFR profiles and block release.

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
| Scenario: allocation | Current `2.8GiB/s` | High suggests temporary churn | A deploy-correlated change supports but does not prove causality |
| Scenario: GC count/pause | Current `38/min, p99=330ms` | High suggests frequency and stop-time impact | A deploy-correlated change supports but does not prove causality |
| Scenario: after-GC baseline | Current `stable 1.3GiB` | High suggests churn rather than retained leak | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `product-5a62`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 11ms -> product 2.62s -> SQL 145ms -> mapping 210ms -> JSON encoding 2.18s
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
2026-09-13T16:39:22.481Z level=ERROR service=product-service instance=product-5a62
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="GET /products/export" downstream="MySQL" latencyMs=5012
error="export_complete rows=18000 responseBytes=14700000 serializationPasses=2 latencyMs=2614" appVersion=2026.09.13.2
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

Worked root cause: **release 4.6 serialized each 18,000-row product response twice, creating temporary arrays and Strings**.

```text
release 4.6 serialized each 18,000-row product response twice, creating temporary arrays and Strings
  -> changes work, waiting, or ownership per request
  -> allocated bytes/request rises 6MiB to 39MiB on 4.6; JFR shows byte[], char[], String under ProductExportController
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
- roll back 4.6 and cap export concurrency at measured safe throughput.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- stream the response once, remove debug payload serialization, and enforce allocation budgets.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **allocation<500MiB/s and <7MiB/request, GC CPU<5%, pause p99<40ms, API p95<350ms**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | allocation=2.8GiB/s, young GC=38/min, GC CPU=29%, p95=2.7s, max pause=410ms | Normal p50/p95/p99 and max below deadline |
| Errors | 504=7%, application 5xx=0.8% | 5xx/timeouts/rejections<1% and business success restored |
| Resource | heap oscillates 45-78%; after-GC baseline stable at 1.3GiB | Leading resource/queue at measured baseline |
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

For high garbage collection, I first correlate GC count, pause p95/p99/max, GC CPU, allocation rate, and after-GC baseline with request latency. Here allocation rose from 420MiB/s to 2.8GiB/s, GC CPU reached 29%, and max pause reached 410ms while the after-GC baseline stayed at 1.3GiB. The matched trace puts 2.18s in JSON encoding, and JFR shows temporary arrays and Strings from double serialization in release 4.6. I mitigate by rolling back 4.6 and capping export concurrency at measured safe throughput. The permanent correction streams once, removes debug serialization, and enforces allocation budgets. I verify equal payload/load against bytes per request, GC CPU, pause percentiles, API tails, and a multi-window soak rather than blindly increasing heap.

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

## Q1. High GC versus leak?

**Answer:** High allocation with stable after-GC baseline is churn; a rising baseline indicates retained live data.

## Q2. Why inspect max pause?

**Answer:** Rare pauses drive p99 and deadlines while averages look healthy.

## Q3. Does GC count prove a problem?

**Answer:** No. Pair it with duration, GC CPU, allocation, collector, and request correlation.

## Q4. Why JFR allocation profiling?

**Answer:** It links object types to allocation stacks with controlled overhead.

## Q5. Would larger heap fix it?

**Answer:** It can change frequency but does not remove 39MiB allocated per request.

## Q6. What does stable after-GC prove?

**Answer:** No accumulating live set in the window, not the absence of every leak.

## Q7. How do you verify?

**Answer:** Equal-load A/B metrics for bytes/request, GC CPU, counts, pauses, and API percentiles.
