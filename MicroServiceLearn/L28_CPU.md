CPU — Metric #3

First understand one thing:

    CPU utilization tells you how much of the available CPU capacity is currently being used.

Example:

    4 CPU cores
    CPU utilization = 90%
    
    means the service is using most of its available CPU capacity.
    
    But high CPU does NOT automatically mean "scale."

We need to understand why CPU is high.

1. CPU troubleshooting master table


| CPU            | RPS       | Latency | Other finding                  | Likely problem                                | What to investigate               | Typical fix                           |
| -------------- | --------- | ------- | ------------------------------ | --------------------------------------------- | --------------------------------- | ------------------------------------- |
| ↑              | ↑         | ↑       | DB healthy                     | **Traffic-driven CPU saturation**             | CPU profile, instance capacity    | Horizontal scaling + optimize         |
| ↑              | ↑         | Normal  | Plenty of headroom             | Increased traffic but still healthy           | Monitor trend/capacity            | Usually no immediate fix              |
| ↑              | Normal    | ↑       | Recent deployment              | **Code regression**                           | Compare versions + CPU profiling  | Rollback/fix code                     |
| ↑              | Normal    | ↑       | GC ↑                           | **Excessive object allocation / GC pressure** | GC logs, allocation profiling     | Reduce allocations/tune JVM           |
| ↑              | Normal    | ↑       | One endpoint dominates         | **Expensive code path**                       | Endpoint-level metrics + profiler | Optimize endpoint                     |
| ↑              | Normal    | ↑       | Infinite/very long computation | **CPU-bound code**                            | Thread dump/profiling             | Fix algorithm/loop                    |
| ↑              | Normal    | ↑       | Thread count high              | **CPU contention / concurrency**              | Thread dump                       | Fix synchronization/concurrency       |
| ↑              | Normal    | Normal  | No latency impact              | CPU is busy but service still has capacity    | Capacity trend                    | Monitor                               |
| ↑              | ↑         | ↑       | DB CPU also ↑                  | **System-wide load**                          | DB queries/connections            | Scale carefully + optimize DB         |
| ↑              | ↑         | ↑       | Downstream overloaded          | **Dependency bottleneck**                     | Trace + downstream metrics        | Protect dependency / optimize         |
| One pod CPU ↑  | Normal    | ↑       | Other pods normal              | **Uneven load / problematic instance**        | Per-pod traffic + profiling       | Fix LB/distribution/restart if needed |
| All pods CPU ↑ | ↑         | ↑       | Similar utilization            | **Genuine service saturation**                | Capacity + profiling              | Scale/optimize                        |
| CPU ↓          | Latency ↑ | —       | DB latency ↑                   | **CPU isn't bottleneck**                      | DB                                | Fix DB                                |
| CPU ↓          | Latency ↑ | —       | Thread queue ↑                 | **Blocking/thread starvation**                | Thread dump                       | Fix blocking                          |
| CPU ↓          | Latency ↑ | —       | Downstream latency ↑           | **Dependency bottleneck**                     | Distributed trace                 | Fix/protect downstream                |


1. CPU profiling

        CPU profiling = finding which methods/classes in your application are consuming the most CPU.

Imagine:

    CPU = 95%


What is using it?


    calculatePrice()       → 40%
    processOrders()        → 30%
    JSON serialization     → 15%
    Other                  → 10%

    A CPU profiler gives you this kind of information.
    
    So instead of saying:
    
    "CPU is high."
    
    you can determine:
    
    "The calculatePrice() method is consuming a large portion of CPU."

How?

Tools commonly used with Java include:

    Java Flight Recorder (JFR)
    Java Mission Control (JMC)
    async-profiler
    VisualVM
    Your APM/profiling tools

The important interview concept is not memorizing every tool.

Just understand:

    CPU profiling → identify CPU-consuming methods → inspect that code → optimize/fix it.

2. Thread dump

        A thread dump is a snapshot of what all Java threads are doing at a particular moment.

Imagine your application has:

    200 threads

A thread dump might show:

    Thread-1 → RUNNABLE → calculateSomething()
    Thread-2 → WAITING → waiting for DB connection
    Thread-3 → BLOCKED → waiting for lock
    Thread-4 → RUNNABLE → processRequest()
    Thread-5 → WAITING → HTTP call
...

    This tells you what your threads are doing.


What do you actually do with the dump?

    This is more important than memorizing the command.

Suppose you take a dump and see:

    100 threads → WAITING for DB connection

You investigate:

    Why are DB connections unavailable?
    
    Or:
    
    50 threads → BLOCKED

You investigate:

        What lock are they waiting for?
        
        Or:
        
        many threads → RUNNABLE

You investigate:

    Are they consuming CPU because of an expensive method?

    You can also take multiple thread dumps a few seconds apart.

For example:

    Dump 1 → Thread A RUNNABLE in calculateReport()
    Dump 2 → Thread A RUNNABLE in calculateReport()
    Dump 3 → Thread A RUNNABLE in calculateReport()

That persistence is a strong clue that the thread is spending significant time there.

3. CPU profiling vs thread dump

This distinction is important.

	|                 | CPU Profiling                       | Thread Dump                                           |
| --------------- | ----------------------------------- | ----------------------------------------------------- |
| Main purpose    | Find **CPU-consuming code**         | Find **what threads are doing/waiting for**           |
| Best for        | High CPU                            | Thread starvation, blocking, deadlocks, stuck threads |
| Example finding | `calculatePrice()` consumes 50% CPU | 100 threads waiting for DB connections                |
| Output          | CPU usage by methods                | Thread states + stack traces                          |
| Think           | **"Who is using CPU?"**             | **"What are my threads doing?"**                      |



1. What does JVM tuning mean?

For a Spring Boot application, JVM tuning mainly means configuring things such as:

Heap size
Garbage Collector
GC behavior
Thread-related settings
JVM/container memory
JVM diagnostic settings

The most common interview discussion is heap + GC.

2. Heap tuning

The JVM has:

JVM Memory
│
├── Heap
│    ├── Young generation
│    └── Old generation
│
└── Non-heap/native memory

You can control heap size using:

-Xms
-Xmx

For example:

-Xms512m
-Xmx2g

means:

Start with 512 MB heap and allow it to grow up to 2 GB.

When would you increase -Xmx?

Suppose:

Heap usage → consistently near max
GC → very frequent
Application → OOM

If you've determined the application genuinely needs more memory, you may increase heap.

But:

Increasing heap is not a fix for a memory leak.

If objects are continuously being retained:

1 GB → 1.5 GB → 2 GB → 3 GB → OOM

Increasing Xmx only delays the crash.

You need to find the leak.

3. GC tuning

Garbage Collector manages unused Java objects.

Modern JVMs have collectors such as:

G1 GC
ZGC
Shenandoah (depending on JVM distribution/version)

For many Spring Boot applications, G1 GC is a common general-purpose choice.

Example:

-XX:+UseG1GC

But don't say:

"Always use G1."

Instead:

"I'd first look at GC behavior and application requirements, then choose/tune the collector accordingly."

4. Example: frequent GC

Suppose metrics show:

Heap → 90%
GC frequency → very high
GC pause → increasing
Latency → increasing

You investigate:

Allocation rate
↓
GC logs / JFR
↓
What objects are being allocated?

You might discover:

Huge temporary objects
Large JSON payloads
Excessive object creation

The best fix may be application code, not JVM configuration.

For example:

Reduce unnecessary object creation → reduce GC pressure → latency improves.

5. GC pause problem

Suppose:

GC pause = 1 second

During a stop-the-world pause, application threads may be paused.

So:

Request
↓
Application processing
↓
GC pause
↓
Request waits
↓
Latency increases

If p99 latency is bad, GC pauses may be the reason.

You investigate:

GC logs
Heap size
Allocation rate
GC frequency
JFR

Then tune appropriately.


Suppose the interviewer says:

"CPU suddenly went to 95%. What do you do?"

Don't immediately say:

"Scale."

Instead:

Step 1 — Check RPS
CPU ↑
↓
RPS ↑ ?
If RPS also increased:
RPS ↑
CPU ↑
Latency ↑

Possible:

Traffic overload

Then check whether the DB and downstream systems can handle additional traffic.

If yes:

Scale horizontally.

If RPS did NOT increase

Example:

RPS = 500 → 500
CPU = 40% → 95%
Latency = 200ms → 2 sec

Now traffic isn't the explanation.

Investigate:

Recent deployment
Expensive code
Infinite loop
CPU-intensive algorithm
Excessive object creation
GC
Lock/contention issues
3. CPU + GC

Suppose:

CPU ↑
GC ↑
Latency ↑
Heap ↑

Now you might have:

Excessive object allocation / memory pressure causing frequent GC.

You investigate:

GC logs
↓
Allocation rate
↓
Heap usage
↓
Heap dump/profiling

Possible fix:

Reduce unnecessary object creation / fix memory issue / tune JVM appropriately.

4. CPU + recent deployment

This is a very common production scenario.

Version 1:
CPU = 40%


Deploy Version 2


CPU = 90%

And:

RPS = same

That is a huge clue.

Likely:

Regression introduced by the new version.

You would:

Compare versions
↓
Check endpoint metrics
↓
CPU profile
↓
Identify expensive method
↓
Fix / rollback
5. One pod vs all pods

This is another important interview concept.

Case A — All pods
Pod 1 → CPU 95%
Pod 2 → CPU 93%
Pod 3 → CPU 96%
Pod 4 → CPU 94%

Likely:

System-wide load/capacity problem.

Think:

Scale / optimize.

Case B — One pod
Pod 1 → 95%
Pod 2 → 40%
Pod 3 → 42%
Pod 4 → 39%

Now don't immediately scale everything.

Investigate:

Is traffic uneven?
Is this pod processing unusual requests?
Is there a problematic thread?
Is this instance behaving differently?
Was it recently deployed/restarted?
6. CPU low but latency high

This is extremely important.

Suppose:

CPU = 30%
Latency = 3 sec

Don't say:

"CPU is fine, so the service is fine."

The service may be waiting rather than computing.

Check:

DB latency
DB connections
Thread pool
Downstream latency
Network
Locks

For example:

CPU = 30%
DB latency = 2.5 sec

Clearly:

DB is likely causing the latency.

7. What does "CPU saturation" actually mean?

This is the concept you need to remember.

CPU saturation = the workload is demanding more CPU processing capacity than the service currently has available.

Symptoms often include:

CPU → near 100%
Latency → ↑
Request queue → ↑
Throughput → may stop increasing

At this point, adding more traffic doesn't necessarily give you more throughput.

You have reached a capacity limit.

CPU ↑
│
├── 1. Is it application CPU or GC CPU?
│
├── 2. Is it all instances or one instance?
│
├── 3. Did a deployment happen recently?
│
├── 4. Which threads are consuming CPU?
│
└── 5. What operation are those threads performing?
