Scenario: API latency suddenly increases

Suppose:

        /orders/{id} normally responds in 200 ms, but suddenly goes to 2–5 seconds.

Your goal is not to immediately look at everything.

    You move from symptom → scope → bottleneck → root cause.

Step 1 — Confirm the latency problem

First, in monitoring/APM, check:

1. Latency

Look at:

    Average latency
    P95
    P99
    Max latency

Example:

        Metric	Normal	Now
        Avg	200 ms	1.2 sec
        P95	300 ms	3 sec
        P99	500 ms	5 sec

This tells you the problem is real.

Why P95/P99?

Average can hide a problem.

For example:

99 requests → 100 ms
1 request  → 10 seconds

Average may still look reasonable, while some users are experiencing terrible latency.

So in production troubleshooting:

P95/P99 are extremely important.

Step 2 — Check traffic

Now ask:

    "Did the API suddenly receive more requests?"

Check:

    Request rate / RPS

Example:

        Normal:  500 RPS
        Now:     2000 RPS

If latency increased at exactly the same time traffic increased, one possible explanation is:

The service is becoming overloaded.

But don't conclude that yet.

You need to check the service resources.

Step 3 — Check CPU

Look at the affected service's:

CPU utilization

Example:

CPU
        
        40% ────────────────
        ↓
        95% ────────────────

If:

    RPS ↑
    Latency ↑
    CPU ↑

then CPU saturation becomes a strong suspect.

Why?

More requests → more processing → CPU becomes saturated → requests wait longer → latency increases.

Possible causes:

    Traffic spike
    Expensive code path
    Infinite/large loop
    Serialization/deserialization overhead
    Encryption/compression
    Bad deployment
    Excessive GC

Step 4 — Check memory

Now check:

    Heap usage
    Memory utilization
    GC activity
    GC pause time

Example:

Heap:       60% → 90%
GC pauses:  20ms → 800ms

Then latency may be caused by GC pressure.

For example:

    Requests
    ↓
    Objects created rapidly
    ↓
    Heap fills
    ↓
    GC runs frequently
    ↓
    Application pauses
    ↓
    Latency increases

So:

High latency + high GC = investigate memory/GC.

Step 5 — Check thread pool

This is very important in Java/Spring Boot.

Look at:

    Active threads
    Thread pool size
    Queue size
    Rejected tasks

Example:

Thread pool:
Max threads = 200
Active      = 200
Queue       = 500

This means the application is fully occupied.

Requests may be waiting for a thread.

So:

Traffic ↑
↓
Threads become busy
↓
Requests wait in queue
↓
Latency ↑
Step 6 — Check database

Now ask:

"Is my application slow, or is the database making my application wait?"

Look at:

DB query latency
DB connection pool
Active connections
Connection acquisition time
DB CPU
Slow queries
Locks
Query throughput

Example:

Application latency: 200 ms → 3 sec


DB query latency:     50 ms  → 2.5 sec

Now you have a strong indication:

Database is the bottleneck.

Very important: DB connection pool

Suppose:

Max connections = 50
Active = 50
Waiting = 100

Your application may not even be executing queries.

Requests are waiting to obtain a DB connection.

So:

Request
↓
Needs DB connection
↓
Pool exhausted
↓
Wait
↓
DB query executes
↓
Response

That waiting time becomes API latency.

Step 7 — Check downstream services

This is where APM/distributed tracing becomes extremely useful.

Suppose:

Order API
↓
Payment Service
↓
Inventory Service
↓
Database

Your API latency is:

4 seconds

But your application itself only spends:

100 ms

Then:

Payment call = 3.5 sec
Inventory    = 200 ms
DB           = 100 ms

Now you know:

The Order API isn't necessarily the root cause. Payment Service is slow.

This is why distributed tracing is so useful.

Step 8 — Look at the trace

A trace might look conceptually like:

Order API
│
├── authentication       20 ms
│
├── DB query             80 ms
│
├── Payment Service     3,200 ms  ← 🚨
│      │
│      └── Payment DB   3,000 ms ← 🚨
│
└── response              20 ms

Now you don't blindly investigate every metric.

You follow the largest contributor to latency.

Step 9 — Check network

If application, DB, and downstream service metrics look normal, investigate network.

Look at:

Network latency
Connection errors
Packet loss
DNS latency
Load balancer latency
TLS handshake time

For example:

Application processing = 100 ms
Downstream response    = 2,000 ms

If downstream service itself says:

processing = 100 ms

then the remaining delay may be between services.

Step 10 — Check recent deployment/configuration

Finally, ask:

"What changed?"

Check:

Recent deployment
Configuration changes
Database changes
Infrastructure changes
Feature flags
Dependency versions
Traffic routing changes

Example:

10:00 → latency normal
10:15 → deployment
10:20 → latency increases

That is a huge clue.

You might compare:

Old version → 200 ms
New version → 2 sec

Then investigate the code/config introduced in the new version.

The important decision tree

Don't memorize 20 random metrics.

Think like this:

             API latency ↑
                    │
                    ▼
             Is traffic ↑?
              /          \
            YES           NO
             │             │
             ▼             ▼
       Check CPU       Check application
       Memory          threads/GC
       Threads              │
             │              │
             └──────┬───────┘
                    ▼
              Where is time
                 spent?
                    │
       ┌────────────┼─────────────┐
       ▼            ▼             ▼
      App           DB       Downstream
       │            │             │
    CPU/GC       queries       latency
    threads      pool          traces
       │            │             │
       └────────────┼─────────────┘
                    ▼
              Find bottleneck
                    │
                    ▼
             Check recent change
                    │
                    ▼
              Fix / rollback
What I want you to remember for interviews

When interviewer says:

"API latency suddenly increased. How will you troubleshoot?"

Don't say:

"I'll check CPU, memory, database, Kafka, network, logs..."

That sounds like you're randomly checking things.

Instead say:

"First I'll confirm the latency increase using P95/P99 and identify which endpoint is affected. Then I'll check request rate to determine whether there was a traffic change. I'll correlate latency with CPU, memory/GC, thread-pool utilization and connection pools to identify resource saturation. Using APM/distributed tracing, I'll break down the request and determine whether the time is spent inside the service, in the database, or in a downstream service. Once I identify the bottleneck, I'll check logs and recent deployments/configuration changes to determine the root cause."

That is the correct troubleshooting flow.

And one crucial distinction

For this scenario, your first four questions are:

1. Is latency actually increasing?
   → P95/P99

2. Is traffic increasing?
   → RPS

3. Is the service getting overloaded?
   → CPU / memory / GC / threads / pools

4. Where exactly is the request spending time?
   → APM/distributed tracing



| #  | Latency issue                       | Typical reason                                        |
| -- | ----------------------------------- | ----------------------------------------------------- |
| 1  | **Traffic spike / overload**        | Too many requests for available capacity              |
| 2  | **CPU saturation**                  | Application has insufficient CPU                      |
| 3  | **Memory pressure / GC**            | JVM spends time doing garbage collection              |
| 4  | **Thread pool exhaustion**          | Requests wait for available threads                   |
| 5  | **DB slow queries**                 | SQL/query execution becomes slow                      |
| 6  | **DB connection pool exhaustion**   | Requests wait for DB connections                      |
| 7  | **DB locks/contention**             | Transactions wait for other transactions              |
| 8  | **Downstream service slow**         | Service B takes too long                              |
| 9  | **Downstream service overloaded**   | Dependency itself is saturated                        |
| 10 | **Network latency**                 | Network communication becomes slow                    |
| 11 | **External API slow**               | Third-party API responds slowly                       |
| 12 | **Retries causing latency**         | Failed/slow calls are retried multiple times          |
| 13 | **Cache miss / cache problem**      | Requests suddenly hit DB instead of Redis/cache       |
| 14 | **Load imbalance**                  | One pod/instance gets disproportionately more traffic |
| 15 | **Insufficient instances**          | Not enough pods/instances to handle load              |
| 16 | **Kubernetes resource throttling**  | CPU/memory limits restrict the application            |
| 17 | **Connection problems**             | HTTP/TCP connection establishment becomes slow        |
| 18 | **Application code regression**     | New code performs expensive processing                |
| 19 | **Lock/synchronization contention** | Java threads wait for shared resources                |
| 20 | **Queue/backlog buildup**           | Requests/tasks wait before processing                 |


CPU SAturation :

| Scenario                                      | Identify                                                                                             | Fix                                                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **1. Traffic spike**                          | RPS increases → CPU reaches 90–100% → latency rises.                                                 | Scale horizontally/add pods; use HPA and rate limiting if needed.                              |
| **2. CPU-heavy application code**             | CPU is high even without a major traffic increase; profiling shows expensive code.                   | Optimize the code/algorithm and remove unnecessary CPU-intensive processing.                   |
| **3. CPU throttling**                         | Container hits its CPU limit and throttling increases; latency rises.                                | Review and increase CPU limits/requests appropriately.                                         |
| **4. Insufficient pod capacity**              | All pods consistently run at high CPU and cannot handle current RPS.                                 | Increase replicas and/or provision more cluster capacity.                                      |
| **5. Uneven load distribution**               | One pod has very high CPU/RPS while other pods are relatively idle.                                  | Fix load balancing/sticky-session issues and investigate why traffic is uneven.                |
| **6. Infinite/long-running processing**       | CPU suddenly goes high because requests/tasks are stuck in expensive processing or loops.            | Identify the code using profiling/thread dumps, then fix/terminate the problematic processing. |
| **7. Too much serialization/deserialization** | CPU rises with large/complex request or response payloads; traces/profiling show serialization work. | Reduce payload size, optimize serialization, or change the data-processing approach.           |
| **8. Excessive logging/processing**           | CPU rises during high-volume requests and profiling shows logging/string processing consuming CPU.   | Reduce unnecessary logs, avoid expensive log construction, and use appropriate log levels.     |
