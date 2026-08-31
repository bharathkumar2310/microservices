1. First confirm what the 502 means

        A 502 Bad Gateway generally means:

        Gateway received the request, tried to communicate with the downstream service, but could not get a valid response.

First check gateway logs for the exact failure:

        Connection refused
        Connection timeout
        Read timeout
        DNS resolution failure
        Connection reset
        Invalid response
        TLS/SSL failure

Also check whether all requests are failing or only some.

2. Check whether the downstream service is healthy

        From gateway → downstream:

Gateway → Service B

Check:

        Service B health/readiness
        Service B instances
        Error rate
        CPU
        Memory
        Thread pool
        Connection pool
        Application logs

For example, if Service B has 5 instances:

    Gateway
    |
    +---- B1 ❌
    +---- B2 ✅
    +---- B3 ✅
    +---- B4 ✅
    +---- B5 ✅

If only B1 is failing, this is likely an instance-specific problem, not a complete Service B outage.

3. Check connectivity from the gateway

This is important.

    Don't just test Service B from your laptop.

Test:

        Gateway host/container
        ↓
        Service B host/container

Check:

    DNS resolution
    Port accessibility
    Network connectivity
    Firewall/security rules
    Kubernetes service/network policy if applicable
    TLS certificate issues

If gateway cannot connect to B, you'll commonly see connection refused/timeout type errors.

4. Check service discovery / routing

If you're using Eureka/service discovery:

    Gateway
    ↓
    Eureka
    ↓
    Service B instances

Check:

        Is Service B registered?
        Are stale/dead instances still registered?
        Is the gateway getting the correct IP/port?
        Is the service name correct?
        Is the route configured correctly?

A particularly important case:

Eureka says:

        B1 → 10.0.0.10:8080  ❌ dead
        B2 → 10.0.0.11:8080  ✅

Gateway may occasionally route requests to the dead instance, producing intermittent 502s.

5. Check gateway configuration

Verify:

    Route configuration
    Target URI
    Path rewriting
    Timeouts
    TLS configuration
    Load-balancing configuration
    Retry configuration

For example:

    /api/orders/**
    ↓
    lb://ORDER-SERVICE
    
    Make sure ORDER-SERVICE is actually the expected service and that its instances are reachable.

6. Check whether the downstream is overloaded

Suppose Service B is technically UP, but:

    CPU        95%
    Thread pool 100%
    DB pool    exhausted
    P99        8 seconds

Gateway might timeout while waiting for B and return an error.

Then investigate Service B's:

    API latency
    ↓
    Thread pool
    ↓
    DB connection pool
    ↓
    DB query latency
    ↓
    Locks
    ↓
    External dependencies

So "Service B is UP" does not mean Service B is healthy from the gateway's perspective.

7. Check traces

If distributed tracing is available:

        Client
        ↓
        Gateway
        ↓
        Service B
        ↓
        Database

Look at the failed request's trace.

You can determine whether the failure happened:

Gateway → B

or

B → Database

or

B → another downstream service

This is much faster than guessing from logs.

8. Compare successful vs failed requests

This is an excellent production-debugging technique.

Suppose:

    90% requests → 200
    10% requests → 502

Ask:

What is different about the failed requests?

Check:

    Same Service B instance?
    Same API?
    Same region/AZ?
    Same request payload?
    Same time period?
    Same gateway instance?

If all failures go to one B instance:
    
    Gateway
    ├── B1 ❌ → 502
    ├── B2 ✅
    ├── B3 ✅
    └── B4 ✅

you've narrowed it down dramatically.

Interview answer

        If the interviewer asks "Gateway returns 502 for a downstream service. How do you debug?", I'd answer:
        
        "First I'd confirm whether the 502 is happening for all requests or only intermittently, and check the gateway logs to identify whether it's a connection timeout, connection refused, DNS, TLS, or invalid response issue. Then I'd check the downstream service health, error rate, latency, CPU, thread pool and connection pool. If there are multiple instances, I'd determine whether failures are isolated to a particular instance. I'd then verify connectivity from the gateway to the downstream, including DNS and network configuration, and check service discovery and gateway routing. If the downstream is reachable but slow, I'd investigate its thread pool, DB connection pool, database latency and any other downstream dependencies. Finally, I'd use distributed tracing to identify exactly where the request is failing and verify the fix by monitoring the 502 rate and latency."1. First confirm what the 502 means
        
        A 502 Bad Gateway generally means:
        
        Gateway received the request, tried to communicate with the downstream service, but could not get a valid response.
        
        First check gateway logs for the exact failure:
        
        Connection refused
        Connection timeout
        Read timeout
        DNS resolution failure
        Connection reset
        Invalid response
        TLS/SSL failure
        
        Also check whether all requests are failing or only some.
        
        2. Check whether the downstream service is healthy
        
        From gateway → downstream:
        
        Gateway → Service B
        
        Check:
        
        Service B health/readiness
        Service B instances
        Error rate
        CPU
        Memory
        Thread pool
        Connection pool
        Application logs
        
        For example, if Service B has 5 instances:
        
        Gateway
        |
        +---- B1 ❌
        +---- B2 ✅
        +---- B3 ✅
        +---- B4 ✅
        +---- B5 ✅
        
        If only B1 is failing, this is likely an instance-specific problem, not a complete Service B outage.
        
        3. Check connectivity from the gateway
        
        This is important.
        
        Don't just test Service B from your laptop.
        
        Test:
        
        Gateway host/container
        ↓
        Service B host/container
        
        Check:
        
        DNS resolution
        Port accessibility
        Network connectivity
        Firewall/security rules
        Kubernetes service/network policy if applicable
        TLS certificate issues
        
        If gateway cannot connect to B, you'll commonly see connection refused/timeout type errors.
        
        4. Check service discovery / routing
        
        If you're using Eureka/service discovery:
        
        Gateway
        ↓
        Eureka
        ↓
        Service B instances
        
        Check:
        
        Is Service B registered?
        Are stale/dead instances still registered?
        Is the gateway getting the correct IP/port?
        Is the service name correct?
        Is the route configured correctly?
        
        A particularly important case:
        
        Eureka says:
        
        B1 → 10.0.0.10:8080  ❌ dead
        B2 → 10.0.0.11:8080  ✅
        
        Gateway may occasionally route requests to the dead instance, producing intermittent 502s.
        
        5. Check gateway configuration
        
        Verify:
        
        Route configuration
        Target URI
        Path rewriting
        Timeouts
        TLS configuration
        Load-balancing configuration
        Retry configuration
        
        For example:
        
        /api/orders/**
        ↓
        lb://ORDER-SERVICE
        
        Make sure ORDER-SERVICE is actually the expected service and that its instances are reachable.
        
        6. Check whether the downstream is overloaded
        
        Suppose Service B is technically UP, but:
        
        CPU        95%
        Thread pool 100%
        DB pool    exhausted
        P99        8 seconds
        
        Gateway might timeout while waiting for B and return an error.
        
        Then investigate Service B's:
        
        API latency
        ↓
        Thread pool
        ↓
        DB connection pool
        ↓
        DB query latency
        ↓
        Locks
        ↓
        External dependencies
        
        So "Service B is UP" does not mean Service B is healthy from the gateway's perspective.
        
        7. Check traces
        
        If distributed tracing is available:
        
        Client
        ↓
        Gateway
        ↓
        Service B
        ↓
        Database
        
        Look at the failed request's trace.
        
        You can determine whether the failure happened:
        
        Gateway → B
        
        or
        
        B → Database
        
        or
        
        B → another downstream service
        
        This is much faster than guessing from logs.
        
        8. Compare successful vs failed requests
        
        This is an excellent production-debugging technique.
        
        Suppose:
        
        90% requests → 200
        10% requests → 502
        
        Ask:
        
        What is different about the failed requests?
        
        Check:
        
        Same Service B instance?
        Same API?
        Same region/AZ?
        Same request payload?
        Same time period?
        Same gateway instance?
        
        If all failures go to one B instance:
        
        Gateway
        ├── B1 ❌ → 502
        ├── B2 ✅
        ├── B3 ✅
        └── B4 ✅
        
        you've narrowed it down dramatically.

Interview answer

If the interviewer asks "Gateway returns 502 for a downstream service. How do you debug?", I'd answer:

"First I'd confirm whether the 502 is happening for all requests or only intermittently, and check the gateway logs to identify whether it's a connection timeout, connection refused, DNS, TLS, or invalid response issue. Then I'd check the downstream service health, error rate, latency, CPU, thread pool and connection pool. If there are multiple instances, I'd determine whether failures are isolated to a particular instance. I'd then verify connectivity from the gateway to the downstream, including DNS and network configuration, and check service discovery and gateway routing. If the downstream is reachable but slow, I'd investigate its thread pool, DB connection pool, database latency and any other downstream dependencies. Finally, I'd use distributed tracing to identify exactly where the request is failing and verify the fix by monitoring the 502 rate and latency."