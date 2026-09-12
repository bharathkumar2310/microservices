JVM Production Troubleshooting – Scenario-Based Interview Guide

This guide covers common JVM production scenarios involving CPU, heap memory, garbage collection, OutOfMemoryError, heap dumps, thread dumps, and excessive object creation.

1. JVM CPU Suddenly Reaches 100%. How Would You Investigate?

First principle

High CPU does not automatically mean a JVM problem.

Possible sources include:

Application business logic

Infinite loops

Excessive garbage collection

Too many requests

Thread contention/spinning

Serialization/deserialization

Database/result processing

Compression/encryption

A library or framework issue

Investigation flow

Step 1: Confirm the CPU spike

Check monitoring:

CPU usage over time

When the spike started

Which deployment/change happened before it

Traffic/request rate

Error rate

API latency

Ask:

Did CPU increase because traffic increased, or did CPU increase without a traffic increase?

Scenario A: Traffic increased

Traffic ↑
↓
More requests processed
↓
More CPU work
↓
CPU ↑

This may be legitimate capacity pressure.

Scenario B: Traffic is normal but CPU is high

Normal traffic
+
CPU suddenly 100%

This is more suspicious.

Possible causes:

Infinite loop

GC activity

CPU-intensive code

Bad deployment

Library issue

Step 2: Check whether GC is responsible

Check:

GC frequency

GC time

Heap usage

Memory after GC

Example:

CPU = 100%
GC frequency = very high
Heap = 95%
GC frees little memory

Likely:

Heap pressure
↓
Frequent GC
↓
GC consumes CPU
↓
High JVM CPU

Step 3: Take a thread dump

A thread dump answers:

Which threads are consuming CPU or actively executing?

Look for many threads in:

RUNNABLE

Inspect their stack traces.

Example:

"worker-15"
RUNNABLE
at com.example.ReportService.processLargeReport()

If many samples show the same code, investigate that code.

Important

A single thread dump is only a snapshot.

For CPU issues, take multiple thread dumps several seconds apart.

If the same thread repeatedly appears:

RUNNABLE
at com.example.SomeService.expensiveMethod()

it is a strong clue.

Step 4: Use CPU profiling when necessary

For a difficult CPU issue, use a profiler/APM to identify:

Hot methods

CPU-consuming methods

Allocation hotspots

Time spent in application code

Interview answer

I would first correlate CPU with traffic, latency, errors, and recent deployments. Then I would check GC metrics to determine whether excessive garbage collection is consuming CPU. If GC is not the cause, I would take multiple thread dumps and inspect RUNNABLE threads and their stack traces. If needed, I would use a CPU profiler/APM to identify hot methods and then investigate the responsible code.

2. Heap Memory Keeps Increasing. What Could Cause It?

Main question

Ask:

Does memory decrease after garbage collection?

Normal pattern

Heap grows
↓
GC
↓
Returns near baseline

Example:

500 MB → GC → 300 MB
550 MB → GC → 320 MB
600 MB → GC → 310 MB

Normal workload may cause this.

Suspicious pattern

500 MB → GC → 400 MB
700 MB → GC → 600 MB
900 MB → GC → 800 MB

The post-GC baseline keeps increasing.

This suggests:

Memory leak/unwanted retention

Unbounded cache

Growing collection

Queue backlog

Too many concurrent requests

Legitimately increasing workload

Common causes

1. Memory leak

Objects are no longer useful but remain referenced.

2. Unbounded cache

Cache
↓
Keeps growing forever

No TTL, eviction, or size limit.

3. Growing collections

list.add(object);

Objects are continuously added and never removed.

4. Unbounded queue

Producer faster than consumer
↓
Queue grows
↓
Heap grows

5. Large DB results

Examples:

findAll()

Loading millions of records

No pagination

6. Large files or HTTP responses

A huge byte[] may consume large heap memory.

7. Too many concurrent requests

Slow requests can keep many object graphs alive simultaneously.

Investigation

Heap trend
↓
Memory after GC
↓
GC frequency
↓
Heap dump
↓
Dominator Tree
↓
Retained Size
↓
Path to GC Roots

3. Frequent GC Causes API Latency. Explain Why.

Garbage Collection requires JVM resources.

When the application allocates many objects:

Object allocation ↑
↓
Heap fills faster
↓
GC runs more often

During some GC phases, application threads may be paused.

Therefore:

Frequent GC
↓
Application spends more time managing memory
↓
Less time processing requests
↓
API latency increases

Example

Normal:
Request = 100 ms

Under GC pressure:
Request processing = 100 ms
GC interruption = 300 ms

Total perceived latency = 400 ms

What to check

Heap usage

Allocation rate

GC frequency

GC pause time

Memory after GC

P95/P99 API latency

Root causes

Excessive object creation

Memory leak

Heap too small

Large requests

Large collections

High traffic

Fix

Do not immediately increase heap.

First identify:

Why is GC happening frequently?

Then fix allocation/retention problems.

4. Full GC Suddenly Increases. What Would You Check?

A sudden increase in Full GC is a warning sign.

Step 1: Check heap usage

Ask:

Is heap close to maximum?

Example:

Heap used = 95%
Heap max = 100%

GC may be struggling to recover enough memory.

Step 2: Check memory after Full GC

Example A:

Before Full GC = 1.9 GB
After Full GC  = 500 MB

A lot was reclaimed.

Investigate temporary allocation spikes.

Example B:

Before Full GC = 1.9 GB
After Full GC  = 1.8 GB

Very little was reclaimed.

Suspect retained objects or memory leak.

Step 3: Check what changed

Traffic spike?

New deployment?

New cache?

Large batch job?

Large report?

Queue backlog?

Step 4: Capture heap dump

Find:

Largest retained objects

Dominators

Growing collections

Caches

byte[]

Strings

Queues

Interview answer

I would check heap occupancy before and after Full GC. If Full GC reclaims very little memory, I would suspect object retention or a memory leak and analyze a heap dump. I would also correlate the increase with traffic, deployments, batch jobs, cache growth, and allocation rate.

5. Application Gets OutOfMemoryError. How Would You Investigate?

Step 1: Identify the exact OOM type

This is the most important first step.

Possible errors:

Java heap space
GC overhead limit exceeded
Metaspace
unable to create new native thread

Each requires a different investigation.

A. Java Heap Space

Check:

Heap usage
GC behavior
Heap trend
Heap dump

Then:

Histogram
Dominator Tree
Retained Size
Path to GC Roots

B. GC Overhead Limit Exceeded

Check:

Heap almost full?

GC frequency?

GC time?

How much memory is freed after GC?

Then analyze the heap similarly.

C. Metaspace

Check:

Metaspace usage trend

Loaded class count

Unloaded class count

ClassLoaders

Dynamic class generation

Possible causes:

ClassLoader leak

Unlimited proxy/class generation

Metaspace limit too low

D. Unable to Create New Native Thread

Check:

Live thread count

Thread trend

Thread dump

Thread pool configuration

Native memory

OS/container limits

Thread stack size

General production workflow

OOM
↓
Identify exact type
↓
Check monitoring
↓
Capture correct diagnostic data
↓
Find root cause
↓
Fix
↓
Verify
↓
Add alerts/prevention

6. Memory Usage Increases Slowly Over Several Hours. What Does That Suggest?

This often suggests gradual object retention.

Possible causes:

Memory leak

Cache continuously growing

Collection accumulating data

Queue backlog

Session data retention

Slow consumer

Strong indicator

Check memory after GC.

Hour 1 → after GC = 300 MB
Hour 2 → after GC = 450 MB
Hour 3 → after GC = 600 MB
Hour 4 → after GC = 800 MB

This is suspicious.

Investigation

Take multiple heap dumps over time:

10 AM → Dump 1
2 PM  → Dump 2
6 PM  → Dump 3

Compare object counts and retained sizes.

Example:

UserSession

Dump 1 = 10,000
Dump 2 = 50,000
Dump 3 = 300,000

Then inspect why these objects remain reachable.

7. CPU Is High After a Deployment. How Would You Determine Whether GC Is Responsible?

This is a common production scenario.

Step 1: Compare before and after deployment

Check:

CPU

Heap usage

Allocation rate

GC frequency

GC time

API latency

Step 2: Look for correlation

Example:

Deployment
↓
Allocation rate ↑
↓
Heap fills faster
↓
GC frequency ↑
↓
CPU ↑

This strongly suggests GC contributes to CPU.

Step 3: Check GC time

If a large percentage of JVM time is spent in GC:

Application work → small percentage
GC work          → large percentage

GC is likely responsible.

Step 4: Check allocation changes

A deployment may introduce:

New code
↓
Creates millions of temporary objects
↓
Allocation rate increases
↓
Frequent GC

Then inspect the changed code.

If GC is NOT responsible

Take thread dumps and profile CPU.

Look for:

Infinite loops

Expensive algorithms

Serialization

Compression

Excessive logging

Busy loops

8. How Would You Use a Heap Dump?

A heap dump is a snapshot of objects currently stored in the Java heap.

Main questions

Question 1

What is consuming memory?

Use:

Histogram

Instance count

Question 2

Which object retains the most memory?

Use:

Dominator Tree

Retained Size

Question 3

Why is this object still alive?

Use:

Path to GC Roots

Example

HashMap
Retained Size = 4 GB
↓
5 million User objects

Then:

Path to GC Roots

GC Root
↓
static Cache
↓
HashMap
↓
Users

Root cause:

Static cache retains users

Then investigate:

Should these users remain?

Is TTL missing?

Is eviction missing?

Heap dump workflow

Open Heap Dump
↓
Leak Suspects Report
↓
Histogram
↓
Dominator Tree
↓
Largest Retained Size
↓
Path to GC Roots
↓
Application code

9. How Would You Use a Thread Dump?

A thread dump is a snapshot of JVM threads.

It shows:

Thread names

Thread states

Stack traces

Locks

Main thread states

RUNNABLE
BLOCKED
WAITING
TIMED_WAITING

What do you look for?

1. Too many threads

Example:

10,000 live threads

Suspect thread leak or pool problem.

2. Repeated thread patterns

pool-1-thread-1
pool-1-thread-2
...
pool-1-thread-5000

Investigate that pool.

3. Many BLOCKED threads

Possible:

Lock contention

Slow synchronized section

4. Many threads waiting at same downstream call

Example:

PaymentClient.call()

Possible:

Slow downstream service

Missing timeout

5. Deadlock

Look for JVM deadlock detection or circular lock waits.

CPU investigation

For CPU issues, take multiple thread dumps.

If the same threads repeatedly appear RUNNABLE in the same stack:

RUNNABLE
at SomeExpensiveMethod()

investigate that method.

10. What Could Cause Excessive Object Creation?

Excessive object creation increases allocation rate.

Objects created quickly
↓
Heap fills quickly
↓
Frequent GC
↓
CPU usage ↑
Latency ↑

Common causes

1. Creating unnecessary temporary objects

Example:

for (...) {
new SomeObject();
}

This is not automatically bad, but millions of unnecessary allocations can create GC pressure.

2. String concatenation in loops

Bad:

String result = "";

for (...) {
result = result + value;
}

Better:

StringBuilder builder = new StringBuilder();

3. Large JSON processing

Repeated:

Serialization

Deserialization

Object mapping

can create many temporary objects.

4. Creating DTO copies repeatedly

Entity
↓
DTO
↓
Another DTO
↓
Response object
↓
Another transformed object

Too much copying increases allocations.

5. Large collections

Creating and copying large Lists, Maps, and Sets.

6. Reading entire files into memory

byte[] data = inputStream.readAllBytes();

Large files can create huge allocations.

Prefer streaming when appropriate.

7. Large database result sets

Loading millions of records simultaneously.

Use:

Pagination

Streaming

Batch processing

8. Excessive logging/object conversion

Building large log messages or repeatedly converting objects can increase allocations.

Final Interview Troubleshooting Framework

For JVM production issues, use this general approach:

1. Confirm the symptom
   ↓
2. Check monitoring metrics
   ↓
3. Correlate with traffic/deployment/errors
   ↓
4. Identify JVM subsystem involved
   ↓
   CPU?
   Heap?
   GC?
   Threads?
   Metaspace?
   ↓
5. Capture diagnostic data
   ↓
   Heap Dump
   Thread Dump
   GC data/logs
   ↓
6. Find root cause
   ↓
7. Fix the underlying problem
   ↓
8. Verify after deployment
   ↓
9. Add monitoring and alerts

Key Rule

Do not jump directly to a fix such as:

Increase -Xmx
Restart application
Increase thread pool

First understand:

What is actually causing the JVM resource problem?

The goal is always:

Symptom
↓
Metrics
↓
Evidence
↓
Root Cause
↓
Correct Fix