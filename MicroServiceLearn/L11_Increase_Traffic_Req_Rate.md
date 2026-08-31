1. What is traffic / request rate?

Simply:

    Request rate = how many requests your service receives per second.
    
    Usually measured as RPS (Requests Per Second) or sometimes RPM (Requests Per Minute).

Example:

    10:00:00 → 100 requests/sec
    10:01:00 → 105 requests/sec
    10:02:00 → 500 requests/sec  ← sudden spike

    So if your API normally receives 100 RPS but suddenly receives 1,000 RPS, the application may become slow because its resources are being consumed faster.

2. Why can request rate suddenly increase?

There are several common reasons.


1. Genuine increase in users / business traffic

   Scenario

        Suppose your Order Service normally receives:
        
        Normal traffic = 200 RPS
        
        A festival sale starts:
        
        Traffic = 2,000 RPS
        
        Nothing is necessarily wrong with the application. There are genuinely more users.

What happens?
    
    More users
    ↓
    More API requests
    ↓
    More CPU / threads / DB connections
    ↓
    Application reaches capacity
    ↓
    Latency increases

How do you identify it?

In monitoring:

    Request rate ↑
    CPU ↑
    Memory may ↑
    Latency ↑

And importantly, the traffic is distributed normally across many users/endpoints.

Solution

Short term:

    Scale horizontally.
    Add more service instances.
    Use load balancing.
    Enable autoscaling.
    
    ┌── Instance 1
    ├── Instance 2
    Gateway ──────┼── Instance 3
    ├── Instance 4
    └── Instance 5

Long term:

    Capacity planning.
    Optimize expensive APIs.
    Optimize DB queries.
    Caching where appropriate.

    
Interview answer

    "If the traffic increase is legitimate due to increased user demand, I would scale the service horizontally and use autoscaling. I would also verify that downstream dependencies such as the database can handle the increased load."


-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


2. One client is sending too many requests

   Scenario

        Normally:
        
        Client A → 50 RPS
        Client B → 50 RPS
        Client C → 50 RPS

Total:

    150 RPS

Suddenly Client C starts sending:

    Client C → 2,000 RPS

Now:

    50 + 50 + 2000 = 2100 RPS

    One client is consuming most of your capacity.

Why could this happen?

Possibilities:

    Client bug
    Incorrect polling
    Client retry problem
    Misconfigured application
    Malicious behavior

How do you identify it?

Don't just look at total RPS.

Break it down:

    Total RPS
    ↓
    Client-wise RPS
    ↓
    Client C = 2000 RPS

You might identify this through:

    API Gateway metrics
    Access logs
    Client ID
    API key
    IP
    User ID
    Solution

Use rate limiting.

For example:

        Client C
        ↓
        Rate limiter
        ↓
        Maximum 100 RPS
        ↓
        Service

    Requests beyond the allowed rate are throttled/rejected.
    
    For HTTP APIs, 429 Too Many Requests is commonly used.
    
    Also contact/fix the client if the traffic is caused by a bug.

Interview answer

    "If one client is generating disproportionate traffic, I would identify it using client ID/API key/IP metrics, apply rate limiting at the gateway, and investigate why that client is generating excessive requests."


-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


3. Retry storm

        This is very important for microservices interviews.

Scenario

Suppose:

    Order Service → Payment Service

Normally:

        Order Service
        ↓
        Payment Service
        ↓
        Response

    Payment Service becomes slow.

Order Service has retry logic:

    Request
    ↓
    Payment Service
    ↓
    Timeout
    ↓
    Retry
    ↓
    Timeout
    ↓
    Retry

Now imagine 1,000 original requests.

Instead of:

1,000 requests

Payment Service may receive:

    1,000 original
    + 1,000 retry
      + 1,000 retry
        = 3,000 requests

Now Payment becomes even slower.

Which causes:

    More timeout
    ↓
    More retries
    ↓
    More traffic
    ↓
    More load
    ↓
    More timeout

This is a feedback loop.

How do you identify it?

You might see:

    Request rate ↑↑
    Latency ↑
    Timeouts ↑
    5xx ↑
    Retry count ↑

You can also inspect application/APM metrics to see that many requests are retries.

Solution

1. Exponential backoff

        Don't retry immediately.

Instead:

    Retry 1 → after 100ms
    Retry 2 → after 200ms
    Retry 3 → after 400ms

2. Jitter

        Add randomness so thousands of clients don't retry simultaneously.

3. Limit retry attempts

       For example:
    
       Maximum retries = 2 or 3

4. Circuit breaker

If the downstream service is unhealthy:

        Service A → Service B
        ↓
        unhealthy

Circuit breaker opens:

    Service A ─X→ Service B

Instead of continuously hammering B.

5. Fix the actual downstream problem

        Retries are not the real solution if the downstream service is broken.

Interview answer

    "If request rate increased because of retries, I would check retry metrics and timeouts. I would prevent retry storms using limited retries, exponential backoff with jitter, and a circuit breaker, while investigating why the downstream service is failing."

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


4. Application bug causing excessive API calls

Scenario

    Suppose frontend normally calls:

GET /products

once.

But a frontend bug causes:

    GET /products
    GET /products
    GET /products
    GET /products
...

    Maybe a polling interval accidentally changed from:

    60 seconds
    
    to:
    
    1 second

What happens?
    
    Frontend bug
    ↓
    Too many API calls
    ↓
    Request rate ↑
    ↓
    Backend load ↑
    ↓
    Latency ↑
How do you identify it?

Look at:

    Endpoint-level traffic

You might see:

    GET /products
    Normal = 100 RPS
    Current = 5000 RPS

Then identify which clients/users are generating it.

Solution

Immediate protection:

    Rate limiting
    Throttling
    Caching

Permanent solution:

Fix the frontend/client bug.

For example, correct:

    poll every 1 second

to:

    poll every 60 seconds

    or use a more appropriate mechanism such as WebSockets/server-sent events if real-time updates are actually required.

Interview answer

    "If a particular endpoint suddenly receives abnormal traffic, I would check client-level and endpoint-level metrics. If it's caused by a client bug such as excessive polling, I would apply temporary rate limiting and fix the client logic."

5. Duplicate requests

        This is slightly different from the previous scenario.

Scenario

A user clicks:

Pay

Frontend accidentally sends:

        POST /payment
        POST /payment
        POST /payment

Maybe because:

    Button can be clicked multiple times
    Network timeout causes client retry
    Frontend doesn't disable the button
    Request handling is incorrect

Why is this dangerous?

For a read:

    GET /products

duplicate requests mainly create load.

But for:

POST /payment

duplicates can cause:

    ₹1000 payment
    ₹1000 payment
    ₹1000 payment

That's a correctness problem, not just a performance problem.

Solution

Use idempotency for operations such as payments/orders.

Example:

POST /payment


    Idempotency-Key: ABC123

If the same request arrives again:

    ABC123

the server recognizes it as the same operation and doesn't execute it again.

Also fix the client behavior.

Interview answer

    "For duplicate requests, especially for POST operations like payments, I would use an idempotency key so retries or duplicate submissions don't create duplicate business operations."

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


6. Bot / malicious traffic

Scenario

    Your API normally receives:

    10,000 requests/minute

Suddenly:

    500,000 requests/minute

And most requests are coming from:

    same IP range
    same API key
    same endpoint

Could be bots or an attack.

What happens?
    
    Malicious traffic
    ↓
    API Gateway
    ↓
    Application
    ↓
    CPU / connections exhausted
    ↓
    Real users become slow

How do you identify it?

Look for patterns:

    Same IP
    Same API key
    Same user agent
    Huge request rate
    Unusual geographic pattern
    Repeated endpoint

Solution

At the edge/gateway:

    Internet
    ↓
    WAF / CDN
    ↓
    Rate limiting
    ↓
    API Gateway
    ↓
    Service

Possible controls:

    Rate limiting
    IP blocking
    WAF rules
    Bot protection
    Authentication
    Request quotas

Interview answer

    "If the traffic appears malicious, I would protect the application at the edge using WAF, rate limiting, bot protection and IP/API-key controls, rather than allowing the traffic to consume application resources."

7. Traffic spike / burst

        This is different from sustained high traffic.

Scenario

Normally:

    100 RPS

Suddenly:

    10:00 → 100
    10:01 → 100
    10:02 → 5000
    10:03 → 5000
    10:04 → 100

Maybe a notification or event caused thousands of users to hit the API simultaneously.

Problem

    Even if the average traffic is manageable:
    
    Average = 100 RPS
    
    the system might not be designed for:
    
    Burst = 5000 RPS
Solution

Depending on the operation:

If synchronous:

    Autoscaling
    Rate limiting
    Load balancing
    Caching

If asynchronous:

Use a queue.

    Users
    ↓
    API
    ↓
    Queue
    ↓
    Consumers
    ↓
    Processing

The queue absorbs the burst.


-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



The flow you should remember
    
    1. Monitoring
       ↓
       Latency ↑
       Traffic/RPS ↑
       ↓
       2. APM
          ↓
          Which endpoint?
          Which client?
          Which service?
          Error rate?
          Trace behavior?
          ↓
       3. Identify the traffic pattern
          ↓
          Genuine increase?
          One client?
          Retries?
          Duplicate calls?
          Bot?
          ↓
       4. Find root cause
          ↓
       5. Apply solution
     
 What exactly do we look at?

Suppose monitoring tells us:

        RPS:      200 → 1500
        Latency:  200ms → 2 sec

Now in APM we drill down.

1. Endpoint-wise RPS
    
       GET /products     1000 RPS
       POST /orders       100 RPS
       GET /users          50 RPS

This tells us which API is responsible.

2. Client-wise RPS

Now:
    
    Client A → 100 RPS
    Client B → 100 RPS
    Client C → 1300 RPS

This strongly suggests one client is responsible.

But we don't yet know why.

3. Look at traces

Suppose traces show:

    Order Service
    ↓
    Payment Service
    ↓
    timeout
    ↓
    retry
    ↓
    Payment Service

Now we know:

    This is likely a retry storm.

So traces are extremely useful here.

4. Look at errors/timeouts

Suppose:

    RPS ↑
    5xx ↑
    Timeouts ↑
    Retry count ↑

That supports the retry-storm hypothesis.

5. Look at client/source patterns

Suppose:

    Client A → 50 RPS
    Client B → 50 RPS
    Client C → 5000 RPS

Then inspect Client C:

    Same API key
    Same IP range
    Same endpoint
    Very high frequency

That could indicate:

    abnormal client behavior / bot / attack.

    But you would investigate further before calling it malicious.

The important thing you're missing

You are thinking:

    Monitoring → APM → client-wise RPS → identify cause

Almost.

It should be:

    Monitoring → APM → break traffic down → investigate the pattern using metrics + traces + logs → identify root cause.

Think of APM as giving you different dimensions:

                            APM
                             │
               ┌─────────────┼─────────────┐
               ↓             ↓             ↓
        Endpoint       Client        Service
        RPS           RPS            RPS
        │             │             │
        └─────────────┼─────────────┘
                    ↓
                    Traces
                    ↓
                    Errors/Timeouts
                    ↓
                    Logs
                    ↓
                    Root cause
Example

If you see:
    
    Traffic ↑
    Latency ↑

You cannot immediately say "retry storm."

You investigate:

    Traffic ↑
    ↓
    Client-wise RPS
    ↓
    One client = huge traffic?
    │
    ├── YES → investigate client
    │           ↓
    │      traces/logs
    │           ↓
    │      retry? bug? malicious?
    │
    └── NO
    ↓
    Traffic increased across clients
    ↓
    Genuine traffic?
    ↓
    Check business event / endpoint pattern

And there's another important case:

Traffic may NOT increase at the entry point

Suppose:

    Gateway
    100 RPS
    ↓
    Order Service
    ↓
    Inventory Service
    
    Order Service receives only 100 requests, but each request suddenly makes 20 calls to Inventory.
    
    Inventory receives:
    
    100 × 20 = 2000 RPS
    
    So you need to look at downstream/service-to-service traffic too, not just client traffic.
    
    So your mental model should be
    
    Monitoring answers:
    
    "Something is wrong."
    
    APM metrics answer:
    
    "Where is the increased load coming from?"
    
    Traces answer:
    
    "What is happening inside the request?"
    
    Logs answer:
    
    "What exactly happened / why did it happen?"
    
    That's why we don't jump directly from traffic increase to a solution.
    
    Traffic increase is a symptom. We still need to identify the cause.