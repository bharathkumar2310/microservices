# Problem

Only one instance is slow in a Java 17/Spring Boot production service.

The affected operation is `GET /accounts/{id}` in `account-service`.

The objective is to find the first constrained stage, prove causality, mitigate safely, and verify a permanent fix.

I do not restart or blindly increase CPU, heap, thread count, queue, pool, or timeout before preserving evidence.

# Production Situation

At 22:08 IST, the alert for `account-service` begins.

- Traffic: six pods total (one bad pod plus five healthy peers), fleet=1,200 requests/s, about 200/s per pod
- Normal: all six pods p95=145-170ms, p99<230ms
- Current: one pod p95=2.9s, p99=5.1s; five healthy peers p95<175ms
- Errors: slow pod 504=11% (22/s); five peers 504=0.08% (0.8/s combined); weighted fleet 504=1.9% (22.8/1,200)
- Resource snapshot: slow pod CPU=31%, heap=49%, GC max=22ms; peers similar
- Highlighted instance: `account-old-2f41`
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
account-service (account-old-2f41)
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

### Step 1 - Preserve pod labels

- **What I check:** Group RED/USE by pod, version, revision, zone, node, and config hash.
- **Why:** Aggregates can hide one route, version, pod, or CPU owner; normalization finds the smallest affected population before deep diagnostics.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** Distributions are tight.
- **Bad:** One pod p95=2.9s while peers<175ms.
- **Meaning:** The fault is local or traffic-skewed.
- **Next:** Compare request mix.

### Step 2 - Check routing skew

- **What I check:** Compare RPS, routes, tenants, payloads, sticky sessions, and connection reuse.
- **Why:** Matched requests eliminate tenant, payload, cache, and routing differences before attributing slowness to a pod.
- **Example command/query/tool:** Prometheus: `sum(rate(http_server_requests_seconds_count[5m])) by (uri,status,version,pod)` plus matching histogram quantiles.
- **Expected:** About 200/s and comparable mix.
- **Bad:** The slow pod gets expensive requests.
- **Meaning:** Routing/hash skew may explain it.
- **Next:** Run a matched safe probe.

### Step 3 - Use matched probes

- **What I check:** Send the same authorized read through a controlled pod route.
- **Why:** Matched requests eliminate tenant, payload, cache, and routing differences before attributing slowness to a pod.
- **Example command/query/tool:** Send the same authorized read to each pod through the approved debug route and record instance plus traceId.
- **Expected:** All pods match.
- **Bad:** Only old-2f41 takes about 2.8s.
- **Meaning:** The issue is reproducibly instance-specific.
- **Next:** Compare trace waterfalls.

### Step 4 - Compare waterfalls

- **What I check:** Match account and payload on slow and healthy pods.
- **Why:** A trace assigns elapsed time to queue, local code, retry, connection acquisition, SQL, or downstream work.
- **Example command/query/tool:** Trace search: `service.name=<service> AND http.route=<route> AND duration>2s`, then compare a matched healthy trace.
- **Expected:** Children are similar.
- **Bad:** Slow pod spends 2.71s acquiring Hikari; SQL is 61ms on both.
- **Meaning:** Local pool wait owns latency.
- **Next:** Compare runtime pool config.

### Step 5 - Compare effective config

- **What I check:** Read sanitized config hash/startup values and Hikari metrics.
- **Why:** Pool utilization without pending/acquisition time cannot distinguish healthy reuse from request starvation.
- **Example command/query/tool:** Query `hikaricp_connections_active`, `idle`, `pending`, `max`, acquisition histograms, DB lock views, and checkout/return counters.
- **Expected:** Every pod max=24.
- **Bad:** Old pod max=4 active=4 pending=37.
- **Meaning:** Runtime configuration drift exists.
- **Next:** Compare owner ReplicaSet and image.

### Step 6 - Reject node/JVM causes

- **What I check:** Compare throttling, node pressure, retransmits, DNS, GC, heap, and hot threads.
- **Why:** A one-pod symptom may follow node pressure, throttling, network, JVM state, config, or revision; comparison identifies which.
- **Example command/query/tool:** Compare `kubectl get pods -o wide`, ReplicaSet revision, image digest, config checksum, limits, node, and deployment events.
- **Expected:** Only pool config differs.
- **Bad:** Symptom follows a node or throttle metric.
- **Meaning:** Investigate node/cgroup/network.
- **Next:** Reschedule only after evidence.

### Step 7 - Check rollout convergence

- **What I check:** Compare image digest, revision, age, config checksum, and rollout status.
- **Why:** Comparing equal traffic across versions separates changed work per request from legitimate traffic-driven demand.
- **Example command/query/tool:** Compare `kubectl get pods -o wide`, ReplicaSet revision, image digest, config checksum, limits, node, and deployment events.
- **Expected:** All desired replicas match.
- **Bad:** One revision 41 pod remains among revision 42.
- **Meaning:** Partial rollout created behavioral heterogeneity.
- **Next:** Find why release gate passed.

### Step 8 - Drain safely

- **What I check:** Remove readiness, verify endpoint removal, wait for active requests, then terminate.
- **Why:** Mitigation must reduce admitted work while preserving evidence and in-flight requests, not merely move waiting into a larger queue.
- **Example command/query/tool:** Remove readiness, verify Service endpoints no longer include the pod, watch in-flight requests reach zero, then terminate.
- **Expected:** New traffic stops and in-flight drains.
- **Bad:** Traffic continues reaching the pod.
- **Meaning:** Endpoint/mesh connection draining is incomplete.
- **Next:** Inspect endpoints and persistent connections.

### Step 9 - Verify homogeneity

- **What I check:** Check every live pod's revision/hash/pool and direct probe.
- **Why:** Recovery must survive the original trigger and duration; a short happy-path check or restart is not proof.
- **Example command/query/tool:** Run the controlled regression workload and query the same RED/USE panels for at least two previous failure windows.
- **Expected:** All match and tails converge.
- **Bad:** Any old replica or config remains.
- **Meaning:** Deployment has not converged.
- **Next:** Block completion.

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
| Scenario: per-pod p95 | Current `one 2.9s, peers<175ms` | High suggests instance outlier | A deploy-correlated change supports but does not prove causality |
| Scenario: Hikari max/active/pending | Current `4/4/37` | High suggests local pool saturation | A deploy-correlated change supports but does not prove causality |
| Scenario: revision/config hash | Current `one old` | High suggests partial rollout | A deploy-correlated change supports but does not prove causality |

Percentiles and load:
- p50 is the typical request; a rise means broad impact.
- p95 reveals queueing that can precede median degradation.
- p99 represents severe tails that trigger deadlines and retries.
- max supplies examples but is noisy; pair it with count and traceId.
- Throughput is completed work, not merely accepted work.
- If arrivals stay flat while completions fall, queue size and age rise.
- More concurrency can be the effect of latency, not additional capacity.

# Distributed Trace Investigation

Representative slow trace for `account-old-2f41`:

```text
traceId=4f921d7b0b3e41d8b355f0237c281a9e
gateway spanId=aa10
  -> gateway 9ms -> account-old-2f41 2.84s -> Hikari acquire 2.71s -> SQL 61ms
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
2026-09-13T16:39:22.481Z level=ERROR service=account-service instance=account-old-2f41
traceId=4f921d7b0b3e41d8b355f0237c281a9e spanId=bc21 requestId=req-842193
endpoint="GET /accounts/{id}" downstream="MySQL" latencyMs=5012
error="HikariPool-1 Connection not available timeoutMs=3000 poolMax=4 revision=41" appVersion=2026.09.13.2
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

Worked root cause: **an old ReplicaSet pod survived a partial rollout with HIKARI_MAXIMUM_POOL_SIZE=4 instead of 24**.

```text
an old ReplicaSet pod survived a partial rollout with HIKARI_MAXIMUM_POOL_SIZE=4 instead of 24
  -> changes work, waiting, or ownership per request
  -> slow pod Hikari max=4 active=4 pending=37; healthy pods max=24 pending=0
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
- mark the old pod unready, drain it after capture, and complete the rollout.
- Preserve representative traces, metric queries, three thread dumps, and bounded JFR when safe.
- Reduce retry amplification and shed overload explicitly with 429/503 where contracts allow.
- Drain before termination and monitor in-flight work.
- State clearly that mitigation is not permanent proof.

## Permanent fix
- gate releases on rollout convergence and alert on image/config-hash heterogeneity.
- Add a regression test for the exact failure path, payload, concurrency, and cancellation/error behavior.
- Size bounded queues/pools/deadlines from measured service time and dependency capacity.
- Never use blind CPU, memory, thread, pool, queue, heap, or timeout increases as the fix.

# Verification

Exact acceptance target: **all six pods use max=24 and same revision/hash; pending=0; per-pod p99<230ms**.

| Signal | Before | After requirement |
|---|---|---|
| Duration | one pod p95=2.9s, p99=5.1s; peers p95<175ms | Normal p50/p95/p99 and max below deadline |
| Errors | slow pod 504=11%, fleet 504=1.9% | 5xx/timeouts/rejections<1% and business success restored |
| Resource | slow pod CPU=31%, heap=49%, GC max=22ms; peers similar | Leading resource/queue at measured baseline |
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

When one instance is slow, I preserve pod labels and compare matched requests before blaming shared dependencies. This fleet has six pods at about 200 requests/s each: one pod reaches p95=2.9s and p99=5.1s while five peers remain below p95=175ms. The slow trace spends 2.71s acquiring Hikari and only 61ms in SQL. Runtime configuration then shows max=4, active=4, pending=37 on the old pod versus max=24 and pending=0 on its peers. That proves an old ReplicaSet survived a partial rollout with stale pool configuration. I mitigate by marking it unready, draining it after evidence capture, and completing the rollout. I prevent recurrence by gating on rollout convergence and config-hash homogeneity, then verify all six pods have matching revisions, zero pending connections, and p99 below 230ms.

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

## Q1. Why group by instance?

**Answer:** Fleet averages hide a bad pod and make failures appear intermittent.

## Q2. How distinguish pod from node?

**Answer:** Compare peers and move only after capture; whether the symptom follows node or app revision narrows ownership.

## Q3. Can direct probing be risky?

**Answer:** Yes. Use authorized safe endpoints and preserve security and representative headers.

## Q4. Why not delete immediately?

**Answer:** Capture evidence, remove readiness, and drain to preserve causality and in-flight work.

## Q5. What proves drift?

**Answer:** Effective runtime values, config hash, image digest, ReplicaSet revision, and matched trace differences.

## Q6. Could equal-traffic pods differ?

**Answer:** Yes: revision/config, cache, throttling, node/network, GC, JIT, or local saturation.

## Q7. How prevent recurrence?

**Answer:** Gate rollout convergence and alert when live pods disagree on image, revision, config, or critical pool limits.
