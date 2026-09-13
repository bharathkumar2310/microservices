# Problem

A thread pool is exhausted in a Java 17/Spring Boot production service.

The affected operation is `POST /payments/authorize` in `payment-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `payment-service` begins.

- Traffic: 260 requests/s
- Normal: p95=190ms; fraudExecutor active=8/32; queue<10
- Current: p95=6.2s; active=32/32; queue=500/500; rejects=84/s
- Errors: 503=14%, client timeout=19%
- Resource snapshot: CPU=24%, heap=51%, GC max=19ms
- Highlighted instance: `payment-69f7`
- Dependency path: fraud-service
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
payment-service (payment-69f7)
  |
  +--> fraud-service
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

### Step 1 - Name the pool

- **What I check:** Inventory Tomcat, async, scheduler, Kafka, event-loop, HTTP, and Hikari pools.
- **Why:** Active workers, queue age, completion, and rejection reveal where arrival rate exceeds service rate.
- **Example command/query/tool:** Query executor active/max, queue size/capacity/age, rejected/completed rates, task duration, and retry spans.
- **Expected:** One named pool aligns with impact.
- **Bad:** Only total thread count is known.
- **Meaning:** Thread count cannot identify queue ownership.
- **Next:** Map thread prefixes and executor beans.

### Step 2 - Read saturation stages

- **What I check:** Graph active/max, queue size/capacity/age, rejected, completed, and task duration.
- **Why:** Active workers, queue age, completion, and rejection reveal where arrival rate exceeds service rate.
- **Example command/query/tool:** Query executor active/max, queue size/capacity/age, rejected/completed rates, task duration, and retry spans.
- **Expected:** Active has headroom, queue near zero, rejects zero.
- **Bad:** 32/32, 500/500, 84 rejects/s.
- **Meaning:** Arrival exceeds completion.
- **Next:** Compare arrival and completion.

### Step 3 - Apply Little's Law

- **What I check:** Calculate concurrency=throughput*time.
- **Why:** Active workers, queue age, completion, and rejection reveal where arrival rate exceeds service rate.
- **Example command/query/tool:** Query executor active/max, queue size/capacity/age, rejected/completed rates, task duration, and retry spans.
- **Expected:** 260/s*0.18s is manageable overall.
- **Bad:** 260/s*6s is about 1,560 in flight.
- **Meaning:** Latency alone can overwhelm unchanged capacity.
- **Next:** Find the expanded wait.

### Step 4 - Read repeated states

- **What I check:** Capture three jcmd Thread.print dumps.
- **Why:** Repeated states and stacks distinguish CPU execution, monitor blocking, pool parking, and downstream I/O waiting.
- **Example command/query/tool:** Run `jcmd <pid> Thread.print -l` three times 10 seconds apart; pair with JFR or OS per-thread CPU.
- **Expected:** Workers progress through varied stacks.
- **Bad:** All remain in socketRead/client wait.
- **Meaning:** Dependency I/O occupies workers.
- **Next:** Inspect fraud spans and client pool.

### Step 5 - Distinguish lock contention

- **What I check:** Count RUNNABLE, BLOCKED, WAITING and common owners.
- **Why:** Repeated states and stacks distinguish CPU execution, monitor blocking, pool parking, and downstream I/O waiting.
- **Example command/query/tool:** Run `jcmd <pid> Thread.print -l` three times 10 seconds apart; pair with JFR or OS per-thread CPU.
- **Expected:** No persistent monitor owner.
- **Bad:** Many BLOCKED point to one synchronized owner.
- **Meaning:** A Java lock, not I/O, may exhaust workers.
- **Next:** Profile the owner.

### Step 6 - Count retry spans

- **What I check:** Inspect attempt count, duration, overlap, deadline, and outcome.
- **Why:** Each attempt extends worker occupancy and dependency load, so retries can be the multiplier rather than the original fault.
- **Example command/query/tool:** Query executor active/max, queue size/capacity/age, rejected/completed rates, task duration, and retry spans.
- **Expected:** One 80ms call.
- **Bad:** Three sequential 2s calls.
- **Meaning:** Retries triple occupancy and load.
- **Next:** Review idempotency and retry budget.

### Step 7 - Inspect dependent pools

- **What I check:** Check HTTP leased/pending/max, Hikari pending, and dependency RED.
- **Why:** Pool utilization without pending/acquisition time cannot distinguish healthy reuse from request starvation.
- **Example command/query/tool:** Query `hikaricp_connections_active`, `idle`, `pending`, `max`, acquisition histograms, DB lock views, and checkout/return counters.
- **Expected:** Other pools have headroom.
- **Bad:** HTTP pending is high before connect.
- **Meaning:** The first smaller pool may be the bottleneck.
- **Next:** Follow the first saturated queue.

### Step 8 - Mitigate without a larger queue

- **What I check:** Disable unsafe retries, bound deadline, circuit-break, and shed.
- **Why:** Mitigation must reduce admitted work while preserving evidence and in-flight requests, not merely move waiting into a larger queue.
- **Example command/query/tool:** Query executor active/max, queue size/capacity/age, rejected/completed rates, task duration, and retry spans.
- **Expected:** Queue drains and useful throughput returns.
- **Bad:** Queue stays full after recovery.
- **Meaning:** Stale tasks or producers continue feeding it.
- **Next:** Inspect cancellation and queue age.

### Step 9 - Verify overload recovery

- **What I check:** Test normal load, slowdown, recovery, and 350/s overload.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** Bounded rejection and recovery without restart.
- **Bad:** Queue outlives deadlines or restart is needed.
- **Meaning:** Cancellation/backpressure is incomplete.
- **Next:** Fix lifecycle before release.

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
| Scenario: executor active/max | Current `32/32` | High suggests worker saturation | A deploy-correlated change supports but does not prove causality |
| Scenario: queue/rejected | Current `500/500 and 84/s` | High suggests arrival exceeds completion | A deploy-correlated change supports but does not prove causality |
| Scenario: task duration | Current `6s versus 100ms` | High suggests Little's Law multiplier | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `payment-69f7`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 13ms -> payment 6.05s -> fraud attempt1 2s -> attempt2 2s -> attempt3 2s
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
2026-09-13T16:39:22.481Z level=ERROR service=payment-service instance=payment-69f7
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="POST /payments/authorize" downstream="fraud-service" latencyMs=5012
error="TaskRejectedException pool=fraudExecutor active=32 max=32 queue=500 remaining=0" appVersion=2026.09.13.2
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

Worked root cause: **fraud-service slowed and three immediate retries held every bounded executor worker for about six seconds**.

```text
fraud-service slowed and three immediate retries held every bounded executor worker for about six seconds
  -> changes work, waiting, or ownership per request
  -> all 32 workers repeatedly wait in synchronous Feign socket reads; completion falls below arrival
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
- disable retries, open the noncritical fraud circuit, shed excess work, and recover/roll back fraud-service.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- use deadline-aware calls, bounded bulkheads, safe backoff/jitter, and rejection-aware admission.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **active has headroom, queue<10, rejected=0, p95<200ms, recovery<60s after induced slowdown**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | p95=6.2s; active=32/32; queue=500/500; rejects=84/s | Normal p50/p95/p99 and max below deadline |
| Errors | 503=14%, client timeout=19% | 5xx/timeouts/rejections<1% and business success restored |
| Resource | CPU=24%, heap=51%, GC max=19ms | Leading resource/queue at measured baseline |
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

For thread-pool exhaustion, I first identify the exact executor and compare active/max, queue size and age, completion rate, rejection rate, and task duration. Here `fraudExecutor` changed from active 8/32 with queue below 10 to 32/32 with queue 500/500 and 84 rejects/s. Repeated dumps show workers waiting in Feign reads, while the trace shows three sequential two-second fraud attempts. At 260 requests/s, Little's Law explains why six-second occupancy overwhelms the pool. I mitigate by disabling retries, opening the noncritical fraud circuit, shedding excess work, and recovering or rolling back fraud-service. The permanent correction uses deadline-aware calls, bounded bulkheads, safe backoff/jitter, and rejection-aware admission. I verify induced slowdown, bounded rejection, queue drain, and recovery without restart.

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

## Q1. What does active=max mean?

**Answer:** All workers are occupied; queue, completion, rejection, and duration show whether that is harmful.

## Q2. Why not add threads?

**Answer:** If workers wait on a slow dependency, more threads increase pressure without increasing completions.

## Q3. Explain Little's Law here.

**Answer:** 260 requests/s times six seconds is about 1,560 in-flight tasks.

## Q4. RUNNABLE versus BLOCKED?

**Answer:** BLOCKED waits for a Java monitor. RUNNABLE may execute or wait in native I/O; stacks give context.

## Q5. Why is an unbounded queue dangerous?

**Answer:** It turns overload into stale latency, memory growth, and delayed failure.

## Q6. How do retries exhaust pools?

**Answer:** Each request occupies a worker longer and creates more dependency calls.

## Q7. How is recovery verified?

**Answer:** Induce slowdown/overload and prove bounded queues, correct rejection, cancellation, and automatic drain.
