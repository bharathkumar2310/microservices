# Problem

CPU is high in a Java 17/Spring Boot production service.

The affected operation is `POST /pricing/quote` in `pricing-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `pricing-service` begins.

- Traffic: 900 requests/s
- Normal: p50=42ms, p95=110ms, p99=180ms, max=600ms
- Current: p50=90ms, p95=780ms, p99=2.4s, max=7.1s
- Errors: HTTP 5xx=1.8%, gateway timeouts=4.2%
- Resource snapshot: container CPU=195% of 2 cores, throttle ratio=31%, heap=48%, GC CPU=3%
- Highlighted instance: `pricing-6c8b`
- Dependency path: catalog-service
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
pricing-service (pricing-6c8b)
  |
  +--> catalog-service
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

### Step 1 - Normalize CPU

- **What I check:** Compare node, container, cgroup limit, JVM process CPU, and cores.
- **Why:** Aggregates can hide one route, version, pod, or CPU owner; normalization finds the smallest affected population before deep diagnostics.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** Process and container CPU agree only on .2.
- **Bad:** Node CPU is high but JVM CPU is low.
- **Meaning:** Another workload may own node CPU.
- **Next:** Inspect per-thread CPU and throttling.

### Step 2 - Apply RED and USE

- **What I check:** Overlay latency/errors with utilization, run queue, and throttled periods.
- **Why:** RED quantifies user impact while USE tests whether a specific resource is utilized, saturated, or failing.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** Latency tracks CPU saturation.
- **Bad:** CPU is high without user impact.
- **Meaning:** High utilization can be useful work, not an incident.
- **Next:** Compare CPU per request.

### Step 3 - Separate traffic from regression

- **What I check:** Compare cores/RPS by route and version at equal payloads.
- **Why:** Comparing equal traffic across versions separates changed work per request from legitimate traffic-driven demand.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** CPU/request remains stable as RPS changes.
- **Bad:** It rises 0.9ms to 2.1ms with flat traffic on .2.
- **Meaning:** Work per request regressed.
- **Next:** Profile the changed version.

### Step 4 - Exclude GC CPU

- **What I check:** Compare GC CPU, allocation, count, pauses, and after-GC heap.
- **Why:** GC symptoms require separating temporary allocation, retained live data, collector behavior, and work per request.
- **Example command/query/tool:** Use GC/allocation Prometheus metrics, unified GC logs, and a bounded JFR allocation recording for the same load window.
- **Expected:** GC CPU=3%, max pause<28ms.
- **Bad:** GC CPU and allocation spike with process CPU.
- **Meaning:** Allocation churn may own CPU.
- **Next:** Capture JFR CPU and allocation.

### Step 5 - Capture repeated runtime evidence

- **What I check:** Take three thread dumps and a 60-second JFR during the plateau.
- **Why:** Repeated states and stacks distinguish CPU execution, monitor blocking, pool parking, and downstream I/O waiting.
- **Example command/query/tool:** Run `jcmd <pid> Thread.print -l` three times 10 seconds apart; pair with JFR or OS per-thread CPU.
- **Expected:** Stacks vary normally.
- **Bad:** The same request stacks repeatedly execute Pattern$Loop.match.
- **Meaning:** A sustained application hotspot is likely.
- **Next:** Map the stack to request input.

### Step 6 - Map OS hot threads

- **What I check:** Use top -H or Process Explorer, convert native ID to Java nid, then inspect jcmd output.
- **Why:** Repeated states and stacks distinguish CPU execution, monitor blocking, pool parking, and downstream I/O waiting.
- **Example command/query/tool:** Run `jcmd <pid> Thread.print -l` three times 10 seconds apart; pair with JFR or OS per-thread CPU.
- **Expected:** No thread dominates.
- **Bad:** Eight threads dominate with the same regex stack.
- **Meaning:** The hotspot is bounded and repeatable.
- **Next:** Correlate them with slow traces.

### Step 7 - Distinguish throttling

- **What I check:** Graph throttled periods/periods and runnable queue against the CPU limit.
- **Why:** CPU utilization shows consumed time; throttling shows runnable time denied by the cgroup limit.
- **Example command/query/tool:** Calculate `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])` and compare limits.
- **Expected:** Throttle ratio stays below 2%.
- **Bad:** Ratio is 31% while runnable work grows.
- **Meaning:** The cgroup denies CPU time and magnifies the regression.
- **Next:** Fix code before capacity tuning.

### Step 8 - Prove the trigger

- **What I check:** Group validation time by input length and invalid reason without logging values.
- **Why:** Linking the hot method to a bounded input class proves why that code became expensive in production.
- **Example command/query/tool:** Group a safe input-length bucket against method latency, then reproduce with synthetic boundary and malformed values.
- **Expected:** All inputs validate below 2ms.
- **Bad:** Malformed 4KB input takes about 2.1s.
- **Meaning:** Input-dependent superlinear work is present.
- **Next:** Reproduce with safe synthetic data.

### Step 9 - Verify mitigation and code fix

- **What I check:** A/B test matched traffic and malformed inputs.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** CPU and p99 return to baseline.
- **Bad:** CPU falls but latency remains high.
- **Meaning:** Another queue/dependency remains constrained.
- **Next:** Inspect trace gaps and queue depth.

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
| Scenario: JVM process CPU | Current `195% of two cores` | High suggests the JVM owns the CPU | A deploy-correlated change supports but does not prove causality |
| Scenario: throttle ratio | Current `31%` | High suggests the cgroup denies runnable time | A deploy-correlated change supports but does not prove causality |
| Scenario: CPU/request | Current `2.1ms versus 0.9ms` | High suggests code cost regression | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `pricing-6c8b`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 12ms -> pricing 2.31s -> promotion validation 2.18s -> catalog 76ms
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
2026-09-13T16:39:22.481Z level=ERROR service=pricing-service instance=pricing-6c8b
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="POST /pricing/quote" downstream="catalog-service" latencyMs=5012
error="quote_slow validationMs=2184 promotionLength=4096 outcome=INVALID" appVersion=2026.09.13.2
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

Worked root cause: **a new promotion validator catastrophically backtracked on malformed 4KB input**.

```text
a new promotion validator catastrophically backtracked on malformed 4KB input
  -> changes work, waiting, or ownership per request
  -> CPU/request rises 0.9ms to 2.1ms only on version .2; repeated RUNNABLE stacks are in java.util.regex.Pattern
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
- reject overlong promotion codes, route traffic from .2, and roll back the validator.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- replace the ambiguous regex with a bounded linear parser and adversarial performance tests.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **CPU<65%, throttle<2%, CPU/request near 0.9ms, p99<190ms**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | p50=90ms, p95=780ms, p99=2.4s, max=7.1s | Normal p50/p95/p99 and max below deadline |
| Errors | HTTP 5xx=1.8%, gateway timeouts=4.2% | 5xx/timeouts/rejections<1% and business success restored |
| Resource | container CPU=195% of 2 cores, throttle ratio=31%, heap=48%, GC CPU=3% | Leading resource/queue at measured baseline |
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

For high CPU, I first lock the time window and scope RED metrics by route, version, and instance. Here normal `p50=42ms, p95=110ms, p99=180ms, max=600ms` became `p50=90ms, p95=780ms, p99=2.4s, max=7.1s`. I compare USE metrics and a matched trace; the key waterfall is `gateway 12ms -> pricing 2.31s -> promotion validation 2.18s -> catalog 76ms`. I correlate its traceId with structured logs and capture bounded JVM evidence before restarting. Together they prove a new promotion validator catastrophically backtracked on malformed 4KB input. I mitigate by rejecting overlong promotion codes, routing traffic away from version .2, and rolling back the validator. The permanent correction replaces the ambiguous regex with a bounded linear parser and adds adversarial performance tests. I verify equal load and failure paths against p50/p95/p99/max, errors, the leading resource signal, every instance, and a multi-window soak. I do not call a restart or resource increase a fix.

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

## Q1. Does 100% CPU always mean saturation?

**Answer:** No. Normalize by cores and limits and check throttling and run queue.

## Q2. How do you separate traffic from regression?

**Answer:** Compare CPU per request across versions under equal route and payload mix.

## Q3. Why multiple thread dumps?

**Answer:** Repeated stacks identify sustained hot code instead of a momentary innocent sample.

## Q4. What does RUNNABLE mean?

**Answer:** It may be executing or eligible to execute, and can include native I/O; stacks plus CPU samples give meaning.

## Q5. Can GC cause high CPU with low heap?

**Answer:** Yes. Allocation churn can trigger frequent young collections while retained heap stays modest.

## Q6. Why not raise CPU first?

**Answer:** It may mitigate after evidence but does not bound exponential work and can conceal the defect.

## Q7. What proves the regex cause?

**Answer:** JFR samples, repeated stacks, input correlation, version comparison, and controlled reproduction.
