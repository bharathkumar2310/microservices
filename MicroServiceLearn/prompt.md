Create a complete, interview-focused **Microservices Production Debugging & Troubleshooting Guide** as a folder of Markdown files.

I am a Java Backend Developer with around 3 years of experience working with:

* Java 17
* Spring Boot
* Spring Cloud
* REST APIs
* Spring Data JPA / Hibernate
* MySQL
* Kafka
* Spring Security / JWT
* Microservices

My goal is to prepare for **real-time production troubleshooting/debugging interview questions**.

IMPORTANT:

Do NOT create a generic microservices notes document.

I want a **practical investigation playbook** where every scenario teaches me exactly:

PROBLEM → WHAT I CHECK FIRST → WHY → WHAT THE RESULT MEANS → NEXT CHECK → METRIC → TRACE → LOGS → ROOT CAUSE → FIX → VERIFY → PREVENT

The explanation must be understandable to someone who has never worked on that exact production issue before.

==================================================

1. CREATE THIS FOLDER STRUCTURE
   ==================================================

Create:

Microservices-Production-Debugging/

├── 00-debugging-framework.md
│
├── 01-service-to-service/
│   ├── 01-service-cannot-connect.md
│   ├── 02-connection-refused.md
│   ├── 03-connection-timeout.md
│   ├── 04-read-timeout.md
│   ├── 05-intermittent-timeout.md
│   ├── 06-502-bad-gateway.md
│   ├── 07-503-service-unavailable.md
│   ├── 08-504-gateway-timeout.md
│   ├── 09-dns-failure.md
│   ├── 10-tls-ssl-failure.md
│   └── 11-gateway-vs-direct-service.md
│
├── 02-performance/
│   ├── 01-api-latency-high.md
│   ├── 02-high-cpu.md
│   ├── 03-high-memory.md
│   ├── 04-out-of-memory.md
│   ├── 05-high-gc.md
│   ├── 06-thread-pool-exhaustion.md
│   └── 07-one-instance-slow.md
│
├── 03-database/
│   ├── 01-slow-database-query.md
│   ├── 02-database-connection-pool-exhaustion.md
│   ├── 03-database-lock.md
│   ├── 04-deadlock.md
│   ├── 05-n-plus-one.md
│   ├── 06-db-cpu-high.md
│   └── 07-db-query-rate-drops.md
│
├── 04-kafka/
│   ├── 01-consumer-lag.md
│   ├── 02-consumer-not-consuming.md
│   ├── 03-producer-failure.md
│   ├── 04-duplicate-messages.md
│   ├── 05-slow-consumer.md
│   ├── 06-message-processing-failure.md
│   └── 07-retry-and-replay.md
│
├── 05-cache/
│   ├── 01-low-cache-hit-ratio.md
│   ├── 02-redis-slow.md
│   ├── 03-redis-unavailable.md
│   └── 04-cache-stampede.md
│
├── 06-distributed-failures/
│   ├── 01-retry-storm.md
│   ├── 02-cascading-failure.md
│   ├── 03-circuit-breaker-open.md
│   ├── 04-one-bad-instance.md
│   └── 05-dependency-failure.md
│
└── 07-observability/
├── 01-distributed-tracing.md
├── 02-distributed-logging.md
├── 03-distributed-monitoring.md
├── 04-missing-trace.md
├── 05-missing-logs.md
└── 06-health-check-ok-business-failing.md

Create ALL files.

==================================================
2. VERY IMPORTANT — DEPTH REQUIREMENT
   =====================================

Do NOT write shallow answers.

Each scenario must be detailed enough that I can:

1. Understand the problem.
2. Explain it in an interview.
3. Actually investigate it in production.
4. Understand why each troubleshooting step is performed.
5. Know what different results mean.
6. Know what to check next.
7. Distinguish similar-looking problems.
8. Identify the likely root cause.
9. Explain the fix.
10. Explain how I would verify that the fix worked.

Every scenario should contain a realistic production example.

For example:

"Service A is calling Service B and getting Connection Refused."

Do NOT answer:

"Check whether Service B is running and check the port."

Instead explain:

Service A
|
| HTTP request
v
Service B
|
X connection refused

Then explain:

STEP 1 — Confirm the problem

Check:

* error rate
* affected endpoint
* affected instances
* timestamp
* whether all requests fail or only some

Explain WHY.

STEP 2 — Determine whether DNS works

Example:

nslookup service-b

Explain:

* successful DNS resolution
* NXDOMAIN
* SERVFAIL
* timeout

Explain what each result tells us.

STEP 3 — Check TCP connectivity

Example Windows commands:

Test-NetConnection service-b -Port 8080

Explain:

* TcpTestSucceeded = True
* TcpTestSucceeded = False

Explain what each means.

STEP 4 — Check Service B

Check:

* process
* application startup
* listening port
* health/readiness
* recent deployment

STEP 5 — Check gateway/load balancer if present.

STEP 6 — Check distributed trace.

STEP 7 — Correlate trace with logs.

STEP 8 — Identify root cause.

STEP 9 — Fix.

STEP 10 — Verify.

STEP 11 — Prevent recurrence.

This level of detail is required.

==================================================
3. EVERY SCENARIO MUST USE THIS TEMPLATE
   ========================================

For EVERY troubleshooting scenario, use the following structure:

# Problem

Clearly describe the production symptom.

# Production Situation

Give a realistic example involving:

Service A → Service B → Database/Kafka/Redis/etc.

Include realistic numbers.

Example:

* Requests: 2,000/min
* Normal P95: 300 ms
* Current P95: 4.8 sec
* Error rate: 18%
* CPU: 35%
* Memory: 58%

Numbers should help explain the investigation.

# Architecture

Show a simple ASCII diagram.

Example:

Client
|
v
API Gateway
|
v
Order Service
|
+----> Payment Service
|
+----> Inventory Service
|
v
MySQL

# What I Check FIRST

Give the first 3–5 things to check.

For every check explain:

WHAT?
WHY?
WHAT RESULT AM I LOOKING FOR?

# Step-by-Step Investigation

This is the MOST IMPORTANT section.

Use:

### Step 1

### Step 2

### Step 3

...

For every step explain:

* What I check
* Why I check it
* Example command/query/tool
* Expected result
* Bad result
* What the bad result means
* What I check next

Do not skip intermediate reasoning.

# Metrics to Check

Include relevant metrics such as:

* request rate
* error rate
* p50
* p95
* p99
* max latency
* CPU
* memory
* GC
* thread pool
* active threads
* queue size
* rejected tasks
* connection pool
* DB latency
* DB locks
* Kafka consumer lag
* retry count
* circuit breaker state
* downstream latency
* downstream error rate

For every metric explain:

"If this metric is HIGH, it suggests..."

"If this metric is LOW, it suggests..."

"If this metric suddenly changes after deployment, it suggests..."

# Distributed Trace Investigation

Explain exactly how I would use:

traceId
spanId
parent/child spans

Show an example trace:

traceId=abc123

Gateway
20ms
|
v
Order Service
50ms
|
+---- Inventory Service 4.8s
|
+---- MySQL 4.7s

Explain how this narrows the problem.

Explain:

* client latency
* server latency
* downstream latency
* DB span
* external API span
* retry spans

Also explain what it means when a child span is missing.

# Distributed Logs

Show realistic logs containing:

timestamp
level
service
instance
traceId
spanId
requestId if relevant
endpoint
downstream service
error
latency

Example:

2026-09-13 10:31:22 ERROR
service=order-service
instance=order-7f9d
traceId=abc123
spanId=xyz456
downstream=inventory-service
endpoint=/inventory/reserve
timeout=3000ms

Then explain how I correlate the logs with the trace.

IMPORTANT:

Explain why a log message alone is NOT always proof of root cause.

# Commands / Tools

Where applicable include realistic commands for Windows and Linux.

Windows:

nslookup
Resolve-DnsName
Test-NetConnection
curl

Linux:

dig
nslookup
getent
curl
nc
ss
netstat
ping

Explain what each command actually proves and what it does NOT prove.

Do not pretend that a successful ping proves HTTP connectivity.

# Root Cause

Give a realistic root cause.

Explain the causal chain.

Example:

Bad deployment
↓
Connection pool leak
↓
DB connections not returned
↓
Pool reaches 40/40
↓
Requests wait for connection
↓
Thread pool fills
↓
Latency increases
↓
Retries occur
↓
More DB requests
↓
System becomes overloaded

Explain every arrow.

# Fix

Explain:

Immediate mitigation
Permanent fix

Examples:

* rollback
* restart affected instance
* remove bad instance
* increase timeout only when appropriate
* fix connection leak
* add missing index
* fix retry policy
* add circuit breaker
* fix DNS/config
* correct Kafka consumer
* optimize SQL

Do NOT recommend blindly increasing timeouts or resources.

# Verification

After fixing, explain exactly what I check.

Example:

Before:

P99 = 5.1 sec
DB pool pending = 186
Error rate = 18%

After:

P99 = 420 ms
DB pool pending = 0
Error rate < 1%

Explain why these numbers prove improvement.

# Prevention

Explain how to prevent recurrence:

* alert
* dashboard
* timeout
* retry/backoff
* circuit breaker
* rate limiting
* resource limits
* health checks
* readiness checks
* tracing
* logging
* metrics
* load testing
* deployment safeguards

# Interview Answer

Give a concise 60–90 second interview answer.

It should sound like a real engineer speaking, not a textbook.

# Interview Follow-up Questions

Give 5–10 likely interviewer follow-up questions with answers.

==================================================
4. SERVICE-TO-SERVICE SCENARIOS
   ===============================

For these scenarios, clearly distinguish:

DNS failure
connection refused
connection timeout
TLS failure
HTTP error
read timeout

Explain the OSI/network progression:

DNS
↓
IP
↓
TCP connection
↓
TLS
↓
HTTP request
↓
HTTP response

Explain where each failure occurs.

For example:

Connection refused:
DNS may work
TCP reaches destination
destination actively rejects connection

Connection timeout:
DNS may work
TCP connection does not complete within timeout

Read timeout:
TCP connection succeeded
request was sent
response was not received in time

Make these distinctions extremely clear.

Also explain:

502 vs 503 vs 504

And:

Gateway → Service B

versus:

Service A → Service B directly

==================================================
5. PERFORMANCE SCENARIOS
   ========================

Explain how to distinguish:

High CPU
High memory
High GC
Thread exhaustion
Slow DB
Slow downstream
Lock contention

Do not assume "high CPU = infinite loop."

Give multiple possible causes and explain how metrics/traces/thread dumps help eliminate them.

For high CPU include:

* CPU metrics
* JVM metrics
* GC
* thread dump
* hot threads
* recent deployment
* traffic increase

For memory include:

heap
non-heap
RSS
GC
heap dump
retained objects
memory leak

For thread exhaustion include:

active threads
maximum threads
queue
rejected tasks
thread dump
blocked threads
WAITING
BLOCKED
RUNNABLE

==================================================
6. DATABASE SCENARIOS
   =====================

Explain:

slow query
connection pool exhaustion
DB lock
deadlock
N+1
high DB CPU

For connection pool exhaustion explain:

application requests
↓
connection pool
↓
database

Example:

max pool = 40
active = 40
idle = 0
pending = 186
acquisition p99 = 4.9 sec

Explain what this means.

Also explain an important scenario:

DB query rate FALLS while application request rate stays constant.

Explain why this can indicate that requests are blocked waiting for connections rather than the DB becoming healthy.

For SQL problems include:

EXPLAIN / execution plan
indexes
full table scan
rows examined
query latency

==================================================
7. KAFKA SCENARIOS
   ==================

Explain:

producer failure
consumer lag
consumer not consuming
slow consumer
duplicate message
processing failure
retry/replay

For consumer lag explain:

producer rate
consumer rate
partition count
consumer count
consumer group
rebalance
processing latency

Give realistic numbers.

Example:

Incoming:
10,000 messages/min

Consumer processing:
7,000 messages/min

Lag:
increasing by 3,000/min

Explain why lag grows.

Also explain how partition count affects parallelism.

Explain duplicate messages using:

at-least-once delivery
offset commit
consumer crash
idempotency
event/message ID

==================================================
8. CACHE SCENARIOS
   ==================

Explain:

cache hit ratio drops
Redis slow
Redis unavailable
cache stampede

Show how cache failure can indirectly overload the database.

Example:

Cache hit ratio:
95% → 60%

Then:

Redis misses
↓
DB requests increase
↓
DB connection pool increases
↓
DB latency increases
↓
API latency increases

==================================================
9. DISTRIBUTED FAILURE SCENARIOS
   ================================

Explain:

retry storm
cascading failure
circuit breaker
one bad instance
dependency failure

Give realistic causal chains.

Example:

Service B slows
↓
Service A times out
↓
Service A retries
↓
B receives more requests
↓
B becomes slower
↓
more retries
↓
cascading failure

Explain why retries can make an outage worse.

==================================================
10. OBSERVABILITY
    =================

Create deep explanations for:

distributed tracing
distributed logging
distributed monitoring

Explain the difference:

Metrics answer:
"Is something wrong?"

Logs answer:
"What happened?"

Traces answer:
"Where did the request spend time / where did it fail?"

Explain:

traceId
spanId
parent span
child span
MDC
structured logging
correlation

Explain:

Micrometer
OpenTelemetry
Prometheus
Grafana
Jaeger / Zipkin

Do not go unnecessarily deep into implementation internals unless useful for production debugging.

Focus on:

WHAT IT TELLS ME
WHEN I USE IT
HOW I CORRELATE IT
WHAT I CAN CONCLUDE

==================================================
11. IMPORTANT REALISTIC SCENARIO
    ================================

Include at least one complete end-to-end scenario like this:

Business success rate suddenly drops.

But:

CPU = 29%
Heap = 56%
GC max = 24 ms
Network = normal

Service-to-service transport is healthy.

Tracing shows:

Service B business latency = 5.1 sec.

Then:

worker queue is increasing.

DB pool:

active = 40/40
idle = 0
pending = 186
acquisition p99 = 4.9 sec

But DB query rate is falling.

Explain why this is NOT necessarily "database is slow."

Trace/log evidence eventually identifies a connection leak on a validation-error path after a deployment.

Explain:

checkout count
return count

and how divergence proves a resource leak.

Then explain:

retry amplification
rollback
instance draining
resource cleanup fix
verification

This should be one of the most detailed examples in the entire guide.

==================================================
12. DO NOT MAKE THESE MISTAKES
    ==============================

Do NOT:

* give only definitions
* say "check logs" without explaining what to search for
* say "check metrics" without explaining what metric means
* say "use tracing" without showing how
* assume every timeout is a network problem
* assume every high CPU problem is a code loop
* assume every DB latency problem is a slow SQL query
* assume ping proves application connectivity
* recommend increasing timeout as the first fix
* recommend increasing CPU/memory without evidence
* treat a health endpoint as proof that business functionality works
* treat one log message as root-cause proof
* skip verification
* skip prevention

==================================================
13. MAKE THE GUIDE INTERVIEW FRIENDLY
    =====================================

At the end of every scenario include:

### What I would say in an interview

Give a natural answer.

Then:

### Common interviewer traps

Explain common mistakes.

Then:

### Quick memory flow

Give a short flow such as:

Symptom
→ Scope
→ Metrics
→ Trace
→ Logs
→ Dependency
→ Root cause
→ Fix
→ Verify

==================================================
14. CROSS-SCENARIO DECISION TREE
    ================================

In:

00-debugging-framework.md

Create a master decision tree.

Start with:

"Production issue reported."

Then branch:

Is it:

* errors?
* latency?
* unavailable?
* resource exhaustion?
* data issue?
* asynchronous processing issue?

Then guide me toward the relevant investigation.

Also include:

RED metrics:

Rate
Errors
Duration

Golden signals:

Latency
Traffic
Errors
Saturation

Explain how they complement each other.

==================================================
15. FINAL CHEAT SHEET
    =====================

At the end of 00-debugging-framework.md create a compact table:

Symptom | First Check | Key Metric | Trace | Likely Areas | Next Step

Include at least:

Connection refused
Connection timeout
Read timeout
DNS failure
TLS failure
502
503
504
High latency
High CPU
High memory
OOM
High GC
Thread exhaustion
DB slow
DB pool exhausted
DB lock
Deadlock
N+1
Kafka lag
Kafka duplicate
Redis slow
Retry storm
Circuit breaker open
One bad instance

==================================================
16. COMMAND EXAMPLES
    ====================

Commands must be realistic and safe.

Windows examples:

nslookup service-b
Resolve-DnsName service-b
Test-NetConnection service-b -Port 8080
curl -v --max-time 5 http://service-b:8080/actuator/health

Linux examples:

dig service-b
getent hosts service-b
nc -vz service-b 8080
curl -v --max-time 5 http://service-b:8080/actuator/health
ss -lntp

Explain what each result means.

Never imply that these commands alone prove the entire root cause.

==================================================
17. JAVA / SPRING BOOT SPECIFIC DEBUGGING
    =========================================

Where relevant include Spring Boot/JVM-specific checks:

Actuator
/actuator/health
/actuator/metrics
/actuator/threaddump
/actuator/heapdump
/actuator/prometheus

Explain when each is useful.

Include realistic examples involving:

RestTemplate
WebClient
Feign
Spring Cloud Gateway
Spring Data JPA
Hibernate
HikariCP
Kafka
Spring Security

Do not force every technology into every scenario.

Only use it when relevant.

==================================================
18. OUTPUT QUALITY
    ==================

The final result should feel like a **production engineer's troubleshooting handbook**, not a college notes document.

Use:

* clear headings
* ASCII diagrams
* tables
* realistic metrics
* realistic logs
* commands
* decision trees
* causal chains
* interview answers

Use simple language first.

Introduce technical terminology only after explaining the idea.

For example:

Instead of immediately saying:

"TCP handshake failed."

First explain:

"The application could not establish the network connection to the destination. Now we need to determine whether DNS, routing, firewall policy, or the destination listener is responsible."

Then introduce TCP terminology.

==================================================
19. IMPORTANT: WRITE COMPLETE FILES
    ===================================

Do not create placeholders like:

"Add detailed explanation here."

Do not write:

"etc."

Do not leave sections incomplete.

Every file must contain useful, complete content.

If the entire task is too large for one response, create the files in batches, but continue until ALL requested files are complete.

Do not reduce the detail merely to finish faster.

==================================================
20. FINAL REVIEW
    ================

After creating all files, review them for consistency.

Make sure:

* terminology is consistent
* troubleshooting flow is consistent
* examples are realistic
* commands are valid
* metrics make sense
* trace examples make sense
* root causes logically follow evidence
* fixes address root causes
* verification proves recovery
* interview answers are concise
* no major production-debugging area is skipped

The final folder should be something I can study from directly for Java/Spring Boot/Microservices production-debugging interviews.
