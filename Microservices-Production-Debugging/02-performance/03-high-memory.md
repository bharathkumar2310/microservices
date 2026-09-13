# Problem

Memory is high in a Java 17/Spring Boot production service.

The affected operation is `GET /catalog/search` in `catalog-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `catalog-service` begins.

- Traffic: 320 requests/s
- Normal: after-GC heap=1.1GiB, RSS=1.8GiB, p95=160ms
- Current: after-GC heap rises 1.2GiB to 3.4GiB in 70min; RSS=4.6GiB; p95=410ms
- Errors: HTTP 5xx=0.4%, no OOM yet
- Resource snapshot: heap=82% of 4GiB, metaspace=210MiB, direct=280MiB, threads=214
- Highlighted instance: `catalog-54bd`
- Dependency path: Redis -> MySQL
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
catalog-service (catalog-54bd)
  |
  +--> Redis -> MySQL
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

### Step 1 - Name the growing memory

- **What I check:** Compare heap, after-GC heap, metaspace, code cache, direct buffers, thread stacks, and RSS.
- **Why:** Heap OOM, native growth, and Kubernetes OOMKilled have different limiters and require different evidence.
- **Example command/query/tool:** Compare pod `lastState`, RSS/limit, JVM memory pools, BufferPoolMXBeans, threads, classloaders, and `jcmd VM.native_memory summary`.
- **Expected:** RSS movement matches one component.
- **Bad:** RSS rises while heap stays flat.
- **Meaning:** Heap dump may be the wrong tool.
- **Next:** Choose component-specific evidence.

### Step 2 - Track after-GC baseline

- **What I check:** Measure old/live heap after collections across time.
- **Why:** The live baseline and retained paths distinguish normal peaks, allocation churn, and unwanted long-lived ownership.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** It returns near 1.1GiB.
- **Bad:** It climbs about 32MiB/min.
- **Meaning:** Objects remain reachable, suggesting retention.
- **Next:** Compare slope by version and endpoint.

### Step 3 - Separate allocation and retention

- **What I check:** Compare allocation, promotion, GC frequency, and after-GC slope.
- **Why:** The live baseline and retained paths distinguish normal peaks, allocation churn, and unwanted long-lived ownership.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** Allocation follows traffic but baseline is flat.
- **Bad:** Baseline rises under flat traffic and remains after idle.
- **Meaning:** Retained objects, not only churn, consume heap.
- **Next:** Take two class histograms.

### Step 4 - Compare histograms

- **What I check:** Run jcmd GC.class_histogram twice 15 minutes apart.
- **Why:** The live baseline and retained paths distinguish normal peaks, allocation churn, and unwanted long-lived ownership.
- **Example command/query/tool:** Graph post-GC live bytes; compare `jcmd <pid> GC.class_histogram`; use a secured MAT dominator analysis only when safe.
- **Expected:** Counts stabilize after warmup.
- **Bad:** DTOs, Strings, and map nodes grow together.
- **Meaning:** A collection is retaining object graphs.
- **Next:** Inspect dominators and GC roots.

### Step 5 - Inspect retained ownership

- **What I check:** Use MAT dominator tree and Paths to GC Roots on an encrypted dump.
- **Why:** Growth alone is correlation; ownership, return/disposal behavior, and a triggering path establish retention causality.
- **Example command/query/tool:** Graph post-GC live bytes; compare `jcmd <pid> GC.class_histogram`; use a secured MAT dominator analysis only when safe.
- **Expected:** No unexpected dominator exists.
- **Bad:** LocalSearchCache retains 2.1GiB via Caffeine nodes.
- **Meaning:** The cache owns the live set.
- **Next:** Validate cache value and policy.

### Step 6 - Measure cache usefulness

- **What I check:** Inspect size, weight, hit/miss, eviction, key cardinality, and entry age.
- **Why:** Growth alone is correlation; ownership, return/disposal behavior, and a triggering path establish retention causality.
- **Example command/query/tool:** Graph post-GC live bytes; compare `jcmd <pid> GC.class_histogram`; use a secured MAT dominator analysis only when safe.
- **Expected:** Size plateaus with useful hit rate.
- **Bad:** Raw variants grow, hit=7%, eviction=0.
- **Meaning:** Mostly one-use entries are retained.
- **Next:** Review key normalization and bounds.

### Step 7 - Rule out native and non-heap growth

- **What I check:** Inspect BufferPoolMXBeans, NMT, classloaders, metaspace, and thread count.
- **Why:** RSS can rise because of direct buffers, Metaspace, native allocations, or thread stacks even when Java heap retention is stable; each source needs different evidence.
- **Example command/query/tool:** Compare `jvm_buffer_memory_used_bytes`, Metaspace usage, live thread count multiplied by configured `-Xss`, container RSS, and `jcmd <pid> VM.native_memory summary` when NMT is enabled.
- **Expected:** Native categories remain stable.
- **Bad:** A native category grows while heap is flat.
- **Meaning:** Investigate direct, classloader, or thread ownership.
- **Next:** Use the matching diagnostic.

### Step 8 - Capture dumps safely

- **What I check:** Drain a replica, verify disk/encryption/access, and prefer histogram/JFR first.
- **Why:** The live baseline and retained paths distinguish normal peaks, allocation churn, and unwanted long-lived ownership.
- **Example command/query/tool:** Graph post-GC live bytes; compare `jcmd <pid> GC.class_histogram`; use a secured MAT dominator analysis only when safe.
- **Expected:** Capture completes without customer impact.
- **Bad:** Pod stalls, disk fills, or dump is exposed.
- **Meaning:** Diagnostics created availability/security risk.
- **Next:** Use a safer controlled replica.

### Step 9 - Verify long enough

- **What I check:** Load varied queries for two previous failure windows.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** Cache and after-GC baseline plateau.
- **Bad:** Baseline resumes climbing.
- **Meaning:** The bound or another retention path is wrong.
- **Next:** Repeat histogram/dominator comparison.

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
| Scenario: after-GC baseline | Current `+32MiB/min` | High suggests live retention | A deploy-correlated change supports but does not prove causality |
| Scenario: RSS minus heap | Current `widens only if native memory grows` | High suggests native/direct/thread candidate | A deploy-correlated change supports but does not prove causality |
| Scenario: cache size/hit/eviction | Current `184k/7%/0` | High suggests unbounded low-value cache | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `catalog-54bd`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 10ms -> catalog 390ms -> cache lookup 3ms -> SQL 155ms -> mapping 198ms
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
2026-09-13T16:39:22.481Z level=ERROR service=catalog-service instance=catalog-54bd
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="GET /catalog/search" downstream="Redis -> MySQL" latencyMs=5012
error="cache_stats name=searchResults estimatedSize=184220 hitRate=0.07 evictions=0" appVersion=2026.09.13.2
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

Worked root cause: **an unbounded Caffeine cache keyed by raw search text retained large SearchResult graphs**.

```text
an unbounded Caffeine cache keyed by raw search text retained large SearchResult graphs
  -> changes work, waiting, or ownership per request
  -> SearchResult, ProductDto, String, and map nodes grow; cache size=184,220, hit=7%, evictions=0
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
- disable the cache and drain only pods near the limit after safe evidence capture.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- normalize keys, set maximumWeight with measured weights and expiry, and expose cache telemetry.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **after-GC heap plateaus<1.4GiB, RSS<2.2GiB, cache weight bounded, evictions occur**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | after-GC heap rises 1.2GiB to 3.4GiB in 70min; RSS=4.6GiB; p95=410ms | Normal p50/p95/p99 and max below deadline |
| Errors | HTTP 5xx=0.4%, no OOM yet | 5xx/timeouts/rejections<1% and business success restored |
| Resource | heap=82% of 4GiB, metaspace=210MiB, direct=280MiB, threads=214 | Leading resource/queue at measured baseline |
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

For high memory, I first lock the time window and scope RED metrics by route, version, and instance. Here normal `after-GC heap=1.1GiB, RSS=1.8GiB, p95=160ms` became `after-GC heap rises 1.2GiB to 3.4GiB in 70min; RSS=4.6GiB; p95=410ms`. I compare heap, non-heap, direct memory, thread memory, RSS, and a matched trace. Histograms and retained paths prove that an unbounded Caffeine cache keyed by raw search text retained large SearchResult graphs. I mitigate by disabling the cache and draining only pods near the limit after safe evidence capture. The permanent correction normalizes keys, sets `maximumWeight` with measured weights and expiry, and exposes cache telemetry. I verify equal load and failure paths against after-GC baseline, RSS, p50/p95/p99/max, errors, every instance, and a multi-window soak. I do not call a restart or larger heap a fix.

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

## Q1. Heap versus RSS?

**Answer:** Heap is JVM object memory; RSS also includes committed pages, metaspace, code, direct buffers, stacks, libraries, and JVM native memory.

## Q2. Does high heap prove a leak?

**Answer:** No. A rising after-GC live baseline plus unwanted retained paths is stronger evidence.

## Q3. Allocation versus retention?

**Answer:** Allocation is creation rate; retention is reachable live data. Churn drives GC, retention drives exhaustion.

## Q4. When is a heap dump unsafe?

**Answer:** On a busy large-heap pod, without disk headroom, encryption, restricted access, and a pause plan.

## Q5. Why can RSS stay high after GC?

**Answer:** The JVM may retain committed pages and native/direct memory is separate from live heap.

## Q6. Why not increase heap?

**Answer:** It delays failure and hides unbounded retention without correcting ownership.

## Q7. How is the cache fix proved?

**Answer:** Stable live baseline under soak, bounded weight, expected evictions, and no growing retained dominator.
