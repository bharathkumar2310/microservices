1. What is API latency?

        Latency = how long one request takes from receiving the request until sending the response.

Example:

    Client
    │
    │ Request
    ▼
    API
    │
    ├── DB       200 ms
    ├── Service B 500 ms
    └── Processing 100 ms
    │
    ▼
    Response


    Total ≈ 800 ms
    
    So if the API normally takes 200 ms and suddenly takes 2 seconds, we have a latency problem.

2. First metrics to look at

Don't just look at "average latency."

| Metric                 | What it tells you                               |
| ---------------------- | ----------------------------------------------- |
| **p50**                | Typical request                                 |
| **p95**                | 95% of requests are faster than this            |
| **p99**                | 99% of requests are faster than this            |
| **RPS**                | How much traffic the service is receiving       |
| **CPU**                | Whether CPU is saturated                        |
| **Memory/GC**          | Whether JVM memory pressure is causing delays   |
| **Thread pool**        | Whether requests are waiting for threads        |
| **DB pool**            | Whether requests are waiting for DB connections |
| **DB latency**         | Whether database operations are slow            |
| **Downstream latency** | Whether another service is slow                 |




3. Why p95/p99 instead of only average?

Suppose you have 100 requests:

95 requests → 100 ms
5 requests  → 5 seconds

The average might hide the problem.

But:

p50 ≈ 100 ms
p95 ≈ 100 ms
p99 ≈ 5 sec

Now you know:

    Most users are fine, but a small percentage are experiencing extremely slow requests.

That's why interviewers often ask about p95/p99.

4. The first decision you make

Suppose you see:

p95 latency: 200 ms → 2 sec
p99 latency: 300 ms → 5 sec

Don't immediately say:

    "The application is slow."

Instead ask:

    Did traffic increase?

Check:

    RPS

Example:

Before:
    
    RPS = 500
    p95 = 200 ms


Now:
        
        RPS = 2,000
        p95 = 2 sec

This suggests load/capacity pressure might be involved.

Then check:

    CPU
    Thread pool
    DB connections
    DB latency
    Downstream latency


5. The most important reasoning pattern

Think of latency as:

    API latency increased
    ↓
    Is traffic higher?
    ↓
    ┌──────┴──────┐
    YES           NO
    ↓              ↓
    Check          Look for
    saturation     internal bottleneck
    ↓              ↓
    CPU             DB
    Threads         Downstream
    DB pool         Threads
    GC

But there's another powerful tool:

    Distributed tracing

Suppose the trace says:

    API = 2.5 sec


    Service A = 100 ms
    ↓
    Service B = 2.2 sec
    ↓
    DB = 2.0 sec

    Now you've narrowed the problem dramatically.
    
    You don't need to investigate everything equally.

You know:

    Service B's database operation is probably responsible for most of the latency.

Then you move to:

    DB metrics
    ↓
    Slow query
    ↓
    EXPLAIN
    ↓
    Query plan
    ↓
    Index / SQL / locking

6. Different latency patterns tell you different things

This is something worth understanding deeply.

Pattern A
p50 ↑
p95 ↑
p99 ↑

Most requests are becoming slower.

Think:

general/systemic bottleneck

Possible causes:

    CPU saturation
    DB slowdown
    downstream slowdown
    traffic increase


Pattern B
p50 normal
p95 ↑
p99 ↑↑

Most requests are fine, but some requests are very slow.

Think:

tail latency problem

Possible causes:

particular DB queries
occasional locks
slow downstream calls
GC pauses
thread contention
specific data/request paths
Pattern C
Latency ↑
RPS ↑
CPU ↑

Strong indication of:

load/capacity problem

Possible fix:

scale + investigate whether the service is efficiently using resources.

Pattern D
Latency ↑
RPS same
CPU normal
DB latency ↑

Now DB becomes your primary suspect.

Pattern E
Latency ↑
RPS same
CPU normal
DB normal
Downstream latency ↑

Investigate the downstream service.

Pattern F
Latency ↑
CPU normal
DB normal
Downstream normal
Thread queue ↑

Now investigate:

thread starvation / blocking operations.

7. What the interviewer may ask next

If you say:

"I'll check CPU."

They may ask:

"CPU is normal. What next?"

You should continue:

"I'd check thread-pool saturation, DB connection-pool utilization and DB latency, downstream service latency, and then use distributed tracing to identify which component contributes most to the request latency."

Then they may say:

"DB connections are exhausted."

Now you're in a DB connection-pool scenario.

They may then ask:

"Why would connections be exhausted?"

Possible causes:

Slow queries
Long-running transactions
Connection leak
Too many concurrent requests
Pool too small
Database itself overloaded

Then you investigate those.



| #  | Latency            | Other metrics / finding                                     | Likely issue                                             | Why this points to it                                               | What to investigate next                                        | Typical fix                                                                  |
| -- | ------------------ | ----------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| 1  | ↑ p50, p95, p99    | **RPS ↑ significantly**                                     | **High traffic / overload**                              | More requests are arriving than the service can comfortably process | CPU, threads, DB connections, downstream load                   | Horizontal scaling, autoscaling, rate limiting, caching, optimize bottleneck |
| 2  | ↑ p50, p95, p99    | **RPS ↑ + CPU 90–100%**                                     | **CPU saturation**                                       | Increased traffic is consuming available CPU                        | CPU profiling, hot methods, thread activity                     | Optimize expensive code, scale instances                                     |
| 3  | ↑ p50, p95, p99    | **RPS normal + CPU 95–100%**                                | **CPU-intensive code/regression**                        | Traffic didn't increase, but CPU did                                | Recent deployment, CPU profile, thread dump                     | Fix inefficient code, rollback, scale if necessary                           |
| 4  | ↑ latency          | **CPU normal + DB latency ↑**                               | **Slow database**                                        | Application is waiting for DB                                       | Slow queries, `EXPLAIN`, DB CPU, locks                          | Query optimization, indexes, reduce DB work                                  |
| 5  | ↑ latency          | **DB connections near max + pending ↑**                     | **DB connection pool exhaustion**                        | Requests are waiting to acquire connections                         | Active connections, query duration, transaction duration        | Fix slow queries/long transactions/leaks; tune pool carefully                |
| 6  | ↑ latency          | **DB connections maxed + DB CPU high**                      | **Database overloaded**                                  | DB cannot process concurrent workload fast enough                   | Top queries, DB CPU/I/O, connection count                       | Optimize queries, scale DB, reduce concurrency                               |
| 7  | ↑ latency          | **DB connections maxed + DB CPU normal**                    | **Long-running queries/transactions or connection leak** | Connections are occupied even though DB isn't CPU-bound             | Active queries, transaction duration, connection leak detection | Fix query/transaction/connection handling                                    |
| 8  | ↑ latency          | **DB lock waits ↑**                                         | **Lock contention**                                      | Requests are waiting for another transaction                        | Blocking transaction, lock information                          | Shorten transactions, fix locking/order, optimize queries                    |
| 9  | ↑ latency          | **Deadlocks ↑**                                             | **Transaction concurrency problem**                      | Transactions are repeatedly conflicting                             | Deadlock logs + transaction order                               | Consistent locking order, smaller transactions, retry carefully              |
| 10 | ↑ latency          | **Downstream Service B latency ↑**                          | **Slow dependency**                                      | Your service is waiting for B                                       | Distributed trace + B's metrics                                 | Fix B, timeout, circuit breaker, fallback/cache                              |
| 11 | ↑ latency          | **Downstream errors ↑ + retries ↑**                         | **Dependency failure/retry storm**                       | Failed calls cause retries and extra waiting                        | Retry count, timeout, B's errors                                | Fix dependency, exponential backoff, limit retries                           |
| 12 | ↑ latency          | **Downstream latency normal + thread queue ↑**              | **Thread-pool saturation**                               | Requests are waiting for execution threads                          | Active threads, queue, thread dump                              | Remove blocking work, tune pool, scale                                       |
| 13 | ↑ latency          | **Thread count ↑ + CPU low**                                | **Blocking I/O / waiting threads**                       | Threads are waiting rather than using CPU                           | Thread dump                                                     | Fix blocking operation, async/non-blocking approach                          |
| 14 | ↑ latency          | **Thread count ↑ + CPU high**                               | **CPU-bound workload**                                   | Threads are actively consuming CPU                                  | CPU/thread profiling                                            | Optimize code, scale                                                         |
| 15 | ↑ latency          | **GC frequency ↑ / GC pauses ↑**                            | **JVM memory pressure**                                  | Application pauses while GC works                                   | Heap usage, allocation rate, GC logs                            | Reduce allocations, investigate heap, tune JVM                               |
| 16 | ↑ latency          | **Heap continuously ↑**                                     | **Possible memory leak**                                 | Objects aren't being released                                       | Heap dump, retained objects                                     | Fix leak                                                                     |
| 17 | ↑ p95/p99 only     | **p50 normal**                                              | **Tail latency problem**                                 | Most requests are fine, but some are very slow                      | Slow traces, specific queries, locks, downstream calls          | Fix problematic path/query/dependency                                        |
| 18 | ↑ p99 dramatically | **Occasional GC/DB lock spikes**                            | **Intermittent bottleneck**                              | Only some requests hit the slow condition                           | Correlate traces with GC/DB locks                               | Fix intermittent bottleneck                                                  |
| 19 | ↑ latency          | **RPS normal + CPU normal + DB normal + downstream normal** | **Application/thread issue**                             | Major external resources look healthy                               | Thread pool, thread dump, code profiling                        | Find blocking/synchronization/inefficient code                               |
| 20 | ↑ latency          | **Request queue ↑**                                         | **Service cannot process requests fast enough**          | Requests are accumulating faster than they're completed             | CPU, threads, DB pool                                           | Scale/optimize bottleneck                                                    |
| 21 | ↑ latency          | **Network latency ↑**                                       | **Network problem**                                      | Time is spent communicating between components                      | Trace network spans, network metrics                            | Network/config/infrastructure fix                                            |
| 22 | ↑ latency          | **Only one pod/instance affected**                          | **Bad/overloaded instance**                              | Other instances are healthy                                         | Per-instance CPU, memory, traffic, logs                         | Replace pod, fix load balancing, investigate instance                        |
| 23 | ↑ latency          | **All instances affected**                                  | **System-wide dependency/config/load issue**             | Problem isn't isolated to one instance                              | RPS, DB, downstream, deployment                                 | Fix shared bottleneck                                                        |
| 24 | ↑ latency          | **Latency increased immediately after deployment**          | **Code/config regression**                               | Timing strongly correlates with new version                         | Compare old/new version, traces, SQL                            | Rollback or fix regression                                                   |
| 25 | ↑ latency          | **DB query count/request ↑ <br/>massively**                      | **N+1 query**                                            | One API request generates excessive DB calls                        | SQL logs/traces                                                 | Fetch join, EntityGraph, batching, query redesign                            |
| 26 | ↑ latency          | **External API response time ↑**                            | **Third-party dependency slow**                          | Your service is waiting for external system                         | External-call traces/logs                                       | Timeout, cache, fallback, circuit breaker                                    |
| 27 | ↑ latency          | **Connection timeout ↑**                                    | **Connection/resource exhaustion**                       | Requests can't obtain required connection                           | DB/HTTP connection pool                                         | Fix pool exhaustion/root cause; tune limits                                  |
| 28 | ↑ latency          | **Kafka consumer lag ↑**                                    | **Async processing falling behind**                      | Consumer can't keep up with incoming events                         | Consumer throughput, processing time, partitions                | Scale consumers, optimize processing                                         |
| 29 | ↑ latency          | **Kafka lag ↑ + DB latency ↑**                              | **Consumer DB bottleneck**                               | Message processing is slowed by DB                                  | Consumer traces + DB queries                                    | Optimize DB/consumer                                                         |
| 30 | ↑ latency          | **Rate limiting/429 ↑**                                     | **Traffic being throttled**                              | Requests are exceeding configured capacity/rate                     | Gateway metrics + client traffic                                | Correct client behavior / tune limits appropriately                          |


