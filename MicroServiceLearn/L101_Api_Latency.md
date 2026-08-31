High traffic / increased request rate

        Sudden traffic spike
        Traffic concentrated on one instance
        Bot/malicious traffic

CPU problems
    
    High CPU utilization
    CPU-intensive business logic
    Infinite/expensive loops
    Heavy serialization/deserialization
    Encryption/compression overhead

Memory / JVM problems
    
    High memory usage
    Excessive GC
    Long GC pauses
    Memory leaks
    Heap pressure

Thread pool problems
    
    Thread pool exhaustion
    Too many concurrent requests
    Threads blocked waiting for DB/downstream
    Incorrect thread-pool configuration

Database problems

        Slow SQL query
        Missing/incorrect indexes
        Large table scans
        N+1 queries
        Database CPU/memory saturation
        Lock contention
        Deadlocks
        Too many DB connections
        DB connection pool exhaustion

Downstream service problems
        
        Slow Service B
        Service B overloaded
        Service B unavailable/restarting
        Slow external API
        Network latency between services

Network problems
        
        Network congestion
        High network latency
        Packet loss
        Connection establishment delay
        DNS resolution delay
        TLS handshake delay

Connection pool problems
    
    HTTP connection pool exhausted
    DB connection pool exhausted
    Waiting for an available connection
    Too many connections

Retries

        Excessive retries
        Retry storms
        Multiple downstream retries
        Poor retry configuration

Locks / concurrency
    
    Database locks
    Java synchronized contention
    Distributed locks
    Concurrent updates waiting on each other

Caching problems

    Cache miss
    Cache unavailable
    Cache eviction causing repeated DB calls
    Slow Redis/cache operations

Application/code problems
        
        Inefficient algorithms
        Large object processing
        Excessive loops
        Unnecessary API calls
        Excessive logging
        Blocking operations

Garbage collection / object creation
        
        Creating too many objects
        Frequent young GC
        Full GC
        Large heap cleanup

Load balancer / gateway
        
        Gateway processing delay
        Gateway overloaded
        Load-balancing issue
        Uneven traffic distribution
        Rate limiting/throttling delays

External dependencies
        
        Payment service slow
        Third-party API slow
        DNS/provider problems
        External database/service slow

Deployment / infrastructure
        
        Instance resource exhaustion
        Container CPU/memory limits
        Kubernetes pod throttling
        Instance restarting
        Autoscaling delay
        Cold starts
Observability/logging overhead
        
        Excessive synchronous logging
        Large log payloads
        Tracing overhead in extreme cases

Serialization / response size

    Very large request/response
    Complex JSON serialization
    Large database result being converted to JSON





1. High Traffic 

        Genuine Traffic
        Malicious User
        retry strom 


| RPS increases | Metric that increases / changes             | What exactly is happening / problem caused                                                                                                                                                                                                                                                                                                                  | How we solve it                                                                                                                                                                                                                                                |
| ------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RPS ↑**     | **CPU utilization ↑**                       | More requests mean more application work: authentication, validation, business logic, serialization, calculations, etc. CPU has to process more work. If CPU reaches ~90–100%, requests compete for CPU time and take longer to execute → **API latency ↑, especially P95/P99**. If demand continues beyond capacity, requests may eventually timeout/fail. | First identify CPU-heavy code using profiling/thread dumps. Optimize expensive logic/queries/serialization. Scale **horizontally** by adding instances. Auto-scale based on CPU/RPS. If appropriate, tune CPU limits/resources.                                |
| **RPS ↑**     | **Thread pool active threads ↑ / queue ↑**  | More concurrent requests need application threads. If all worker threads are busy, new requests have to **wait in the queue**. The request may not even start processing immediately → **latency ↑**. If the queue becomes full, requests can be rejected → **errors ↑**.                                                                                   | Identify why threads are busy. If CPU-bound, optimize/scale. If threads are blocked on DB/downstream calls, fix that dependency. Tune thread-pool size carefully; don't blindly increase it because too many threads can cause CPU/context-switching overhead. |
| **RPS ↑**     | **DB connection pool active ↑ / pending ↑** | More requests reach the database and need connections. If all DB connections are occupied, new requests **wait for a connection**. Application threads sit waiting → **latency ↑**. If connection-acquisition timeout occurs → errors/timeouts.                                                                                                             | Optimize DB usage and query duration. Fix connection leaks. Tune pool size based on DB capacity. Optimize indexes/queries. Scale DB if necessary. Don't simply keep increasing the pool because the DB itself has finite capacity.                             |
| **RPS ↑**     | **DB query latency ↑**                      | More requests generate more database work. DB CPU, I/O, locks, buffer/cache pressure, etc. can increase. Queries take longer → application threads wait longer for DB → **API latency ↑**.                                                                                                                                                                  | Use slow-query analysis and `EXPLAIN`. Add/fix indexes, optimize SQL, reduce unnecessary queries/N+1 queries, caching, read replicas where appropriate, DB scaling.                                                                                            |
| **RPS ↑**     | **DB CPU ↑**                                | Database receives more queries and therefore performs more CPU work. When DB CPU approaches saturation, query execution slows down. This causes application requests waiting on DB → **API latency ↑**.                                                                                                                                                     | Optimize expensive queries/indexes, reduce unnecessary DB calls, cache frequently accessed data, scale DB resources/read replicas where appropriate.                                                                                                           |
| **RPS ↑**     | **Downstream service latency ↑**            | Service A receives more requests and sends more requests to Service B. Service B may not have enough capacity. B becomes slow → A waits for B → **A's latency increases**. If B times out, A may also return errors.                                                                                                                                        | Scale downstream service, optimize it, use caching, asynchronous processing where appropriate, circuit breakers/timeouts, bulkheads, and controlled retries.                                                                                                   |
| **RPS ↑**     | **HTTP connection pool active/pending ↑**   | More requests require connections to downstream services. If the connection pool is exhausted, requests wait for an available connection. This waiting time becomes part of API latency.                                                                                                                                                                    | Tune connection pool based on downstream capacity, reuse connections/keep-alive, investigate slow downstream responses, scale downstream service, set sensible timeouts.                                                                                       |
| **RPS ↑**     | **GC frequency / GC pause time ↑**          | More requests can create more temporary Java objects. More object allocation means the JVM has more garbage to clean. GC runs more frequently or takes longer → application threads may pause → **latency spikes**, especially P95/P99.                                                                                                                     | Reduce unnecessary object creation, optimize application code, investigate memory/heap usage, tune JVM/GC only after identifying the cause, and scale instances if needed.                                                                                     |
| **RPS ↑**     | **Heap usage ↑**                            | More concurrent requests can create more objects and retain more data. If heap pressure becomes high, GC becomes more frequent. Severe pressure can cause long GC pauses or `OutOfMemoryError`.                                                                                                                                                             | Find memory leaks/unnecessary retention, optimize object usage, configure appropriate heap, tune GC, and scale resources if required.                                                                                                                          |
| **RPS ↑**     | **Lock contention ↑**                       | More requests concurrently access the same shared resource. Threads may compete for synchronized blocks, locks, database rows, etc. Threads spend time **waiting instead of doing work** → latency increases.                                                                                                                                               | Identify the contended lock using thread dumps/profiling. Reduce critical-section size, improve concurrency design, optimize DB transactions, reduce unnecessary locking.                                                                                      |
| **RPS ↑**     | **Queue/message backlog ↑**                 | More work is being produced than the consumer can process. Requests/jobs accumulate in a queue. The work waits longer before processing → **end-to-end latency increases**.                                                                                                                                                                                 | Increase consumers/workers if the downstream system can handle it, optimize processing, partition work, apply backpressure, rate-limit producers.                                                                                                              |
| **RPS ↑**     | **Network latency / bandwidth ↑**           | More requests/responses mean more network traffic. If network capacity is insufficient, packets can be delayed or retransmitted. Service-to-service communication becomes slower → API latency increases.                                                                                                                                                   | Check network utilization, packet loss and latency. Increase capacity, reduce payload size, compression where appropriate, connection reuse, and improve network topology.                                                                                     |
| **RPS ↑**     | **Cache hit rate ↓ / cache load ↑**         | More requests may cause more cache operations. If cache misses increase, requests fall back to the DB. That creates additional DB load → DB becomes slower → API latency increases.                                                                                                                                                                         | Improve cache strategy/TTL, cache frequently accessed data, prevent cache stampedes, scale cache infrastructure where required.                                                                                                                                |
| **RPS ↑**     | **Error rate ↑ (429/5xx/timeouts)**         | The system has reached some capacity limit. Requests are being rejected, timing out, or failing because CPU, threads, DB, downstream services, etc. cannot handle the load.                                                                                                                                                                                 | Identify the underlying saturated component. Apply rate limiting/backpressure, scale horizontally, optimize bottleneck, and use circuit breakers/bulkheads where appropriate.                                                                                  |
| **RPS ↑**     | **P95/P99 latency ↑**                       | Not necessarily every request is slow. Under increased load, a portion of requests may experience queueing, DB waits, GC pauses, or downstream delays. Therefore **tail latency** increases significantly.                                                                                                                                                  | Use tracing/metrics to find where the slow requests spend their time. Fix the actual bottleneck rather than simply increasing timeouts.                                                                                                                        |


"RPS increased, so I would check CPU, thread pools, database, connection pools, and downstream latency to identify which resource became saturated."\


1.  First I would look into metrices like rps , P95, p99 latencies,  cpu, threadpools, db connection pool, gc etc and try to narrow down the problem
2. Say for instace if rps has increases first I would try to understand why that has happened, it might be because of an 





2. High CPU


| Reason                                 | What is happening                                                              | Example                                                |
| -------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------ |
| **More requests**                      | More requests mean more application work, so CPU naturally increases           | 100 RPS → 1,000 RPS                                    |
| **CPU-intensive business logic**       | Each request itself requires significant computation                           | Complex calculations, pricing, recommendation logic    |
| **Inefficient algorithm**              | Code performs much more work than necessary                                    | O(n²) loop processing a large list                     |
| **Large data processing**              | Application processes large payloads/results in memory                         | Processing 100,000 records per request                 |
| **Serialization/deserialization**      | Converting large/complex objects to/from JSON consumes CPU                     | Huge JSON request/response                             |
| **Encryption/hashing**                 | Cryptographic operations consume CPU                                           | JWT/signature validation, encryption, password hashing |
| **Compression**                        | Compressing/decompressing large payloads requires CPU                          | GZIP response compression                              |
| **Excessive object creation**          | Application creates huge numbers of objects                                    | Creating thousands of DTOs/temporary objects           |
| **Excessive GC**                       | Lots of objects become garbage → JVM spends CPU cleaning them                  | High allocation rate → frequent GC                     |
| **Infinite/very expensive loop**       | A bug causes a thread to continuously consume CPU                              | `while` loop that never terminates                     |
| **Too many threads/context switching** | Excessive threads compete for CPU and the OS spends CPU switching between them | Thread count grows abnormally                          |
| **Busy waiting**                       | Threads repeatedly check something instead of blocking/sleeping                | Polling in a tight loop                                |
| **Logging overhead**                   | Huge amounts of logging/formatting can consume CPU                             | Logging large objects on every request                 |
| **Traffic to one instance**            | Load isn't distributed evenly, so one instance gets overloaded                 | Instance A = 90% CPU, B/C = 30%                        |
