1. API normally takes 200 ms but suddenly takes 5 seconds. How do you investigate?
   First, confirm the problem

I would first check:

Is the increase real or just a monitoring anomaly?
When did latency increase?
Is it affecting all requests or only some?
Did it happen suddenly or gradually?
Did a deployment/configuration change happen around that time?

Then check:

Metric	What I'm looking for
RPS	Did traffic suddenly increase?
P50	Is normal traffic affected?
P95	Are many requests slower?
P99	Are only tail requests affected?
Error rate	Are failures also increasing?
CPU	CPU saturation?
Memory	Memory pressure?
GC	Long GC pauses?
Thread pool	Queueing/starvation?
DB latency	Slow queries?
DB pool	Connection exhaustion?
Redis	Slow commands/timeouts?
Downstream latency	Another service slow?
Then use distributed tracing

Suppose:

Client
↓
Gateway
↓
Service A
↓
├── DB
├── Redis
└── Service B

A 5-second request might actually be:

Gateway       20 ms
Service A     30 ms
DB            100 ms
Redis         50 ms
Service B     4.7 sec
---------------------
Total         ~5 sec

Then the problem isn't Service A's CPU—it is Service B.

Possible causes
Traffic spike
Slow DB query
DB connection pool exhaustion
Downstream service latency
Thread pool exhaustion
GC pause
Network problem
Lock contention
Retry storm
Cache failure
Deployment/configuration change
External API slowdown
Important interview point

Don't immediately say:

"CPU is high, so increase CPU."

First identify where the 5 seconds are actually being spent.

2. CPU is normal but API latency increased. What do you check?

This is a very important scenario.

Normal CPU means:

The application isn't necessarily CPU-bound.

I would investigate waiting/blocked time.

Check thread pools

For example:

Request arrives
↓
Worker thread
↓
Waiting for DB connection
↓
DB connection unavailable
↓
Request waits

CPU can remain at 30%, while latency becomes 5 seconds.

Check:

Active threads
Idle threads
Queue size
Rejected tasks
Thread-pool saturation
Request concurrency
Check DB

Look at:

Query latency
Slow queries
Connection acquisition time
Active connections
Maximum connections
Connection pool exhaustion
Locks
Deadlocks
DB CPU
DB I/O
Check downstream services

Service A may be waiting:

A → B
A → C
A → External API
Check Redis/cache

Possible problems:

Redis latency
Redis unavailable
Cache miss rate increased
Cache eviction
Connection pool exhaustion
Check GC

CPU may appear normal overall, but:

Application threads
↓
GC pause
↓
Requests wait
Check network
DNS latency
Connection establishment
TLS handshake
Network packet loss
Connection timeout
Read timeout
Core idea

Normal CPU + high latency usually makes me investigate waiting, blocking, queueing and downstream dependencies.

3. P50 is normal but P99 increased significantly. What could be happening?

This means the majority of requests are still fast, but a small percentage are extremely slow.

Example:

P50 = 200 ms
P95 = 400 ms
P99 = 5 sec

This is a tail-latency problem.

Possible causes
1. Some requests hit slow DB queries
   99% → normal query
   1% → expensive query

Could depend on:

Customer
Data size
Query parameters
Missing index
Large result set
2. Specific downstream calls

Maybe:

Service B → normally 100 ms
occasionally → 5 sec

That affects P99.

3. Thread starvation

Most requests get threads immediately.

Some requests wait:

Request 1 → immediate
Request 2 → immediate
...
Request 100 → waits 4 sec
4. GC pauses

A few requests may overlap with long GC pauses.

5. Network issues

Intermittent packet loss or connection problems.

6. Retries

Some requests might experience:

A → B timeout
A → B retry
A → B retry

Those requests become very slow.

7. Large/complex requests

Only some customers may send huge payloads.

What I would investigate

Compare slow requests against normal requests using tracing:

Normal request:
A → DB = 50 ms

Slow request:
A → DB = 4.5 sec

That immediately narrows the problem.

4. API is slow only during peak traffic. How do you debug?

This strongly suggests capacity, saturation or contention.

First compare:

Normal traffic:
RPS = 100
Latency = 200 ms

Peak:
RPS = 1,000
Latency = 3 sec

Then check:

Application
CPU
Memory
GC
Thread pools
Request queues
Connection pools
Database
DB CPU
Connections
Query latency
Lock contention
I/O
Slow queries
Infrastructure
Load balancer
Gateway
Kubernetes pods
Network
Autoscaling
Important question

Does capacity scale with traffic?

For example:

Traffic ↑
↓
Pods remain same
↓
Threads/DB connections become saturated
↓
Requests queue
↓
Latency ↑
Fixes

Depending on the bottleneck:

Horizontal scaling
Increase resources
Optimize DB queries
Tune connection pools
Cache frequently accessed data
Rate limiting
Queue asynchronous work
Optimize thread pools
Fix downstream bottleneck
5. Only one endpoint is slow. What do you investigate?

This usually means the problem is endpoint-specific rather than the entire application.

Suppose:

GET /users       → 100 ms
GET /orders      → 120 ms
POST /payment    → 4 sec

Focus on /payment.

Investigate its execution path
POST /payment
↓
Validation
↓
DB
↓
Payment service
↓
Kafka
↓
Response

Check each step.

Compare DB queries

Maybe /payment executes:

SELECT ...
UPDATE ...
SELECT ...

while other endpoints only perform one query.

Check:

SQL execution time
Number of queries
Indexes
N+1 queries
Locks
Check downstream calls

Maybe only this endpoint calls:

Payment API
Fraud API
Inventory API
Check payload

Perhaps this endpoint receives huge request bodies.

Check recent code changes

Use deployment/version comparison.

Key point

Don't investigate the entire application equally. Trace the specific endpoint's execution path.

6. Latency increased after deployment. How do you identify the cause?

This is a classic production scenario.

First establish correlation:

Deployment
↓
Latency increased

Check:

What changed?
Which version is running?
Was database schema changed?
Configuration changed?
Dependency version changed?
Thread-pool settings changed?
Connection-pool settings changed?
Compare old vs new version

Suppose:

Version 1:
P95 = 300 ms

Version 2:
P95 = 1.5 sec

Use tracing and metrics broken down by service/version.

Look for code changes

Examples:

Before:
1 DB query

After:
10 DB queries

Or:

Before:
Redis cache hit

After:
Cache key changed → cache miss

Or:

Before:
Async processing

After:
Synchronous external API call
Database migration

A deployment may have:

Removed an index
Changed query
Changed schema
Added expensive join
Best mitigation

If the deployment clearly caused the issue and impact is severe:

Roll back first if safe, then investigate the root cause.

This is an important production mindset.

7. API latency is high but there are no errors. Why?

This is very common.

Latency and errors are different signals.

A request can succeed after 10 seconds.

Example:

Request
↓
Wait 8 sec for DB
↓
DB succeeds
↓
HTTP 200

No error, but terrible user experience.

Possible causes
Slow DB
Slow downstream service
Thread queueing
Connection pool waiting
Lock contention
GC
Network latency
External API slow
Large payload processing
Serialization/deserialization
Cache misses
Retry delays that eventually succeed
What to check

Don't stop at:

Error rate = 0%

Check:

P50
P95
P99
RPS
thread pool
DB latency
downstream latency
connection pools
GC

-----------------------------------------------------------------------------------------------------------------------------------------------

8. Latency increased across all microservices. Where do you start?

This is different from one endpoint being slow.

If:

    A latency ↑
    B latency ↑
    C latency ↑
    D latency ↑

I would first suspect a shared dependency or infrastructure issue.

Investigation order
    
    Client
    ↓
    Gateway / Load Balancer
    ↓
    Network
    ↓
    Services
    ↓
    Shared dependencies

Check:

1. Gateway
   Request latency
   Upstream latency
   Connection issues
   Rate limiting
   Routing
2. Infrastructure
   CPU
   Memory
   Network
   Kubernetes nodes
   Container issues
3. Shared DB

If every service uses the same DB:

DB latency ↑
↓
A latency ↑
B latency ↑
C latency ↑
4. Shared Redis
5. Shared external service
6. Network/DNS
7. Service mesh/load balancer
   Important insight

When many independent services become slow simultaneously, don't start debugging each service individually. Look for a common dependency.

9. Service A calls B and C. A is slow. How do you identify whether B or C is responsible?

Use distributed tracing.

Example:

A
├── B → 100 ms
└── C → 4 sec

Clearly C is responsible.

Without tracing

Check application metrics for outgoing requests:

A → B latency
A → C latency

You want:

B P95 = 100 ms
C P95 = 4 sec
Also consider parallel vs sequential calls
Sequential
A → B → 500 ms
→ C → 500 ms

Total ≈ 1000 ms
Parallel
A
├── B → 500 ms
└── C → 500 ms

Total ≈ 500 ms

So understanding the call structure matters.

Also check
B's DB
C's DB
B/C thread pools
B/C downstream calls
Network latency
Timeouts
Retries

Don't stop at "C is slow." Find why C is slow.

10. Traffic increases 10× and latency increases. How do you investigate?

First establish whether the traffic increase is expected.

Possible reasons:

Marketing campaign
Sale
New feature
Client bug
Retry storm
Duplicate requests
Bot traffic
Attack
Traffic routed from another region

Then compare:

Before:
RPS = 100
Latency = 200 ms

After:
RPS = 1000
Latency = 3 sec
Check capacity
Traffic ↑
↓
CPU?
↓
Thread pool?
↓
DB connections?
↓
DB capacity?
↓
Downstream capacity?
Important distinction

Traffic itself isn't necessarily the problem.

The problem may be:

Traffic exceeded the capacity of one component.

For example:

1000 RPS
↓
Service handles 1000
↓
DB can handle only 300
↓
DB queue grows
↓
API latency increases
Solutions
Scale service
Scale DB appropriately
Cache
Rate limiting
Backpressure
Queueing
Optimize expensive operations
Fix unexpected traffic source
11. CPU, memory and DB look normal but API is slow. What next?

This is where many candidates get stuck.

If:

CPU = normal
Memory = normal
DB = normal

I would investigate:

1. Thread pools

Maybe threads are blocked/waiting.

2. Connection pools

Especially:

HikariCP active = max
idle = 0
pending = high

Application may be waiting for a DB connection even though DB itself looks healthy.

3. Downstream services

Maybe:

A → B = 4 sec
4. Redis
5. External APIs
6. Network
   DNS
   TCP
   TLS
   packet loss
   connection establishment
7. Locks

Application-level locks:

synchronized

or distributed locks.

8. GC

Look at:

GC pause duration
Frequency
Old-generation collection
9. Serialization

Large JSON/XML payloads can consume significant time even without high sustained CPU.

10. Thread dump

If metrics don't explain it:

Take a thread dump and see what request threads are waiting on.

12. P99 suddenly spikes for only a few requests. What could cause it?

This is usually intermittent behavior.

Possible causes:

Slow DB for particular data
Customer A → 100 ms
Customer B → 5 sec
Large payload
Normal payload → 50 KB
Slow request → 20 MB
Specific downstream request
99% → downstream 100 ms
1% → downstream 5 sec
GC

Some requests happen during GC pause.

Connection creation

Some requests may need new TCP/TLS connections.

Cache miss
Cache hit → 10 ms
Cache miss → DB → 2 sec
Lock contention

One request holds a lock while another waits.

Retries/timeouts

Some requests encounter temporary downstream failures.

Best tool

Distributed tracing with request correlation.

Compare:

Fast request trace
vs
Slow request trace

That is much more useful than looking only at aggregate CPU.

13. API latency increases gradually over several hours. What could cause it?

Gradual degradation is different from a sudden spike.

I would investigate resource accumulation.

1. Memory leak
   Memory
   ↑
   ↑
   ↑
   GC increasingly frequent
   ↓
   Latency ↑

Check:

Heap usage
Old-gen
GC frequency
GC pause time
2. Thread leak

Threads gradually increase.

Eventually:

Thread pool saturation
→ requests wait
→ latency ↑
3. Connection leak

DB connections aren't returned properly.

Eventually:

Available connections ↓
Waiting requests ↑
Latency ↑
4. Cache growth

Cache becomes too large or eviction behavior changes.

5. Increasing queue depth

Background jobs accumulate.

6. DB degradation
   Table grows
   Query scans more rows
   Fragmentation
   Lock contention
   Increasing I/O
7. Traffic gradually increasing

Maybe:

RPS 100 → 200 → 300 → 500

Eventually capacity is reached.

Key clue

Gradual degradation makes me think of leaks, accumulation, resource exhaustion, growing data, or slowly increasing traffic.

14. Latency is high only for certain customers. What could be different?

This is a segmentation problem.

Don't only look at overall P99.

Break latency down by:

Customer
Region
API parameters
Tenant
Payload size
Data volume
User type
Account configuration
Example
Customer A → 100 ms
Customer B → 5 sec

Why?

Customer B might have:

10 records
vs
10 million records

Then the query may behave differently.

Possible causes
Large customer dataset
Missing index
Different DB shard
Different region
Different downstream
Different configuration
Cache behavior
Large payload
Tenant-specific lock
Customer-specific feature enabled
Very important

Use tenant/customer ID in tracing/logging carefully, respecting privacy and cardinality concerns.

15. Read APIs are fast but write APIs are slow. Why?

Writes generally involve more work.

Example:

GET /orders
↓
SELECT

POST /orders
↓
Validation
↓
INSERT
↓
UPDATE inventory
↓
UPDATE payment
↓
Kafka event
↓
Transaction commit
Investigate DB writes
INSERT/UPDATE latency
Index maintenance
Locks
Deadlocks
Transactions
Commit latency
Foreign-key checks
Triggers
Connection pool

Writes may hold DB connections longer.

Transaction scope

Bad:

BEGIN TRANSACTION

DB
↓
External API
↓
Another DB call
↓
COMMIT

The DB connection/transaction remains open while waiting for the external API.

Downstream calls

Writes often trigger:

Payment
Inventory
Notification
Audit
Event publishing
Kafka

Check:

Producer latency
Acks
Broker latency
Retries
Batch configuration

-----------------------------------------------------------------------------------------------------------------------------------------

16. API is slow only for large payloads. What would you check?

First compare:

Payload 10 KB → 100 ms
Payload 10 MB → 3 sec

Then investigate the complete payload lifecycle.

1. Network transfer

Large payload takes longer to transfer.

2. Serialization/deserialization

JSON/XML processing.

3. Request validation

Large number of fields/objects.

4. Memory

Large objects increase heap pressure.

5. GC

Large allocations can trigger GC.

6. Database

Maybe the large payload results in:

1000 DB inserts

instead of one operation.

7. Response payload

Don't only investigate request size.

Maybe the response is huge.

8. Compression

Check compression/decompression overhead.

9. Gateway/load balancer limits

Check:

Maximum request size
Buffering
Timeout
Upload handling


-----------------------------------------------------------------------------------------------------------------------------------
17. API became slow without any deployment. What possibilities do you investigate?

No deployment doesn't mean no change.

Many things can change outside your application.

Traffic
Traffic spike
New client
Bot
Retry storm
Database
Data growth
Slow query
Index issue
Locks
DB resource saturation
Infrastructure
Node problem
Container resource contention
Network problem
Load balancer issue
Dependencies
Downstream service changed
External API degraded
Redis degraded
Configuration

Configuration could change independently.

Cache
Cache eviction
Cache outage
Cache hit rate decreased
Certificates/DNS/network

Potential intermittent network behavior.

Background jobs

A scheduled job may suddenly consume:

CPU
DB connections
Threads
I/O
Important interview answer

"I wouldn't assume application code is responsible just because the API became slow. I would compare application metrics, infrastructure metrics, dependency metrics and traffic around the exact time the latency changed."

--------------------------------------------------------------------------------------------------------------------------------------------------

18. How do you distinguish application latency from downstream latency?

Distributed tracing is the best approach.

Suppose:

Total API = 3 sec

Service A processing = 100 ms
DB = 50 ms
Redis = 20 ms
Service B = 2.8 sec

Then downstream B is responsible.

Application latency

Time spent inside the service:

Business logic
Computation
Serialization
Validation
Internal locks
Downstream latency

Time spent waiting for:

DB
Redis
Another microservice
External API
Kafka
Network
Example
Request
↓
Service A
├── business logic = 50 ms
├── DB = 100 ms
└── Service B = 2 sec

A's CPU might still be completely normal.

Metrics

Look for:

http.server.requests
outgoing HTTP client latency
DB query latency
Redis command latency
Kafka producer latency

Tracing gives the clearest picture.

--------------------------------------------------------------------------------------------------------------------------------------

19. How do you identify whether DB, Redis or another microservice is responsible?

Start with the request trace.

Example:

API = 4 sec

DB = 50 ms
Redis = 20 ms
Service B = 3.8 sec

Service B is the first suspect.

Then investigate B.

DB

Check:

Query latency
Slow query log
EXPLAIN
DB CPU
DB connections
Lock waits
Deadlocks
I/O
Redis

Check:

Command latency
Hit/miss ratio
Connection pool
Memory
Evictions
Network
Microservice

Check:

Incoming latency
Outgoing latency
CPU
Memory
Thread pool
DB
Redis
Errors
Retries
Don't just ask:

"Is DB slow?"

Ask:

"How much time did this request spend waiting for DB?"

That's a much stronger troubleshooting approach.



------------------------------------------------------------------------------------------------------------------------------------------

20. What metrics would you check first when investigating API latency?

This is probably the most important interview question.

I would group metrics into layers.

Layer 1 — API traffic and latency

First:

RPS
P50
P95
P99
Error rate
Request count

Why?

Because I need to understand:

What changed?

For example:

RPS ↑ 10x
P99 ↑ 5x
Errors normal

This already tells me a lot.

Layer 2 — Application resources

Check:

CPU
Memory
GC
Thread pools
Request queue

Ask:

Is the application saturated or waiting?

Layer 3 — Connection pools

Check:

DB pool
HTTP connection pool
Redis pool

Especially:

Active
Idle
Pending
Maximum
Timeouts

A healthy DB can still have an unhealthy application connection pool.

Layer 4 — Database

Check:

Query latency
Slow queries
Connections
DB CPU
DB memory
I/O
Locks
Deadlocks
Transactions
Layer 5 — Cache

For Redis:

Latency
Hit rate
Miss rate
Evictions
Memory
Connections
Commands
Layer 6 — Downstream services

For every dependency:

Request count
Latency
P95/P99
Errors
Timeouts
Retries
Connection failures
Layer 7 — Infrastructure/network

Check:

Load balancer
Gateway
Network
DNS
TLS
Kubernetes nodes
Pods
Container restarts
Layer 8 — Logs + tracing

Finally correlate the exact slow requests.

Example:

Request ID: 123

Gateway       20 ms
Service A     30 ms
DB            40 ms
Redis         10 ms
Service B     4 sec

Now you know where to continue.

----------------------------------------------------------------------------------------------------------------------------


The complete interview framework

For almost every API latency question, you can use this structure:

Step 1 — Confirm
When did latency increase?
Is it all requests or only some?
Step 2 — Check traffic
RPS ↑?
Concurrency ↑?
Unexpected traffic?
Step 3 — Check latency distribution
P50
P95
P99

This tells you whether the problem is general or tail-specific.

Step 4 — Check errors
4xx
5xx
timeouts
retries
Step 5 — Check application
CPU
Memory
GC
Threads
Queues
Step 6 — Check pools
DB connection pool
HTTP connection pool
Redis pool
Thread pool
Step 7 — Trace downstream dependencies
DB
Redis
Service B
Service C
External APIs
Kafka
Step 8 — Check infrastructure/network
Gateway
Load balancer
DNS
Network
Kubernetes
Step 9 — Compare

Always compare:

Healthy vs unhealthy
Before vs after
Fast request vs slow request
Customer A vs customer B
Normal traffic vs peak traffic
Old version vs new version
Step 10 — Fix the actual bottleneck

Don't blindly:

increase CPU
increase timeout
increase thread pool
increase DB connections

First identify the bottleneck.

Step 11 — Verify

After the fix:

P95 ↓
P99 ↓
RPS stable
Errors stable
Resource utilization healthy
One mental model to remember

When an API becomes slow, think:

                    API LATENCY
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
    TRAFFIC          APPLICATION       DOWNSTREAM
       │                 │                 │
      RPS              CPU              DB
Concurrency        Memory           Redis
Retries            GC              Service B
Bots               Threads         Service C
Queues           External API
Locks
│
↓
CONNECTION POOLS
│
┌───────────┼───────────┐
↓           ↓           ↓
DB          HTTP       Redis
pool         pool        pool
│
↓
INFRASTRUCTURE
│
Gateway / LB / Network

The most important distinction for your interviews is this:

High latency does not automatically mean high CPU.

A service can have 30% CPU and 5-second latency because its threads are sitting around waiting for a DB connection, downstream service, Redis, network response, lock, or another resource.

And when you get stuck in an interview, return to:

Traffic → P50/P95/P99 → Errors → CPU/Memory/GC → Threads → Connection pools → DB/Redis → Downstream → Network → Traces → Logs.