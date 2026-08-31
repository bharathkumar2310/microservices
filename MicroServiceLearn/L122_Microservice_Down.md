Question 1: Service Down

You prioritize:

Is the process/pod running? → Is it healthy? → Is it reachable? → Did it crash? → Are instances available?

Typical checks:

        Pod/container status
        Application startup/crash logs
        Health/readiness
        CPU/memory
        Deployment problems
        Network/service discovery



1. First confirm what "down" means

Ask/check:

        Is the service completely unreachable?
        Are all requests failing?
        Are only some users affected?
        Is it returning errors or timing out?

This is important because "service down" can mean different things.

2. Check whether the service instances are running

For example:

        Is the application process running?
        Is the container/pod running?
        Are instances restarting?
        Are they healthy and ready?

If no instances are available, investigate:

Application crash
Out-of-memory
Bad deployment
Configuration problem
3. Check the routing/intermediate layers

If the service itself is running, check whether traffic can actually reach it:

Client
↓
Gateway / Load Balancer
↓
DNS / Service Discovery
↓
Network
↓
Service

Check:

API Gateway routing
Load balancer health checks
DNS/service discovery
Network/firewall/security rules
Correct port and endpoint configuration
4. Check whether it is actually slow rather than down

This is where your earlier point is important.

Check:

Request latency
Timeouts
CPU/memory
Thread pool exhaustion
Connection pool exhaustion

A service might be running but unable to process requests.

5. Check dependencies

The service may be healthy but unavailable from the user's perspective because:

Database is down/slow
Downstream service is unavailable
External API is failing
Cache is unavailable



Question 2: API Slow

You prioritize:

Which part of the request is slow?

Typical checks:

P95/P99 latency
Distributed tracing
Slow database queries
Downstream API latency
Thread pool saturation
Connection pool waiting
CPU/memory/GC



3. Service A calls Service B, but the request is timing out. What could be causing it?

"First, I would identify whether this is a connection timeout or a response/read timeout because they indicate different failure points. For a connection timeout, I would investigate the network path, DNS/service discovery, load balancer, firewall/security rules, and whether Service B is reachable and listening on the correct port. For a read timeout, I would investigate why Service B is taking too long by checking latency metrics, CPU, memory and GC, thread pools, database latency and connection pools, and downstream dependencies using distributed tracing. If the issue is intermittent, I would compare instances and check whether only specific instances or periods of high load are affected. Finally, I would use logs, metrics, and traces to identify the root cause, fix it, and verify the timeout is resolved."


5. One downstream service becomes slow. How can it affect other microservices?

"If a downstream service becomes slow, upstream services that synchronously call it will spend longer waiting for responses. Their threads, HTTP connections, or other resources can become occupied, causing request queues and latency to increase. Eventually resource pools can become exhausted, leading to timeouts and failures in the upstream services. If multiple services depend on the same downstream service, the problem can spread across the system and cause a cascading failure. Retries can make the situation worse by increasing load on the already slow service. To prevent this, I would use appropriate timeouts, circuit breakers, bulkheads/resource isolation, controlled retries with exponential backoff and jitter, and monitor downstream latency and resource saturation."


-----------------------------------------------------------------------------------------------------------------------------------------------------------

7. Your application works normally, but after deploying a new version, errors suddenly increase. How would you investigate?


. Confirm the correlation with the deployment

First check:

    Exactly when did errors start?
    Exactly when was the deployment completed?
    Did the error increase immediately after deployment?

Example:

2:00 PM → Version 1 working normally ✅
2:10 PM → Version 2 deployed
2:11 PM → Error rate increases ❌

This gives a strong correlation.

But remember:

Correlation is not yet proof.

2. Check what kind of errors increased

Check:

    HTTP 4xx?
    HTTP 5xx?
    Timeouts?
    Database errors?
    Connection errors?

This immediately narrows the investigation.

Example
        
        Before deployment:
        5xx = 0.1%
        
        After deployment:
        5xx = 15%

Then check:

        Which endpoints are producing those 500 errors?

3. Compare the new version with the old version

Check the changes introduced:

    Code changes
    Configuration changes
    Environment variables
    Dependency/library changes
    Database changes
    API contract changes

Ask:

What changed between Version 1 and Version 2?

This is one of the most important debugging questions.

4. Check application logs

        Look for new exceptions:
        
        Version 2 deployed
        ↓
        New exception appears
        ↓
        NullPointerException

Or:

    Database connection error
    Configuration missing
    Authentication failure

Compare logs before and after deployment.

5. Check whether all instances are affected ⭐

This is extremely important in real deployments.

Suppose:

        Version 1 → 5 instances
        Version 2 → 5 instances
        
        During a rolling deployment:
        
        Request 1 → Version 1 → ✅
        Request 2 → Version 2 → ❌
        Request 3 → Version 1 → ✅
        Request 4 → Version 2 → ❌

Now the errors may appear intermittently.

So check:

Which version handled failed requests?
Are only new instances failing?
Are all new instances affected?
6. Check deployment and configuration issues

Common problems after deployment:

❌ Wrong environment variable

        DB_URL = wrong
❌ Missing secret

    JWT_SECRET missing
❌ Wrong database configuration
❌ Wrong service URL
❌ Incompatible dependency
❌ Incorrect API configuration

The code may be correct, but the deployment configuration may be wrong.

7. Check database compatibility

A common production problem:

Application Version 2
↓
Expects new DB column
↓
Column doesn't exist ❌

Or:

Database schema changed
↓
Old application incompatible ❌

Check:

Database migrations
Schema compatibility
Deployment order
8. Use metrics and tracing

Check what changed after deployment:

Error rate ↑
Latency ↑
CPU ↑
Memory ↑

Then use tracing to identify whether the new version introduced:

Slow downstream calls
New database queries
Dependency failures
9. Mitigate first if users are affected ⭐

If production is severely affected:

Error rate = 40%

Don't spend one hour debugging while users are impacted.

Consider:

Rollback
Version 2 ❌
↓
Rollback
↓
Version 1 ✅

Or stop traffic to the bad version.

This is often the fastest way to restore service.

10. Find root cause after stabilizing

Once the service is stable:

Restore service
↓
Investigate root cause
↓
Fix
↓
Test
↓
Deploy safely
⭐ Strong interview answer

"Since the errors started immediately after a new deployment, I would first correlate the error spike with the deployment timeline and treat the recent change as the primary suspect. I would check which error types and endpoints are affected, then compare the new version with the previous version, including code, configuration, dependencies, environment variables, and database changes. I would check application logs and determine whether failures are occurring only on the newly deployed instances, which is especially important during a rolling deployment. If the production impact is significant, I would first mitigate the issue by rolling back or stopping traffic to the faulty version. After stabilizing the system, I would investigate the root cause, fix it, verify it through testing and monitoring, and use safer deployment practices to prevent recurrence."