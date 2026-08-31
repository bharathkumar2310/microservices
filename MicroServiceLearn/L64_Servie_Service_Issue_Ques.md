1. Service A cannot communicate with Service B. Troubleshoot.

First, I would identify exactly what type of failure A is getting.

Possible errors:

DNS resolution failure
Connection refused
Connection timeout
Read timeout
TLS/SSL error
HTTP 4xx/5xx
Connection reset

My investigation flow would be:

Service A
|
| 1. Can it resolve B?
↓
DNS
|
| 2. Can it establish TCP connection?
↓
Network / Firewall / Load Balancer
|
| 3. Is B listening?
↓
Service B
|
| 4. Can B process the request?
↓
Application / Thread Pool / DB / Dependencies
Step 1: Check the error from Service A

For example:

UnknownHostException
→ DNS problem

Connection refused
→ Host reachable, but nothing accepting connection on port

Connect timeout
→ Cannot establish TCP connection in time

Read timeout
→ Connection established, but response didn't arrive in time

SSLHandshakeException
→ TLS/Certificate problem

HTTP 500
→ B received request but failed internally
Step 2: Check whether B is reachable

From the environment where A is running:

Can A resolve B's hostname?
Can A connect to B's port?
Can A call B's health endpoint?
Step 3: Check infrastructure

Depending on architecture:

Service Discovery
DNS
Kubernetes Service
Load Balancer
Firewall
Security Groups
Network Policies
Ingress
Step 4: Check Service B

Check:

Is B running?
Is it listening on the expected port?
Are all instances healthy?
Is CPU high?
Is memory exhausted?
Are threads exhausted?
Is the DB slow?
Is the connection pool exhausted?
2. Service B is intermittently unavailable. Why?

This is different from being completely unavailable.

If B works sometimes and fails sometimes, I would suspect something that affects only certain requests or certain instances.

Major possibilities
Possible Cause	Why it causes intermittent failure
One unhealthy instance	Requests routed to that instance fail
Load balancer issue	Some requests go to a bad backend
CPU spikes	During spikes, requests become slow/fail
Thread pool exhaustion	Sometimes capacity is available, sometimes not
DB connection pool exhaustion	Requests fail when all connections are busy
Network instability	Some connections are dropped
DNS inconsistency	Different requests resolve differently
Deployment/restart	Requests fail during pod restart
GC pauses	JVM temporarily stops processing
Traffic spikes	System works normally until load increases
Very common real-world scenario

Suppose:

Service B has 3 instances

B1 → Healthy
B2 → Healthy
B3 → Broken

Load balancer distributes:

Request 1 → B1 → Success
Request 2 → B2 → Success
Request 3 → B3 → Failure
Request 4 → B1 → Success
Request 5 → B3 → Failure

From Service A's perspective:

"Service B is intermittently failing."

But the actual issue is:

One instance of Service B is unhealthy.

How to investigate

Break down metrics per instance, not just at service level.

Check:

Error rate by instance
Latency by instance
CPU by instance
Memory by instance
Restart count
Health status
Logs
3. A → B works sometimes and times out sometimes. Why?

This usually means the request path is available, but sometimes something becomes slow enough to exceed the timeout.

Important distinction:

Sometimes success
Sometimes timeout

does not automatically mean a network problem.

Common causes
1. B is slow under load
   Normal traffic:

A → B → Response in 100 ms ✅

High traffic:

A → B → Response takes 10 seconds ❌ timeout

Check:

RPS
P95 latency
P99 latency
CPU
thread pools
DB latency
2. Thread pool exhaustion

Suppose B has:

100 request threads

All 100 are busy waiting for the database.

New requests must wait.

Request
↓
Thread pool
↓
No free thread
↓
Request waits
↓
A timeout occurs
3. DB becomes slow

Example:

A
↓
B
↓
Database

B itself may be healthy.

But:

Normal DB query → 20 ms

Slow DB query → 10 seconds

Therefore:

A waits for B
B waits for DB

Eventually A gets a timeout.

4. Bad instance

Again:

B1 → 100 ms
B2 → 120 ms
B3 → 20 seconds

Requests routed to B3 time out.

5. Network packet loss/congestion

Network problems can cause:

retransmissions
delayed packets
failed connections
Investigation

Compare successful and failed requests.

Successful:
P50 = 100 ms

Failed:
Timeout after 5 seconds

Then trace:

A
↓
B
↓
Database
↓
External Service

Find where time is being spent.

This is where distributed tracing is extremely useful.

4. Connection refused from B. What does it indicate?

Connection refused means the network reached the target host, but the target port rejected the connection.

For example:

A → B:8080

A tries to establish a TCP connection.

The host responds:

No application is accepting connections on this port.
Common causes
1. B is not running
   Service B stopped

No process is listening.

2. Wrong port

Example:

B actually runs on 8081

A calls:

B:8080

Result:

Connection refused
3. Application started but isn't listening correctly

For example, B may bind to:

localhost

instead of an accessible network interface.

4. Container/pod problem

The container may be running, but the application inside it may have crashed.

5. Load balancer sends traffic to an unavailable backend

Sometimes infrastructure incorrectly routes requests to an instance that isn't ready.

Important comparison
Connection refused

usually means:

I reached the machine, but nothing is accepting the connection on that port.

Whereas:

Connection timeout

usually means:

I could not establish the connection in time.

5. Connection timeout vs read timeout

This is one of the most important interview questions.

Type	What happened?	Connection established?
Connection Timeout	Could not establish connection	❌ No
Read Timeout	Connection established but response didn't arrive	✅ Yes
Connection timeout
A
|
| ---- tries TCP connection ---->
|
|        No connection established
|
Timeout

Possible causes:

Firewall dropping packets
Network problem
Wrong routing
Server unreachable
Load balancer unreachable
Security group blocking traffic
Read timeout
A
|
| ---- TCP connection ----> B
| <--- Connection established
|
| ---- HTTP Request -----> B
|
|        B processing...
|        B processing...
|        B processing...
|
Timeout

Possible causes:

Slow API
Slow DB
Thread pool exhaustion
External dependency slow
Deadlock
Long-running operation
6. B health endpoint is UP but requests fail. Why?

This is extremely common.

A health endpoint being UP does not necessarily mean the service can successfully handle business requests.

Example

Health endpoint:

GET /actuator/health

Returns:

{
"status": "UP"
}

But:

POST /orders

returns:

500 Internal Server Error

Why?

Because health checks may only check basic things.

Possible causes
1. Health check is too shallow

It might only mean:

Application process is running

It may not check:

database
Kafka
external APIs
business functionality
2. DB is technically UP but queries are slow/failing
   Health endpoint → UP

Business request:
B → DB query → timeout
3. Dependency is failing

Example:

B Health → UP

Actual request:
B → Payment Service → Failure
4. Thread pool exhaustion

Health endpoint may still respond because it uses a different execution path.

Business requests may be stuck.

5. Partial failure

For example:

GET /health → Works

GET /users → Works

POST /payment → Fails

The service is partially functional.

Interview answer

"I would not rely only on the health endpoint. Health indicates whether the service is alive or ready according to configured health checks, but business requests can still fail because of dependency failures, thread exhaustion, database problems, or application-level errors. I would check error rates, latency, traces, logs, dependency health, and instance-level metrics."

7. A waits indefinitely for B. What do you check?

First question:

Why is A allowed to wait indefinitely?

Normally, it should not.

Check timeout configuration

Check:

Connect timeout
Read timeout
Feign client timeout
HTTP client timeout
Gateway timeout
Load balancer timeout

Example:

A → B

If there is no timeout:

Thread 1 → waiting for B
Thread 2 → waiting for B
Thread 3 → waiting for B
Thread 4 → waiting for B
...

Eventually:

All A threads are waiting

Then A itself becomes unavailable.

Check thread pools

Look for:

Blocked threads
Waiting threads
Thread pool active count
Queue size
Rejected requests

A thread dump can show:

Thread → waiting on HTTP call to B
Check B

Why isn't B responding?

Trace:

A
↓
B
↓
DB
↓
External API

Maybe B is waiting for something else.

Correct architecture

Always use bounded waiting:

A → B

Connect Timeout
Read Timeout
Circuit Breaker
Retry (when appropriate)
Fallback
8. B is slow and causes A to become slow. How do you isolate it?

This is called failure propagation or cascading failure.

Example:

User
↓
A
↓
B (Slow)

B becomes slow.

Then A's threads wait for B:

A Thread 1 → waiting
A Thread 2 → waiting
A Thread 3 → waiting
A Thread 4 → waiting

Eventually:

A Thread Pool = FULL

Now even requests that don't need B may become slow.

How to isolate
1. Timeout

Don't allow A to wait forever.

A → B

Wait maximum 2 seconds

After that:

Fail fast
2. Circuit breaker

Suppose B is failing.

Instead of:

A → B ❌
A → B ❌
A → B ❌
A → B ❌
A → B ❌

Circuit breaker opens:

A

X ---- B

A immediately returns fallback/failure.

3. Bulkhead

This is especially important.

Suppose A has:

100 threads

Without isolation:

B consumes all 100 threads

With a bulkhead:

Total A capacity = 100

B calls maximum = 20 threads

Other operations = remaining capacity

Therefore B cannot destroy the entire Service A.

4. Rate limiting

If B is overloaded:

Too many requests → B

Limit requests.

9. B starts returning 500. What should A do?

First, understand:

500 = B received the request but encountered an internal server error.
A should not blindly retry every 500

Because the error might be:

Database constraint problem
NullPointerException
Business validation bug
Permanent failure

Retrying can make the situation worse.

Step 1: Determine whether failure is transient

Examples of potentially transient failures:

Temporary DB failure
Temporary dependency issue
Short network issue

Retry may help.

Examples of non-transient failures:

Application bug
Invalid state
Permanent business failure

Retry will not help.

Recommended behavior
500 received
↓
Classify failure
↓
Transient?
/      \
Yes       No
↓         ↓
Retry     Return error

If retrying:

Retry
+ Exponential Backoff
+ Jitter
+ Limited attempts

Example:

Attempt 1 → immediately

Attempt 2 → wait 200ms

Attempt 3 → wait 500ms

Attempt 4 → wait 1 second

Don't do:

Retry immediately
Retry immediately
Retry immediately
Retry immediately

That can create a retry storm.

10. B is completely unavailable. How should architecture behave?

The answer depends on whether B is critical.

Case 1: B is critical

Example:

Payment Service unavailable

You may need to:

Fail fast
Return meaningful error
Do not keep waiting

Example:

503 Service Unavailable
Case 2: B is non-critical

Example:

Recommendation Service unavailable

You can use fallback:

Show page without recommendations
Case 3: Operation can be asynchronous

Instead of:

A → B synchronously

Use:

A → Kafka → B

If B is down:

Message remains in Kafka
B processes it when available

This depends on business requirements.

Ideal architecture
A
│
├── Timeout
│
├── Circuit Breaker
│
├── Bulkhead
│
├── Limited Retry
│
└── Fallback / Graceful Degradation
11. DNS resolution fails between services. How do you troubleshoot?

Example:

A tries:

http://service-b

But gets:

UnknownHostException

This means:

The hostname could not be translated into an IP address.

Investigation
Step 1: Verify service name

Maybe A calls:

service-b

But actual name is:

service-b-service
Step 2: Check DNS configuration

Depending on environment:

Kubernetes DNS
Corporate DNS
Cloud DNS
Service Discovery
/etc/resolv.conf
Step 3: Test resolution from A's environment

Important:

Don't only test from your laptop.

Test from where Service A runs.

Because:

Laptop DNS ≠ Container DNS

or:

Laptop Network ≠ Kubernetes Network
Kubernetes example

Check:

Service exists
Service name
Namespace
DNS
Endpoints

For example:

service-b.default.svc.cluster.local

Maybe A is using the wrong namespace.

Important distinction
DNS failure

happens before connecting.

DNS
↓
IP address
↓
TCP connection
↓
HTTP request

If DNS fails, you never even reach the TCP connection stage.

12. Connection reset occurs. What could cause it?

A connection reset means an existing TCP connection was abruptly terminated.

Usually:

A ↔ B

Connection exists.

Then suddenly:

Connection RESET
Possible causes
1. Service B crashed
   Connection active
   ↓
   B crashes
   ↓
   Connection terminated
2. Load balancer/proxy closed the connection

Examples:

Nginx
API Gateway
Load Balancer
Service Mesh

may terminate connections.

3. Idle timeout

Suppose connection is idle:

Connection open
↓
No traffic for 60 seconds
↓
Load balancer closes it

A tries to reuse it.

Result:

Connection reset
4. Network device

Firewall or proxy may reset connections.

5. Application forcibly closes connection

The server might close the socket unexpectedly.

Investigation

Check timestamps across:

Service A logs
Service B logs
Load Balancer logs
Proxy logs
Network logs

Correlate the exact request.

13. TLS handshake fails. How do you investigate?

TLS happens before the actual HTTP request.

A
|
| ---- TLS Handshake ---->
|
| <---- Certificate ------
|
| ---- Verify certificate
|
| ---- Encryption setup
|
↓
HTTP Request

If handshake fails, the HTTP request may never reach the application.

Common causes
1. Certificate expired

Check:

Certificate validity
Expiration date
2. Certificate not trusted

Service A doesn't trust the certificate authority.

SSLHandshakeException
3. Hostname mismatch

Certificate says:

service-b.company.com

But A calls:

internal-service.company.com

Certificate validation fails.

4. TLS version mismatch

Example:

A supports TLS 1.2

B requires TLS 1.3

Handshake may fail.

5. Cipher suite mismatch

Both sides cannot agree on encryption algorithms.

Investigation

Check:

Certificate expiration
Certificate chain
Hostname/SAN
Truststore
Keystore
TLS versions
Cipher suites
Proxy/load balancer TLS configuration

Also determine where TLS terminates:

A → HTTPS → Load Balancer → HTTP → B

or:

A → HTTPS → B

This is very important.

The TLS problem might be in the:

Service
Gateway
Ingress
Load Balancer
Service Mesh

and not in Service B itself.

14. How do you decide timeout values between services?

There is no universal answer like:

Every API = 5 seconds

Timeouts should be based on actual latency and business requirements.

Look at latency metrics

Example:

B latency:

P50 = 100 ms
P95 = 300 ms
P99 = 800 ms

A reasonable timeout might be above expected P99, with an appropriate buffer.

But don't simply make it huge.

Bad:

P99 = 800 ms

Timeout = 60 seconds

If B is failing, A waits 60 seconds and destroys its own resources.

Consider the full request chain
User
↓
Gateway
↓
A
↓
B
↓
C
↓
Database

Suppose user timeout is:

10 seconds

You cannot give every service:

10 seconds

Because delays accumulate.

You need a timeout budget.

Example:

Total budget = 10 seconds

Gateway → A = 9 seconds

A → B = 3 seconds

B → C = 1 second

The exact values depend on architecture.

Separate timeout types

Don't use one timeout for everything.

Connect timeout

How long to establish connection.

Usually relatively short.

Read/response timeout

How long to wait for response.

Based on expected API processing time.

15. How do you prevent one slow downstream service from consuming all resources?

This is one of the most important microservice resilience questions.

Suppose:

A → B

B becomes slow.

Without protection:

A Request 1 → waits
A Request 2 → waits
A Request 3 → waits
A Request 4 → waits
...

Eventually:

A threads exhausted

Then:

A becomes slow

Then services calling A also become slow.

This creates:

Cascading Failure
B Slow
↓
A Threads Blocked
↓
A Slow
↓
Services calling A Slow
↓
Entire System Impacted
Protection mechanisms
1. Timeout

Never wait indefinitely.

A → B

Maximum wait = X seconds
2. Circuit breaker

If B keeps failing:

Circuit OPEN

A stops calling B temporarily.

3. Bulkhead

Limit resources allocated to B.

Example:

A has 100 threads

Maximum allowed for B = 20

Even if B hangs:

Only 20 threads affected

The rest of A continues functioning.

4. Bounded queues

Don't allow infinite requests to wait in memory.

Bad:

Infinite queue

Result:

Memory grows
Latency grows
Eventually crash

Use:

Bounded queue

When full:

Reject quickly
5. Limited retries

Retries must be:

Limited
Exponential backoff
Jitter

Otherwise a failing B receives even more traffic.

6. Rate limiting

Protect B from excessive traffic.

7. Async messaging where appropriate

For operations that don't need an immediate response:

A → Kafka → B

This reduces synchronous dependency.

⭐ The complete troubleshooting mental model

Whenever:

Service A cannot communicate with Service B

Think in layers:

┌─────────────────────────────┐
│ 1. DNS                      │
│ Can A find B?               │
└──────────────┬──────────────┘
↓
┌─────────────────────────────┐
│ 2. Network / TCP            │
│ Can A connect to B?         │
└──────────────┬──────────────┘
↓
┌─────────────────────────────┐
│ 3. TLS                      │
│ Can secure connection form? │
└──────────────┬──────────────┘
↓
┌─────────────────────────────┐
│ 4. HTTP                     │
│ Did B return 4xx/5xx?       │
└──────────────┬──────────────┘
↓
┌─────────────────────────────┐
│ 5. Application              │
│ Is B processing correctly?  │
└──────────────┬──────────────┘
↓
┌─────────────────────────────┐
│ 6. Dependencies             │
│ DB / Kafka / External APIs  │
└─────────────────────────────┘
The most important interview approach

For every communication problem, start with:

1. What exactly is the error?
   DNS?
   Connection refused?
   Connection timeout?
   Read timeout?
   TLS?
   HTTP 500?
   Connection reset?
2. Is it affecting all requests or only some?
   All requests fail
   ↓
   Check common infrastructure / complete outage

Some requests fail
↓
Check specific instances / load / intermittent dependencies
3. Where does the request stop?

Use:

Metrics
↓
Distributed Tracing
↓
Logs
↓
Infrastructure checks
4. Protect Service A

Even while investigating B:

Timeout
Circuit Breaker
Bulkhead
Retry
Fallback
Rate Limiting

The biggest concept tying almost all of these questions together is:

First identify exactly at which layer the failure occurs. Then use metrics, tracing, logs, and infrastructure checks to isolate the component. Finally, ensure resilience mechanisms prevent that failure from cascading to other services.