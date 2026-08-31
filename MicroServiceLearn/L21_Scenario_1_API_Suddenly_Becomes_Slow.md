Scenario 1: API suddenly becomes slow


      Step 1 → Confirm the problem
      Step 2 → Monitoring & metrics
      Step 3 → APM
      Step 4 → Logs
      Step 5 → Distributed tracing
      Step 6 → Database
      Step 7 → Downstream services
      Step 8 → JVM / threads / CPU / memory
      Step 9 → Network
      Step 10 → Fix + prevention


--------------------------------------------------------------------------------------------------------------------------------------------

1. Confirming the problem :

Check the API directly

      Call the API and measure its response time.

For example:

      GET /orders/123


      Expected: 200 ms
      Actual:   8 seconds

You can use:

      Postman
      curl
      Browser (for simple GET APIs)
      Load-testing tools

But this only tells us our current request is slow.




Case 1: API is working normally now

Example:

      You call /orders/123
      Response: 200 OK
      Response time: 180 ms

But the user reported:

      "It was taking 8 seconds 10 minutes ago."

      Don't conclude "there is no problem."

The issue may be intermittent.

So we check monitoring/metrics for the historical time period:

      10:00 → 200 ms
      10:10 → 8 sec  ← incident
      10:20 → 200 ms
      10:30 → 190 ms  ← now

Now we know:

      The problem happened, but it has recovered.

      Then we investigate what happened around 10:10.

Case 2: API is still slow

You call:

      /orders/123

Response time: 8 seconds

      Now we have reproduced the problem.
      
      So we can continue investigating while the problem is happening.

We move toward:

      API slow
      ↓
      Monitoring / Metrics
      ↓
      APM
      ↓
      Logs
      ↓
      Tracing
      ↓
      Database / Downstream / JVM / Network

The important interview point

"First, I would try to reproduce the issue. If it is currently working, I would check historical monitoring and metrics to determine whether the reported latency actually occurred and whether it was intermittent. 
If it is still slow, I would investigate the live request."


------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Step 2 → Monitoring & metrics

For an API suddenly becoming slow, while doing monitoring, focus on these 5 main things:

| # | What to monitor            | What you're checking                      |
| - | -------------------------- | ----------------------------------------- |
| 1 | **Latency**                | How long API requests take                |
| 2 | **Traffic / Request rate** | How many requests are coming              |
| 3 | **Error rate**             | Are requests failing?                     |
| 4 | **CPU**                    | Is the application CPU overloaded?        |
| 5 | **Memory**                 | Is the application running out of memory? |


1. Latency
      
         Normal: 200 ms
         Now:    8 seconds ❌

Confirms how severe the slowdown is.

2. Traffic
      
         Normal: 500 req/sec
         Now:    500 req/sec

or:

         Normal: 500 req/sec
         Now:    5,000 req/sec ❌

Tells us whether a traffic spike could be causing the problem.

3. Error rate
      
         Normal: 0.1%
         Now:    0.1%

or:

Normal: 0.1%
Now:    15% ❌

Tells us whether the slowdown is accompanied by failures.

4. CPU
      
         Normal: 40%
         Now:    95% ❌

High CPU can cause requests to wait for processing.

5. Memory
      
         Normal: 60%
         Now:    95% ❌

High memory can lead to GC pressure, OOM, etc.

1. Request count is HIGH

Example:

      Normal: 500 req/sec
      Current: 5,000 req/sec

What does it mean?

      There is a traffic spike.

What do we do?

We investigate:

      Traffic spike
      ↓
      Why did traffic increase?
      ↓
      Normal business traffic?
      OR
      Unexpected traffic?

Then check whether our application has enough capacity.

Possible actions:

      Scale up/out the application.
      Check autoscaling.
      Check for abnormal/repeated requests.
      Check whether a recent change caused request amplification.
      Rate limiting if appropriate.

2. CPU is HIGH

Example:

CPU: 40% → 95%
What does it mean?

The application/server is using a lot of CPU.

What do we do?

First determine what is consuming CPU.

We investigate:

      High CPU
      ↓
      Which instance?
      ↓
      Which process/thread?
      ↓
      Why is CPU being consumed?

Possible causes:

      Traffic increase
      CPU-intensive code
      Infinite/very expensive loop
      Excessive serialization/deserialization
      Heavy computation
      GC activity

Possible actions depend on the root cause.

Don't simply restart the server immediately, because that may hide the real problem.

3. Memory is HIGH

Example:

Memory: 60% → 95%
What does it mean?

The application may be under memory pressure.

We investigate:

      High Memory
      ↓
      Heap usage?
      ↓
      GC activity?
      ↓
      Objects accumulating?
      ↓
      Memory leak?

Possible causes:

      Memory leak
      Large objects/data being loaded
      Too many requests holding objects
      Excessive caching
      Heap too small
      GC pressure

Possible actions depend on the root cause.

4. Error rate is HIGH

Example:

      Normal: 0.2%
      Current: 15%

Now we have:

      Latency ↑
      Errors ↑

We check:

      What HTTP status codes are increasing?
      4xx or 5xx?
      Which API?
      What exceptions are occurring?
      Application logs
      Recent deployments/configuration changes

For example:

      500 errors ↑
      ↓
      Check application logs
      ↓
      Find exception
      ↓
      Find root cause

5. Everything looks NORMAL

This is also very important.

      Latency       ↑ ❌
      Requests      Normal ✅
      CPU           Normal ✅
      Memory        Normal ✅
      Errors        Normal ✅

Now monitoring hasn't given us the cause.

So we go deeper:

        Monitoring
            ↓
       No obvious clue
            ↓
           APM
            ↓
       Distributed Trace
            ↓
┌────────┼─────────┐
↓        ↓         ↓
Database  Downstream  Code


----------------------------------------------------------------------------------------------------------------------------------------


3. What is APM?

      APM = Application Performance Monitoring.

Monitoring tells us:

      "The API is slow."

APM helps us answer:

      "Why is this API slow, and where is the time being spent?"

Simple example

Monitoring might show:

      /orders
      Latency = 8 seconds ❌

But it doesn't tell us where those 8 seconds went.

APM can break it down:

      /orders
      |
      ├── Application code → 100 ms
      ├── Database        → 6.5 sec ❌
      ├── Payment service  → 1.2 sec
      └── Other            → 200 ms
      ───────
      8 sec

Now we immediately have a direction:

The database is taking 6.5 seconds.

What does APM typically show?

For each request, APM can provide things like:

      1. Response time
         /orders → 8 sec
         2. Breakdown of time
            Controller      100 ms
            Service         200 ms
            Database        6.5 sec
            HTTP call       1.2 sec
         3. Errors
            /orders
            500 errors ↑
            NullPointerException
         4. JVM/application information

Depending on the APM product, you can see:

      CPU
      Memory
      GC
      Threads
      Monitoring vs APM

This distinction is very important for interviews.

Monitoring
"Something is wrong."

Example:

      /orders latency = 8 sec
      CPU = 40%
      Memory = 60%
      APM
"Here is where the request is spending its time."

Example:

      /orders
      ↓
      Controller → 100 ms
      ↓
      Service → 200 ms
      ↓
      DB → 6.5 sec ❌

So:

      Monitoring gives us the high-level picture. APM gives us application-level details.

-------------------------------------------------------------------------------------------------------------------------------------------

4. Logs

         A log is a record of something that happened inside the application.

For example:

      2026-08-16 20:10:15
      OrderService
      Fetching order 101

Or an error:

      2026-08-16 20:10:16
ERROR
      
      Database connection timeout
      Monitoring vs APM vs Logs

This distinction is important:

      Monitoring
      ↓
      WHAT is wrong?
      "API latency is 5 seconds"
      

      APM / Tracing
      ↓
      WHERE is it slow?
      "Database operation took 4.8 seconds"


      Logs
      ↓
      WHY did it happen?
      "Connection acquisition timed out"

That's the relationship we're building.

In our scenario

We have:

      API suddenly slow
      ↓
      Monitoring
      ↓
      Latency increased
      ↓
      APM / Trace
      ↓
      Database operation is slow
      ↓
      LOGS ← We are here

Now we want to find what happened around that database operation.

What do we look for in logs?

1. Timestamp

First, correlate the time.

Suppose Jaeger says:

      Request:
      20:10:15
      Duration:
      5 seconds

Don't search the entire log file blindly.

Look around:

      20:10:15
      20:10:16
      20:10:17
      20:10:18
      20:10:19
2. Log level

Common levels:

      TRACE
      DEBUG
      INFO
      WARN
      ERROR

For production troubleshooting, ERROR and WARN are usually the first places to look.

Example:

      ERROR Database connection timeout

3. Exception / error message

Suppose the log says:

      ERROR
      SQLTransientConnectionException:
      Connection is not available

Now we have a strong clue.

We can investigate:

      Why couldn't the application obtain a DB connection?
      
      That takes us toward connection-pool investigation.

4. Stack trace

         The exception message tells us what failed.

The stack trace helps us understand where it failed in our code.

For example:

      OrderController
      ↓
      OrderService
      ↓
      OrderRepository
      ↓
      HikariDataSource
      ↓
      Connection timeout

Now we can correlate the application path with our Jaeger trace.

Very important: Trace ID

      This is one of the most important things to understand for Microservices interviews.

Suppose a request gets:

      Trace ID:
      abc123

That same trace ID can be included in application logs.

Then we can search:

      abc123

and find all logs belonging to that particular request.

So:

      Jaeger
      ↓
      Trace ID = abc123
      ↓
      Search logs for abc123
      ↓
      Find exactly what happened
      
      This is called correlation between traces and logs.

In Microservices this becomes extremely useful

Imagine:

      Client
      ↓
      Order Service
      ↓
      Payment Service
      ↓
      Inventory Service

Request gets:

      Trace ID = abc123
      
      You can follow:
      
      Order Service
      Trace abc123
      ↓
      Payment Service
      Trace abc123
      ↓
      Inventory Service
      Trace abc123
      
      And search logs using that trace ID.

That's how you avoid trying to understand thousands of unrelated log lines.

So our Step 4 flow is
      
      APM / Trace
      ↓
      Identify slow component
      ↓
      Get timestamp + Trace ID
      ↓
      Search logs
      ↓
      Check WARN / ERROR
      ↓
      Check exception
      ↓
      Check stack trace
      ↓
      Correlate with trace
      ↓
      Understand WHY
For our current scenario

We haven't reached the root cause yet.

We currently know:

      /orders/{id}
      ↓
      Trace
      ↓
      Database spans
      ↓
      Now investigate logs


------------------------------------------------------------------------------------------------------------------------------------------------------


Step 5 — Identify root cause

We combine the evidence:

      Monitoring → WHAT?
      APM        → WHERE?
      Logs       → WHY?

Then determine the actual root cause.

For our scenario:

      API slow
      ↓
      Monitoring → latency high
      ↓
      APM → DB operation is slow
      ↓
      Logs → connection timeout
      ↓
      Root cause → DB connection pool exhausted

That's Step 5.

Step 6 — Fix

      The fix depends entirely on the root cause.

If:

      Connection pool exhausted

we investigate/fix the connection issue.

If instead:

      Slow SQL query

we might investigate:

      Execution plan
      Indexes
      Locks
      Query design

So there is no universal Step 6 answer.

Step 7 — Verify

After fixing:

      Before:
      API = 5 sec ❌


After:

      API = 100 ms ✅

We verify:

      latency
      error rate
      DB time
      CPU/memory
      affected API

Basically:

      Did the original problem actually disappear?

Step 8 — Prevent recurrence

      Again, based on what caused it.

For example, if the root cause was a DB connection issue:

      Alert on connection pool usage
      Monitor DB connections
      Improve connection handling
      Load testing

If the root cause was a missing index:

      Query-performance monitoring
      DB review
      Index analysis
      Performance testing


--------------------------------------------------------`