1. Check traffic

        First check whether request volume increased.

        Request rate / RPS
        Number of concurrent requests
        P95/P99 latency

Even with normal CPU, higher concurrency can cause waiting elsewhere.

2. Check thread pool

CPU can be normal while application threads are blocked/waiting.

Check:

        Active threads
        Pool utilization
        Queue size
        Rejected tasks
        Thread dumps

For example:

    
API thread → waiting for DB connection

CPU remains low, but requests become slow.

3. Check database

This is one of my first suspects.

Check:

DB query latency
Slow queries
Connection pool usage
Connection pool exhaustion
DB CPU/load
Lock contention
Deadlocks
Missing indexes
N+1 queries

Example:

API → DB query → 2 seconds

The application CPU can remain completely normal.

4. Check downstream services

If the API calls another microservice:

Service A → Service B

Check:

Service B latency
Connection timeout
Read timeout
Connection pool
Number of concurrent calls
Service B errors
Retries

A slow Service B makes Service A slow even though Service A's CPU is normal.

5. Check external dependencies

For example:

API → Payment API

or

API → Redis

or

API → Kafka

Check dependency latency and connection problems.

6. Check thread dumps / waiting states

This is especially useful when CPU is normal.

You might discover many threads in:

WAITING
TIMED_WAITING
BLOCKED

For example:

200 request threads
↓
150 waiting for DB connection
↓
30 waiting for downstream service
↓
20 processing

CPU could be only 30%, while P99 latency is extremely high.



-----------------------------------------------------------------------------------------------------------------



Complete investigation flow

1. Confirm the latency problem

Check P50, P95, P99.
Compare with the normal baseline.
Identify whether all APIs or only one endpoint is affected.
Check when the latency started.
Check whether the problem affects all instances or only some.
Latency ↑
↓
Which API?
Which instances?
When did it start?
P50/P95/P99?

2. Check traffic

Check:

RPS/request rate
Concurrent requests
Traffic per instance
Traffic distribution

Then ask:

Why did traffic increase?

Possible reasons:

Genuine user increase
Business event
Client retrying
Retry storm
Misbehaving client
Duplicate requests
Malicious traffic

If traffic is normal, move on.

3. Check CPU

You already know CPU is normal, so:

CPU saturation is probably not the bottleneck.

But verify CPU per instance, because one instance could be abnormal while the average is normal.

4. Check thread-pool metrics

Look at:

Active threads
Maximum threads
Queue size
Pool utilization
Rejected tasks

If thread pool is saturated:

Find why the threads are occupied.

Take a thread dump if necessary.

Look for:

RUNNABLE       → doing CPU work
WAITING        → waiting for something
TIMED_WAITING  → waiting with timeout
BLOCKED        → waiting for a lock

5. Check DB connection-pool metrics

For example, HikariCP:

Active connections
Idle connections
Maximum pool size
Pending/waiting threads
Connection acquisition time

If:

Active connections = max
Waiting threads ↑

then requests are waiting for DB connections.

6. Check database performance

If DB is involved, check:

Query latency
Slow queries
DB CPU
DB memory
DB I/O
Connection count
Lock contention
Deadlocks
Transaction duration

If a query is slow, investigate:

Slow query
↓
EXPLAIN
↓
Index?
N+1?
Bad execution plan?
Large scan?
Too much data?
Lock?

7. Check downstream services

For every important dependency:

Service A → Service B
Service A → Redis
Service A → External API

Check:

Downstream latency
Error rate
Timeout rate
Connection pool
Connection failures
Retry count
Circuit-breaker state

If B is slow:

Continue investigating Service B using the same process.

8. Check network

If DB and downstream application metrics look healthy, check:

Network latency
Packet loss
Connection failures
DNS resolution
TLS/SSL handshake
Connection establishment time

You can have:

Service B healthy
DB healthy
CPU healthy

but still:

A → B network latency ↑

9. Check JVM memory and GC

Even with normal CPU, check:

Heap usage
Old Gen
Allocation rate
GC frequency
GC pause duration
Full GC

For example:

Allocation ↑
↓
GC frequency ↑
↓
GC pauses ↑
↓
Requests delayed
↓
Latency ↑

If GC is high, investigate why allocations increased.

10. Check retries

This is very important in microservices.

Check:

Retry count
Timeout count
Circuit-breaker events
Downstream errors

Example:

A → B
↓ timeout
A → B retry
↓ timeout
A → B retry

One user request may become three downstream requests.

That can create a retry storm and increase latency.

11. Check queues / Kafka if applicable

If asynchronous processing is involved:

Queue depth
Consumer lag
Producer rate
Consumer rate
Processing time
Consumer errors

For Kafka:

Producer rate ↑
↓
Consumer rate normal
↓
Consumer lag ↑

Now processing is falling behind.

12. Check recent changes

Look at:

Application deployment
Configuration changes
Database changes
Infrastructure changes
Dependency/library upgrades
Gateway changes
Connection-pool configuration
Timeout/retry configuration

Especially check:

Did the latency increase immediately after a change?

13. Use distributed tracing

Now trace an actual slow request.

For example:

Request = 2.5 sec

Gateway       20 ms
Service A     30 ms
Database     100 ms
Service B   2,300 ms  ← bottleneck

Now you don't need to guess.

You know where the latency is.

14. Drill into the identified bottleneck

If tracing says DB:

DB
↓
Slow query?
Index?
N+1?
Execution plan?
Locks?
Connection pool?
DB capacity?

If tracing says Service B:

Service B
↓
B traffic?
B CPU?
B thread pool?
B DB?
B downstream?
B connection pool?

If tracing says network:

Network
↓
DNS?
Connection establishment?
TLS?
Packet loss?
Network latency?

If tracing says application processing:

Application
↓
Thread dump
↓
CPU profile
↓
Expensive operation?
Large payload?
Serialization?
Regex?
Loop?

15. Decide whether it is a capacity problem

Only after finding the bottleneck ask:

Can this component handle the current load?

If application tier is the bottleneck:

Horizontal scaling
Autoscaling
Optimize processing
Increase appropriate pool size

If DB is the bottleneck:

Optimize queries
Indexes
Caching
Read replicas
DB scaling

If downstream is the bottleneck:

Caching
Reduce calls
Parallelize independent calls
Rate limiting
Circuit breaker
Backpressure

16. Apply the fix

Examples:

Slow DB query → optimize/index
DB pool exhausted → tune pool carefully
Downstream slow → timeout/circuit breaker/cache
Thread pool exhausted → fix blocking dependency
Traffic spike → scale/rate limit
Retry storm → exponential backoff + jitter
GC pressure → reduce allocations/tune JVM
Network issue → fix infrastructure

17. Verify the fix

After fixing:

P50/P95/P99
Error rate
Traffic
CPU
Thread pool
DB latency
DB pool
Downstream latency
GC
Queue/lag

Confirm that latency returns to the baseline.

18. Prevent recurrence

Finally:

Add alerts
Add dashboards
Set appropriate timeouts
Configure retries correctly
Add circuit breakers
Capacity planning
Autoscaling
Load testing
Improve tracing
Document the incident/root cause