# Problem

The service runs out of memory in a Java 17/Spring Boot production service.

The affected operation is `POST /documents/render` in `document-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `document-service` begins.

- Traffic: 45 requests/s with 12 concurrent large renders
- Normal: after-GC heap=1.4GiB, RSS=2.3GiB, p95=900ms
- Current: pod restarts every 18min; RSS reaches 3.95GiB of a 4GiB limit
- Errors: 502=22% during restart windows
- Resource snapshot: heap=52% of 2GiB; direct buffer capacity rises to 1.55GiB
- Highlighted instance: `document-77d9`
- Dependency path: object storage
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
document-service (document-77d9)
  |
  +--> object storage
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

### Step 1 - Classify the termination

- **What I check:** Read pod lastState, reason, exit code, JVM fatal logs, and previous application logs.
- **Why:** Heap OOM, native growth, and Kubernetes OOMKilled have different limiters and require different evidence.
- **Example command/query/tool:** Compare pod `lastState`, RSS/limit, JVM memory pools, BufferPoolMXBeans, threads, classloaders, and `jcmd VM.native_memory summary`.
- **Expected:** A named OOM boundary is present.
- **Bad:** Only restart count is known.
- **Meaning:** Liveness, crash, kill, and OOM can look alike.
- **Next:** Inspect events and termination details.

### Step 2 - Check OOM signatures

- **What I check:** Search Java heap space, GC overhead, Metaspace, Direct buffer memory, unable to create native thread, and requested array size.
- **Why:** Heap OOM, native growth, and Kubernetes OOMKilled have different limiters and require different evidence.
- **Example command/query/tool:** Compare pod `lastState`, RSS/limit, JVM memory pools, BufferPoolMXBeans, threads, classloaders, and `jcmd VM.native_memory summary`.
- **Expected:** One message names the failed allocation.
- **Bad:** No JVM OOM but Kubernetes reports OOMKilled.
- **Meaning:** The cgroup killed total process memory.
- **Next:** Compare RSS with JVM pools.

### Step 3 - Compare RSS and limit

- **What I check:** Graph working set/RSS, limit, heap committed/used, direct, metaspace, and threads.
- **Why:** Heap OOM, native growth, and Kubernetes OOMKilled have different limiters and require different evidence.
- **Example command/query/tool:** Compare pod `lastState`, RSS/limit, JVM memory pools, BufferPoolMXBeans, threads, classloaders, and `jcmd VM.native_memory summary`.
- **Expected:** Headroom remains safe.
- **Bad:** RSS reaches 3.95/4GiB while heap is 1.04GiB.
- **Meaning:** Non-heap/native memory owns the risk.
- **Next:** Break down direct/thread/metaspace/native.

### Step 4 - Inspect direct buffers

- **What I check:** Read BufferPoolMXBean, Netty allocator metrics, and NMT when enabled.
- **Why:** Heap OOM, native growth, and Kubernetes OOMKilled have different limiters and require different evidence.
- **Example command/query/tool:** Compare pod `lastState`, RSS/limit, JVM memory pools, BufferPoolMXBeans, threads, classloaders, and `jcmd VM.native_memory summary`.
- **Expected:** Capacity falls after requests.
- **Bad:** Direct reaches 1.55GiB and never returns after cancellation.
- **Meaning:** Buffers are retained or unreleased.
- **Next:** Correlate growth with failure paths.

### Step 5 - Reject thread and Metaspace OOM

- **What I check:** Check thread names/count/stacks, loaded classes/classloaders, and Metaspace after GC.
- **Why:** Repeated states and stacks distinguish CPU execution, monitor blocking, pool parking, and downstream I/O waiting.
- **Example command/query/tool:** Run `jcmd <pid> Thread.print -l` three times 10 seconds apart; pair with JFR or OS per-thread CPU.
- **Expected:** 214 threads and 210MiB remain stable.
- **Bad:** Thousands of threads or classloaders grow.
- **Meaning:** Investigate native stacks or classloader leak.
- **Next:** Trace creator/retainer.

### Step 6 - Prove cancellation ownership

- **What I check:** Group direct delta by response size, timeout, cancel, reset, and version.
- **Why:** Growth alone is correlation; ownership, return/disposal behavior, and a triggering path establish retention causality.
- **Example command/query/tool:** Group direct-buffer capacity delta by `appVersion`, response-size bucket, and outcome (`success`, `timeout`, `cancel`, or `reset`), then compare version .2 with a healthy version.
- **Expected:** Memory returns for every outcome.
- **Bad:** Each cancelled large body leaves direct capacity only on version .2.
- **Meaning:** The release path is incomplete.
- **Next:** Review doOnDiscard/release handling.

### Step 7 - Choose capture safely

- **What I check:** Use NMT/buffer metrics first; heap dump only if wrappers/roots matter and headroom permits.
- **Why:** The live baseline and retained paths distinguish normal peaks, allocation churn, and unwanted long-lived ownership.
- **Example command/query/tool:** Graph post-GC live bytes; compare `jcmd <pid> GC.class_histogram`; use a secured MAT dominator analysis only when safe.
- **Expected:** A drained replica captures safely.
- **Bad:** Dump triggers kill or contains no native bytes.
- **Meaning:** The method is unsuitable or unsafe.
- **Next:** Use controlled Netty leak tests.

### Step 8 - Mitigate fleet risk

- **What I check:** Remove version .2, limit body size/concurrency, and stop retrying cancelled work.
- **Why:** Mitigation must reduce admitted work while preserving evidence and in-flight requests, not merely move waiting into a larger queue.
- **Example command/query/tool:** Remove readiness, verify Service endpoints no longer include the pod, watch in-flight requests reach zero, then terminate.
- **Expected:** Healthy capacity remains and errors fall.
- **Bad:** All replicas approach the limit.
- **Meaning:** Shared load/code affects the fleet.
- **Next:** Shed admission and reduce concurrency.

### Step 9 - Verify failure paths

- **What I check:** Repeat success, timeout, cancel, reset, and oversized-body tests.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** Direct/RSS baseline returns each cycle.
- **Bad:** Baseline grows per cancellation.
- **Meaning:** A release path remains missing.
- **Next:** Instrument disposal and block release.

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
| Scenario: RSS/limit | Current `3.95/4GiB` | High suggests cgroup kill risk | A deploy-correlated change supports but does not prove causality |
| Scenario: direct capacity | Current `1.55GiB rising` | High suggests native direct leak | A deploy-correlated change supports but does not prove causality |
| Scenario: termination | Current `OOMKilled/137` | High suggests kernel cgroup kill, not proof of heap OOM | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `document-77d9`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 14ms -> document 11.8s -> storage download 10.9s -> client cancellation
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
2026-09-13T16:39:22.481Z level=ERROR service=document-service instance=document-77d9
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="POST /documents/render" downstream="object storage" latencyMs=5012
error="pod terminated reason=OOMKilled exitCode=137 previousLogsEnd=cancelled_download" appVersion=2026.09.13.2
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

Worked root cause: **release 2026.09.13.2 introduced a WebClient cancellation/error path that failed to release pooled DataBuffers, exhausting native direct memory until cgroup OOMKilled**.

```text
release 2026.09.13.2 introduced a WebClient cancellation/error path that failed to release pooled DataBuffers
  -> unreleased direct memory accumulated until the cgroup limit caused OOMKilled
  -> changes work, waiting, or ownership per request
  -> lastState reason=OOMKilled exitCode=137; no Java heap exception; each cancelled 100MiB download leaves about 96MiB direct capacity
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
- Stop large renders on version .2, drain it, and roll back after capturing termination and buffer evidence.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- release/discard DataBuffers on success, error, and cancellation; stream with strict size limits.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **direct capacity returns after cancellations, RSS plateaus<2.6GiB, no OOMKilled for three prior failure windows**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | pod restarts every 18min; RSS reaches 3.95GiB of a 4GiB limit | Normal p50/p95/p99 and max below deadline |
| Errors | 502=22% during restart windows | 5xx/timeouts/rejections<1% and business success restored |
| Resource | heap=52% of 2GiB; direct buffer capacity rises to 1.55GiB | Leading resource/queue at measured baseline |
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

When a service runs out of memory, I first classify the failure from the JVM message, Kubernetes `lastState`, reason, and exit code. Here heap stayed at 52%, but RSS reached 3.95GiB of a 4GiB limit and version .2 was OOMKilled. The trace ends after a large object-storage download is cancelled, and direct-buffer capacity remains elevated only on .2. That proves its WebClient cancellation path failed to release pooled DataBuffers. I mitigate by stopping large renders on .2, draining it, and rolling it back after preserving evidence. The permanent correction releases buffers on success, error, discard, and cancellation and streams with strict size limits. I verify cancellation and reset paths until direct memory and RSS repeatedly return to baseline, with no OOMKilled for three prior failure windows.

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

## Q1. OOMKilled versus heap OOM?

**Answer:** OOMKilled is the kernel/cgroup enforcing total memory; heap OOM is thrown by the JVM for a failed managed allocation.

## Q2. Relevant OOM types?

**Answer:** Java heap space, GC overhead limit, Metaspace, Direct buffer memory, unable to create native thread, and requested array size.

## Q3. Should every OOM get a heap dump?

**Answer:** No. Dumps can pause, contain data, require disk, and miss native memory.

## Q4. Why is exit 137 insufficient?

**Answer:** SIGKILL has other causes; Kubernetes reason and events establish OOMKilled.

## Q5. Can larger Xmx worsen it?

**Answer:** Yes. It removes container headroom from direct, metaspace, code cache, and stacks.

## Q6. How do cancellation leaks happen?

**Answer:** Reference-counted buffers must be released on success, error, discard, and cancellation.

## Q7. What proves the fix?

**Answer:** Repeated cancellation load with stable direct/RSS baseline and no kill over several failure windows.
