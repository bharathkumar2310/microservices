1. High Traffic / Request Rate

Suppose monitoring shows:

        Normal traffic: 500 requests/sec
        Current traffic: 5,000 requests/sec

The API is slow.

Our question is:

Why did the request rate increase?

Step 1 — Find where the traffic increased

First break the traffic down by:

        Overall traffic
        ↓
        Which service?
        ↓
        Which API/endpoint?
        ↓
        Which client/source?

Example:

    /orders       → 4,500 req/sec ❌
    /users        →   300 req/sec
    /products     →   200 req/sec

Now we know /orders is responsible for most of the increase.

Step 2 — Check whether the traffic increase is legitimate

Now ask:

    Why are 5,000 requests coming to /orders?

There are several possibilities.

A. Genuine traffic increase

        For example, a sale/event causes many more users to call the API.
        
        500 → 5,000 req/sec
        
        Nothing is technically wrong with the requests.
        
        What do we do?
        
        Increase application capacity:
        
        2 instances
        ↓
        5 instances
        
        Use horizontal scaling if appropriate.

Also verify:

    Load balancer distribution
    Autoscaling
    CPU/memory after scaling
    Database capacity
    Downstream capacity

Because simply adding application instances won't help if the database becomes the bottleneck.

B. Client is sending requests repeatedly

Example:

        Client
        ↓
        API request
        ↓
        No response quickly
        ↓
        Client retries
        ↓
        Another request
        ↓
        Another retry
        ↓
        ...
        
        You can get a retry storm.

Traffic becomes:

    500 req/sec → 5,000 req/sec
    What do we check?

Look at:

    Requests by client
    Retry patterns
    HTTP status codes
    Request timestamps
    API gateway/load-balancer metrics
    Client logs if available

If one client is responsible for:

        Client A → 4,500 req/sec
        Client B → 200 req/sec
        Client C → 300 req/sec

we investigate Client A.

C. One API is being called excessively

Example:

        /orders
        ↓
        500 req/sec normally
        ↓
        5,000 req/sec
        
        We check why that endpoint is being called so frequently.
        
        Possibilities:
        
        Bug in client application
        Polling too frequently
        Retry loop
        Duplicate requests
        Misconfigured job
        Step 3 — Check whether our system can handle the traffic

Even if the traffic is legitimate, we need to determine whether our infrastructure is overloaded.

Check:

        Traffic
        ↓
        CPU?
        Memory?
        Thread pool?
        Connection pool?
        Database?
        Downstream services?

For example:

        Traffic ↑
        ↓
        CPU ↑
        ↓
        Thread pool ↑
        ↓
        DB connections ↑
        ↓
        DB becomes slow
        ↓
        API latency ↑
        
        Now we have a possible chain of events.
        
        So what do we DO when traffic is high?
        
        We don't immediately just "increase servers."
        
        Our flow is:
        
        Traffic HIGH
        ↓
        Which API?
        ↓
        Which client?
        ↓
        Why did traffic increase?
        ↓
        ┌───────────────┬────────────────┐
        │ Legitimate    │ Unexpected     │
        │ traffic       │ traffic        │
        └───────┬───────┴────────┬───────┘
        ↓                ↓
        Scale capacity    Find source
        ↓                ↓
        Check DB/downstream   Retry/client bug
        ↓
        Monitor again
        Interview answer
        
        If asked:
        
        "Traffic suddenly increased and your API became slow. What would you do?"


---------------------------------------------------------------------------------------------------------------------------------------


2. CPU Usage.

        Suppose monitoring shows:

        Normal CPU: 40%
        Current CPU: 95%
        API latency: 200 ms → 8 sec

We need to answer two questions:

What should we check?
What should we do?

1. First: identify where CPU is high
    
       CPU 95%
       ↓
       Which server/pod/instance?
       ↓
       Is it one instance or all instances?

Example:

    Instance 1 → 95% ❌
    Instance 2 → 42% ✅
    Instance 3 → 40% ✅

Now the problem may be specific to Instance 1.

2. Check which process/thread is consuming CPU

For a Java application:

    High CPU
    ↓
    Find high-CPU process
    ↓
    Find high-CPU threads
    ↓
    Thread dump / profiling

Example:

    Thread-25 → 80% CPU ❌
    Thread-31 → 5%
    Others    → 15%

Now we investigate what Thread-25 is doing.

3. Find WHY that thread is consuming CPU

Possible causes:

    High CPU
    ├── Traffic increase
    ├── Expensive computation
    ├── Infinite/very large loop
    ├── Bad algorithm
    ├── Excessive serialization/deserialization
    └── GC activity

So we correlate CPU with other metrics.

For example:

            Traffic ↑
            CPU ↑

→ High traffic may be consuming the CPU.

Or:

    Traffic normal
    CPU ↑

→ Something inside the application may be consuming CPU.

Or:

    CPU ↑
    GC ↑

→ Garbage collection may be consuming CPU.

4. What do we do?

If traffic caused it:

    Scale horizontally

If a particular code path caused it:

Find inefficient code
    
    → optimize/fix
    
    If GC caused it:
    
    Investigate memory allocation / GC
    
    If one instance is abnormal:

Investigate that instance
    
    → remove/replace it if necessary
    → find why it became abnormal
    The complete CPU investigation
    CPU HIGH
    ↓
    Which instance?
    ↓
    Which process?
    ↓
    Which thread?
    ↓
    What is the thread doing?
    ↓
    Traffic? Code? GC? Other?
    ↓
    Fix root cause


----------------------------------------------------------------------------------------------------------------------------------------


Memory Usage.

Suppose:

Memory: 60% → 95%
Latency: 200 ms → 8 sec

We investigate why memory is high.

1. Find where memory is high
    
       Memory HIGH
       ↓
       Which instance/pod?
       ↓
       One instance or all?

Example:

    Instance 1 → 95% ❌
    Instance 2 → 60% ✅
    Instance 3 → 62% ✅

If only one instance is high, investigate that instance specifically.

2. Check JVM heap

For Java:

    Heap Used
    Heap Max

Example:

    Heap: 7.5 GB / 8 GB

Now we know the JVM is under memory pressure.

3. Check Garbage Collection

Look at:

    GC frequency
    GC pause duration
    Young GC
    Old/Full GC

Example:

Before:

    GC pause → 20 ms


During issue:
    
    GC pause → 2 sec
    GC → happening continuously

Now memory pressure is likely affecting latency.

4. If memory keeps increasing

Example:

    2 GB
    ↓
    4 GB
    ↓
    6 GB
    ↓
    7.5 GB
    ↓
    8 GB

This is suspicious.

We investigate what objects are consuming the heap.

Use:

    Heap dump
    JVM monitoring
    Profiling/APM

For example, we might discover:

    Huge List objects
    Large cache
    Unreleased objects

That can point toward a memory leak or excessive memory usage.

5. What do we do?

It depends on the cause.

Temporary protection:

    Scale/restart affected instance if necessary

But that's not the final solution.

Then:

        Memory HIGH
        ↓
        Heap?
        ↓
        GC?
        ↓
        Object allocation?
        ↓
        Heap dump
        ↓
        Find root cause
        ↓
        Fix code/configuration


------------------------------------------------------------------------------------------------------------------------------------


Error Rate.

Suppose:

        Normal error rate: 0.2%
        Current:           15%
        API latency:       8 sec

Now we investigate what errors are increasing and why.

1. Check which status codes increased

        Don't just look at "15% errors."
        
        Break it down:
        
        4xx → Client-side errors
        5xx → Server-side errors
        
        Example:
        
        400 → normal
        401 → normal
        404 → normal
        500 → ↑↑
        502 → normal
        503 → ↑
        504 → ↑↑

Now we know 500/503/504 are the problem.

2. Check which API is producing errors
    
       /orders   → 20% errors ❌
       /users    → 0.1%
       /products → 0.2%

Now focus on /orders.

3. Check application logs

        If it's a 500, look at the logs around the same timestamp.

Example:

        500
        ↓
        Application log
        ↓
        SQLException
        ↓
        Database connection timeout

Now we have a direction.

Another example:

    500
    ↓
    NullPointerException
    ↓
    Application code

4. Check whether errors are coming from a downstream service

Example:

        Our API
        ↓
        Payment Service
        ↓
        Timeout
        ↓
        Our API returns 504

So our API may not be the original problem.

We need to check the downstream service's:

    Error rate
    Latency
    Availability

5. What do we do?

    It depends on the error.

    500
    ↓
    Check logs → exception → fix application
    
    
    503
    ↓
    Check service availability/capacity
    
    
    504
    ↓
    Check timeout/downstream latency
    
    
    401/403
    ↓
    Check authentication/authorization
    
    
    400
    ↓
    Check client request
    Complete flow
    Error Rate HIGH
    ↓
    Which status code?
    ↓
    Which API?
    ↓
    Which instance?
    ↓
    Application logs
    ↓
    Exception?
    ↓
    Our code or downstream service?
    ↓
    Find root cause
    ↓
    Fix

------------------------------------------------------------------------------------------------------------------------------------


GC (Garbage Collection)

    GC is especially important for a Java/Spring Boot application.

Suppose we see:

        Latency: 200 ms → 8 sec
        Memory: 60% → 90%
        GC activity: ↑↑

Now we investigate whether GC is contributing to the latency.

1. Check GC frequency

Normally:

    GC: occasional

During the issue:

        GC
        GC
        GC
        GC
        GC

If GC is happening extremely frequently, the JVM may be under memory/allocation pressure.

2. Check GC pause duration

This is very important.

Example:

    Normal:
    GC pause → 20 ms


    During issue:
    GC pause → 2 seconds

During a significant GC pause, application processing can be delayed, depending on the collector and situation.

So:

    GC pause ↑
    ↓
    Request processing delayed
    ↓
    Latency ↑

3. Check which type of GC is happening

At a high level, look at:

    Young GC
    Old/Mixed GC
    Full GC

If we suddenly see frequent old/full collections, that's a stronger signal of memory pressure.

We then investigate why objects are surviving and accumulating.

4. Check heap before and after GC

This is very useful.

Suppose:

    Before GC: 7.8 GB
    After GC:  3.0 GB

A lot of garbage was collected.

But suppose:

    Before GC: 7.8 GB
    After GC:  7.2 GB

Only a small amount was reclaimed.

If this pattern continues:

    6 GB → 5.8 GB
    7 GB → 6.8 GB
    7.8 GB → 7.2 GB

We investigate whether too many objects are long-lived, potentially because of a leak, excessive caching, or some other allocation pattern.

5. What do we do?

If GC is abnormal:

    GC HIGH
    ↓
    Check pause duration
    ↓
    Check frequency
    ↓
    Check heap before/after GC
    ↓
    Find why memory is being allocated/retained

Possible causes:

    Too many objects being created
    Large objects
    Excessive caching
    Memory leak
    Heap too small
    High traffic causing high allocation

Then we fix the underlying cause, rather than simply increasing the heap.

Complete GC investigation
    
    GC abnormal
    ↓
    Frequency?
    ↓
    Pause duration?
    ↓
    GC type?
    ↓
    Heap before/after?
    ↓
    Objects being retained?
    ↓
    Heap dump / profiling
    ↓
    Root cause



-------------------------------------------------------------------------------------------------------------------------------------

Thread Count / Thread Pool

This is important because an API can become slow even when CPU and memory look completely normal.

Suppose:

    Latency:       200 ms → 8 sec
    CPU:           40%   ✅
    Memory:        60%   ✅
    Traffic:       Normal
    Errors:        Normal
    Threads:       100 → 500 ❌

Now we investigate the threads.

1. Check thread count

First ask:

    Are we creating too many threads?

Example:

    Normal: 100 threads
    Now:    500 threads

Too many threads can cause contention and scheduling overhead.

2. Check thread states

This is more important than just the count.

Threads can be:

    RUNNABLE
    WAITING
    BLOCKED
    TIMED_WAITING

Suppose:

    100 threads → RUNNABLE
    300 threads → WAITING
    100 threads → BLOCKED

Now we need to understand why they're waiting/blocking.

3. Check the thread pool

For a Spring Boot application, requests are typically handled by a server thread pool.

We check:

    Active threads
    Maximum threads
    Queued requests
    Idle threads

Example:

    Max threads:       200
    Active threads:    200
    Queue:             500

Now requests are waiting for an available thread.

That directly contributes to latency:

    Request arrives
    ↓
    No available thread
    ↓
    Request waits in queue
    ↓
    Thread becomes available
    ↓
    Request executes
    ↓
    Response
4. Why are threads blocked?

This is where we investigate further.

Common reasons:

    Threads blocked
    ↓
    Database connection waiting
    ↓
    Slow DB query
    
    or:
    
    Threads waiting
    ↓
    Slow downstream API
    
    or:
    
    Threads BLOCKED
    ↓
    Lock contention
    
    or:
    
    Threads waiting
    ↓
    Connection pool exhausted
    
    So thread problems often point us toward another underlying problem.

5. Take a Thread Dump

If threads are behaving abnormally, we can take a thread dump.

It shows things like:

    Thread-1 → WAITING
    Thread-2 → BLOCKED on lock
    Thread-3 → waiting for DB connection
    Thread-4 → executing code

Now we can investigate exactly what the threads are doing.



---------------------------------------------------------------------------------------------------------------------------

Database Connection Pool

Suppose we see:

    API Latency:       200 ms → 8 sec
    CPU:               Normal
    Memory:            Normal
    Traffic:            Normal
    DB Connections:    Pool exhausted ❌

Now we investigate why the application cannot get a database connection.

1. Check the connection pool

For example, suppose HikariCP is configured:

    Maximum connections: 20
    Active connections:  20
    Idle connections:    0
    Waiting requests:    100

This means:

    All 20 connections are currently being used, and new requests are waiting for a connection.

That waiting time can directly increase API latency.

2. Why are all connections being used?

This is the important question.

We investigate:

    Connection pool exhausted
    ↓
    Why?
    ┌────┼─────┬─────────┐
    ↓    ↓     ↓         ↓
    Slow   Long  Leak    Too many
    query  tx             requests

A. Slow database queries

Example:

    Request
    ↓
    Gets DB connection
    ↓
    Runs query
    ↓
    Query takes 8 seconds
    ↓
    Connection remains occupied

If many requests do this, the pool gets exhausted.

    B. Long transactions
    
    A connection may remain occupied because a transaction is taking too long.

    Transaction starts
    ↓
    DB connection occupied
    ↓
    Some long operation
    ↓
    Transaction doesn't finish
    ↓
    Connection unavailable

We investigate transaction duration and what is happening inside it.

C. Connection leak

    A connection is obtained but isn't properly returned to the pool.
    
    Conceptually:
    
    Get connection
    ↓
    Use connection
    ↓
    Should return connection
    ↓
    ❌ Doesn't return
    
    Over time:
    
    20 connections
    ↓
    19 available
    ↓
    15 available
    ↓
    5 available
    ↓
    0 available

Now the pool is exhausted.

D. Traffic increased

Maybe the pool is actually functioning correctly, but there are suddenly many more requests.

    Traffic ↑
    ↓
    DB requests ↑
    ↓
    Connections occupied ↑
    ↓
    Pool exhausted

So we correlate this with request rate.

3. What do we check internally?

For a Spring Boot application using HikariCP, we'd look at:

    Active connections
    Idle connections
    Maximum pool size
    Pending/waiting requests
    Connection acquisition time
    Connection usage time

Then investigate the database side:

    Slow queries?
    Locks?
    Long transactions?
    DB CPU?
    DB connection limit?
4. What do we do?

If slow queries:

    Find slow query
    → execution plan
    → indexes/query optimization
    
    If long transactions:
    
    Find transaction
    → understand why it's long
    → reduce transaction duration
    
    If connection leak:
    
    Find code that doesn't release connections
    → fix resource handling
    
    If legitimate high traffic:
    
    Scale application if appropriate
    +
    ensure DB can handle increased load
    
    Don't simply increase the connection pool from 20 → 100.
    
    If the database can only handle 20 efficiently, increasing the pool can make the database itself collapse.
    
    Complete investigation
    Connection Pool Exhausted
    ↓
    Active connections?
    ↓
    Why are connections occupied?
    ↓
    ┌────────┼─────────┬──────────┐
    ↓        ↓         ↓          ↓
    Slow DB  Long TX   Leak     Traffic
    query
    ↓
    Investigate root cause
    ↓
    Fix

The key idea:

A connection pool problem is often a symptom. We need to find what is holding the connections.




---------------------------------------------------------------------------------------------------------------------------------