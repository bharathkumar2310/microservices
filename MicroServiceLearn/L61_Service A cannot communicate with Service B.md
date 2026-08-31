Suppose:

        Client
        |
        |  ❌ DNS failure / Client network issue
        ↓
        API Gateway
        |
        |  ❌ Gateway timeout
        |  ❌ 4xx / 5xx
        |  ❌ Rate limiting
        |  ❌ Gateway unavailable
        ↓
        Load Balancer
        |
        |  ❌ No healthy instances
        |  ❌ Wrong routing
        |  ❌ LB timeout
        |  ❌ Connection refused
        ↓
        Service A
        |
        |  ❌ High CPU / Memory
        |  ❌ Thread pool exhausted
        |  ❌ Application errors
        |  ❌ Slow processing
        ↓
        Service Discovery / DNS
        |
        |  ❌ DNS resolution failure
        |  ❌ Service not registered
        |  ❌ Stale/incorrect IP
        ↓
        Load Balancer / Service
        |
        |  ❌ No healthy B instances
        |  ❌ Connection timeout
        |  ❌ Connection refused
        |  ❌ Routing problem
        ↓
        Service B instances
        |
        |  ❌ B is down
        |  ❌ High CPU / Memory
        |  ❌ Thread pool exhausted
        |  ❌ GC pressure
        |  ❌ Application exception
        |  ❌ Slow processing
        ↓
        DB / Downstream
        |
        |  ❌ DB slow
        |  ❌ DB connection pool exhausted
        |  ❌ DB unavailable
        |  ❌ Query/locking problem
        |  ❌ Downstream timeout
        ↓
        Response




| Layer / Issue                                         | Why does it happen?                                                                                                                                                    | How do you identify it?                                                                                                                    | What do you check?                                                                                                          | How do you fix it?                                                                                                         |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **DNS failure**                                       | Service A cannot resolve Service B's hostname to an IP. DNS server may be unavailable, wrong hostname, stale record, or service discovery registration may be missing. | Logs show `UnknownHostException`, `Name or service not known`, DNS resolution errors. Distributed trace may show the call never reached B. | Check DNS resolution from A, service-discovery registration, DNS records, CoreDNS if Kubernetes, hostname configuration.    | Fix hostname/record, register B correctly, fix DNS/CoreDNS, remove stale records, correct service-discovery configuration. |
| **Connection refused**                                | A successfully reaches B's IP, but nothing is listening on the target port. B may be down, crashed, still starting, or configured with the wrong port.                 | A logs `Connection refused` / `ConnectException`. B may show no corresponding request because connection never reached the application.    | Check B instance/pod status, application process, listening port, container port, service port and LB target configuration. | Start/restart B, fix port configuration, fix container/service mapping, remove unhealthy instances from LB.                |
| **Connection timeout**                                | A cannot establish a TCP connection to B within the configured connection timeout. Network, firewall, routing, LB, or overloaded B can cause it.                       | Logs show `ConnectTimeoutException` or connection timeout. Trace shows failure before B receives the request.                              | Check network connectivity, firewall/security groups, routes, LB, B availability and connection saturation.                 | Fix network/firewall/routing/LB, restore B, scale B if overloaded. Don't blindly increase timeout.                         |
| **Read timeout**                                      | TCP connection was established, but B did not send a response before A's read timeout expired. Usually B is slow rather than unreachable.                              | Logs show `ReadTimeoutException` / socket timeout. Trace shows A waiting on B for a long time.                                             | Check B P95/P99 latency, thread pool, CPU, GC, DB pool, DB query latency and downstream services.                           | Fix the slow operation/dependency, optimize DB queries, scale B, use appropriate timeout + circuit breaker.                |
| **Connection reset**                                  | An established connection was unexpectedly closed. B may have crashed/restarted, LB/proxy may have terminated it, or network interruption occurred.                    | Logs show `Connection reset`, `SocketException`, or similar. Correlate A and B timestamps.                                                 | Check B restarts/crashes, LB/proxy logs, network errors and connection handling.                                            | Fix crashing B, LB/network configuration, connection management, or infrastructure issue.                                  |
| **HTTP 400**                                          | Request reached B but B rejected it as invalid. Request format, parameters or headers may be wrong.                                                                    | A receives HTTP 400. Trace confirms request reached B.                                                                                     | B logs, request payload, parameters, headers and API contract.                                                              | Fix request construction or B's validation/contract if incorrect.                                                          |
| **HTTP 401**                                          | Authentication is missing or invalid.                                                                                                                                  | A receives 401 from B.                                                                                                                     | JWT/token, expiration, authentication configuration and headers.                                                            | Send valid credentials/token and fix authentication configuration.                                                         |
| **HTTP 403**                                          | Authentication succeeded, but caller doesn't have permission.                                                                                                          | A receives 403.                                                                                                                            | Roles, scopes, authorization rules and security configuration.                                                              | Correct permissions/roles/scopes.                                                                                          |
| **HTTP 404**                                          | Requested endpoint/resource doesn't exist at B. Could also be wrong URL or API version.                                                                                | A receives 404.                                                                                                                            | Service URL, endpoint path, API version and B routing.                                                                      | Correct URL/route/version or restore missing endpoint.                                                                     |
| **HTTP 429**                                          | B or an intermediate gateway is rate limiting A because request volume exceeded configured limits.                                                                     | 429 responses increase; rate-limit metrics/logs may show throttling.                                                                       | Request rate, gateway/B rate limits, quotas and traffic spikes.                                                             | Reduce traffic, use backoff, tune limits appropriately, scale capacity.                                                    |
| **HTTP 500**                                          | Request reached B, but B encountered an internal application error.                                                                                                    | A receives 500. B logs contain exception/stack trace.                                                                                      | B logs, traces, exception, DB/downstream calls and recent deployments.                                                      | Fix application bug/config/dependency causing the exception.                                                               |
| **HTTP 502**                                          | Usually a gateway/LB/proxy received an invalid or failed response from its backend.                                                                                    | A receives 502, often from gateway/LB rather than B directly.                                                                              | Gateway/LB logs, backend health, routing and B response.                                                                    | Fix backend availability, routing, proxy/LB configuration.                                                                 |
| **HTTP 503**                                          | Service is unavailable or overloaded. B may have no healthy instances or intentionally reject traffic.                                                                 | 503 responses increase; health/LB metrics may show no healthy backends.                                                                    | B instance health, LB health checks, CPU, thread pool, deployments.                                                         | Restore/scale B, fix health checks, rollback bad deployment if needed.                                                     |
| **HTTP 504**                                          | Gateway/proxy waited too long for B and timed out.                                                                                                                     | A receives 504; gateway logs show upstream timeout.                                                                                        | Gateway timeout, B latency, DB/downstream latency.                                                                          | Fix B's latency/dependency; configure sensible timeout values.                                                             |
| **TLS/SSL handshake failure**                         | A and B cannot establish a secure TLS connection due to certificate, truststore, TLS version or mTLS configuration problems.                                           | Logs show `SSLHandshakeException`, certificate errors, PKIX errors.                                                                        | Certificate expiry, certificate chain, truststore, TLS versions, mTLS configuration.                                        | Renew/fix certificate, trust CA, correct TLS/mTLS configuration.                                                           |
| **Certificate expired**                               | B's HTTPS certificate has passed its validity period.                                                                                                                  | TLS handshake fails; certificate inspection shows expired certificate.                                                                     | Certificate expiry date and certificate chain.                                                                              | Renew and deploy the certificate; verify trust chain.                                                                      |
| **Service discovery failure**                         | B is running but isn't correctly registered/discoverable.                                                                                                              | A cannot find B or gets no valid instances.                                                                                                | Eureka/Consul/Kubernetes service, registrations, health status.                                                             | Fix registration/heartbeat/configuration and service naming.                                                               |
| **Stale service-discovery information**               | A or discovery layer has an old IP/instance that no longer exists.                                                                                                     | Calls intermittently go to invalid instances.                                                                                              | Discovery registry, DNS cache, instance lifecycle and TTL.                                                                  | Remove stale registration, fix TTL/cache behavior, correct instance registration.                                          |
| **Load balancer has no healthy instances**            | All B instances fail health checks or are unavailable.                                                                                                                 | LB reports zero healthy targets; A receives 503/502/timeouts.                                                                              | LB target health, health-check endpoint, B instance health.                                                                 | Fix B/health check, restore instances, scale B.                                                                            |
| **Load balancer sends traffic to unhealthy instance** | Health checks are incorrect or LB hasn't detected a failed instance.                                                                                                   | Some requests succeed while others fail depending on which B instance receives traffic.                                                    | Per-instance metrics, LB target health, health-check configuration.                                                         | Fix health checks and remove unhealthy instances.                                                                          |
| **Wrong LB routing**                                  | Routing rules send traffic to wrong service/version/port.                                                                                                              | Requests consistently or selectively reach the wrong backend.                                                                              | LB rules, host/path routing, target groups.                                                                                 | Correct routing rules and target configuration.                                                                            |
| **Network/firewall blocked**                          | Network policy, firewall, security group or Kubernetes NetworkPolicy blocks A → B traffic.                                                                             | Connection timeout is common; network tools fail to reach B.                                                                               | Firewall/security rules, network policies, routes and connectivity from A.                                                  | Allow required traffic, fix routes/policies/security rules.                                                                |
| **Service B is down**                                 | B process/container/pod crashed, deployment failed, or all instances are unavailable.                                                                                  | Health checks fail; no healthy B instances; A gets connection errors/503.                                                                  | Pod/instance status, process status, startup logs, deployment history.                                                      | Restart/replace instances, fix startup/deployment issue, rollback if necessary.                                            |
| **Service B CPU saturated**                           | Too much traffic or CPU-heavy processing consumes available CPU. Requests take longer and may eventually timeout.                                                      | CPU close to 100%, latency and thread activity increase.                                                                                   | CPU, request rate, endpoint latency, profiling/thread dump.                                                                 | Optimize CPU-heavy code, reduce traffic, scale horizontally.                                                               |
| **Service B memory pressure**                         | Excessive allocations, leak, cache growth, or insufficient heap causes memory pressure.                                                                                | Heap usage high, GC increases, latency rises, possible OOM.                                                                                | Heap, GC metrics, allocation rate, memory usage, heap dump if needed.                                                       | Fix leak/allocation, tune heap/GC, control caches, scale appropriately.                                                    |
| **Service B thread pool exhausted**                   | Threads are blocked on DB/downstream calls, locks, slow processing, or traffic is too high. New requests wait in the queue and eventually timeout.                     | Active threads near max, queue grows, request latency rises.                                                                               | Executor metrics, thread dump, traces, DB pool and downstream latency.                                                      | Fix blocking dependency/root cause, tune pool carefully, scale service.                                                    |
| **Excessive GC in B**                                 | Too many temporary objects or insufficient heap causes frequent GC. Application threads spend time in GC instead of processing requests.                               | GC frequency/pause time increases, CPU may increase, API latency rises.                                                                    | JVM GC metrics, heap usage, allocation rate.                                                                                | Reduce allocations, fix leaks, tune heap/GC, scale if necessary.                                                           |
| **B DB connection pool exhausted**                    | Requests hold DB connections for too long, queries are slow, connections leak, or concurrency is too high.                                                             | Hikari `pending`/active connections increase; requests wait for DB connections.                                                            | Active/idle/pending connections, query latency, connection leak detection.                                                  | Fix slow queries/leaks/transactions, tune pool size carefully, optimize DB usage.                                          |
| **B DB slow**                                         | Missing indexes, bad query plans, large scans, locks, high DB load or excessive queries.                                                                               | Trace shows B spending most time in DB; DB query latency increases.                                                                        | `EXPLAIN`, slow query logs, DB CPU, locks, indexes.                                                                         | Optimize query/indexes, fix locks, reduce unnecessary queries.                                                             |
| **DB locking**                                        | One transaction holds a lock while another transaction waits.                                                                                                          | Query/transaction wait time increases.                                                                                                     | DB lock/transaction monitoring, blocking sessions.                                                                          | Reduce transaction duration, improve query/indexing, fix locking strategy.                                                 |
| **B downstream service slow**                         | B is waiting for another microservice. B's threads remain occupied while waiting.                                                                                      | Distributed trace shows B → C consuming most of the request time.                                                                          | Tracing, B thread pool, C latency/errors.                                                                                   | Fix C, introduce timeout/circuit breaker/fallback, optimize communication.                                                 |
| **B downstream unavailable**                          | Dependency C is completely unavailable. Without protection, B may accumulate waiting threads and become unavailable itself.                                            | B shows downstream connection errors/timeouts and rising thread usage.                                                                     | B → C errors, timeout metrics, C health.                                                                                    | Circuit breaker, timeout, fallback/degraded response, retry carefully.                                                     |
| **Connection pool exhaustion**                        | Too many concurrent requests consume available connections faster than they are released.                                                                              | Waiting/pending connection count rises; request latency increases.                                                                         | Pool active/idle/pending metrics and connection acquisition time.                                                           | Fix slow consumers/leaks, tune pool size, reduce unnecessary connections.                                                  |
| **Recent deployment failure**                         | New code/config introduced after deployment causes errors, latency or connectivity problems.                                                                           | Error/latency increase starts immediately after deployment.                                                                                | Deployment timeline, logs, metrics before/after release.                                                                    | Roll back quickly if confirmed, then fix and redeploy.                                                                     |
| **Configuration mismatch**                            | A has wrong B URL, port, protocol, credentials, timeout or endpoint.                                                                                                   | Failures begin after configuration change; logs show wrong endpoint/connection errors.                                                     | Environment variables, config files, secrets, deployment config.                                                            | Correct configuration and redeploy/reload safely.                                                                          |


Users are reporting that an API in Service A is failing because Service A cannot call Service B.

1. First confirm the problem

        I would first check Service A's metrics and logs.

I want to answer:

                Is Service A actually failing to communicate with B, or is some other problem causing the API failure?
        
        Check:
        
        Request rate
        Error rate
        P95/P99 latency
        HTTP status codes
        Timeout count
        Connection errors
        Recent deployment/configuration changes

For example, Service A logs:

    Connection refused: service-b:8080

or

    Read timed out calling service-b

or:

    HTTP 500 from service-b

These are different problems, so I don't treat them the same way.

2. Identify exactly what type of failure we have

This is one of the most important parts.

Case A — Connection refused

Example:

    java.net.ConnectException: Connection refused

Meaning:

    Service A reached the destination, but nothing is accepting connections on that port.

Possible causes:

    Service B is down
    B's instance/container crashed
    B hasn't started properly
    Wrong port
    Application isn't listening on the expected port
    Load balancer sent traffic to an unhealthy instance
    How I debug

Check Service B:

    Is the process running?
    Are instances healthy?
    Is port 8080 listening?
    Are containers/pods running?

Then check load balancer/service discovery.

Fix

Depending on root cause:

    Restart/replace unhealthy instance
    Fix incorrect port configuration
    Fix deployment/startup problem
    Remove unhealthy instance from load balancer
    Fix service discovery configuration

3. Connection timeout

Suppose A gets:

        Connection timed out

This is different from connection refused.

It generally means:

    A couldn't establish the connection within the configured connection timeout.

I would investigate:

    A
    |
    |---- DNS?
    |
    |---- Network?
    |
    |---- Firewall/security rules?
    |
    |---- Load balancer?
    |
    v
    B

Possible causes:

    Network connectivity issue
    Firewall/security-group rule
    Network routing problem
    Load balancer problem
    B is overloaded and unable to accept connections
    Network congestion

I would check network/connectivity metrics and infrastructure logs.

Fix

Depending on root cause:

        Correct network/security configuration
        Fix routing
        Fix load balancer
        Scale B if it is overloaded
        Remove unhealthy instances

4. Read timeout

Suppose:

    Read timed out

This is another important distinction.

    Here the connection was generally established, but:

Service A sent the request and didn't receive a response from B within the configured read timeout.

For example:

    A --------------------> B
    connection OK
    
    A ---- request -------> B

                       B processing...
                       B processing...
                       B processing...

    A <--- timeout --------

Now I investigate Service B.

Check:

    B's P95/P99 latency
    CPU
    Memory
    Thread pool
    GC
    DB connection pool
    DB query latency
    Downstream services
    Locks/contention

For example:

    A latency increased
    ↓
    Tracing
    ↓
    A → B = 8 seconds
    ↓
    B → DB = 7.5 seconds

Now the communication problem is actually caused by B's slow database.

    So I don't simply increase the timeout.

5. Use distributed tracing

This is where your original answer was good.

I would trace:

    Client
    ↓
    Service A
    ↓
    Service B
    ↓
    Database

Suppose the trace shows:

    Service A       200 ms
    |
    └── Service B       9 sec
    |
    └── DB       8.7 sec

Then B isn't necessarily "unavailable."

B is slow because its DB call is slow.

But suppose:

    Service A       10 sec
    |
    └── Service B       no response
    
    Then I investigate B itself.

6. Check Service B health

Now check:

    Number of healthy instances
    Number of running instances
    CPU
    Memory
    JVM metrics
    Thread pool
    GC
    HTTP request metrics
    Error rate
    DB connection pool
    Downstream calls

For example:

    Service B
    CPU       95%
    Threads   200/200
    DB pool   50/50

This tells me B may be resource saturated.

7. Check Service B logs

        I correlate the timestamp from Service A with B's logs.
        
        For example:
        
        Service A:
        
        12:01:05 Request to B timed out
        
        Service B:
        
        12:01:05 Waiting for DB connection
        12:01:05 Waiting for DB connection
        12:01:06 Waiting for DB connection
        
        Now we know:
        
        A timeout
        ↓
        B couldn't process request
        ↓
        B waiting for DB connection
        ↓
        DB connection pool exhausted
        
        Then I investigate why the DB pool is exhausted.
        
        Maybe:
        
        Slow queries
        Too many concurrent requests
        Connection leak
        Long transactions
        DB itself overloaded
   8. Check Service Discovery / DNS

           Suppose Service A calls:
        
           http://service-b:8080
        
           I need to verify that:
        
           service-b
           ↓
           correct IP/instances
        
           Possible problem:
        
           Service discovery
           ↓
           returns old/invalid instance
           ↓
           A connects to wrong destination
        
           Or DNS resolution itself may fail.
        
           Then the fix could be:
        
           Correct service registration
           Remove stale instances
           Fix DNS
           Fix service discovery configuration
9. Check Load Balancer

Suppose B has 5 instances:

    B1 → healthy
    B2 → healthy
    B3 → unhealthy
    B4 → healthy
    B5 → healthy

If the load balancer continues sending traffic to B3, A may see intermittent failures.

I would check:

    LB health checks
    Backend instance status
    Routing
    Connection limits
    LB errors
    Recent configuration changes

Fix:

Remove unhealthy instances and fix the health-check/configuration problem.

10. Check recent changes

        This is extremely important in production.

Ask:

    "When did the problem start?"

Then correlate with:

    Service B deployment
    Service A deployment
    Configuration change
    Infrastructure change
    Network change
    Database change
    Certificate change
    Service discovery change

For example:

    12:00 deployment B
    12:05 errors start
    
    That is a strong clue.
    
    Potential solution:
    
    Roll back the deployment if the new release is confirmed to be responsible.

11. Don't forget TLS/certificates

If A communicates with B over HTTPS, check:

    Certificate expiry
    Certificate trust
    TLS configuration
    Mutual TLS configuration

For example:

SSLHandshakeException

means this isn't a thread-pool or database problem.

It's potentially a TLS/certificate/configuration problem.

12. Finally identify the root cause

Suppose our investigation produces:

    Service A
    ↓
    calls B
    ↓
    B is slow
    ↓
    B thread pool exhausted
    ↓
    because DB queries became slow
    ↓
    missing index

Then the actual root cause is:

A missing DB index caused slow queries, which caused B's threads to remain occupied longer, which exhausted B's thread pool, which caused requests from A to time out.

That's the level of reasoning interviewers want.

13. Fix the actual root cause

Don't just do:

Increase timeout from 2 sec → 30 sec

That may hide the problem and make resource exhaustion worse.

Instead:

        Slow DB query
        ↓
        Optimize query
        ↓
        Add/fix index
        ↓
        DB latency decreases
        ↓
        B processes requests faster
        ↓
        Thread pool recovers
        ↓
        A gets responses normally

Depending on the root cause, solutions could include:  

| Root cause            | Possible solution                        |
| --------------------- | ---------------------------------------- |
| B instance down       | Restart/replace instance                 |
| Wrong port            | Correct configuration                    |
| DNS/service discovery | Fix registration/DNS                     |
| LB routing issue      | Fix health checks/routing                |
| Network issue         | Fix network/security/routing             |
| B CPU saturated       | Optimize/scale                           |
| Thread pool exhausted | Find why threads are blocked; tune/scale |
| DB pool exhausted     | Fix slow queries/leaks/contention        |
| Slow downstream       | Timeout + circuit breaker + fallback     |
| Deployment issue      | Rollback/fix release                     |
| TLS issue             | Renew/fix certificate/config             |
| Application exception | Fix application bug                      |


14. Verify after fixing

I would not stop after applying the fix.

Check:

        Error rate ↓
        Latency ↓
        Timeouts ↓
        Service B healthy
        Thread pool normal
        DB pool normal
        Trace successful

Then monitor for some time to ensure the issue doesn't return.

Interview-ready structure

For this question, remember this simple flow:

1. Confirm
   → Metrics + logs

2. Identify failure type
   → DNS / connection refused / connection timeout / read timeout / HTTP 5xx

3. Trace
   → A → B → B's dependencies

4. Check infrastructure
   → DNS → LB → network → B instances

5. Check B
   → CPU → memory → GC → threads → DB pool → downstream

6. Find root cause
   → Don't assume B itself is the problem.

7. Fix
   → Fix root cause, not just increase timeout.

8. Verify
   → Errors, latency, traces and resource metrics return to normal.




1. First confirm the problem

Before changing anything, determine:

      Is Service B actually down?
      Is the problem affecting all requests from A, or only some?
      Did it start recently?


      "First I would check Service B's health endpoint and its instance/container status. 
      Then I would check B's application logs and verify that the expected port is listening. 
      However, I wouldn't conclude that B is down just because A cannot reach it. 
      I would test B locally or from the infrastructure side. 
      If B is healthy locally but A cannot connect, I would investigate DNS, service discovery, network connectivity, firewall, security groups, or network policies."


Is A getting:
      
      connection timeout?
      connection refused?
      DNS error?
      HTTP 4xx?
      HTTP 5xx?
      TLS/SSL error?

The exact error immediately narrows the investigation.

For example:

      Connection refused

is very different from:

      Connection timed out

and both are different from:

      UnknownHostException

2. Check Service A logs first

         Service A is the caller, so first look at the outbound call.

Example:

      Service A
      |
      | HTTP request
      ↓
      Service B

Look for:

Connection refused
Connection timeout
Read timeout
UnknownHostException
SSLHandshakeException
HTTP 401
HTTP 403
HTTP 404
HTTP 429
HTTP 500
HTTP 503

Also check:

request URL
hostname
port
HTTP method
timeout
retry count
response status
exception stack trace

For example:

POST http://service-b:8080/orders
Connection refused

Now we know A is trying to reach:

service-b:8080
3. Verify Service B is running

Now move to B.

Check:

Is Service B running?
Are its instances healthy?
Is its application started successfully?

For Spring Boot:

/actuator/health

You might see:

{
"status": "UP"
}

If B is completely down:

A → X B

then investigate why B is down:

application crash
OOM
deployment failure
startup failure
configuration problem
dependency failure
4. Check whether B is listening on the expected port

This is a very important distinction.

Suppose A calls:

service-b:8080

But B is actually listening on:

8081

Then:

A → service-b:8080
X
B listens on 8081

So verify:

Application port
Container port
Service port
Target port

For Spring Boot:

server.port=8080

If using Docker/Kubernetes, also verify the networking configuration.

5. Check DNS/service discovery

Suppose A calls:

http://service-b:8080

Before HTTP communication can happen, service-b must resolve to an IP.

Conceptually:

service-b
↓
DNS / Service Discovery
↓
10.x.x.x

If DNS fails:

UnknownHostException

then investigate:

service name
DNS
Eureka/service discovery
Kubernetes Service
Consul
configuration
namespace

For example:

Service A
|
| "service-b"
↓
DNS
|
X
Cannot resolve service-b

At this point there is no reason to investigate the database yet.

6. Test network connectivity from A to B

This is one of the most important interview points.

Don't test connectivity from your laptop and assume it proves A can reach B.

You need to test from the environment where Service A is running.

Conceptually:

A ─────────────→ B

Check:

Can A resolve B?
Can A establish a TCP connection to B's port?

For example:

DNS resolution
↓
TCP connection
↓
HTTP request

If DNS works but TCP fails:

service-b → IP
↓
TCP :8080
X

investigate:

firewall
security group
network policy
Kubernetes NetworkPolicy
wrong port
service configuration
routing
load balancer
network infrastructure
7. Check firewall / network policy

If B is running and listening, but A cannot establish a connection, check whether something between them is blocking traffic.

For example:

Service A
|
↓
Network
|
X ← firewall / network policy
|
↓
Service B

Possible causes:

firewall rule
security group
Kubernetes NetworkPolicy
blocked port
subnet/routing issue
proxy configuration
service mesh policy
8. Check Service B logs

Now look at B.

This is extremely useful.

There are two important cases.

Case 1 — B has NO request logs

A says:

I called B

but B says:

I received nothing

Then the problem is likely before the request reaches B.

Investigate:

DNS
↓
Network
↓
Firewall
↓
Port
↓
Load balancer/service
Case 2 — B DOES receive the request

For example:

B logs:

POST /orders
request received

Now network connectivity is working.

The problem is probably inside B.

Then investigate:

A → B ✓

B
↓
controller
↓
service
↓
database / downstream
9. If B receives request but returns 5xx

Now investigate B internally.

For example:

A
↓
B
↓
Database
X

Possible causes:

database unavailable
connection pool exhausted
SQL error
downstream service failure
thread pool exhaustion
application exception
timeout
memory pressure
CPU saturation

Check B's:

application logs
exception stack traces
CPU
memory
thread pool
DB connection pool
DB latency
downstream calls
10. If B returns 401/403

Then communication itself is working.

The problem is authentication/authorization.

For example:

A → B
↓
HTTP 403

Check:

JWT/token
token expiration
scopes
roles
service-to-service credentials
OAuth configuration
gateway/security configuration

Don't call this a network problem.

11. If B returns 404

Again, connectivity is working.

Now check:

URL
HTTP method
API path
API version

For example:

A calls:

POST /api/v1/order

but B exposes:

POST /api/v2/order

Then the network is perfectly healthy.

12. If B returns 429

This usually indicates throttling/rate limiting.

Flow:

A → B
↓
429 Too Many Requests

Investigate:

rate limiter
traffic spike
retry storm
gateway limits
service limits

If A retries aggressively:

A
↓ request
B → 429
↓ retry
B → 429
↓ retry
B → 429

A's retries can make the situation worse.

13. If it is a timeout

This requires more careful investigation.

There are different timeout types.

Connection timeout
A → B
X

A cannot establish the connection.

Investigate:

network
firewall
routing
overloaded B
wrong endpoint
connection pool
Read timeout

Connection was established:

A → B ✓

but B doesn't respond within the configured time.

Then investigate:

B
↓
slow code
↓
slow DB
↓
slow downstream

This distinction is very important.

14. Check connection pools

Suppose B is healthy but A has exhausted its HTTP connection pool.

Example:

Service A
HTTP connection pool

10 connections
10 currently busy
0 available

New requests wait.

Eventually:

Connection pool timeout

So check A's:

HTTP client connection pool
active connections
idle connections
pending requests
connection acquisition timeout

This can make it look like:

"A cannot communicate with B"

when B itself is perfectly healthy.

15. Check load balancer / gateway if present

The actual architecture may be:

A
↓
Gateway / Load Balancer
↓
B

Then don't assume A connects directly to B.

Check:

A → Gateway ✓?
Gateway → B ✓?

For example:

A → Gateway ✓
Gateway → B X

Then the problem is between Gateway and B.

16. Check service discovery/load-balancer endpoints

If B has multiple instances:

          ┌─ B1
A → LB ───┼─ B2
└─ B3

You could have:

B1 ✓
B2 ✓
B3 X

Then some requests succeed and some fail.

This is a very important production scenario.

If:

8 requests succeed
2 requests fail

don't immediately conclude that the entire B service is down.

Check whether one unhealthy instance is behind the load balancer.

17. Check recent changes

Once you narrow down the failure, check what changed around the time it started.

Look at:

deployment
configuration change
DNS change
firewall change
certificate change
service discovery change
scaling
gateway configuration
code release
infrastructure change

For example:

10:00 → everything healthy

10:05 → B deployed

10:06 → A cannot reach B

The deployment becomes a strong suspect.

18. Use tracing if available

If you have distributed tracing:

Client
↓
A
↓
B
↓
DB

A trace can show:

A → B
2ms

or:

A → B
5000ms

or:

A → B
ERROR

This helps identify exactly where the request stopped.

Complete interview flow

I would answer the interviewer like this:

"First I would confirm the exact failure from Service A logs and identify whether it is DNS failure, connection failure, timeout, or an HTTP error. Then I would verify that Service B is running and healthy and that A is using the correct hostname and port.

Next I would verify DNS/service discovery from A's environment and test TCP connectivity from A to B. If TCP connectivity fails, I would investigate firewall, network policies, routing, security groups, or incorrect ports.

If B is reachable, I would check B's logs to see whether the request actually reached it. If B has no request logs, the issue is somewhere in the network path or service routing. If B received the request, I would investigate B internally, including application errors, database calls, downstream calls, thread pools, CPU, memory, and connection pools.

I would also check whether the issue affects all B instances or only some instances behind a load balancer/service discovery mechanism. Finally, I would check recent deployments or configuration changes, use distributed tracing if available, fix the root cause, and then verify successful requests and monitor the service to ensure the issue is resolved."



                   SERVICE A CANNOT REACH B
                             │
                             ↓
                      What error?
                             │
              ┌──────────────┼───────────────┐
              ↓              ↓               ↓
            DNS           Network          HTTP
              │              │               │
      hostname?       TCP works?      status code?
      │              │               │
      discovery       firewall?       4xx / 5xx
      port?
      policy?
      │
      ↓
      Did B receive it?
      /           \
      NO             YES
      │               │
      network path      B internals
      / routing         / DB / downstream
      / threads / CPU


Important causes  of connceetion Timeout

| Cause                              | What actually happens                                                        | Example                          |
| ---------------------------------- | ---------------------------------------------------------------------------- | -------------------------------- |
| **B is down**                      | A tries to connect, but there is no functioning server accepting connections | B crashed                        |
| **Wrong IP/hostname**              | A connects to an incorrect destination where nothing responds                | DNS points to old IP             |
| **Wrong port**                     | A tries the wrong port                                                       | A → `B:8080`, but B uses `8081`  |
| **Firewall blocking**              | Network device silently drops A's packets                                    | Firewall blocks `A → B:8080`     |
| **Security group blocking**        | Cloud network rules prevent traffic                                          | AWS/Azure rule doesn't allow A   |
| **Kubernetes NetworkPolicy**       | Cluster policy blocks A → B                                                  | Namespace/pod traffic denied     |
| **Routing problem**                | Packets cannot find a route to B                                             | Broken route/subnet              |
| **B overloaded**                   | B/infrastructure cannot accept new connections fast enough                   | Connection backlog exhausted     |
| **Load balancer problem**          | LB cannot successfully route A's connection                                  | Bad LB/backend configuration     |
| **Service discovery problem**      | A receives an unreachable/stale B endpoint                                   | Eureka/DNS returns dead instance |
| **Network congestion/packet loss** | TCP packets are delayed/dropped repeatedly                                   | Network issue between A and B    |
| **Proxy/service-mesh issue**       | Sidecar/proxy cannot establish upstream connection                           | Envoy/network proxy problem      |


      Service A
      │
      │ 1. Application / HTTP Client
      ↓
      HTTP Connection Pool
      │
      │ 2. DNS / Service Discovery
      ↓
      Service B hostname
      │
      ↓
      IP Address
      │
      │ 3. Network routing
      ↓
      Route / Subnet / VPC / VNet
      │
      │ 4. Firewall / Security Rules
      ↓
      Security Group / NACL / Firewall
      │
      │ 5. Load Balancer / Gateway (if present)
      ↓
      Load Balancer 
      │
      │ 6. Service / Kubernetes networking
      ↓
      Service B
      │
      │ 7. B's network interface
      ↓
      B's IP : Port
      │
      │ 8. TCP connection
      ↓
      B's application server
      │
      ↓
      B's Controller




| Part where the issue happens    | Can it cause `ConnectTimeoutException`? | What exactly happens?                                                                       |
| ------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Service A / HTTP client**     | ✅ Yes                                   | A starts a connection attempt but cannot establish it within the configured connect timeout |
| **Wrong/stale destination IP**  | ✅ Yes                                   | Hostname resolves successfully, but A tries to connect to an unreachable IP                 |
| **A's network**                 | ✅ Yes                                   | A's outgoing traffic cannot properly leave A's host/network                                 |
| **Routing between A and B**     | ✅ Yes                                   | Packets cannot find a valid path to B's network                                             |
| **Firewall**                    | ✅ Yes                                   | Firewall silently drops the TCP connection packets                                          |
| **Security Group / NSG / NACL** | ✅ Yes                                   | Infrastructure security rules drop/block the traffic                                        |
| **Kubernetes NetworkPolicy**    | ✅ Yes                                   | Pod A is not allowed to establish communication with Pod B                                  |
| **Load Balancer**               | ✅ Yes, depending on the hop             | A may be unable to connect to the LB itself                                                 |
| **Gateway**                     | ✅ Yes, depending on the hop             | A may be unable to connect to the Gateway itself                                            |
| **Network between A and B**     | ✅ Yes                                   | Packet loss or network failure prevents TCP handshake completion                            |
| **Service B host/VM/node**      | ✅ Yes                                   | The destination machine is unreachable or not responding                                    |
| **Service B overloaded**        | ⚠️ Possible                             | Under severe overload, new TCP connections may not be accepted/responded to                 |
| **TCP connection backlog full** | ⚠️ Possible                             | New connection attempts may be dropped when the server cannot accept more connections       |
| **Service mesh/proxy**          | ✅ Yes                                   | A proxy/sidecar cannot establish the required upstream connection                           |


| Which connection fails?    | Where is the actual problem? | Who experiences the connection timeout? |
| -------------------------- | ---------------------------- | --------------------------------------- |
| A → LB                     | Before/reaching LB           | Service A                               |
| LB → B                     | Backend path                 | Load Balancer/Gateway                   |
| LB → one bad B instance    | Specific backend             | Load Balancer                           |
| A → LB blocked by firewall | Network before LB            | Service A                               |
| LB → B blocked by firewall | Backend network              | Load Balancer                           |




Connection Refused :


| #  | Issue                                                        | Where the problem is             | Why it causes Connection Refused                                     |
| -- | ------------------------------------------------------------ | -------------------------------- | -------------------------------------------------------------------- |
| 1  | **Service B is not running**                                 | Service B application            | Nothing is accepting connections on that port                        |
| 2  | **Service B crashed/stopped**                                | Service B application/process    | OS has no application listening on the port                          |
| 3  | **Wrong port**                                               | Service A configuration          | A reaches the host but connects to a port where nothing is listening |
| 4  | **Application listening on a different port**                | Service B configuration          | Example: A calls `8080`, but B listens on `8081`                     |
| 5  | **Application bound only to localhost**                      | Service B server configuration   | B listens on `127.0.0.1`, not on the network interface               |
| 6  | **Container/pod is running but application isn't listening** | Inside container/pod             | Container may be UP, but the Java application failed                 |
| 7  | **Wrong Kubernetes Service targetPort**                      | Kubernetes Service configuration | Traffic reaches the pod/host but is forwarded to the wrong port      |
| 8  | **Wrong Load Balancer target port**                          | Load Balancer configuration      | LB connects to a port where backend isn't listening                  |
| 9  | **Service restarted and temporarily isn't listening**        | Deployment/application lifecycle | During startup/restart, port may not yet be open                     |
| 10 | **Explicit REJECT firewall rule**                            | Firewall                         | Firewall actively rejects instead of silently dropping traffic       |



Read TimeOut

      Service A  ───── TCP Connected ✅ ─────→ Service B
      
      Service A  ───── HTTP Request ─────────→ Service B
      
      Service A  ←──── waiting for response...
      
                   waiting...
                   waiting...
      
      Read Timeout ❌




| #  | Where the issue is          | Issue                             | Why it causes Read Timeout                              |
| -- | --------------------------- | --------------------------------- | ------------------------------------------------------- |
| 1  | **Service B application**   | Business logic is slow            | B received the request but takes too long to process it |
| 2  | **Service B application**   | Infinite loop / stuck code        | Request processing never completes                      |
| 3  | **Service B CPU**           | High CPU / CPU saturation         | B processes requests very slowly                        |
| 4  | **Service B thread pool**   | Thread pool exhausted             | Request waits for a free thread                         |
| 5  | **Service B thread pool**   | Threads blocked                   | Threads are waiting on DB, locks, or other operations   |
| 6  | **Database**                | Slow query                        | B waits too long for the database                       |
| 7  | **Database**                | Database overloaded               | Queries wait in queues or execute slowly                |
| 8  | **Database**                | Connection pool exhausted         | B waits for a DB connection                             |
| 9  | **Database**                | Database lock                     | Query waits for another transaction to release a lock   |
| 10 | **Downstream Service C**    | C is slow                         | B is waiting for C before responding to A               |
| 11 | **Downstream Service C**    | C hangs / doesn't respond         | B keeps waiting for C                                   |
| 12 | **Network during response** | Response packets are delayed/lost | Response doesn't reach A within the timeout             |
| 13 | **Large response**          | Very large payload                | Sending/reading the response takes too long             |
| 14 | **Service B JVM**           | Long Garbage Collection pause     | B temporarily stops processing requests                 |
| 15 | **External API**            | Third-party API is slow           | B waits for the external API response                   |
| 16 | **Lock contention**         | Thread waits for a lock           | Request cannot continue processing                      |


400 and 500

Service A
│
│ 1. TCP Connection established ✅
▼
Service B
│
│ 2. HTTP Request received ✅
│
├── Request problem → HTTP 4xx
│
└── Server problem  → HTTP 5xx


Why TLS is needed	What it protects against
🔒 Encryption	Others reading sensitive data
🪪 Authentication	Connecting to a fake server
🛡️ Integrity	Data being modified during transit


      Service A
      │
      │ 1. DNS
      ▼
      Resolve hostname
      │
      │ 2. TCP Connection
      ▼
      TCP Connected ✅
      │
      │ 3. TLS/SSL Handshake
      ▼
      TLS established? ❌
      │
      └── TLS/SSL Error
