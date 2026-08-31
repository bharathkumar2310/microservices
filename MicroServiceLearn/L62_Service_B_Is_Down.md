Scenario: Service B is completely down

Assume:

    Service A
    |
    ↓
    Service B

X  ← DOWN

Service A is getting errors when calling B.

1. How do I identify that B is down?

        I wouldn't assume it immediately. I would confirm it using multiple signals.


| What I check                  | What I might see                              | What it tells me                  |
| ----------------------------- | --------------------------------------------- | --------------------------------- |
| **Service B health endpoint** | `/actuator/health` is DOWN/unreachable        | B may be unavailable              |
| **Service B instance count**  | 0 running instances/pods                      | B is definitely unavailable       |
| **Load Balancer**             | 0 healthy targets                             | No B instance can receive traffic |
| **Service A metrics**         | Connection refused/timeouts/503 increase      | A cannot communicate with B       |
| **Service B metrics**         | No requests, no metrics, instance disappeared | B may be completely down          |
| **Service B logs**            | No new logs / startup crash                   | B may have crashed                |
| **Deployment/orchestrator**   | Pods repeatedly restarting                    | B has startup/crash problem       |
| **Distributed tracing**       | A → B span fails                              | Confirms A's call to B is failing |



For example:

        A
        ↓
        B call → Connection refused
        ↓
        LB → 0 healthy B instances
        ↓
        Kubernetes → B pods = 0
        
        Now I can confidently say:
        
        Service B is unavailable.

2. Now WHY is Service B down?

This is the important part.

    A service can be down for many different reasons.

A. Application crashed

For example:

        Service B
        ↓
        Unhandled exception
        ↓
        Process crashes
        ↓
        Instance disappears

How identify?

Check B's logs:

        Exception
        OutOfMemoryError
        Fatal error
        Application startup failure
Fix

    Find and fix the application exception.

    If production is impacted:

    Roll back to the last known-good version if the issue was introduced by a deployment.

3. OutOfMemoryError

Example:

    java.lang.OutOfMemoryError: Java heap space

    B consumes all available heap.

    Memory
    ████████████████████ 100%
    ↓
    OOM
    ↓
    B crashes
How identify?

Check:

    JVM memory
    Heap usage
    GC metrics
    OOM logs
    Container memory limits
Fix

Don't simply keep increasing memory.

Investigate:

    Memory leak
    Unbounded cache
    Excessive object creation
    Large responses
    Incorrect heap configuration

Then fix the root cause and scale memory if appropriate.

4. CPU saturation

Suppose B isn't technically crashed, but CPU reaches 100%.

        CPU = 100%
        ↓
        Requests become extremely slow
        ↓
        Health checks fail
        ↓
        LB removes B
        ↓
        B appears unavailable
Identify

Check:

    CPU
    Request rate
    Latency
    Thread activity
    CPU profiling if necessary
    Fix

Depending on root cause:

        Optimize CPU-heavy code
        Reduce unnecessary processing
        Scale horizontally
        Investigate traffic spike
5. Thread pool exhaustion

This is especially important for your current preparation.

Suppose:

B thread pool = 200

        200 threads
        ↓
        waiting on DB/downstream/locks
        ↓
        No threads available
        ↓
        Health check/request waits
        ↓
        LB health check times out
        ↓
        B marked unhealthy
Identify

Check:

        Active threads
        Maximum threads
        Queue size
        Thread dump
        DB connection pool
        Downstream latency
Fix

Don't immediately increase the thread pool.

Find why threads are blocked.

For example:

        Thread pool exhausted
        ↓
        Threads waiting for DB
        ↓
        DB connection pool exhausted
        ↓
        Slow DB queries

Fix the slow query/root cause.

6. Database dependency is down

B itself may be running.

But suppose B's startup requires DB:

    B starts
    ↓
    Connect to DB
    ↓
    DB unavailable
    ↓
    B startup fails
    ↓
    B goes down
Identify

Check B startup logs:

    Unable to connect to database
    Connection refused
    Database unavailable

Also check DB health.

Fix

Restore DB connectivity.

Depending on architecture, ideally B should not necessarily become completely unavailable just because a non-critical dependency is down.

Use appropriate:

        Timeouts
        Circuit breakers
        Fallbacks
        Graceful degradation
7. Bad deployment

This is one of the first things I would check in production.

Suppose:

    10:00 → Deployment B v2
    10:02 → B instances start crashing
    10:03 → B unavailable

That's a very strong correlation.

Identify

Check:

    Deployment history
    Error rate before/after deployment
    Startup logs
    Configuration changes
    New dependencies
Fix

If confirmed:

    Rollback to the last known-good version.
    
    Then investigate and fix v2 before redeploying.

8. Configuration problem

B may fail because of incorrect configuration.

For example:

    DB_URL = wrong
    PORT = wrong
    SECRET = wrong
    DOWNSTREAM_URL = wrong
Identify

Check startup logs and deployment configuration.

For example:

    Failed to bind to port
    Unable to connect to downstream
    Missing environment variable
    Invalid configuration
Fix

Correct the configuration and redeploy/restart.

9. Infrastructure / container problem

If using Kubernetes/container infrastructure:

    Pod
    ↓
    CrashLoopBackOff
    
    or:
    
    Pod
    ↓
    OOMKilled
    
    or:
    
    Readiness probe failed
    Identify
    
    Check:
    
    Pod status
    Restart count
    Events
    Readiness/liveness probes
    Container logs
    Resource limits
    Fix
    
    Depends on the reason:
    
    Fix application crash
    Increase resources if genuinely insufficient
    Fix health probes
    Fix container configuration
    Fix deployment
10. Network problem

Sometimes B is running perfectly but A cannot reach it.

This is an important distinction.

B = healthy
B = running
B = accepting requests

BUT

A ─────X────→ B
network

Possible causes:

Firewall
Security group
Network policy
Routing problem
LB issue
Service discovery issue

So don't say:

"B is down"

just because A cannot connect.

You need to confirm B's own health.

The complete interview answer

If interviewer asks:

"Service B is down. How do you identify the problem and fix it?"

You can answer:

"First I would confirm that B is actually down rather than assuming it from A's errors. I would check B's health endpoint, instance or pod count, load-balancer target health, B's metrics and logs, and distributed traces from A to B. Once I confirm B is unavailable, I would investigate why—whether it's an application crash, OOM, CPU or thread-pool exhaustion, failed startup due to DB or another dependency, bad deployment, configuration issue, or infrastructure/network problem. I would correlate the failure with recent deployments or configuration changes. If a recent deployment caused it, I would roll back to the last known-good version. Otherwise I would fix the specific root cause—for example memory leak, slow dependency, incorrect configuration, unhealthy instance, or network issue. After the fix, I would verify that B becomes healthy, LB health checks pass, A → B requests succeed, and error/latency metrics return to normal."

Remember this flow
B appears DOWN
↓
CONFIRM
↓
Health + instances + LB + metrics + logs
↓
WHY?
↓
Crash?
OOM?
CPU?
Threads?
DB?
Deployment?
Config?
Infrastructure?
Network?
↓
FIX ROOT CAUSE
↓
VERIFY
↓
B healthy + A → B successful


| Why Service B goes down            | What actually happens                                                                                    | How you identify it                                                  | Root-cause fix                                                                   |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Application crash**              | B process terminates because of an unhandled exception/fatal error                                       | B logs, stack trace, process/pod restart count                       | Fix the application bug; deploy corrected version                                |
| **OutOfMemoryError / OOMKilled**   | B consumes available heap/container memory and gets killed                                               | OOM logs, heap usage, GC metrics, container status                   | Fix memory leak/excessive allocation/unbounded cache; tune memory if required    |
| **CPU saturation**                 | B consumes near 100% CPU and becomes extremely slow/unresponsive; health checks may fail                 | CPU ~100%, latency increases, profiling                              | Optimize CPU-heavy code, reduce unnecessary processing, scale instances          |
| **Thread-pool exhaustion**         | All request threads are occupied, often waiting on DB/downstream/locks; new requests cannot be processed | Active threads = max, queue grows, thread dump                       | Find what threads are waiting for; fix DB/downstream/lock issue; scale if needed |
| **Excessive GC**                   | JVM spends too much time doing garbage collection, leaving little time for request processing            | GC frequency/pause time increases, CPU/latency increases             | Reduce object allocation, fix leaks/cache growth, tune heap/GC                   |
| **DB unavailable during startup**  | B starts but cannot establish required DB connection, so startup fails                                   | Startup logs show DB connection errors                               | Restore DB/connectivity; fix DB configuration                                    |
| **DB connection pool exhausted**   | B's threads wait for DB connections; eventually B becomes unresponsive                                   | Hikari active/pending connections high, thread dump shows DB waits   | Fix slow queries/leaks/long transactions; tune pool appropriately                |
| **Slow DB / locking**              | B's requests remain blocked waiting for DB responses/locks; thread pool can eventually exhaust           | Tracing shows DB consuming most latency; DB lock/query metrics       | Optimize queries/indexes, reduce transaction duration, fix locking               |
| **Downstream service unavailable** | B waits/retries for another service; enough blocked requests can exhaust B's resources                   | Tracing shows B → downstream failures/timeouts; thread pool grows    | Add proper timeout, circuit breaker, fallback; fix downstream                    |
| **Bad deployment**                 | New B version crashes or fails health checks after deployment                                            | Errors start immediately after deployment; deployment history + logs | Roll back to last known-good version; fix release                                |
| **Bad configuration**              | B cannot start because URL, port, credentials, environment variable, etc. are wrong                      | Startup logs/config comparison                                       | Correct configuration and redeploy/restart                                       |
| **Health-check failure**           | B may actually be running, but readiness/health check fails, so LB/orchestrator removes it               | Instance is running but marked unhealthy; health endpoint fails      | Fix health-check endpoint/dependency or probe configuration                      |
| **Container/pod failure**          | Container crashes, gets evicted, or enters `CrashLoopBackOff`                                            | Container/pod status, events, restart count, logs                    | Fix underlying application/resource/configuration problem                        |
| **Infrastructure failure**         | VM/node/container host itself fails                                                                      | Multiple B instances on same host disappear; infrastructure alerts   | Replace/recover node/VM and restore instances                                    |
| **Traffic spike**                  | Sudden traffic exceeds B's capacity, exhausting CPU, threads, connections or memory                      | Request rate suddenly increases with resource saturation             | Rate limiting, autoscaling, load shedding, optimization                          |
| **Dependency/resource exhaustion** | B exhausts a resource such as DB connections, file descriptors, sockets or connection pools              | Resource-specific metrics/errors                                     | Release/leak resources correctly; tune limits and fix root cause                 |

    "First, I'll confirm that Service B is actually unavailable by checking its health endpoint, instance/pod status, load-balancer health, and metrics. Then I'll check whether there was a recent deployment or configuration change, because a bad deployment or startup failure is a common cause of sudden unavailability.
    
    Then I'll check application logs for crashes or startup exceptions and look at resource metrics such as memory/OOM, CPU, GC and thread-pool utilization. I'll also compare traffic with the normal baseline to see whether a sudden traffic spike caused resource exhaustion.
    
    If B itself looks healthy but is becoming unavailable under load, I'll investigate its dependencies—downstream services and database—checking their latency, errors, connection pools and failures. Finally, I'll check infrastructure and health-check issues if necessary.
    
    Once I identify the root cause, I'll apply the appropriate fix—for example rollback a bad deployment, fix a memory issue, optimize a slow database query, resolve a downstream failure, or scale the service. Then I'll verify that B is healthy again and monitor error rate, latency and resource utilization.