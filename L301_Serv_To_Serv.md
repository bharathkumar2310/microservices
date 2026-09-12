# Microservices Production Troubleshooting --- Service-to-Service Connectivity

## Scope

This document covers the first 15 service-to-service communication
scenarios:

1.  Service A is unable to connect to Service B.
2.  Connection Refused.
3.  Connection Timeout.
4.  Read Timeout.
5.  Intermittent timeouts.
6.  Service B is running but Service A cannot reach it.
7.  Ping works but API call fails.
8.  DNS resolution failure.
9.  Hostname works from laptop but not from Service A.
10. HTTP 502 from API Gateway.
11. HTTP 503 from API Gateway.
12. HTTP 504 from API Gateway.
13. Only one Service B instance is failing.
14. Requests routed to one particular Service B instance fail.
15. Service B health endpoint is healthy but actual APIs fail.

------------------------------------------------------------------------

# 1. The Core Mental Model

When Service A calls Service B, do not immediately jump to application
logs.

Think about the request path:

``` text
Client / Service A
       |
       | 1. DNS
       v
Service B hostname
       |
       | 2. Network / routing
       v
Service B IP
       |
       | 3. TCP connection
       v
Service B host:port
       |
       | 4. HTTP request
       v
Service B application
       |
       | 5. Application processing
       +------> DB
       +------> Kafka
       +------> Other services
```

There are several different failure stages:

``` text
DNS failure
    ↓
Cannot resolve hostname

Network/connectivity failure
    ↓
Cannot establish TCP connection

Connection refused
    ↓
Destination actively rejected TCP connection

Connection timeout
    ↓
TCP connection could not be established within timeout

Read timeout
    ↓
Connection was established, but response did not arrive in time

HTTP 502/503/504
    ↓
An intermediary such as a gateway/proxy/load balancer is reporting a
problem communicating with or receiving a response from the backend
```

The most important question is:

> **At which stage does the request fail?**

That determines what you investigate next.

------------------------------------------------------------------------

# 2. A Generic Investigation Flow

For almost every connectivity incident, use this order.

## Step 1 --- Understand the exact error

Do not accept:

> "Service B is down."

Ask:

-   What exact exception/error are we seeing?
-   Connection refused?
-   Connection timeout?
-   Read timeout?
-   DNS error?
-   HTTP 4xx?
-   HTTP 5xx?
-   Is the failure from Service A directly or from a gateway?
-   Is it happening for every request or only some requests?
-   When did it start?
-   Did anything change before it started?

Examples:

``` text
java.net.ConnectException: Connection refused
```

``` text
java.net.SocketTimeoutException: connect timed out
```

``` text
java.net.SocketTimeoutException: Read timed out
```

``` text
java.net.UnknownHostException
```

These are not interchangeable.

------------------------------------------------------------------------

# 3. Scenario 1 --- Service A Is Unable to Connect to Service B

## Interview Question

> Service A is unable to connect to Service B. How would you
> investigate?

## Strong interview approach

I would first determine whether the failure is happening during DNS
resolution, TCP connection establishment, or after the connection has
already been established.

Then I would check:

1.  The exact error in Service A.
2.  Whether Service B is actually running and listening on the expected
    port.
3.  DNS resolution from the Service A environment.
4.  Network connectivity from Service A to Service B.
5.  Whether the correct host and port are configured.
6.  Whether a firewall, security group, network policy, proxy, or load
    balancer is blocking the connection.
7.  If a gateway/load balancer is involved, whether it can reach Service
    B.
8.  If Service B has multiple instances, whether only one instance is
    failing.
9.  Service B logs and metrics.
10. Recent deployments or configuration changes.

## Example investigation

Suppose:

``` text
Service A
   |
   | HTTP GET
   v
http://service-b:8080/orders
```

First check the error.

If:

``` text
UnknownHostException
```

I investigate DNS/service discovery.

If:

``` text
Connection refused
```

I investigate the destination host/port and whether something is
listening.

If:

``` text
Connect timed out
```

I investigate network reachability, routing, firewall/security rules,
and whether packets are being dropped.

If:

``` text
Read timed out
```

I investigate Service B's processing time, thread pool, DB, downstream
services, GC, CPU, and request-specific behavior.

This classification prevents random troubleshooting.

------------------------------------------------------------------------

# 4. Scenario 2 --- Connection Refused

## Interview Question

> Service A is getting "Connection Refused" while calling Service B.
> What could be the reasons?

## Meaning

Connection refused generally means the client was able to reach the
destination network location, but the TCP connection was actively
rejected.

A common interpretation is:

``` text
Service A
   |
   | TCP connect
   v
Service B host:8080
   |
   X ----> connection refused
```

The most common reason is:

> Nothing is listening on that IP/port.

But that is not the only possibility.

## Possible causes

### 1. Service B is not running

For example:

``` text
Service B expected:
10.0.0.5:8080
```

But the process has stopped.

### 2. Service B is running on another port

Service B may actually be listening on:

``` text
8081
```

while Service A calls:

``` text
8080
```

### 3. Service B is listening only on localhost

For example:

``` text
127.0.0.1:8080
```

instead of an address/interface reachable from Service A.

The application may be running, but remote clients cannot connect.

### 4. Wrong IP address

DNS/service discovery may resolve the hostname to an incorrect
destination.

### 5. Load balancer/service discovery points to an unhealthy instance

If there are multiple Service B instances:

``` text
          Load Balancer
          /     |      \
       B1      B2      B3
       OK      FAIL     OK
```

Requests sent to B2 can fail.

### 6. Container/pod restarted

The service may have restarted and the endpoint may temporarily not be
listening.

### 7. Service startup is incomplete

The process may exist, but the server has not started listening yet.

### 8. Incorrect service configuration

For example:

``` properties
service.b.url=http://service-b:8080
```

when the actual port is different.

### 9. Network/security rule actively rejects the connection

Depending on the environment, network controls can contribute to
connection failures. The exact behavior depends on the networking layer.

------------------------------------------------------------------------

## How to investigate

### Check Service B

Determine:

-   Is the process running?
-   Is the application started?
-   Is the expected port configured?
-   Is anything listening on that port?

On Linux:

``` bash
ss -lntp
```

or:

``` bash
netstat -lntp
```

On Windows:

``` cmd
netstat -ano | findstr :8080
```

Then identify the process if necessary.

### Test from Service A's environment

Do not test only from your laptop.

The important question is:

> Can the environment where Service A runs connect to the Service B
> endpoint?

Examples:

``` bash
curl http://service-b:8080/health
```

For TCP connectivity:

``` powershell
Test-NetConnection service-b -Port 8080
```

The exact command depends on the operating system/environment.

### Check configuration

Verify:

``` text
hostname
port
protocol
base URL
proxy configuration
service discovery configuration
```

### Check recent changes

Ask:

-   Was Service B deployed?
-   Did its port change?
-   Did configuration change?
-   Did DNS/service discovery change?
-   Did infrastructure/network rules change?

------------------------------------------------------------------------

## Interview-ready answer

> "I would first confirm the exact exception. If it is Connection
> Refused, I would check whether Service B is running and listening on
> the expected host and port. Then I would test the endpoint from the
> Service A runtime environment, verify DNS and configuration, and check
> whether a load balancer or service discovery is routing traffic to an
> unhealthy instance. I would also check recent deployments and Service
> B startup logs. If multiple instances exist, I would compare healthy
> and failing instances."

------------------------------------------------------------------------

# 5. Scenario 3 --- Connection Timeout

## Interview Question

> Service A is getting a connection timeout while calling Service B. How
> would you troubleshoot it?

## Meaning

A connection timeout occurs while trying to establish the TCP
connection.

Conceptually:

``` text
Service A
   |
   | SYN
   v
Service B
   |
   | no successful TCP connection
   X
```

The important distinction:

``` text
Connection timeout
=
could not establish connection

Read timeout
=
connection was established,
but response was too slow
```

## Possible causes

### 1. Service B is unreachable

The destination cannot be reached from Service A.

### 2. Firewall/security rule drops traffic

Packets may be silently dropped.

### 3. Network routing problem

There may be a routing problem between the environments.

### 4. Wrong IP/hostname

Service A may be trying to connect to an incorrect destination.

### 5. Wrong port

Service B may not expose the expected port.

### 6. Network congestion

Connectivity may be severely degraded.

### 7. Service B infrastructure is overloaded

Depending on the infrastructure, an overloaded destination can
contribute to inability to establish connections.

### 8. Kubernetes/network-policy/service configuration issue

In a containerized environment, service/network configuration may
prevent connectivity.

------------------------------------------------------------------------

## Investigation

Start from Service A's environment.

### DNS

``` bash
nslookup service-b
```

or:

``` bash
dig service-b
```

### TCP connectivity

Windows:

``` powershell
Test-NetConnection service-b -Port 8080
```

Linux:

``` bash
nc -vz service-b 8080
```

### HTTP test

``` bash
curl -v http://service-b:8080/health
```

The `-v` output can help identify where the request gets stuck.

### Check network path

Depending on environment:

``` bash
traceroute service-b
```

or:

``` cmd
tracert service-b
```

But remember:

> ICMP/traceroute behavior does not always represent application TCP
> connectivity.

### Check infrastructure

Investigate:

-   firewall
-   security groups
-   network ACLs
-   network policies
-   routing
-   load balancer
-   proxy
-   service discovery

------------------------------------------------------------------------

# 6. Scenario 4 --- Read Timeout

## Interview Question

> Service A connects to Service B, but the response takes too long and
> eventually gets a Read Timeout. What could be happening?

## Meaning

This is different from connection timeout.

The connection has already been established.

``` text
Service A
   |
   | TCP connection established
   v
Service B
   |
   | processing request
   |
   |-------------------- too long --------------------X
```

Possible causes:

### 1. Service B is processing slowly

Maybe the endpoint has expensive business logic.

### 2. Slow database query

``` text
A → B → DB
         |
         | slow query
         X
```

### 3. B is waiting for another downstream service

``` text
A → B → C
         |
         | slow
```

### 4. Thread pool exhaustion

Requests may be waiting for an available thread.

### 5. Connection pool exhaustion

B may be waiting for:

-   DB connection
-   HTTP connection
-   another resource

### 6. High CPU

Application processing can become slow.

### 7. Garbage collection

Long or frequent GC pauses can delay responses.

### 8. Kafka or external dependency delay

B may be waiting for another operation.

### 9. Large response/processing

Serialization, data retrieval, or network transfer can take longer.

### 10. Timeout configuration is too aggressive

The service may legitimately take longer than the configured read
timeout.

------------------------------------------------------------------------

## Investigation

Use distributed tracing if available.

Example:

``` text
A
|
| 50ms
v
B
|
| 4.8 sec
v
DB
```

Then the DB span immediately becomes suspicious.

Or:

``` text
A
|
v
B
|
| 4.7 sec
v
C
```

Then investigate C.

Check:

-   response time
-   CPU
-   memory
-   GC
-   thread pools
-   DB connection pool
-   HTTP connection pool
-   downstream latency
-   database query latency
-   application logs

------------------------------------------------------------------------

# 7. Scenario 5 --- Intermittent Timeouts

## Interview Question

> Service A → Service B works sometimes but times out sometimes. How
> would you investigate?

This is more difficult than a complete outage.

The key clue is:

> The system works for some requests.

Therefore, investigate differences between successful and failed
requests.

## Possible causes

### 1. One unhealthy instance

``` text
             Load Balancer
             /    |    \
            B1   B2     B3
            OK   FAIL   OK
```

Some requests succeed, some fail.

### 2. Uneven load

One instance may be overloaded.

### 3. Intermittent network problem

Connectivity may fail only occasionally.

### 4. Slow database queries

Certain requests may hit expensive queries.

### 5. Connection pool exhaustion

Under bursts of traffic, the pool may temporarily run out.

### 6. Thread pool exhaustion

Same principle at the application level.

### 7. GC pauses

One instance may periodically pause.

### 8. Downstream intermittent failures

B may call C, which intermittently fails.

### 9. DNS/service discovery issues

Some DNS resolutions or endpoints may lead to bad destinations.

### 10. Load balancer routing issue

Requests may be distributed incorrectly.

------------------------------------------------------------------------

## Investigation strategy

Do not only inspect one failed request.

Compare:

``` text
Successful request
vs
Failed request
```

Use:

``` text
traceId
timestamp
instance ID
source
destination
endpoint
latency
HTTP status
```

If failures always correspond to:

``` text
instance=B2
```

you have a very strong clue.

If failures correspond to:

``` text
specific endpoint
```

investigate application behavior.

If failures correspond to:

``` text
specific database query
```

investigate DB.

------------------------------------------------------------------------

# 8. Scenario 6 --- Service B Is Running, But Service A Cannot Reach It

## Interview Question

> Service B is running, but Service A cannot reach it. What would you
> check?

"Running" does not mean "reachable."

Check these layers:

``` text
1. Is B running?
2. Is B listening?
3. Is DNS correct?
4. Is the IP correct?
5. Is the port correct?
6. Can A reach that IP/port?
7. Is traffic blocked?
8. Is a proxy/gateway involved?
9. Is load balancing correct?
10. Is B actually accepting the request?
```

### Example

Service B:

``` text
Process: RUNNING
Port: 8080
```

But Service A calls:

``` text
service-b:9090
```

B is running, but A cannot reach it because the endpoint is wrong.

Another example:

``` text
B listens on 127.0.0.1:8080
```

A is on another machine.

The application is healthy locally but not remotely reachable.

------------------------------------------------------------------------

# 9. Scenario 7 --- Ping Works, But API Call Fails

## Interview Question

> Service A can ping Service B's server, but the API call fails. Why?

This is a very common interview question.

## Important concept

`ping` uses ICMP.

An API call normally uses TCP, followed by HTTP.

Therefore:

``` text
ping success
≠
TCP port success
≠
HTTP API success
```

Example:

``` text
A
 |
 | ICMP
 v
B server
 ✓ ping works

A
 |
 | TCP :8080
 X
B server
```

Possible reasons:

-   port 8080 is not listening
-   firewall blocks TCP 8080
-   wrong port
-   wrong protocol
-   HTTP server is down
-   TLS/SSL problem
-   reverse proxy problem
-   authentication failure
-   incorrect URL/path
-   application error
-   service is bound only to localhost

Use a TCP test:

``` powershell
Test-NetConnection service-b -Port 8080
```

Then HTTP:

``` bash
curl -v http://service-b:8080/health
```

------------------------------------------------------------------------

# 10. Scenario 8 --- DNS Resolution Is Failing

## Interview Question

> DNS resolution for Service B is failing. How would you troubleshoot
> it?

First identify the exact error.

Typical Java error:

``` text
java.net.UnknownHostException
```

This indicates the hostname could not be resolved.

## Investigation

### 1. Test DNS from Service A environment

``` bash
nslookup service-b
```

or:

``` bash
dig service-b
```

### 2. Check hostname

Make sure:

``` text
service-b
```

is actually the correct service name.

### 3. Check DNS configuration

Investigate:

-   DNS server
-   resolver configuration
-   search domain
-   DNS records
-   service discovery
-   TTL/cache
-   environment-specific DNS

### 4. Compare environments

For example:

``` text
Laptop → resolves
Service A → does not resolve
```

That suggests an environment-specific DNS/service-discovery issue.

### 5. If only some instances fail

Compare DNS configuration between those instances.

------------------------------------------------------------------------

# 11. Scenario 9 --- Hostname Works From Laptop but Not From Service A

## Interview Question

> The hostname works from your laptop but doesn't work from Service A.
> What could be wrong?

The important principle is:

> Your laptop and Service A are not necessarily using the same
> DNS/network environment.

Possible differences:

-   different DNS server
-   different network
-   VPN
-   corporate DNS
-   private DNS
-   container DNS
-   Kubernetes DNS
-   proxy
-   firewall
-   routing
-   service discovery
-   `/etc/hosts`
-   Windows hosts file
-   environment-specific configuration

### Investigation

Run DNS lookup from the same environment as Service A.

Do not conclude:

> "DNS works because it works on my laptop."

Instead:

``` text
Laptop
   |
   | DNS
   ✓

Service A
   |
   | DNS
   X
```

The second environment is the one that matters.

------------------------------------------------------------------------

# 12. Scenario 10 --- HTTP 502 From API Gateway

## Interview Question

> Service A receives HTTP 502 from the API Gateway. How would you
> investigate?

## Meaning

502 generally means:

> The gateway/proxy received an invalid or unsuccessful response while
> communicating with its upstream/backend.

Exact behavior varies by gateway/proxy.

Typical path:

``` text
Service A
   |
   v
API Gateway
   |
   v
Service B
```

If Gateway cannot properly communicate with B, it may return:

``` text
502 Bad Gateway
```

## Possible causes

-   Service B unavailable
-   Gateway cannot connect to B
-   wrong upstream host/port
-   DNS problem between Gateway and B
-   connection refused
-   invalid upstream response
-   TLS problem
-   load balancer issue
-   bad gateway configuration
-   unhealthy backend instance

## Investigation

First identify:

> Is Service A calling B directly, or through the Gateway?

Then:

``` text
A → Gateway → B
```

Investigate each hop separately.

### From Gateway to B

Check:

-   DNS
-   TCP connectivity
-   port
-   route
-   TLS
-   backend health
-   gateway logs

### Check B

-   application logs
-   startup status
-   listener port
-   health
-   instance status

### Compare with a working request

If:

``` text
A → Gateway → B
```

fails, but direct:

``` text
A → B
```

works, the Gateway becomes a major suspect.

------------------------------------------------------------------------

# 13. Scenario 11 --- HTTP 503 From Gateway

## Interview Question

> Service A receives HTTP 503 from the Gateway. What could be the cause?

503 generally means:

> Service unavailable.

Again, exact semantics depend on the gateway/proxy.

Common causes:

-   no healthy backend instances
-   Service B is unavailable
-   all backend instances failed health checks
-   gateway has no available upstream
-   service discovery returned no usable instances
-   temporary overload
-   maintenance
-   backend capacity issue

Typical scenario:

``` text
Gateway
   |
   +---- B1 unhealthy
   +---- B2 unhealthy
   +---- B3 unhealthy

No healthy backend
        ↓
       503
```

## Investigation

Check:

1.  Gateway logs.
2.  Backend health status.
3.  Number of healthy instances.
4.  Service discovery.
5.  Load balancer target health.
6.  Service B logs.
7.  Recent deployment.
8.  Capacity/resource metrics.

### Important distinction

A 503 often points toward:

``` text
No usable service/backend
```

while 502 often points toward:

``` text
Gateway had trouble communicating with or receiving a valid
response from the upstream.
```

But do not memorize HTTP status codes as absolute rules. Always check
the specific gateway's semantics.

------------------------------------------------------------------------

# 14. Scenario 12 --- HTTP 504 From Gateway

## Interview Question

> Service A receives HTTP 504 from the Gateway. How would you
> investigate?

## Meaning

504 generally means:

> The gateway/proxy did not receive a response from the upstream within
> its allowed time.

Typical flow:

``` text
A
|
v
Gateway
|
v
B
|
|---- processing for too long ----|
|
X
Gateway timeout
|
v
504
```

## Possible causes

### 1. Service B is slow

### 2. B's database is slow

``` text
A → Gateway → B → DB
                  |
                  X slow
```

### 3. B is waiting for another service

``` text
A → Gateway → B → C
                  |
                  X
```

### 4. Thread pool exhaustion

### 5. Connection pool exhaustion

### 6. High CPU / GC

### 7. Network delay

### 8. Gateway timeout is too short

For example:

``` text
B normally takes 8 seconds
Gateway timeout = 5 seconds
```

Then Gateway may return 504 even though B eventually completes.

------------------------------------------------------------------------

## Investigation

Use tracing:

``` text
Gateway
  20ms
   |
   v
B
  7.8s
   |
   v
DB
  7.5s
```

The trace points toward DB.

Without tracing, correlate:

-   gateway logs
-   Service B logs
-   traceId/correlation ID
-   timestamps
-   endpoint latency
-   DB metrics
-   downstream latency

------------------------------------------------------------------------

# 15. Scenario 13 --- Only One Service B Instance Is Failing

## Interview Question

> Only one instance of Service B is failing while other instances work.
> How would you identify the problem?

Example:

``` text
Load Balancer
   |
   +---- B1 ✓
   +---- B2 ✓
   +---- B3 ✗
   +---- B4 ✓
```

This explains intermittent failures if traffic is distributed across
instances.

## Investigation

First identify whether failed requests are correlated with B3.

Use:

-   instance ID
-   pod name
-   hostname
-   IP
-   trace data
-   load balancer logs

Then compare B3 against healthy instances.

### Compare configuration

``` text
B3 configuration
vs
B1/B2/B4
```

Look for:

-   environment variables
-   application properties
-   secrets
-   service URLs
-   database configuration
-   certificates
-   feature flags

### Compare resources

-   CPU
-   memory
-   GC
-   threads
-   connection pools
-   network
-   disk

### Check application logs

Look for:

``` text
exceptions
timeouts
startup errors
connection errors
OutOfMemoryError
```

### Check deployment/version

Maybe:

``` text
B1/B2/B4 → version 1.5
B3 → version 1.6
```

That is a strong clue.

------------------------------------------------------------------------

# 16. Scenario 14 --- Requests Routed to One Particular Instance Fail

## Interview Question

> Requests routed to one particular Service B instance are failing. What
> could cause this?

This is closely related to the previous scenario, but the interviewer
wants you to think about routing.

Possible causes:

### 1. Instance is unhealthy

### 2. Instance has different configuration

### 3. Instance has a different application version

### 4. Instance has resource exhaustion

### 5. Instance cannot reach its dependencies

For example:

``` text
B3 → DB
   X
```

while:

``` text
B1 → DB ✓
B2 → DB ✓
B4 → DB ✓
```

### 6. Bad network path

### 7. Corrupted/stale local state

### 8. Load balancer incorrectly considers the instance healthy

### 9. Health check is too shallow

For example:

``` text
/health
```

only confirms that the Java process is alive.

It does not necessarily prove:

``` text
DB ✓
Kafka ✓
downstream C ✓
business API ✓
```

------------------------------------------------------------------------

# 17. Scenario 15 --- Health Endpoint Is Healthy, But Actual APIs Fail

## Interview Question

> Service B is healthy according to its health endpoint, but actual API
> requests are failing. Why?

This is a very important production concept.

## Health does not always mean business availability.

Suppose:

``` text
GET /health
→ 200 OK
```

But:

``` text
GET /orders/123
→ 500
```

Possible reasons:

### 1. Health endpoint checks only application process

The JVM is alive, but a dependency is broken.

### 2. Specific database query is failing

Health may only check:

``` text
DB connection exists
```

while the actual query is failing.

### 3. Specific endpoint has a code bug

### 4. Authentication/authorization problem

Health endpoint may be public while business endpoints require JWT.

### 5. Request-specific data problem

One customer/order may trigger an exception.

### 6. Specific downstream service is unavailable

### 7. Thread/resource exhaustion affects business endpoints

### 8. Health endpoint is too shallow

------------------------------------------------------------------------

# 18. Liveness vs Readiness

This distinction is especially important.

## Liveness

Question:

> Is the application process alive?

Conceptually:

``` text
Application process alive?
       |
      yes
       ↓
Liveness = UP
```

If liveness fails, the platform may restart the instance.

## Readiness

Question:

> Is this instance ready to receive traffic?

Readiness can consider whether important dependencies are available,
depending on how it is configured.

``` text
Application alive
        +
Ready to receive traffic
        ↓
Traffic can be routed
```

A service can be:

``` text
Liveness = UP
Readiness = DOWN
```

That can be preferable to sending traffic to an instance that is alive
but not ready.

------------------------------------------------------------------------

# 19. The Most Important Distinction: Refused vs Connect Timeout vs Read Timeout

Memorize the conceptual difference.

  ---------------------------------------------------------------------------------------
Error                   What happened?          First area to investigate
  ----------------------- ----------------------- ---------------------------------------
DNS failure             Hostname couldn't be    DNS/service discovery
resolved

Connection refused      TCP connection actively Destination host/port/listener
rejected

Connection timeout      TCP connection wasn't   Network/routing/firewall/reachability
established in time

Read timeout            Connection established  Application/dependencies/performance
but response was too    
slow

502                     Gateway couldn't        Gateway ↔ backend
obtain/accept a valid   
upstream response

503                     Service/backend         Healthy backend availability
unavailable

504                     Gateway waited too long Backend latency/dependencies/timeout
for upstream
  ---------------------------------------------------------------------------------------

These are investigation starting points, not absolute laws.

------------------------------------------------------------------------

# 20. Commands You Should Know

## DNS

### Windows

``` cmd
nslookup service-b
```

### Linux

``` bash
nslookup service-b
dig service-b
```

------------------------------------------------------------------------

## Ping

``` cmd
ping service-b
```

Remember:

> Ping tests ICMP reachability, not whether the API's TCP port is
> reachable.

------------------------------------------------------------------------

## TCP port connectivity

### Windows PowerShell

``` powershell
Test-NetConnection service-b -Port 8080
```

### Linux

``` bash
nc -vz service-b 8080
```

------------------------------------------------------------------------

## HTTP

``` bash
curl -v http://service-b:8080/health
```

The `-v` option is useful because it shows connection/HTTP details.

------------------------------------------------------------------------

## Port listener

### Windows

``` cmd
netstat -ano | findstr :8080
```

### Linux

``` bash
ss -lntp
```

------------------------------------------------------------------------

## Route

### Windows

``` cmd
tracert service-b
```

### Linux

``` bash
traceroute service-b
```

Remember that firewalls and network configuration can make traceroute
incomplete or misleading.

------------------------------------------------------------------------

# 21. A Practical Decision Tree

Use this mental model during interviews.

``` text
Service A cannot call B
        |
        v
What exact error?
        |
        +----------------------+
        |                      |
        v                      v
DNS error?                Network/TCP error?
        |                      |
        v                      v
Check DNS              Refused or Timeout?
                               |
                  +------------+------------+
                  |                         |
                  v                         v
             Refused                  Connect timeout
                  |                         |
                  v                         v
          Check listener,             Check network,
          port, process,              routing, firewall,
          config, instance            reachability

If TCP connection succeeds:
        |
        v
Read timeout?
        |
        v
Investigate B processing,
DB, downstream services,
threads, pools, CPU, GC
```

For gateway responses:

``` text
Gateway returns 5xx
       |
       +---- 502 → upstream communication/response problem
       |
       +---- 503 → no usable service/backend
       |
       +---- 504 → upstream response took too long
```

Again, exact semantics depend on the gateway.

------------------------------------------------------------------------

# 22. How to Use Logs, Metrics and Traces

Do not treat these as three unrelated tools.

Use them together.

## Logs

Tell you:

> What happened?

Example:

``` text
ConnectException: Connection refused
```

## Metrics

Tell you:

> How often and how severely is it happening?

Examples:

``` text
5xx rate ↑
latency ↑
CPU ↑
DB connections ↑
```

## Traces

Tell you:

> Where in the distributed request did time/failure occur?

Example:

``` text
A
 |
 | 20 ms
 v
B
 |
 | 4.8 sec
 v
C
 |
 | 4.7 sec
 v
DB
```

The combination is powerful:

``` text
Trace → identifies slow component
Logs  → explains what happened
Metrics → shows whether it is systemic
```

------------------------------------------------------------------------

# 23. What If There Is No Distributed Tracing?

You can still investigate.

Use:

``` text
correlation ID / request ID
trace ID if available
timestamp
service name
instance ID
endpoint
```

Example:

``` text
requestId=abc123
```

Search across:

``` text
Gateway logs
Service A logs
Service B logs
Service C logs
```

Then reconstruct:

``` text
A received request
    ↓
A called B
    ↓
B called C
    ↓
C failed
```

Tracing makes this easier, but it is not the only way.

------------------------------------------------------------------------

# 24. Common Interview Mistakes

## Mistake 1

> "I will check logs."

Too generic.

Better:

> "First I identify the exact exception to determine whether failure
> occurs during DNS resolution, TCP connection establishment, or after
> the connection is established."

------------------------------------------------------------------------

## Mistake 2

> "Connection refused means Service B is down."

Too absolute.

Better:

> "Connection refused commonly means nothing is listening on the target
> port, but I would also verify the host, port, listener binding,
> service discovery, routing, and whether the request was routed to a
> bad instance."

------------------------------------------------------------------------

## Mistake 3

> "Ping is successful, so network is working."

Incorrect.

Ping uses ICMP.

Your API uses TCP/HTTP.

------------------------------------------------------------------------

## Mistake 4

> "504 means Service B is down."

Not necessarily.

504 generally indicates that a gateway/proxy did not receive an upstream
response within its timeout.

Service B could be alive but slow.

------------------------------------------------------------------------

## Mistake 5

> "Health endpoint is 200, so Service B is healthy."

Not necessarily.

A shallow health check may only prove that the application process is
alive.

------------------------------------------------------------------------

# 25. A Strong Production Investigation Example

Suppose the interviewer says:

> "Service A sometimes gets 504 while calling Service B."

A strong investigation would sound like this:

``` text
1. Confirm the exact failure rate and timestamps.

2. Determine whether 504 is generated by the gateway,
   load balancer, or application.

3. Use traceId/requestId to follow failed requests.

4. Check whether failures are associated with a particular
   Service B instance.

5. Check Service B latency, CPU, memory, thread pools,
   connection pools and GC.

6. Check Service B's database and downstream calls.

7. Compare successful and failed traces.

8. Check whether the gateway timeout is shorter than
   the actual backend processing time.

9. Check recent deployments/configuration/network changes.

10. Fix the actual bottleneck rather than simply increasing
    the timeout.
```

------------------------------------------------------------------------

# 26. How to Think Like a Production Debugger

The biggest skill is not memorizing commands.

It is narrowing the problem.

Start with:

``` text
What exactly failed?
```

Then:

``` text
Where did it fail?
```

Then:

``` text
Which component owns that failure?
```

Then:

``` text
Why did that component fail?
```

Then:

``` text
What evidence proves my hypothesis?
```

Then:

``` text
How do I fix it and prevent recurrence?
```

For example:

``` text
API failed
   ↓
504
   ↓
Gateway waited for B
   ↓
B took 8 seconds
   ↓
B waited for DB
   ↓
DB query took 7.5 seconds
   ↓
Missing index
   ↓
Query optimized
   ↓
Latency returns to normal
```

That is much stronger than:

> "I will check Service B logs."

------------------------------------------------------------------------

# 27. Final Interview Cheat Sheet

When you hear:

### "Connection refused"

Think:

``` text
Is B listening?
Correct port?
Correct IP?
Correct binding?
Correct instance?
```

### "Connection timeout"

Think:

``` text
Can A reach B?
DNS?
Routing?
Firewall?
Network policy?
Port?
```

### "Read timeout"

Think:

``` text
Connection succeeded.
Why isn't B responding quickly?
DB?
Downstream?
Threads?
Pools?
CPU?
GC?
Slow code?
```

### "Intermittent timeout"

Think:

``` text
Which requests?
Which instance?
Which endpoint?
Which dependency?
Which time?
```

### "Ping works but API fails"

Think:

``` text
ICMP ≠ TCP ≠ HTTP
```

### "502"

Think:

``` text
Gateway ↔ upstream communication/response
```

### "503"

Think:

``` text
No usable backend/service
```

### "504"

Think:

``` text
Upstream took too long
```

### "Only one instance fails"

Think:

``` text
Compare bad instance vs good instances
```

### "Health is UP but API fails"

Think:

``` text
Health check may be shallow.
Check actual endpoint and dependencies.
```

------------------------------------------------------------------------

# 28. Interview Answer Template

For almost any connectivity scenario, use this structure:

> **"First, I would identify the exact error and determine whether the
> failure is happening during DNS resolution, TCP connection
> establishment, or after the connection is established. Then I would
> test the endpoint from the Service A runtime environment, verify the
> destination host and port, and check Service B's listener and
> application status. I would also check DNS, routing, firewall/network
> policies, load balancer or service discovery if applicable. If the
> connection succeeds but the response is slow, I would investigate
> Service B's logs, traces, database, downstream calls,
> thread/connection pools, CPU, memory and GC. If multiple instances
> exist, I would compare successful and failed requests by instance.
> Finally, I would check recent deployments or configuration changes and
> use logs, metrics and traces to confirm the root cause."**

This template should **not** be blindly memorized. The goal is to
understand why each step exists.

------------------------------------------------------------------------

# 29. What We Will Do Next

These 15 questions are really one connected topic:

``` text
DNS
 ↓
Network
 ↓
TCP
 ↓
HTTP
 ↓
Gateway
 ↓
Load Balancer
 ↓
Service Instance
 ↓
Application
 ↓
Dependencies
```

Once you understand this flow deeply, many other production scenarios
become much easier.

The next set should build on this foundation:

**API latency → CPU → memory → threads → connection pools → database →
downstream services.**
