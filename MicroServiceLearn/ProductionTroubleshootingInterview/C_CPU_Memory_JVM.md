# Production Troubleshooting Study Chapter: CPU, Memory, JVM, and Threads

## Purpose

This chapter explains how a service uses CPU, memory, garbage collection, threads, pools, and container resources, then turns those concepts into production-safe investigation workflows. It treats JVM evidence as one part of the operating system and container picture rather than assuming every Java incident is a heap or GC problem.

The examples emphasize Java and Spring Boot, but the resource and concurrency reasoning applies to most server runtimes.

> Safety: Replace placeholders such as `<namespace>`, `<pod>`, `<container>`, `<pid>`, and `<secure-path>` with approved values. Diagnostic files can contain source names, object data, URLs, headers, and customer information. Store them in access-controlled locations, collect them for the shortest useful duration, and follow retention policy. Prefer read-only evidence before changing a process.

## Learning goals

After studying this chapter, you should be able to:

1. Distinguish CPU utilization, CPU saturation, per-core hotspots, and container CPU throttling.
2. Explain Java heap, non-heap, native memory, RSS, working set, virtual memory, and cgroup limits.
3. Recognize important `OutOfMemoryError` variants and distinguish them from a kernel/container OOM kill.
4. Interpret allocation rate, live set, GC frequency, pause time, promotion, and concurrent-GC CPU.
5. Capture and compare thread dumps, identify deadlocks and persistent blocking, and avoid misreading normal waiting threads.
6. Diagnose worker, database, and HTTP connection-pool exhaustion from queue and wait evidence.
7. Explain periodic Kubernetes/container restarts using exit reason, exit code, events, probes, and application evidence.
8. Choose immediate mitigation without blindly restarting or increasing heap, threads, pools, or timeouts.
9. Convert a production finding into a root-cause fix, capacity model, alert, and test.

---

# 1. Foundational mental model

## 1.1 CPU: work, waiting, saturation, and throttling

CPU metrics answer different questions:

- **CPU time** is time scheduled on a processor.
- **CPU utilization** is CPU time used divided by available CPU time over a window.
- **CPU saturation** means runnable work waits because available CPU cannot schedule it promptly.
- **Run queue** approximates runnable tasks waiting for CPU.
- **CPU throttling** is forced waiting after a container consumes its cgroup quota during a quota period.

A container limited to one CPU can use its quota and be throttled while the node is only 30 percent busy. A dashboard that shows host CPU or a long-window average can therefore look normal while request latency suffers.

CPU percentages need a denominator:

```text
100% of one logical CPU
100% of a four-CPU container
100% of the host
```

Tools disagree about presentation. A process shown as 300 percent by one tool may be using three cores, while another chart normalizes that to 75 percent of a four-core limit.

### Common CPU patterns

| Pattern | Likely meaning |
|---|---|
| CPU and throughput rise together, latency stable | Useful scaling within capacity |
| CPU near capacity, run queue and latency rise | CPU saturation |
| Host CPU low, container throttling high | Cgroup quota limits the container |
| One core hot, total CPU moderate | Serial hotspot, single event loop, lock owner, or nonparallel phase |
| CPU high, throughput flat/falling | Waste, contention, GC, retries, spin, or inefficient work |
| CPU low, service unresponsive | Waiting, deadlock, pool exhaustion, network/dependency, or stopped request intake |

## 1.2 Memory: heap is not process memory

A Java process consumes several kinds of memory:

```text
Process resident memory (RSS / working set)
|
+-- Java heap
|   +-- young-generation objects
|   `-- old-generation/live retained objects
|
+-- JVM non-heap
|   +-- Metaspace and compressed class space
|   +-- JIT code cache
|   `-- JVM internal structures
|
+-- Native/off-heap
|   +-- direct byte buffers
|   +-- thread stacks
|   +-- GC data structures
|   +-- JNI/native libraries
|   +-- libc allocator arenas/fragmentation
|   +-- memory-mapped files
|   `-- TLS, compression, database/client buffers
|
`-- Shared/file-backed pages and other mapped regions
```

Definitions:

- **Heap used** is memory occupied inside the Java object heap.
- **Heap committed** is virtual memory currently committed for heap use.
- **Heap max** is the configured/effective upper bound.
- **Live set** is memory still reachable after a collection capable of reclaiming old objects.
- **RSS** is resident physical pages attributed to the process. Exact accounting varies by OS/tool.
- **Working set** is a container/platform view of actively resident memory; its treatment of inactive file cache varies.
- **Virtual memory size** is address space reserved or mapped, not the same as physical memory used.
- **Cgroup memory usage/limit** governs container enforcement and includes more than Java heap.

Therefore:

```text
-Xmx < container memory limit
```

is necessary but not sufficient. Native memory, threads, direct buffers, code cache, metaspace, and runtime overhead also need headroom.

## 1.3 Allocation, retention, leak, and pressure

- **Allocation rate** is bytes of new objects created per unit time.
- **Churn** means objects are allocated quickly and die quickly.
- **Retention** means objects remain reachable and survive collection.
- **Memory leak** is unintended retention or unreleased native/resource ownership.
- **Memory pressure** means demand approaches available memory, causing frequent collection, allocation stalls, paging, or OOM.

A rising heap before GC is normal. A useful signal is the post-old/full-GC floor:

```text
healthy sawtooth:
used rises -> GC -> used returns near a stable floor

possible retention:
used rises -> GC -> used falls, but the post-GC floor trends upward
```

This pattern is only a clue. A heap dump and ownership path, or equivalent allocation/retention evidence, is needed to prove what is retaining memory.

## 1.4 Garbage collection is a consequence, not automatically the cause

Garbage collection reclaims unreachable heap objects. Important dimensions are:

- Allocation rate.
- Young/minor collection rate and duration.
- Old/mixed/full collection rate and duration.
- Pause-time distribution.
- Concurrent GC CPU.
- Promotion rate and premature promotion.
- Heap occupancy before and after collection.
- Live-set size after old-generation collection.
- Allocation stalls or evacuation/to-space failures.

High GC count can be harmless if pauses and CPU cost are small and throughput is healthy. Low GC count can be dangerous if one pause is extremely long. Diagnose user impact and the allocation/live-set mechanism, not a collector label alone.

## 1.5 Threads and states

A thread dump is a snapshot of thread stacks and states:

| Java state | Meaning | Important caution |
|---|---|---|
| `NEW` | Not started | Rare in normal diagnostics |
| `RUNNABLE` | Eligible to run or executing native work | May be on CPU, in syscall, or socket I/O; not proof of CPU use |
| `BLOCKED` | Waiting to enter a Java monitor held by another thread | Identify the monitor and owner |
| `WAITING` | Waiting indefinitely for a signal/condition/join/park | Often normal for idle pool threads |
| `TIMED_WAITING` | Waiting with a timeout | Often normal for sleep, poll, or timed park |
| `TERMINATED` | Finished | Usually absent from live dumps |

One dump shows one moment. Capture at least three dumps several seconds apart during the symptom. Repeated stacks reveal persistent waits or loops.

### Deadlock, contention, starvation, and exhaustion

- **Deadlock:** a cycle of owners and waiters means none can progress.
- **Lock contention:** threads eventually progress but queue behind a lock.
- **Starvation:** a task rarely obtains CPU, lock, thread, or another resource.
- **Pool exhaustion:** all finite resources are leased/busy and new work waits or rejects.
- **Thread leak:** threads accumulate because lifecycle cleanup is missing.
- **Thread explosion:** excessive creation increases stacks, scheduling, and context switching.

## 1.6 Pools are queues around finite resources

Common pools include server workers, executors, DB connections, HTTP connections, and message-consumer workers.

For any pool, ask:

```text
capacity:
active/leased:
idle/available:
queued/pending:
wait duration:
rejections/timeouts:
resource hold duration:
arrival rate:
completion rate:
```

If all DB connections are active, the cause may be slow SQL, locks, long transactions, a leak, or database capacity. Increasing the pool can make the database slower and increase failure impact.

## 1.7 Containers and Kubernetes add another control plane

A process can stop because:

- It exited normally or with an application error.
- The JVM terminated after an uncaught fatal error or OOM.
- The kernel/cgroup killed it for memory.
- Kubernetes restarted it after liveness/startup probe failures.
- A controller replaced it during rollout, scaling, or node maintenance.
- The node evicted it for memory/disk/PID pressure.
- An operator or automation deleted it.
- A preemptible/spot node disappeared.

`restartCount` alone does not identify the cause. Read the previous container's terminated state, exit code, reason, events, previous logs, node events, and JVM artifacts.

## 1.8 Glossary

| Term | Meaning |
|---|---|
| CPU-bound | Throughput primarily limited by CPU work |
| I/O-bound | Work primarily waits on disk, network, database, or external I/O |
| Context switch | CPU changes which thread/process runs |
| Throttling | Cgroup pauses CPU execution after quota is consumed |
| Safepoint | JVM state where selected VM operations require threads to reach a safe state |
| Stop-the-world | Application threads are paused for a JVM operation |
| Young/old generation | Logical heap areas organized around object lifetime |
| Promotion | Moving surviving objects toward/into old generation |
| Metaspace | Native memory for class metadata |
| Direct buffer | Off-heap buffer often used for network/file I/O |
| Native Memory Tracking (NMT) | JVM accounting of many native allocation categories; must be enabled at startup |
| Dominator | An object that controls reachability to other objects in a heap graph |
| Retained size | Memory that would become collectible if an object were removed |
| RSS | Resident physical memory attributed to a process |
| OOMKilled | Container termination due to kernel/cgroup memory enforcement, not necessarily a Java exception |
| CrashLoopBackOff | Kubernetes delays repeated restarts; it is a state, not the original cause |
| Liveness probe | Decides whether Kubernetes should restart a container |
| Readiness probe | Decides whether a pod should receive traffic |

---

# 2. Essential metrics and evidence

## 2.1 CPU metrics

| Metric | Why it matters | How to interpret it |
|---|---|---|
| Process/container CPU usage | CPU time consumed | Normalize against cores/limit and compare with throughput |
| Per-thread/per-core CPU | Finds concentrated work | One hot thread/core can hide in fleet averages |
| Load average/run queue | Runnable or uninterruptible work pressure | Compare with available cores; investigate I/O state too |
| Cgroup throttled periods/seconds | Forced CPU wait | Rising ratio with latency indicates quota pressure |
| Context switches | Scheduling overhead/blocking changes | High values need workload baseline; not a cause by themselves |
| CPU per request | Service demand | Rising value means each request became more expensive |
| GC CPU | Runtime collection cost | Separate from application execution |

## 2.2 Memory and GC metrics

| Metric | Why it matters | Diagnostic pattern |
|---|---|---|
| Heap used/committed/max by pool | Heap occupancy | Used touching max repeatedly indicates pressure |
| Post-GC old occupancy/live set | Retained heap trend | Rising floor suggests retention or growing legitimate state |
| Allocation rate | Object churn | High rate can drive frequent young GC without a leak |
| GC pause count/duration | Stop-the-world impact | Correlate exact pauses with latency |
| Time/CPU in GC | Throughput cost | High cost with little reclamation is dangerous |
| Promotion rate | Survivor pressure | High promotion can fill old generation |
| Metaspace/class count | Class metadata growth | Rising class loaders/classes can indicate loader leak |
| Direct buffer count/bytes | Off-heap use | Can OOM while heap is healthy |
| Thread count | Stack/native memory and scheduling | Trend and states matter more than count alone |
| RSS/working set | Total resident footprint | Compare with cgroup limit and heap |
| Cgroup memory events | Limit enforcement | `oom`, `oom_kill`, and pressure are decisive container evidence |
| Page faults/swap | Memory pressure | Major faults/swap can severely increase latency |

## 2.3 Thread and pool metrics

| Metric                                             | Meaning |
|----------------------------------------------------|---|
| Live, daemon, peak thread count                    | Thread footprint and trend |
| Threads by state                                   | Broad symptom; stacks provide cause |
| Deadlock count/detection                           | Cyclic lock failure |
| Server busy/max/queued/rejected                    | Request execution saturation |
| Executor active/core/max/queue/rejected            | Application pool saturation |
| Hikari active/idle/max/pending/acquisition/timeout | DB connection availability and wait |
| HTTP client leased/available/pending               | Downstream connection availability |
| Task age/queue oldest age                          | User impact hidden by count alone |

## 2.4 Kubernetes and process-lifecycle metrics

- Pod restart count and last termination reason/exit code.
- OOM kill and cgroup memory events.
- Liveness, readiness, and startup probe failures.
- Evictions and node memory/disk/PID pressure.
- Desired versus available replicas.
- Container CPU request, CPU limit, memory request, and memory limit.
- Deployment/rollout and node-drain events.
- Time from container start to ready.

## 2.5 Important OOM types

| Evidence/message | What exhausted | Typical mechanisms | First evidence |
|---|---|---|---|
| `OutOfMemoryError: Java heap space` | Java heap cannot satisfy allocation | Retention/leak, legitimate live set too large, huge request, insufficient heap, excessive concurrency | GC log, heap-after-GC trend, heap dump/class histogram |
| `OutOfMemoryError: GC overhead limit exceeded` | JVM spends nearly all time collecting with little recovery | Heap nearly full with ineffective GC, often retention or undersized heap for live set | GC CPU, reclaimed bytes, heap dump |
| `OutOfMemoryError: Metaspace` | Native class-metadata space | Dynamic class generation, class-loader leak, too-low max metaspace | Class count/loaders, NMT, heap dump loader analysis |
| `OutOfMemoryError: Compressed class space` | Space for compressed class pointers | Very high loaded-class/class-loader count or restrictive setting | Class-loading metrics, NMT |
| `OutOfMemoryError: Direct buffer memory` | Direct/off-heap buffer allowance or native availability | Buffers retained, high I/O concurrency, pool/cache issue, low direct-memory cap | Buffer-pool metrics, NMT, allocation ownership |
| `OutOfMemoryError: unable to create native thread` | OS/cgroup PID/thread capacity or native memory for stack | Thread leak/explosion, PID limit, low native headroom | Thread count/trend, `pids` limit, RSS, thread dump |
| `OutOfMemoryError: Requested array size exceeds VM limit` | One requested array is impossible/too large | Integer/length bug, unbounded payload/result, attempted giant allocation | Exception stack and input size |
| `OutOfMemoryError: Map failed` or native allocation failure | Native address space/commit/map resource | Large mappings, native fragmentation, OS commit/cgroup pressure | Error log, NMT, maps, OS memory |
| Exit code 137 / `OOMKilled` | Kernel/cgroup killed the process | Total container memory exceeded limit; heap may be normal | Pod terminated state, cgroup events, RSS components |
| JVM fatal native-memory error | JVM/native allocator could not reserve/commit | Host/cgroup pressure, native leak, address-space/commit limit | `hs_err_pid*.log`, NMT, OS/cgroup evidence |

An `OOMKilled` container may leave no Java `OutOfMemoryError` because the kernel terminates the process externally. Conversely, a Java heap OOM can be caught or can terminate with an exit code other than 137. Use actual termination evidence.

## 2.6 Production-safe evidence ladder

Use the least invasive evidence that answers the question:

1. Existing metrics, traces, logs, deployment events, and cgroup/container status.
2. Thread count/states, class loading, buffer-pool, GC, and pool metrics.
3. Repeated thread dumps.
4. Short JFR or sampling profiler capture.
5. Class histograms or NMT summaries.
6. Heap dump/core/native profiler only with capacity, security, and impact approval.

Useful commands:

```bash
# Kubernetes current status
kubectl -n <namespace> get pod <pod> -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> -c <container> --previous --since=30m
kubectl -n <namespace> top pod <pod> --containers

# Java inventory and low-impact summaries
jcmd <pid> VM.version
jcmd <pid> VM.flags
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print -l
jcmd <pid> VM.native_memory summary

# Linux process and thread views
ps -o pid,ppid,nlwp,rss,vsz,etime,cmd -p <pid>
top -H -p <pid>
pidstat -p <pid> -t 1 10
```

`VM.native_memory` works only when NMT was enabled at JVM startup, for example with `-XX:NativeMemoryTracking=summary`. NMT has overhead, does not account for every native allocation, and should be enabled intentionally.

Heap dumps can pause the process, require substantial disk, and contain secrets and customer data. Configure an approved path and retention policy before relying on:

```text
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=<secure-path>
-XX:ErrorFile=<secure-path>/hs_err_pid%p.log
```

---

# 3. Generic CPU, memory, JVM, and thread investigation workflow

## Step 1 - Define impact and time

Record the first bad time, user-facing symptom, affected endpoints, traffic, instance/pod/node, version, restart history, resource requests/limits, and recent changes. Keep CPU, memory, GC, latency, errors, and throughput on the same timeline.

## Step 2 - Separate fleet, instance, container, and process views

Determine whether the anomaly affects:

- Every instance or one instance.
- One node/zone or the fleet.
- One revision.
- The Java process, sidecar, or another container.
- Host resources or only cgroup-limited resources.

Fleet averages can hide one saturated or leaking pod.

## Step 3 - Classify CPU, memory, or waiting

```text
High CPU?
  -> compare throughput, per-thread CPU, run queue, throttling, GC CPU

Rising memory?
  -> compare heap-after-GC, RSS, native categories, direct buffers, threads

Unresponsive with normal CPU/memory?
  -> inspect queues, pools, locks, deadlocks, I/O waits, probes, dependencies
```

These categories can coexist. A leak can cause GC CPU, which causes queueing, which causes timeouts and retries.

## Step 4 - Look for the first causal change

Compare the start of the user symptom with:

- Request and retry rate.
- CPU throttling and run queue.
- Allocation rate and GC pause/CPU.
- Heap-after-GC and RSS.
- Thread count/states.
- Pool active/pending/wait.
- Queue depth and task age.
- Dependency latency.
- Deployment/config/resource/node events.

The first change is a lead, not proof.

## Step 5 - Capture targeted runtime evidence

- High CPU: per-thread CPU plus JFR/sampling profile and repeated dumps.
- Rising heap: GC trend, class histograms, then heap dump if justified.
- Rising RSS with stable heap: NMT, direct buffers, threads/stacks, native libraries, mappings.
- Unresponsive: repeated thread dumps, deadlock check, pool/queue state, network/dependency evidence.
- Restarts: previous termination state/logs, events, probes, `hs_err`/heap dump.

## Step 6 - Mitigate without erasing the diagnosis

Possible bounded actions include pausing a rollout, shifting traffic from a proven bad instance, reducing retry/fan-out load, shedding optional work, disabling a leaking feature, or scaling a proven horizontally scalable bottleneck. Capture evidence before controlled replacement when safe.

Do not blindly:

- Restart processes.
- Increase heap.
- Add threads.
- Add DB/HTTP connections.
- Increase timeouts.

Each can delay or amplify failure and obscure the root cause.

## Step 7 - Prove recovery and steady state

Verify user-facing latency/errors and the causal resource signal. For memory/backlog issues, run long enough to prove a stable post-GC floor or queue equilibrium. For CPU, verify CPU per request and throughput, not just lower utilization after traffic fell.

---

# 4. Original interview questions

## 1. A production service suddenly has 95% CPU utilization. How would you troubleshoot it?

### What the symptom means and does not prove

It means a measured CPU scope used about 95 percent of its denominator during a window. It does not prove CPU saturation, application fault, or that 5 percent headroom remains. Determine whether the metric is host, container, process, one core, or quota-relative; whether work waits in a run queue; and whether the CPU is doing useful throughput.

### Where it can happen

In application methods, serialization/compression/encryption, regex/parsing, logging, GC, JIT compilation, lock spinning, retry loops, monitoring agents, TLS, kernel/network handling, a sidecar, or another process on the node.

### Detailed causes and mechanisms

| Cause | Mechanism |
|---|---|
| Traffic or heavier requests | More CPU service demand per second |
| Inefficient/infinite loop | A thread repeatedly executes without useful completion |
| Algorithm/data growth | Complexity rises with larger collections or payloads |
| Serialization/compression/crypto | Per-byte CPU rises with payload or settings |
| High allocation and GC | CPU is spent allocating, scanning, copying, and reclaiming |
| Retry/error loop | Failed work repeats rapidly and may log excessively |
| Lock spin/contention | Threads consume CPU attempting progress or context-switch heavily |
| JIT/class loading/startup | New code paths compile/load after rollout or scale-up |
| Observability agent/logging | Instrumentation or formatting consumes CPU |
| CPU throttling | Quota causes latency; usage may appear at the limit |
| Sidecar/kernel work | Proxy encryption/network processing is outside Java process |

### Ordered investigation

1. Define CPU denominator, window, affected pod/node/revision, and user impact.
2. Compare CPU with RPS, completions, latency, errors, request mix, and payload size.
3. Check CPU limit, throttled periods/seconds, node contention, and run queue.
4. Separate Java process, sidecar, and other process CPU.
5. Identify hot OS threads with `top -H`/`pidstat`, then map to Java `nid` values.
6. Capture a short JFR or sampling profile during the incident.
7. Compare multiple thread dumps for repeated runnable stacks, loops, or contention.
8. Separate application CPU from GC, JIT, logging, and agent CPU.
9. Correlate with releases, flags, traffic, data shape, scheduled jobs, and dependency failures.

### Commands, tools, metrics, and interpretation

```bash
top -H -p <pid>
pidstat -p <pid> -t 1 10

# Convert a decimal native thread ID to hex for comparison with Java nid
printf '%x\n' <decimal-thread-id>

jcmd <pid> Thread.print -l > <secure-path>/threads.txt
jcmd <pid> JFR.start name=cpu settings=profile duration=60s \
  filename=<secure-path>/cpu.jfr
```

- One hot thread whose hex ID matches a repeated `RUNNABLE` stack: inspect that code path.
- Many hot request threads: workload/service demand or parallel hot path.
- High GC CPU/allocation events: investigate allocation and live set before collector tuning.
- High throttled time with a CPU limit: quota pressure, even if node has idle CPU.
- CPU high but completions low: waste, contention, GC, retry, or pathological work.

### Immediate mitigation

Pause/roll back a proven regression, disable a CPU-expensive optional feature, stop retry storms, shed low-priority load, isolate a hot tenant/key, or scale the CPU-bound stateless tier if downstream systems have headroom.

### Root-cause fixes

Optimize the profiled hotspot and algorithm, bound payload/work, eliminate loops/retries, reduce allocation, cache safely, move suitable batch work, correct CPU requests/limits, and establish concurrency/backpressure.

### Prevention and alerting

Alert on CPU saturation with latency/run-queue/throttling context, CPU per request, anomalous retry/log rate, and SLO burn. Keep low-overhead continuous profiles where approved and performance-test worst-case data.

### Common mistakes

- Treating 95 percent as the cause without checking saturation or useful throughput.
- Reading a single long-window fleet average.
- Guessing from thread names instead of profiling.
- Restarting before capturing evidence.
- Adding CPU without fixing an infinite loop or retry storm.

### Concise interview-ready answer

> I first clarify whether 95 percent is host, process, container, or quota-relative and whether latency, throughput, run queue, or throttling show saturation. I correlate CPU with traffic, request mix, errors, GC, and the deployment timeline, then identify hot native threads and use a short JFR or sampling profile plus repeated thread dumps to locate application, GC, JIT, logging, or sidecar work. I mitigate the proven source and verify CPU per request and SLO recovery rather than blindly restarting.

---

## 2. CPU is continuously high even though traffic has not increased. What could be happening?

### What the symptom means and does not prove

Stable request count with high CPU means CPU demand per request increased, non-request work increased, completions fell while attempts/retries rose, or the available CPU denominator changed. It does not prove an infinite loop.

### Where it can happen

In changed request data, background jobs, message consumers, cache behavior, retry/error loops, GC, JIT, agents, logging, security, sidecars, node contention, or a reduced container CPU limit.

### Detailed causes and mechanisms

- Requests are the same count but larger or routed through a costlier feature path.
- Cache hit ratio falls, adding deserialization, queries, or computation.
- A scheduled batch, consumer backlog, compaction, or maintenance job runs.
- A bug spins after an error, busy-polls an empty queue, or retries without backoff.
- Dependency failures trigger more attempts and exception/log formatting.
- Allocation rate or live set rises, increasing GC CPU.
- A lock-free algorithm spins under contention.
- Dynamic class generation/JIT or a monitoring/security agent adds work.
- CPU limit is reduced, so the same CPU seconds become a higher percentage and more throttling.
- A sidecar performs more TLS, retries, or telemetry.

### Ordered investigation

1. Compare offered requests, completed requests, outbound attempts, message rate, and background-task activity.
2. Normalize CPU by completed transaction and payload/data-size class.
3. Compare current versus previous CPU limit, throttling, node, sidecars, and version.
4. Split CPU by process and thread; capture a profile.
5. Compare GC CPU/allocation, cache hit ratio, exception/log rate, and retry ratio.
6. Inspect schedulers, consumers, maintenance tasks, and stuck loops.
7. Correlate with config/flag/data/dependency/agent changes.
8. Reproduce the hot stack or workload and fix the mechanism.

### Commands, tools, metrics, and interpretation

- `pidstat -p <pid> -t 1 10` identifies persistent hot threads.
- JFR execution samples identify methods consuming sampled CPU.
- Request count flat but outbound call count rises indicates fan-out/retry amplification.
- Request count flat but completed count falls indicates accumulation; "traffic" may count only accepted/completed work.
- Higher GC CPU with higher allocation but stable live set indicates churn; rising live set suggests retention.
- A manifest diff showing a lower CPU limit explains a changed denominator but still requires capacity correction.

### Immediate mitigation

Stop or throttle the proven background task, disable the costly feature, suppress retry/error loops, restore a mistakenly reduced CPU allocation through change control, or shed optional work.

### Root-cause fixes

Fix spin/poll/retry logic, use blocking/backoff appropriately, optimize new data paths, restore cache effectiveness, isolate background pools and quotas, reduce allocation/logging, and capacity-plan sidecars and scheduled work.

### Prevention and alerting

Monitor CPU per completed operation, outbound attempts per inbound request, background job CPU, cache hit ratio, exception/log volume, GC CPU, and resource-spec changes. Alert when CPU rises without throughput.

### Common mistakes

- Equating inbound RPS with all workload.
- Ignoring message consumers and scheduled jobs.
- Assuming flat request count means flat request cost.
- Raising the CPU limit before finding runaway work.
- Tuning GC without allocation evidence.

### Concise interview-ready answer

> With flat traffic, I would ask whether work per request or non-request work changed. I would compare completed throughput, outbound attempts, payload/data shape, cache hit rate, schedulers/consumers, exception and retry rate, GC CPU, sidecars, and CPU limits. A per-thread profile identifies loops or new hotspots. I would stop the proven runaway or optional work, then fix the cost, retry, cache, background isolation, or resource-spec cause.

---

## 3. Memory usage keeps increasing and eventually the service crashes with OutOfMemoryError. How would you investigate?

### What the symptom means and does not prove

Some allocation failed and the JVM reported an OOM variant. Rising "memory" does not prove a Java heap leak. First capture the exact OOM message and determine whether the graph is heap used, heap after GC, RSS, working set, or cgroup usage. Also distinguish a Java OOM from `OOMKilled`.

### Where it can happen

Java heap, metaspace/class space, direct buffers, native allocations, thread stacks/PID limits, mapped memory, OS commit, or total container memory.

### Detailed causes and mechanisms

- A map/cache/listener/session retains objects after their useful lifetime.
- Static collections, `ThreadLocal`, callbacks, class loaders, or pending futures retain object graphs.
- Legitimate concurrency/live data exceeds the heap design.
- One unbounded request/result attempts a huge array or graph.
- High allocation causes GC overhead with too little reclamation.
- Dynamic class loaders/proxies/classes fill metaspace.
- Direct buffers are retained or concurrency exceeds off-heap capacity.
- Threads leak; each consumes native stack and PID resources.
- JNI/native library or allocator fragmentation raises RSS with stable heap.
- Container limit leaves insufficient non-heap headroom, producing OOM kill.

### Ordered investigation

1. Record exact OOM text, stack, timestamp, exit code, termination reason, and affected memory metric.
2. Preserve GC logs, heap dump, `hs_err` file, previous container logs, and cgroup events.
3. Plot heap used/max, post-GC old occupancy, allocation, GC CPU/pause, RSS, direct buffers, metaspace, class count, and threads.
4. If heap retention is indicated, compare class histograms and analyze a secure heap dump by dominators, retained size, and paths to GC roots.
5. If metaspace rises, inspect class count and class-loader ownership.
6. If direct memory rises, inspect buffer-pool metrics, I/O libraries, pooling, and lifecycle.
7. If thread count rises, inspect creators, stacks, executor lifecycle, PID limits, and native headroom.
8. If heap is stable but RSS rises, use NMT summaries/diffs and OS mapping/native evidence.
9. Reproduce with a soak test and verify post-GC/RSS steady state.

### Commands, tools, metrics, and interpretation

```bash
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram > <secure-path>/classes-1.txt
jcmd <pid> VM.native_memory summary
jcmd <pid> Thread.print -l > <secure-path>/threads.txt
```

- Rising post-old-GC floor plus a growing class/dominator supports heap retention.
- High allocation with stable post-GC floor supports churn, not a leak.
- Class count and metaspace rise with repeated class loaders: class-loader leak hypothesis.
- Direct-buffer bytes rise while heap stays stable: off-heap ownership issue.
- RSS approaches cgroup limit while heap remains below max: total-memory budgeting/native issue.
- `OOMKilled`/137 with no Java OOM: kernel/cgroup kill; inspect cgroup and RSS components.

### Immediate mitigation

Disable or limit the leaking/unbounded feature, cap request/concurrency/queue size, reduce retry-driven in-flight work, route away from a proven degraded instance, and perform a controlled rolling replacement only after preserving evidence. Temporarily increasing memory is justified only as a measured containment with verified node/cgroup headroom and an explicit expiration.

### Root-cause fixes

Remove unintended references, bound caches/queues/results, close native resources, clean up `ThreadLocal` and listeners, correct executor/class-loader lifecycle, stream/paginate data, reduce concurrency memory, and set JVM/container memory budgets across heap and native components.

### Prevention and alerting

Enable secure OOM artifacts, GC logs, heap-after-GC alerts, RSS-to-limit alerts, direct-buffer/metaspace/thread trends, and long soak tests. Alert early enough to capture evidence, not only at the limit.

### Common mistakes

- Increasing `-Xmx` without knowing the OOM type.
- Calling RSS growth a heap leak.
- Taking unplanned heap dumps on a nearly full filesystem.
- Comparing shallow object size instead of retained ownership.
- Scheduling restarts as the permanent fix.

### Concise interview-ready answer

> I start with the exact OOM variant and distinguish Java heap OOM from metaspace, direct memory, native thread, huge allocation, native failure, or container `OOMKilled`. I correlate post-GC heap, allocation and GC cost with RSS, direct buffers, metaspace/classes, and thread count. For heap retention I use histograms and a secured heap dump with dominator and GC-root analysis; for stable heap but rising RSS I use NMT and native evidence. I contain the leaking path after preserving evidence, then fix ownership and prove a steady state in a soak test.

---

## 4. A service restarts periodically without any code deployment. What would you check?

### What the symptom means and does not prove

The workload is being terminated and recreated, or the process exits and the runtime restarts it. "No deployment" rules out only one cause. It does not prove OOM, a liveness failure, or a platform fault.

### Where it can happen

Application/JVM exit, OOM, fatal native crash, container runtime, Kubernetes probes/controller, node eviction/reboot/drain, autoscaling, scheduled automation, certificate/config reload behavior, or infrastructure preemption.

### Detailed causes and mechanisms

- Java OOM or uncaught fatal exception exits the process.
- Kernel/cgroup OOM kills the container when total memory crosses its limit.
- Liveness or startup probe repeatedly fails due to pause, dependency-coupled health, wrong threshold, or blocked server.
- JVM fatal error, `SIGSEGV`, native library failure, or forced signal terminates it.
- Node memory/disk/PID pressure evicts the pod.
- Node reboot, drain, upgrade, spot/preemptible loss, or runtime failure replaces it.
- A cron/automation policy deliberately recycles workloads.
- File descriptor/PID exhaustion makes probes or the application fail.
- A scheduled traffic/job/GC pattern causes periodic unresponsiveness.
- Controller scaling can create/delete pods without incrementing the same pod's restart count.

### Ordered investigation

1. Determine whether the same pod's container restarted or the pod was replaced.
2. Read last terminated reason, exit code, signal, start/finish timestamps, and restart count.
3. Read previous container logs and application shutdown/fatal/OOM records.
4. Inspect Kubernetes events, probe failures, eviction messages, and controller events.
5. Inspect node conditions, reboot/drain/runtime events, and resource pressure.
6. Correlate interval with memory/GC, CPU throttling, scheduled jobs, traffic, certificate refresh, and automation.
7. Search for heap dumps, `hs_err` files, and core artifacts in approved storage.
8. Validate liveness/readiness/startup probe purpose and dependency coupling.
9. Fix the confirmed termination source and observe beyond the prior period.

### Commands, tools, metrics, and interpretation

```bash
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" reason="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{" signal="}{.lastState.terminated.signal}{" started="}{.lastState.terminated.startedAt}{" finished="}{.lastState.terminated.finishedAt}{" restarts="}{.restartCount}{"\n"}{end}'
kubectl -n <namespace> logs <pod> -c <container> --previous --since=2h
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.metadata.creationTimestamp
```

- `OOMKilled`/137: inspect total cgroup memory, heap versus native, and node evidence.
- Probe failure followed by kill: determine why the probe failed; do not merely relax it.
- Exit code 0: application or lifecycle automation may be exiting intentionally.
- `Error` plus `hs_err_pid`: JVM/native crash.
- Pod UID changed but restart count is zero: replacement, not in-container restart.

### Immediate mitigation

Remove a proven bad node through approved operations, pause harmful automation, correct a clearly broken probe, reduce triggering load, or roll instances safely after evidence capture. Keep enough healthy replicas and avoid disabling liveness globally without a failure-containment plan.

### Root-cause fixes

Fix OOM/native crash/application exit, separate shallow liveness from dependency readiness, add startup allowance for slow boot, correct resource budgets, remove scheduled recycle policies, and make shutdown/grace periods and node disruption policies explicit.

### Prevention and alerting

Alert on restart and replacement rate, termination reason, probe failures, node pressure, RSS-to-limit, fatal error artifacts, and crash loops. Centralize previous logs and termination metadata.

### Common mistakes

- Assuming restart count captures pod replacements.
- Reading only current logs.
- Treating `CrashLoopBackOff` as the cause.
- Increasing probe timeout before understanding blocked work.
- Calling every exit 137 a Java heap OOM.

### Concise interview-ready answer

> I distinguish a container restart from pod replacement, then inspect the previous terminated state, exit code/signal, previous logs, events, probe failures, node pressure, and JVM artifacts. I correlate the period with heap/RSS, GC, CPU throttling, jobs, traffic, and automation. `OOMKilled`, probe kill, JVM crash, application exit, eviction, and node replacement need different fixes, so I use the actual lifecycle evidence rather than assuming a deployment or blind restart.

---

## 5. GC activity suddenly becomes very high. How would you investigate?

### What the symptom means and does not prove

The JVM is collecting more frequently, pausing longer, using more concurrent CPU, or some combination. It does not by itself prove a memory leak or that GC tuning is required. First identify collector, event type, allocation rate, reclaimed memory, live set, pause impact, and CPU cost.

### Where it can happen

Young-generation churn, old-generation pressure, humongous/large allocations, promotion, remembered-set/card processing, metadata/class unloading, direct effects of heap sizing, CPU throttling, and application allocation behavior.

### Detailed causes and mechanisms

- Higher request rate or payload size increases temporary allocation and young GC.
- New code creates excessive temporary objects, boxing, copies, strings, or buffers.
- Retained live set grows, leaving less free heap and causing frequent old collections.
- Bursty concurrency allocates faster than GC can reclaim.
- Large/humongous objects create fragmentation or special-region pressure.
- Too-small effective heap for the legitimate live set reduces breathing room.
- CPU throttling slows concurrent GC threads and application progress.
- Promotion/survivor pressure moves objects old too early.
- Explicit `System.gc()` or tooling requests full collections.
- Collector/JDK/flag change alters behavior.

### Ordered investigation

1. Confirm which GC metric rose: count, pause p95/p99, total pause, concurrent CPU, or full collections.
2. Correlate exact GC events with latency and throughput.
3. Identify JVM/JDK/collector and any recent flag/resource changes.
4. Compare allocation rate, request/payload mix, heap occupancy before/after GC, and promotion.
5. Examine post-old-GC live-set trend and reclamation efficiency.
6. Look for full-GC triggers, allocation stalls, humongous allocations, evacuation failures, or explicit GC in logs/JFR.
7. Check container CPU throttling and memory headroom.
8. Use JFR/allocation profiling to identify allocating code.
9. Fix allocation/retention/capacity first; tune collector only with measured goals and tests.

### Commands, tools, metrics, and interpretation

```bash
jcmd <pid> GC.heap_info
jcmd <pid> JFR.start name=gc settings=profile duration=120s \
  filename=<secure-path>/gc.jfr
```

- Frequent short young GCs, stable live set, low pause and CPU: possibly healthy throughput behavior.
- Allocation rate jumps with a deployment: profile allocation stacks.
- Post-old-GC occupancy rises: investigate retention.
- Full GC reclaims little and repeats: heap/live-set crisis.
- Pauses align with throttling: GC lacks CPU quota.
- Humongous allocation events align with a route: bound/stream large objects.

### Immediate mitigation

Reduce the allocation-heavy workload, disable the implicated feature, limit large requests/concurrency, stop retries, restore a proven resource regression, or shift traffic while preserving evidence. A larger heap may temporarily postpone OOM but is not a default mitigation and can violate the container budget.

### Root-cause fixes

Reduce allocation and copying, stream large data, bound concurrency/caches, remove retention, right-size heap plus native headroom, correct CPU quota, upgrade/fix runtime issues where proven, and tune collector only against pause/throughput objectives.

### Prevention and alerting

Monitor allocation, post-GC occupancy, pause histograms, full/mixed collection rate, GC CPU, promotion, humongous allocation, and RSS/cgroup headroom. Keep GC logs with rotation and run allocation/soak tests.

### Common mistakes

- Alerting only on GC count.
- Switching collectors before finding allocation or retention.
- Increasing heap without container headroom.
- Assuming every stop-the-world pause is GC; safepoints can have other causes.
- Reading a GC log without correlating user latency.

### Concise interview-ready answer

> I clarify whether "high GC" means more collections, longer pauses, more GC CPU, or full collections, then correlate events with user latency. I compare allocation rate, pre/post-GC occupancy, live-set trend, promotion, large allocations, collector/JDK flags, CPU throttling, and workload changes. JFR or allocation profiling identifies the allocating path. I fix churn, retention, large objects, concurrency, or resource budget before considering collector tuning.

---

## 6. One instance has much higher memory usage than other instances. What could be the reason?

### What the symptom means and does not prove

The instance's measured heap, RSS, or cgroup memory differs. It does not prove a leak until age, workload, role, cache, GC timing, and measurement are normalized. A recently collected instance and one just before collection can differ normally.

### Where it can happen

Instance-local heap/native state, sticky sessions, load balancing, cache ownership, partition assignment, message backlog, version/config, node/kernel accounting, traffic/data skew, or a leak triggered by a particular request.

### Detailed causes and mechanisms

- The instance is older and has accumulated retained state.
- Sticky traffic, hot tenant/key, or uneven balancing sends it more/larger work.
- It owns more cache partitions, broker partitions, or scheduled tasks.
- It runs a different version, feature flag, JVM option, or resource limit.
- One request built a large retained cache/result/direct buffer.
- A connection/thread/file/native leak occurs only on a triggered path.
- It has not recently completed the same GC phase as peers.
- Node/page-cache/shared-memory accounting differs.
- Readiness/routing makes peers idle while this instance is hot.

### Ordered investigation

1. Identify whether the difference is heap used, post-GC heap, RSS, or cgroup working set.
2. Compare process age, version, flags, limits, node, GC phase, and traffic.
3. Compare route/tenant/payload distribution, cache/partition ownership, jobs, and connection/thread counts.
4. Compare post-GC heap and class histograms, not arbitrary instantaneous heap.
5. Compare direct buffers, metaspace/classes, NMT, thread stacks, and mappings.
6. Check sticky sessions, load balancer weights, readiness, and hot partitions.
7. Preserve instance-specific evidence before draining/replacing it.
8. Reproduce the trigger or prove age-based divergence with a soak.

### Commands, tools, metrics, and interpretation

```bash
kubectl -n <namespace> get pods -l app=<service> -o wide
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram > <secure-path>/classes.txt
jcmd <pid> VM.native_memory summary
```

- Higher traffic plus proportional memory/in-flight requests may be load skew.
- Higher post-GC heap at equal age/workload points to retained heap.
- Equal heap but higher RSS points to native/direct/thread/mapping differences.
- More assigned partitions/cache entries may be legitimate but still need capacity bounds.
- Different image/config invalidates an apples-to-apples comparison.

### Immediate mitigation

Correct a proven load imbalance, move a hot partition through an approved process, disable the triggering feature, or drain the affected instance after capturing evidence. Ensure remaining instances can absorb traffic.

### Root-cause fixes

Fix the leak, bound instance-local state, rebalance partitions/traffic, remove sticky sessions where inappropriate, make config/version uniform, and capacity-plan legitimate role differences.

### Prevention and alerting

Use per-instance outlier alerts for post-GC heap, RSS minus heap, direct buffers, threads, traffic, and age-normalized memory. Track version/config and partition ownership.

### Common mistakes

- Comparing one pod before GC with another after GC.
- Treating fleet average as proof all pods are safe.
- Deleting the pod before collecting evidence.
- Assuming equal replica names imply equal workload.
- Increasing every pod's heap for one outlier.

### Concise interview-ready answer

> I first identify which memory measurement differs and normalize by process age, GC phase, version, limits, node, and workload. I compare route/tenant traffic, sticky sessions, cache or partition ownership, jobs, threads, direct buffers, metaspace, post-GC heap, and NMT. If equal workloads show a rising post-GC class/dominator on one pod, I investigate retention; if heap is equal but RSS differs, I investigate native memory. I preserve evidence before draining the outlier.

---

## 7. The application is not responding, but CPU and memory appear normal. What would you investigate?

### What the symptom means and does not prove

Requests are not completing at the observed boundary, while coarse CPU and memory metrics are not high. It does not prove the process is healthy, idle, or deadlocked. It may not be receiving requests, may be waiting, or may be unable to accept new work.

### Where it can happen

Gateway/routing/readiness, listen/accept backlog, server worker or event loop, deadlock/locks, DB/HTTP pools, database/downstream, DNS/network, file descriptors, disk/logging, safepoint, pause, or a dependency-coupled health check.

### Detailed causes and mechanisms

- Every request thread waits on a slow dependency or pool.
- Deadlock creates a wait cycle with almost no CPU.
- Lock contention or one stuck owner serializes work.
- Event-loop thread blocks on synchronous I/O.
- DB/HTTP pool has no available connection.
- File descriptor, socket, or PID exhaustion prevents accepts/connections.
- Accept queue/backlog or load balancer path fails.
- Long JVM safepoint/GC pause is missed by coarse sampling.
- Synchronous logging/disk I/O blocks threads.
- Readiness/liveness route shares exhausted resources.
- Network packets do not reach the process or responses do not return.

### Ordered investigation

1. Test the exact endpoint from the same path and separate DNS/connect/TLS from read timeout.
2. Confirm whether requests reach gateway, pod, listener, access log, and application handler.
3. Check readiness/endpoints/load balancer membership and listen/accept state.
4. Inspect in-flight requests, server queue, active/max workers, event-loop lag, and rejections.
5. Check DB/HTTP pool pending and dependency latency.
6. Capture three thread dumps and use built-in deadlock detection.
7. Inspect file descriptors, sockets, disk latency, network retransmits, and cgroup throttling.
8. Inspect GC/safepoint logs around the exact gap.
9. Mitigate based on the blocked layer and verify real business traffic.

### Commands, tools, metrics, and interpretation

```bash
jcmd <pid> Thread.print -l > <secure-path>/threads-1.txt
sleep 10
jcmd <pid> Thread.print -l > <secure-path>/threads-2.txt
sleep 10
jcmd <pid> Thread.print -l > <secure-path>/threads-3.txt

ss -lntp
ls /proc/<pid>/fd | wc -l
```

- No request at the pod: routing, readiness, gateway, network, or DNS.
- Request logged but no handler completion: process queue/lock/pool/dependency.
- Dumps report a Java deadlock: preserve owner/wait chain and fix lock ordering.
- Many threads in one socket read: downstream stall.
- Many threads waiting for Hikari connection: pool/dependency problem.
- Server busy=max, queue rising: worker exhaustion.
- Threads absent from CPU because all wait: normal CPU is expected.

### Immediate mitigation

Fail over or route away from the proven stuck instance, trip an existing circuit breaker for a failed dependency, stop retry load, shed optional requests, or perform controlled replacement after dumps and state are captured.

### Root-cause fixes

Fix lock ordering and cancellation, bound dependency calls, correct pool/resource lifecycle, isolate workloads, remove blocking from event loops, improve readiness semantics, raise OS limits only when usage/capacity evidence justifies it, and repair routing/network.

### Prevention and alerting

Use end-to-end synthetic checks, in-flight/queue/pool alerts, deadlock detection, event-loop lag, FD/PID headroom, dependency SLOs, and probe metrics. A CPU/memory-only health model is inadequate.

### Common mistakes

- Restarting immediately and losing the deadlock/wait evidence.
- Calling all `WAITING` threads deadlocked.
- Checking a shallow health endpoint only.
- Increasing worker threads when all wait on the same pool.
- Ignoring that the request may never reach the application.

### Concise interview-ready answer

> Normal CPU and memory suggest I should trace reachability and waiting. I verify whether the request reaches the listener and handler, then inspect readiness/routing, accept and worker queues, event-loop lag, DB/HTTP pool waits, dependency latency, file descriptors, network and disk. I take multiple thread dumps to find persistent blocked stacks or a deadlock. I mitigate the specific failed layer after preserving evidence and fix the lock, pool, dependency, event-loop, or routing cause rather than adding threads.

---

## 8. The application has many threads and becomes unresponsive. What could be happening?

### What the symptom means and does not prove

Thread count is high relative to baseline and the service is not making expected progress. It does not prove the thread count itself is the cause. Many threads may be a consequence of blocked I/O, leaked executors, unbounded creation, or a dump that includes legitimately idle threads.

### Where it can happen

Server worker pools, custom executors, schedulers, `CompletableFuture`, async clients, message consumers, DB drivers, HTTP clients, libraries that create per-request threads, and native/JVM service threads.

### Detailed causes and mechanisms

- Unbounded thread-per-request/task design creates threads faster than they finish.
- Executor/scheduler instances are repeatedly created and never shut down.
- Requests block on DB/downstream calls, so pool threads accumulate or remain busy.
- Deadlock or lock convoy stops completion.
- Cached thread pools expand under blocking work.
- Stuck tasks ignore cancellation and outlive requests.
- ThreadLocal/state increases per-thread memory and retention.
- Native thread stacks consume memory and hit PID/thread limits.
- Too many runnable threads cause context-switching and CPU cache loss.
- A common fork-join pool is blocked by synchronous work, starving dependent tasks.

### Ordered investigation

1. Plot thread count over time and compare with traffic, latency, pool queues, RSS, and deployments.
2. Break down thread names, states, and owning executors/libraries.
3. Capture three dumps and group identical stacks.
4. Run deadlock detection and identify monitor owners/waiters.
5. Check OS/cgroup PID limits, native memory, stack size, and context switches.
6. Check server/custom executor configured sizes, queue policy, rejection, and lifecycle.
7. Inspect DB/HTTP pool waits and downstream duration causing blocked threads.
8. Trace where threads are created and whether executors shut down.
9. Fix lifecycle/blocking/backpressure and verify thread count reaches a stable bound.

### Commands, tools, metrics, and interpretation

```bash
ps -L -p <pid> -o pid,tid,stat,pcpu,rss,comm
jcmd <pid> Thread.print -l > <secure-path>/threads.txt

# Linux cgroup v2 PID controls, when accessible
cat /sys/fs/cgroup/pids.current
cat /sys/fs/cgroup/pids.max
```

- Hundreds of identically named idle pool threads: inspect configured pool creation and count.
- Repeated stacks blocked on one monitor: contention/deadlock owner.
- Repeated socket-read stacks to one dependency: dependency/pool timeout/cancellation.
- Rising threads and RSS with `unable to create native thread`: leak/PID/native headroom.
- Many runnable threads plus high context switching: oversubscription.

### Immediate mitigation

Stop the task source or retry storm, isolate the failing dependency, shed new work, disable the leaking feature, or replace the instance after collecting dumps. Do not increase thread limits.

### Root-cause fixes

Use shared bounded executors, close them on lifecycle shutdown, avoid thread-per-task where unsuitable, bound queues and concurrency, propagate deadlines/cancellation, remove blocking from common event pools, fix locks, and set resource-aware thread budgets.

### Prevention and alerting

Alert on thread-count slope, executor active/queue/rejection, state distribution, PID headroom, native memory, context switches, and stuck-task age. Review every custom executor's owner and shutdown policy.

### Common mistakes

- Assuming a high thread count equals deadlock.
- Increasing `maxThreads`.
- Reading only names without stacks and states.
- Ignoring library-created executors.
- Reducing stack size blindly to fit more threads.

### Concise interview-ready answer

> I treat many threads as a symptom. I compare thread growth with workload, queues, RSS and pool waits, then group multiple thread dumps by name, state, and repeated stack. I check deadlock owners, blocked dependencies, executor creation/lifecycle, PID and native-stack headroom, and context switching. I stop the source of new work if needed, then use bounded shared executors, cancellation, backpressure, and corrected locks or dependency behavior rather than increasing thread limits.

---

## 9. Thread pool exhaustion occurs in production. How would you troubleshoot it?

### What the symptom means and does not prove

All usable workers in a named pool are busy or unavailable, and new work queues, rejects, or times out. It does not prove the pool is too small. The workers may be blocked, slow, deadlocked, running CPU-heavy tasks, or waiting on another exhausted pool.

### Where it can happen

HTTP server workers, Spring `TaskExecutor`, scheduled executors, fork-join/common pools, reactive event loops, message consumers, database pools, HTTP client pools, and chained pools where one waits on another.

### Detailed causes and mechanisms

- Traffic/concurrency exceeds sustainable completion capacity.
- Tasks block on slow DB/downstream operations.
- A DB/HTTP connection pool is exhausted, pinning worker threads.
- Long CPU tasks occupy request workers.
- Deadlock/lock contention prevents task completion.
- Unbounded/large tasks share a pool with latency-sensitive work.
- Nested submission waits for work queued to the same pool, causing starvation deadlock.
- Timeouts do not cancel tasks, so abandoned work holds workers.
- Executor size/queue/rejection policy is inappropriate for workload and resource limits.
- Thread or task leak prevents reuse/completion.

### Ordered investigation

1. Name the exact exhausted pool and its role.
2. Capture active/core/max, queue depth/capacity, oldest wait, completed count, rejection and timeout metrics.
3. Compare arrival rate with completion rate and user latency.
4. Capture repeated thread dumps and classify active workers: CPU, socket read, pool acquisition, lock, sleep, or nested wait.
5. Check downstream DB/HTTP pool active/pending and dependency latency.
6. Check CPU saturation/throttling and context switching.
7. Inspect queue and rejection policy, task timeouts, cancellation, retries, and nested executor usage.
8. Determine whether workload isolation or backpressure failed.
9. Mitigate the task source, then fix completion time/concurrency before resizing.

### Commands, tools, metrics, and interpretation

For Spring's `ThreadPoolTaskExecutor`, export and inspect pool size, active count, queue size/capacity, completed tasks, and rejections. For embedded Tomcat, inspect current/busy/max threads and accept queue. For Hikari, inspect active/max/pending/acquisition.

Thread-dump patterns:

```text
Workers in socket read        -> downstream is slow or timeout/cancellation is weak
Workers in DB pool acquisition -> DB pool is the next exhausted resource
Workers BLOCKED on one lock    -> contention/owner problem
Workers RUNNABLE in same method -> CPU-heavy path; profile it
Workers waiting on Future.get  -> nested task/dependency or starvation risk
```

### Immediate mitigation

Rate-limit or shed new work, stop retries, disable heavy optional tasks, isolate a failed dependency, pause batch work, or add instances only if the pool is per-instance and dependencies/CPU have headroom. Preserve dumps before replacement.

### Root-cause fixes

Reduce task duration, fix downstream/query/lock issues, propagate deadlines and cancellation, separate bulkheads for independent workloads, use bounded queues and explicit rejection, avoid nested same-pool blocking, and calculate concurrency from latency and capacity. Resize only after this evidence.

### Prevention and alerting

Alert on active/max ratio plus queue wait/oldest age and rejection, not utilization alone. Load-test overload behavior, document every pool, and verify cancellation and bulkhead isolation.

### Common mistakes

- Increasing pool size while workers wait on a smaller DB pool.
- Using an unbounded queue that hides overload until latency explodes.
- Treating zero rejections as health when the queue is enormous.
- Forgetting abandoned requests still run.
- Mixing batch and request work in one executor.

### Concise interview-ready answer

> I identify the exact pool and collect active/max, queue depth and age, completion rate, rejection, and wait time. Multiple thread dumps show whether workers are on CPU, blocked on a lock, waiting for DB/HTTP connections, or stuck in nested futures. I correlate the next downstream pool, CPU throttling, retries, and cancellation. I contain incoming or optional work, then shorten tasks, fix the dependency or lock, isolate workloads, and use bounded queues/backpressure before considering a measured resize.

---

# 5. Related interview questions

## 5.1 Heap usage is stable, but the container is OOMKilled. How is that possible?

The cgroup enforces total container memory, not Java heap alone. Direct buffers, thread stacks, metaspace, code cache, JNI/native libraries, allocator arenas, memory-mapped files, sidecars in some accounting views, and file-backed pages can consume the remaining budget. The kernel may kill the JVM before it can throw a Java OOM.

Confirm `OOMKilled`, exit code, cgroup memory events, memory limit, RSS/working set, heap committed/used, direct-buffer metrics, thread count, metaspace, and NMT. Budget:

```text
container limit
  > heap
  + metaspace/code cache
  + direct/native buffers
  + thread stacks
  + GC/JVM/native overhead
  + safety margin
```

Fix the growing native owner or concurrency and set a deliberate heap-to-container budget. Do not use all container memory for `-Xmx`.

## 5.2 How do you distinguish high allocation from a memory leak?

High allocation is rapid object creation; a leak is unintended retention. With high churn, heap rises and falls while the post-GC live-set floor stays stable. With retention, that floor trends upward and old collections reclaim less.

Use allocation rate/JFR to find allocation stacks. Use class histograms and a heap dump to find retained size, dominators, and paths to GC roots. Both can coexist: a leaking request path may also allocate heavily.

## 5.3 What do you look for in three thread dumps?

Compare dumps taken during the symptom:

1. Which threads remain in the same application stack?
2. Which locks and owners recur?
3. Does the JVM report a deadlock cycle?
4. Are request workers waiting on one dependency or pool?
5. Are hot native thread IDs mapped to repeated runnable stacks?
6. Are queue consumers idle while producers report backlog, suggesting wiring/signal issues?

Normal idle pool threads often remain parked in the same framework stack. Focus on application impact, ownership, and repeated non-idle waits.

## 5.4 What is the difference between deadlock and thread-pool starvation?

A deadlock is a cycle of waits: A owns lock 1 and waits for lock 2, while B owns lock 2 and waits for lock 1. No participant can progress without external termination or code correction.

Thread-pool starvation occurs when tasks needed for progress are queued behind tasks that wait for them. For example, every worker submits child work to the same fixed pool and blocks on `Future.get`; no worker remains to run the children. It may not appear as a Java monitor deadlock. Diagnose executor ownership, queue, and future dependencies, then avoid blocking nested work in the same constrained pool.

## 5.5 Why can a larger heap make an incident worse?

A larger heap can:

- Consume native/container headroom and trigger `OOMKilled`.
- Allow a leak to retain more data before failing, extending diagnosis time.
- Increase collection work or worst-case pause depending on collector/workload.
- Hide an unbounded cache or request.
- Increase dump size, disk use, and diagnostic impact.

Increase heap only when measured live-set and allocation behavior show legitimate need, the collector meets pause goals, and the container/node budget has safe headroom.

## 5.6 How should liveness, readiness, and startup probes differ?

- **Startup** allows initialization to finish before liveness takes effect.
- **Readiness** removes an instance from traffic when it cannot serve; it may include critical serving dependencies but should avoid synchronized fleet removal from a shared transient issue.
- **Liveness** should detect a locally unrecoverable process state where restart is likely to help. It should not fail merely because a remote database briefly slows.

A deep liveness check can turn one dependency incident into a fleet restart storm. Probe endpoints need bounded, isolated execution and telemetry.

## 5.7 When is a heap dump appropriate in production?

Use it when heap retention evidence is needed and the operational/security cost is accepted. Ensure enough disk, secure permissions, encryption/transfer, retention, and expected pause/CPU impact. Prefer an automatic on-OOM dump to an approved path or collect from a safely isolated replica. A dump is not the first tool for direct-memory, native-thread, or cgroup OOM.

## 5.8 How do virtual threads change troubleshooting?

Virtual threads reduce the cost of blocking-style concurrency but do not create database connections, CPU, or downstream capacity. A large number can still pin carrier threads through certain blocking/native/monitor patterns, create excessive in-flight work, consume memory, and overwhelm dependencies.

Monitor request concurrency, pinning events where supported, DB/HTTP pool wait, downstream limits, queueing, and deadlines. Keep bulkheads and backpressure. Do not replace a bounded capacity model with unlimited virtual-thread creation.

---

# 6. Decision trees

## 6.1 High CPU decision tree

```text
CPU is high
|
+-- Is the denominator known?
|   +-- No -> identify host/process/container/cores/limit/window
|   `-- Yes
|
+-- Is there user impact or saturation?
|   +-- No -> compare throughput and capacity headroom
|   `-- Yes -> run queue, latency, completion rate, throttling
|
+-- Is container throttling high?
|   +-- Yes -> inspect quota and CPU service demand
|   `-- No
|
+-- Which process/thread consumes CPU?
|   +-- Application -> JFR/profile; algorithm, payload, loop, crypto
|   +-- GC -> allocation, live set, collector, throttling
|   +-- JIT/class loading -> startup/new path
|   +-- Sidecar/agent/logging -> proxy or observability path
|   `-- Kernel -> network/disk/system evidence
|
+-- Did work change?
|   +-- RPS/payload/request mix
|   +-- retries/errors
|   +-- jobs/consumers
|   +-- deployment/config/limit
|
`-- Mitigate proven work; verify CPU per completion and SLO
```

## 6.2 Rising memory or OOM decision tree

```text
Memory rises or process dies
|
+-- What exact evidence?
|   +-- Java OOM message -> classify heap/metaspace/direct/thread/array/native
|   +-- OOMKilled/137 -> cgroup total memory
|   +-- JVM fatal file -> native/JVM crash
|   `-- Unknown -> terminated state, previous logs, events
|
+-- Does post-GC heap trend upward?
|   +-- Yes -> histogram/dump, dominators, GC roots, legitimate live set
|   `-- No
|
+-- Does RSS/working set trend upward?
|   +-- Direct buffers -> buffer metrics/ownership
|   +-- Threads -> count, stacks, PID/native stack
|   +-- Metaspace/classes -> loader/class trend
|   +-- NMT category -> JVM/native owner
|   +-- NMT unexplained -> JNI/allocator/maps/OS evidence
|   `-- No -> transient allocation or accounting/GC phase
|
+-- Does memory reach steady state under a soak test?
|   +-- Yes -> capacity/headroom validation
|   `-- No -> continue ownership analysis
|
`-- Fix retention/lifecycle/bounds; do not mask with scheduled restarts
```

## 6.3 Unresponsive service decision tree

```text
Service is unresponsive; CPU and memory look normal
|
+-- Does TCP/TLS connect?
|   +-- No -> listener, routing, network, readiness
|   `-- Yes
|
+-- Does request reach application access log/trace?
|   +-- No -> gateway/LB/accept queue/request upload
|   `-- Yes
|
+-- Are server workers/event loops saturated?
|   +-- Yes -> repeated dumps
|   |          +-- socket/DB wait -> dependency
|   |          +-- pool acquisition -> next pool
|   |          +-- BLOCKED/deadlock -> lock owner/order
|   |          +-- RUNNABLE hot -> CPU/profile
|   |          `-- Future wait -> starvation/nested task
|   `-- No
|
+-- Are FD/PID/socket/disk/network limits healthy?
|   +-- No -> fix proven resource/OS path
|   `-- Yes
|
`-- Inspect safepoints, probe behavior, instrumentation gaps, and response path
```

## 6.4 Periodic restart decision tree

```text
Service appears to restart
|
+-- Same pod UID?
|   +-- Yes -> container restart; read lastState, exit, signal, previous logs
|   `-- No -> controller replacement, eviction, node, scale, automation
|
+-- Termination evidence?
|   +-- OOMKilled/137 -> total cgroup memory breakdown
|   +-- Java OOM -> exact OOM taxonomy and artifacts
|   +-- Probe failure -> why process stopped responding
|   +-- hs_err/core -> JVM/native crash
|   +-- Exit 0 -> intentional application/lifecycle exit
|   +-- Evicted/node lost -> node pressure/infrastructure
|   `-- Signal/operator -> audit controller/automation
|
+-- Why periodic?
|   +-- memory/resource slope
|   +-- scheduled job/traffic
|   +-- probe threshold
|   +-- certificate/config automation
|   `-- node lifecycle
|
`-- Fix source and observe longer than previous restart interval
```

---

# 7. Final cheat sheet

## CPU checklist

```text
[ ] Denominator: host, process, container, cores, limit, window
[ ] User impact: latency, errors, throughput, queue
[ ] Saturation: run queue and CPU throttling
[ ] Per-process and per-thread CPU
[ ] RPS, completed work, payload, request mix, retries
[ ] GC/JIT/agent/sidecar/background work
[ ] Short profile/JFR plus repeated dumps
[ ] CPU per completed operation after the fix
```

## Memory and OOM checklist

```text
[ ] Exact OOM message or termination reason/exit code
[ ] Heap used/max and post-GC live-set trend
[ ] Allocation rate, GC CPU, pause and reclamation
[ ] RSS/working set versus cgroup limit
[ ] Metaspace/class loaders and direct buffers
[ ] Thread count, stack/native/PID headroom
[ ] NMT/native mappings when applicable
[ ] Secure heap dump only for justified heap-retention analysis
[ ] Soak test proves steady state
```

## Thread dump checklist

```text
[ ] Capture at least three dumps during impact
[ ] Group identical stacks by thread name and state
[ ] Identify lock IDs, owners, and waiters
[ ] Check JVM deadlock report
[ ] Map hot native thread IDs to Java nid
[ ] Find DB/HTTP pool acquisition waits
[ ] Find repeated socket/file I/O destinations
[ ] Distinguish idle parked pool workers from blocked request workers
[ ] Check nested Future/get/join and same-pool starvation
```

## Fast evidence interpretation

| Evidence | Strong next hypothesis |
|---|---|
| CPU high + run queue high | CPU saturation |
| Host CPU low + throttling high | Container CPU quota |
| CPU high + throughput flat | Waste, GC, retry, contention, expensive work |
| Heap post-GC floor rises | Retained heap or growing legitimate live set |
| Allocation high + post-GC floor stable | Object churn |
| Heap stable + RSS rises | Direct/native/thread/mapped memory |
| `OOMKilled`/137 | Cgroup/kernel total-memory kill |
| `unable to create native thread` | Thread/PID/native-memory exhaustion |
| Workers active=max + queue rises | Worker pool exhaustion |
| Hikari pending + active=max | DB connections unavailable; find hold reason |
| Many `BLOCKED` on same monitor | Lock contention; identify owner |
| JVM reports lock cycle | Deadlock |
| Many socket-read stacks to one host | Slow/unbounded downstream wait |
| Pod UID changes, restart count zero | Replacement, not container restart |

## Java/Spring evidence map

| Question | Evidence |
|---|---|
| Which code consumes CPU? | JFR execution samples, sampling profiler, hot-thread mapping |
| Which code allocates? | JFR allocation samples, allocation profiler |
| What is retained? | Post-GC trend, histogram, heap dump dominators/GC roots |
| What uses native memory? | NMT, direct-buffer metrics, threads, mappings, native tools |
| Why are requests stuck? | Multiple thread dumps, server/executor queues, pool waits |
| Why did the container restart? | Last terminated state, previous logs, events, probes, JVM artifacts |
| Is GC hurting users? | Pause timestamps versus latency, GC CPU, allocation/live set |

## Actions that need evidence, not reflex

- **Restarting:** containment after evidence; never the root-cause explanation.
- **Increasing heap:** only for proven legitimate live-set need with native/cgroup headroom.
- **Increasing threads:** only when workers, not dependencies/CPU/locks, are the constrained resource.
- **Increasing connection pools:** only when the dependency has capacity and hold time/leaks are understood.
- **Increasing timeouts:** only when valid service objectives and the complete deadline budget require it.
- **Changing GC:** only after allocation, live set, pauses, CPU, and collector goals are measured.

## Interview answer structure

```text
1. Clarify the metric scope and exact symptom.
2. Correlate user impact, traffic, throughput, and recent changes.
3. Split fleet/pod/node/container/process/thread views.
4. Classify CPU work, memory ownership, or waiting.
5. Collect the least invasive evidence that proves the next hypothesis.
6. Apply a bounded mitigation after preserving evidence.
7. Fix the causal mechanism, not the alarm.
8. Verify SLO recovery and long-term steady state.
9. Add metrics, artifacts, alerts, capacity tests, and a runbook.
```
