Common reasons a 500 occurs

1. Unhandled application exception

Example:

    User user = userRepository.findById(id).get();
    If the user doesn't exist, an exception may occur.

    NoSuchElementException
    ↓
    not handled
    ↓
    500

2. Database failure

Your API tries:

    Order Service
    ↓
    MySQL

    But MySQL is unavailable, connection pool is exhausted, query fails, etc.

    DB failure
    ↓
    exception
    ↓
    API cannot complete request
    ↓
    500

3. Downstream microservice failure

For example:

    Order Service
    ↓
    Payment Service
    ↓
    Payment Service fails

If Order Service doesn't handle that failure properly, it may return:

500 Internal Server Error

4. External API failure
    
       Our API
       ↓
       External API
       ↓
       timeout / failure

If the failure isn't handled appropriately, your API may return 500.

5. Configuration problems

For example:

    Application
    ↓
    DB_URL = wrong

or:

    Missing environment variable
    Missing secret
    Invalid configuration

The application may fail while processing a request.

6. Resource exhaustion

For example:

    Traffic increases
    ↓
    Too many DB connections
    ↓
    Connection pool exhausted
    ↓
    Requests fail
    ↓
    500

Similarly, memory/resource problems can lead to application failures.



----------------------------------------------------------------------------------------------------------------------------------------------



Step 1 — Confirm the 500s in Monitoring

    First look at your monitoring dashboard.

You want to establish:

Is this really an increase in 500s?

Look at:

    Request rate
    Error rate
    Latency
    CPU
    Memory
    Instance/pod health

For example:

    Before:
    Requests: 1,000 RPS
    5xx:      2 RPS
    
    
    Now:
    Requests: 1,000 RPS
    5xx:      300 RPS

Clearly something changed.

But don't immediately assume the application code is broken.



----------

















-------------------------------------------------------------------------------------------------------------------------------------------------------------------

Step 2 — Find the scope

Now ask:

    Is every API affected?

For example:

        POST /orders     → 500
        GET /orders      → 200
        GET /products    → 200

Then the problem is probably specific to the order flow.

Or:

    GET /orders      → 500
    GET /products    → 500
    POST /payment    → 500

Multiple APIs failing could indicate:

    shared database problem
    shared downstream service
    infrastructure problem
    bad deployment
    configuration problem
    authentication/service dependency issue

Step 3 — Check whether it is instance-specific

Suppose you have:

    Pod-1 → 200
    Pod-2 → 200
    Pod-3 → 500
    Pod-4 → 200

This is a very important clue.

The problem may be specific to Pod-3:

    corrupted state
    bad configuration
    memory issue
    connection pool problem
    bad instance
    incomplete deployment

You can remove the unhealthy instance from traffic and investigate it.

But suppose:

    Pod-1 → 500
    Pod-2 → 500
    Pod-3 → 500
    Pod-4 → 500

Then it's probably a common dependency or application-level issue.

Step 4 — Check deployment/change history

Ask:

    "Did anything change immediately before the errors started?"

Check:

    application deployment
    configuration change
    database change
    infrastructure change
    feature flag
    dependency upgrade

Example:

    10:00 → deployment
    10:03 → 500 errors start

That's a very strong signal.

    In production, if the new deployment is clearly responsible, rollback may be the fastest mitigation.

Step 5 — Go to APM

Now we want to understand where the request failed.

Suppose:

    Client
    ↓
    API Gateway
    ↓
    Order Service
    ↓
    Payment Service
    ↓
    Database

APM/distributed tracing can show:

Order Service
    
    20 ms
    ↓
    Payment Service
    3000 ms
    ↓
    Payment Service → 500

Now you know the Order Service may only be propagating a downstream failure.

This is why we don't immediately blame the API itself.

Step 6 — Check application logs

Now correlate the failing request using:

    timestamp
    trace ID
    request ID
    correlation ID

You might see:

    ERROR PaymentService
    java.sql.SQLTransientConnectionException:
    Connection pool exhausted

Now we've moved from:

    "API returns 500"

to:

    "Payment Service is returning 500 because its database connection pool is exhausted."

That's a much stronger interview answer.

Step 7 — Determine the actual root cause

There are several possibilities.

Case A — Application exception

    NullPointerException
    IllegalArgumentException

Check the recent code change and fix/rollback.

Case B — Database failure
    
    Connection timeout
    Too many connections
    Deadlock
    Database unavailable

Investigate DB health, connections, locks, queries, etc.

Case C — Downstream service failure
    
    Order Service
    ↓
    Payment Service → 500
    
    Investigate Payment Service rather than blindly changing Order Service.

Case D — External API failure
    
    Our API
    ↓
    External provider
    ↓
    500 / timeout
    
    Check timeout, retry, circuit breaker and fallback behavior.

Case E — Configuration problem

For example:

    Database URL changed
    Secret missing
    Environment variable incorrect

The application may start successfully but fail when processing requests.

Step 8 — Mitigate first, then permanently fix

    This is important in production interviews.
    
    You don't necessarily wait for the complete root-cause fix before reducing customer impact.

Depending on the situation:

    rollback deployment
    remove unhealthy instance
    disable feature flag
    scale service if resource exhaustion is involved
    activate fallback
    fix configuration
    restore dependency
    temporarily reduce problematic traffic

Then perform the permanent fix.

Step 9 — Verify

After remediation:

    500 rate
    ↓
    300 RPS
    ↓
    50 RPS
    ↓
    5 RPS
    ↓
    normal

Also check:

    latency
    throughput
    downstream errors
    logs
    affected endpoints
    instance health

Don't stop at:

"The deployment succeeded."

You need to verify the customer-facing error rate recovered.


---------------------------------------------------------------------------------------------------------------------------------------


Common flow
        
        500 errors increased
        ↓
        Check monitoring
        ↓
        Where are the 500s concentrated?
        ↓
        ┌──────────────┬──────────────┬──────────────┐
        Endpoint?      Pod?           All services?
        ↓             ↓                  ↓
        App flow       Instance        Shared dependency
        ↓             ↓                  ↓
        APM + logs    Pod/resources    DB/downstream/etc.

Example 1 — One endpoint has 500s
    
    POST /orders → 500 = 40%
    GET /orders  → 200

Then investigate:

    APM → trace → logs → code/database/downstream dependency

Maybe /orders has a new bug.

Example 2 — One pod has 500s
    
    Pod-1 → 200
    Pod-2 → 200
    Pod-3 → 500
    Pod-4 → 200

Now the mechanism is different.

Investigate:

    Pod logs
    CPU/memory
    restarts
    OOM
    configuration
    connection pools
    deployment state

Potential immediate mitigation:

Remove/restart the unhealthy pod if appropriate, then investigate why it became unhealthy.

Example 3 — All pods have 500s
    
    Pod-1 → 500
    Pod-2 → 500
    Pod-3 → 500
    Pod-4 → 500

Now don't waste time investigating individual pods.

Look for shared dependencies:

                 ┌─ Pod 1 ─┐
                 ├─ Pod 2 ─┤
    Gateway ─────┼─ Pod 3 ─┼──→ Database
    └─ Pod 4 ─┘       ↓
    FAILURE

Check:

    Database
    Redis
    Kafka
    downstream microservices
    external APIs
    shared configuration
    certificates/secrets
    recent deployment

Example 4 — 500s started immediately after deployment
    
    10:00 deployment
    10:02 500s increase

That's a strong deployment correlation.

Your first mitigation consideration is:

    Rollback, if the evidence supports the deployment as the cause.

Then investigate the exact code/configuration issue.

Example 5 — 500s happen only for one client
    
    Client A → 200
    Client B → 200
    Client C → 500

Now investigate:

    request pattern
    payload
    authentication
    headers
    unusual traffic
    client-specific retries

Don't assume the entire API is broken.


--------------------------------------------------------------------------------------------------------------------------------------

METRICS :

   1. "High traffic can overload a service and exhaust resources such as CPU, threads, memory, or database connections. That can cause internal exceptions or dependency failures, which may result in 500 responses depending on how the application handles those failures."

    2. 5xx Error Rate

            This is the most direct metric for our scenario.

            Example:
            
            5xx = 0.5% → 30%
            What does it tell us?
            
            How much of our traffic is failing with server-side errors.
            
            Now break it down.
            
            POST /orders → 40%
            GET /orders  → 0%
            GET /users   → 0%
            What did we learn?
            
            Only the order creation flow is failing.
            
            Solution?
            
            Don't restart everything.
            
            Go to the POST /orders request flow in APM/tracing.



3. Latency

Look at:

p50
p95
p99

For example:

p95: 200ms → 5 seconds
Why?

Because latency tells us whether requests are becoming slow before they fail.

Pattern A
RPS       ↑
Latency   ↑
5xx       ↑
CPU       ↑

This looks like:

Overload/resource saturation

Possible solution:

scale
reduce traffic
rate limit
investigate resource bottleneck
Pattern B
RPS       normal
Latency   normal
5xx       suddenly ↑

This doesn't look like overload.

Possible causes:

application exception
bad deployment
configuration issue
database error
downstream failure

Next step:

APM + logs.

4. CPU

Example:

CPU: 40% → 98%
What does it tell us?

Your service is consuming almost all available CPU.

But high CPU doesn't automatically mean it's the root cause.

You need correlation.

If:

RPS ↑
CPU ↑
Latency ↑
500 ↑

Then high CPU is probably contributing.

Solution?

Depending on the cause:

Scale horizontally
Increase CPU capacity
Find CPU-heavy code
Investigate expensive queries
Check for traffic spikes
5. Memory

Example:

Memory:
50% → 95% → OOMKilled
What does it tell us?

The application is running out of memory.

Possible causes:

memory leak
huge payloads
excessive caching
too many objects
traffic increase
Solution?

Immediate:

Restart/replace unhealthy instances if appropriate.

Permanent:

Find the memory leak/resource issue and fix it; tune memory limits if justified.

6. Pod/Instance Restarts

Suppose:

Pod-1 → 0 restarts
Pod-2 → 0
Pod-3 → 15
Pod-4 → 0
What does it tell us?

Pod-3 is unhealthy.

Maybe:

OOM
Crash
Failed health check
Application exception
What do you do?

First:

Remove/restart the unhealthy instance if appropriate.

Then investigate:

Pod logs
OOM events
CPU
Memory
deployment/configuration
7. Database Connection Pool

This is a very important microservices metric.

Suppose:

Max connections = 100
Active = 100
Idle = 0

And API 500s increased.

What does it tell us?

The application cannot obtain database connections.

Requests may do:

API
↓
get DB connection
↓
NO CONNECTION AVAILABLE
↓
exception/timeout
↓
500
Solution?

Immediate:

reduce traffic if necessary
scale appropriately
recover DB connectivity

Permanent:

investigate connection leaks
tune pool size
optimize slow queries
investigate DB capacity
8. Database Latency

Suppose:

API latency: 200ms → 4 sec
DB latency:  100ms → 3.5 sec
What does it tell us?

The database is taking much longer.

Then:

API becomes slow
↓
DB is slow
↓
requests timeout/fail
↓
possibly 500
Solution?

Investigate:

slow queries
missing indexes
locks
deadlocks
DB CPU
DB connections
DB capacity
9. Downstream Service 5xx

Suppose:

Order Service → 500 = 30%


Payment Service → 500 = 35%
What does it tell us?

Order Service may not actually be the original problem.

The chain could be:

Client
↓
Order Service
↓
Payment Service
↓
500

Order Service may simply be propagating the failure.

Solution?

Move investigation to Payment Service.

10. Timeouts

Suppose:

Payment Service timeout count ↑

And Order Service 500s increase.

Flow:

Order
↓
Payment
↓
Payment takes too long
↓
timeout
↓
Order fails
↓
500
Solution?

Investigate why Payment is slow.

And use:

appropriate timeout
circuit breaker
fallback
controlled retry

rather than allowing requests to pile up.

Now put everything together

Imagine monitoring gives you:

RPS        ↑↑
5xx        ↑↑
p95        ↑↑
CPU        ↑↑
Memory     normal
Restarts   normal
DB pool    normal

What do you think?

Answer:

Traffic spike → CPU saturation → latency increases → requests start failing.

Likely overload.

Action:
Scale service
+
investigate traffic source
+
rate limit if necessary
+
check whether retries amplified traffic
