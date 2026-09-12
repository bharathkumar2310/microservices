
1. A microservice is responding slowly. How do you identify whether the issue is code, DB, or network?
   Answer: Use distributed tracing (Zipkin/Jaeger), logs, metrics (Micrometer), and isolate latency source.

2. One dependent service is down. How do you prevent cascading failures?
   Answer: Circuit Breaker using Resilience4j with fallback logic.

3. Your API is called by multiple clients and traffic suddenly spikes. How do you protect the service?
   Answer: Rate limiting via API Gateway, load balancing, autoscaling.

4. You deployed a new version and users report errors. How do you roll back safely?
   Answer: Blue-green or canary deployment strategy.

5. Multiple microservices need the same configuration. How do you manage it?
   Answer: Spring Cloud Config Server with centralized configs.

6. A service restarts and loses its IP. How do other services still find it?
   Answer: Service Discovery using Eureka or Consul.

7. You need async communication between services. What approach do you use?
   Answer: Message broker like Kafka or RabbitMQ.

8. One request needs data from three services. Where should aggregation happen?
   Answer: API Gateway or a dedicated aggregator service.

9. How do you handle distributed transactions across services?
   Answer: Saga pattern (choreography or orchestration).

10. How do you secure internal microservice communication?
    Answer: OAuth2/JWT, mutual TLS, or service mesh security.

11. Database is shared by two microservices. Is this correct?
    Answer: No, each microservice should own its database.

12. How do you version APIs without breaking clients?
    Answer: URI versioning or header-based versioning.

13. Logs are scattered across services. How do you debug production issues?
    Answer: Centralized logging using ELK stack.

14. How do you ensure backward compatibility during deployment?
    Answer: Contract testing and tolerant readers.

15. One service consumes too much memory under load. How do you analyze it?
    Answer: JVM metrics, heap dump, GC logs.

16. How do you test microservices end-to-end?
    Answer: Integration tests, Testcontainers, contract tests.

17. You need to deploy services independently. What enables this?
    Answer: Loose coupling and independent CI/CD pipelines.

18. How do you handle timeouts between services?
    Answer: Configure timeouts + retries with circuit breakers.

19. When would you avoid microservices and choose monolith?
    Answer: Small teams, low scale, early-stage products.

20. How do you monitor system health in real time?
    Answer: Actuator endpoints, Prometheus, Grafana.

| #  | Scenario-based question                                                                               | Main concept tested             |
| -- | ----------------------------------------------------------------------------------------------------- | ------------------------------- |
| 1  | An API suddenly becomes slow. How would you investigate it?                                           | API latency / troubleshooting   |
| 2  | API latency increased from 200ms to 3 seconds after a deployment. What would you check?               | Regression                      |
| 3  | API P95 latency is high but P50 is normal. What does it indicate?                                     | Tail latency                    |
| 4  | API traffic suddenly increases 10x. What problems can occur?                                          | Traffic / saturation            |
| 5  | CPU of your service reaches 95% during peak traffic. How would you investigate?                       | CPU saturation                  |
| 6  | Memory usage continuously increases and eventually the service crashes. What would you investigate?   | Memory / GC / leak              |
| 7  | Your API is returning 500 errors intermittently. How would you debug it?                              | Production troubleshooting      |
| 8  | API returns 429 errors during high traffic. Why might this happen?                                    | Rate limiting                   |
| 9  | One particular endpoint is slow while all other endpoints are normal. How do you isolate the problem? | Endpoint-level debugging        |
| 10 | API response time is fine locally but very slow in production. What could cause this?                 | Environment/network differences |


| #  | Scenario-based question                                                                               | Main concept tested         |
| -- | ----------------------------------------------------------------------------------------------------- | --------------------------- |
| 11 | An API became slow and you discover the DB query takes 5 seconds. How do you investigate?             | DB troubleshooting          |
| 12 | A query performs a full table scan on a table with millions of rows. What would you do?               | Indexing / EXPLAIN          |
| 13 | A query is slow even though an index exists. Why?                                                     | Query optimizer / index     |
| 14 | One API suddenly creates thousands of DB queries for a single request. What could be happening?       | N+1                         |
| 15 | DB connection pool is exhausted. What could cause it?                                                 | Connection pooling          |
| 16 | DB CPU suddenly reaches 100%. How would you investigate?                                              | DB saturation               |
| 17 | After a new release, DB load increases significantly. What would you check?                           | Query regression            |
| 18 | Two requests update the same record at the same time and data becomes incorrect. How do you solve it? | Concurrency                 |
| 19 | A transaction remains open for a long time and other queries are blocked. How would you investigate?  | Locks / transactions        |
| 20 | Pagination becomes extremely slow when users request page 100,000. How would you improve it?          | Offset vs keyset pagination |


| #  | Scenario-based question                                                                                       | Main concept tested          |
| -- | ------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| 21 | Service A calls Service B, but B becomes slow. What happens to A?                                             | Cascading latency            |
| 22 | Service B is completely down. How should Service A behave?                                                    | Failure handling             |
| 23 | Service A calls B and the request times out. Should A retry?                                                  | Retry strategy               |
| 24 | Retries themselves cause the system to become overloaded. How can you prevent this?                           | Retry storm                  |
| 25 | Service B returns intermittent 500 errors. How would you handle them?                                         | Resilience                   |
| 26 | One downstream service is consistently failing. How can you prevent requests from reaching it?                | Circuit breaker              |
| 27 | A request passes through Gateway → Service A → B → C. The overall API is slow. How do you find exactly where? | Distributed tracing          |
| 28 | Service A is healthy, but requests from A to B are timing out. What would you check?                          | Network / downstream         |
| 29 | Service A receives a response from B after its timeout has already expired. What problems can occur?          | Timeout / resource handling  |
| 30 | A downstream service has a strict rate limit. How should your service communicate with it safely?             | Rate limiting / backpressure |



| #  | Scenario-based question                                                                                       | Main concept tested               |
| -- | ------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 31 | A Kafka consumer processes the same message twice. How do you prevent duplicate business operations?          | Idempotency                       |
| 32 | A Kafka consumer crashes after processing a message but before committing the offset. What happens?           | Offset / at-least-once            |
| 33 | Kafka consumer lag suddenly increases. How would you investigate?                                             | Consumer lag                      |
| 34 | One Kafka partition is receiving much more traffic than others. What problem is this?                         | Partition skew                    |
| 35 | A consumer is slower than the producer. What happens and how do you solve it?                                 | Backpressure / scaling            |
| 36 | A message fails repeatedly and blocks processing. How would you handle it?                                    | DLQ                               |
| 37 | An event is published but the database transaction later rolls back. How can you avoid inconsistent data?     | Transactional messaging / outbox  |
| 38 | Two consumers process related events in the wrong order. How would you guarantee ordering where required?     | Kafka ordering                    |
| 39 | Kafka is unavailable when your service tries to publish an event. What should happen?                         | Failure handling                  |
| 40 | Your service processes a payment event twice and the customer gets charged twice. How would you prevent this? | Idempotency / distributed systems |



| #  | Scenario-based question                                                                                                    | Main concept tested |
| -- | -------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| 41 | One client sends thousands of requests per second and affects other users. How do you protect the system?                  | Rate limiting       |
| 42 | Gateway itself becomes a bottleneck. How would you troubleshoot and scale it?                                              | Gateway scalability |
| 43 | A service suddenly receives a huge amount of malicious traffic. What layers can protect it?                                | Traffic protection  |
| 44 | A service is repeatedly calling an unhealthy downstream service despite failures. What would you implement?                | Circuit breaker     |
| 45 | Your service has 10 downstream dependencies. One dependency is slow and causes thread exhaustion. How do you prevent this? | Bulkhead / timeout  |


| #  | Scenario-based question                                                                                                         | Main concept tested      |
| -- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 46 | A deployment was successful, but error rate increased immediately afterward. What is your investigation process?                | Production debugging     |
| 47 | One instance of a service is failing while other instances are healthy. What would you investigate?                             | Instance-level issue     |
| 48 | A service works correctly but suddenly starts returning stale data. What could cause this?                                      | Cache                    |
| 49 | Multiple instances update the same business object concurrently and overwrite each other's changes. How would you solve it?     | Distributed concurrency  |
| 50 | Everything looks healthy individually—CPU, memory, DB, Kafka—but the end-to-end API is slow. How would you find the bottleneck? | End-to-end observability |


Microservices Troubleshooting — Complete Scenario Bank
1. API Latency / Slow API
   An API normally takes 200 ms but suddenly takes 5 seconds. How would you investigate?
   API latency increased but CPU is normal. What would you check?
   P50 is normal but P99 increased significantly. What could be happening?
   API is slow only during peak traffic. How would you debug?
   API is slow for only one endpoint. What would you investigate?
   API latency increased after a new deployment. How would you identify the cause?
   API latency is high but there are no errors. What could cause this?
   API latency increased across all microservices. Where would you start?
   API A calls B and C. A is slow. How would you identify whether B or C is responsible?
   API suddenly becomes slow after traffic increases 10×. What would you check?
2. Traffic / Load Problems
   Traffic suddenly increases 10×. What happens to your system?
   One service receives much more traffic than other services. Why?
   CPU reaches 100% during a traffic spike. How would you handle it?
   Requests are increasing but throughput isn't increasing. Why?
   Traffic returns to normal but latency remains high. Why?
   A malicious client sends thousands of requests. How would you protect the service?
   A flash sale causes millions of requests. How would you handle the load?
   How would you identify whether increased traffic is legitimate or malicious?
   Load balancer distributes traffic unevenly. What would you investigate?
   One instance has high CPU while other instances are idle. Why?
3. API Gateway
   Gateway returns 502 for a downstream service. How do you debug?
   Gateway returns 504 Gateway Timeout. What could cause it?
   Gateway itself becomes slow. What would you check?
   Gateway CPU reaches 100%. What could cause it?
   One service is unreachable through Gateway but works directly. Why?
   Gateway is returning 429 responses. What does that indicate?
   Rate limiting suddenly blocks legitimate users. How would you investigate?
   Gateway configuration changed and multiple APIs stopped working. How would you debug?
   How would you prevent Gateway from becoming a single point of failure?
   Gateway receives a huge traffic spike. How would you protect downstream services?
4. Service-to-Service Communication
   Service A cannot communicate with Service B. How do you troubleshoot?
   Service B is intermittently unavailable. How would you investigate?
   Service A calls B successfully sometimes and times out sometimes. Why?
   Service A receives connection refused from B. What could cause it?
   Service A receives connection timeout from B. Difference between connection timeout and read timeout?
   Service B is healthy according to its health endpoint but requests still fail. Why?
   Service A is waiting indefinitely for Service B. What would you check?
   One downstream service is slow and causing your service to slow down. How do you isolate it?
   A dependency suddenly starts returning 500 errors. What should your service do?
   A downstream service is completely unavailable. How should your architecture behave?
5. Service Discovery
   Service A cannot discover Service B. What would you check?
   Service is registered in Eureka but requests still fail. Why?
   Eureka shows an instance as UP but it isn't responding. What could happen?
   Service instances keep registering and deregistering. Why?
   Service discovery has stale instances. How would you handle it?
   One instance receives traffic even though it is unhealthy. Why?
   Service discovery itself becomes unavailable. What happens?
   New instances aren't receiving traffic. What would you investigate?
6. Timeouts
   API requests are taking exactly 30 seconds before failing. What does that tell you?
   Downstream calls frequently hit read timeout. How would you debug?
   Connection timeout increased suddenly. What could cause it?
   Timeout is configured to 5 seconds but API takes 20 seconds. Why?
   Should you always increase timeout when requests are timing out?
   Multiple services have cascading timeouts. How do you stop the problem?
7. Retries
   A downstream service fails and your service retries the request. Traffic suddenly doubles. Why?
   Retry storm occurs in production. How would you stop it?
   Three services each retry a failed request three times. How many requests can reach the downstream service?
   Retries are causing high CPU. What would you investigate?
   When should you NOT retry?
   How does exponential backoff help?
   Why do we need jitter?
   Retry succeeds but duplicate transactions occur. How do you prevent them?
8. Circuit Breaker
   Circuit breaker suddenly opens. What would you investigate?
   Circuit breaker keeps opening and closing. Why?
   Circuit breaker is OPEN but downstream service is healthy. Why?
   Circuit breaker doesn't open even though downstream is failing. What could be wrong?
   What happens when the circuit breaker moves to HALF_OPEN?
   How does circuit breaker prevent cascading failure?
   Circuit breaker configuration is too aggressive. What problems can occur?
9. Rate Limiting
   API receives 100,000 requests per second. How would you protect it?
   Rate limiter starts rejecting legitimate users. How would you debug?
   How would you implement per-user rate limiting?
   How would you implement per-IP rate limiting?
   Rate limiter is distributed across multiple instances. How do you maintain consistency?
   Redis used for rate limiting becomes unavailable. What happens?
   How do you decide what rate limit to configure?
10. Thread Pool
    Thread pool active threads suddenly reach maximum. What could cause it?
    Thread pool queue keeps increasing. What does that indicate?
    CPU is low but thread pool is exhausted. Why?
    Thread pool exhaustion causes API latency. Explain the chain.
    How can slow DB queries exhaust application threads?
    How can slow downstream APIs exhaust application threads?
    Increasing thread-pool size doesn't solve the problem. Why?
    How would you troubleshoot thread-pool exhaustion?
11. JVM / GC / Memory
    JVM CPU suddenly reaches 100%. How would you investigate?
    Heap memory keeps increasing. What could cause it?
    Frequent GC causes API latency. Explain why.
    Full GC suddenly increases. What would you check?
    Application gets OutOfMemoryError. How would you investigate?
    Memory usage increases slowly over several hours. What does that suggest?
    CPU is high after a deployment. How would you determine whether GC is responsible?
    How would you use a heap dump?
    How would you use a thread dump?
    What could cause excessive object creation?
12. Database Connection Pool
    HikariCP connection pool is exhausted. How would you investigate?
    DB connection wait time suddenly increases. What could cause it?
    Connection pool has 100 connections but requests are still waiting. Why?
    Increasing DB pool size doesn't fix the problem. Why?
    Connections aren't being returned to the pool. What could cause it?
    Database itself is healthy but application DB pool is exhausted. Why?
    How can slow queries exhaust the connection pool?
    How can long transactions exhaust DB connections?
13. Slow Database Queries
    API latency increased because DB queries are slow. How do you investigate?
    Query was fast with 10,000 rows but slow with 10 million rows. Why?
    How would you use EXPLAIN?
    Query isn't using an index. What would you check?
    Adding an index didn't improve performance. Why?
    Database CPU is 100%. What would you investigate?
    Database I/O is very high. What could cause it?
    One query is consuming most DB resources. How would you identify it?
    Query suddenly became slow without code changes. What could cause it?
    Database statistics are stale. How can that affect performance?
14. N+1 Query
    One API request generates hundreds of SQL queries. How would you detect it?
    Why does N+1 happen with JPA/Hibernate?
    How would you fix N+1?
    API becomes slow after increasing the number of records. Could N+1 be responsible?
    How would you identify N+1 in production?
15. Database Locking / Concurrency
    API requests are waiting on database locks. How do you investigate?
    Two transactions update the same row. What happens?
    Deadlocks suddenly increase. What would you check?
    How would you resolve a database deadlock?
    Optimistic locking failure suddenly increases. Why?
    Pessimistic locking causes performance problems. Why?
    One transaction takes several seconds while others wait. What could be happening?
    How would you prevent unnecessary locking?
16. Redis / Caching
    Cache hit ratio suddenly drops. Why?
    Redis latency increases. How would you investigate?
    Redis goes down. What happens to your application?
    Database gets overloaded after Redis failure. Explain why.
    Cache contains stale data. How would you fix it?
    Cache stampede occurs. What happened?
    Cache expires and thousands of requests hit DB simultaneously. How do you prevent it?
    Redis memory reaches maximum. What would you check?
    Cache works in one instance but not another. Why?
    How would you decide what data should be cached?
17. Kafka Consumer
    Kafka consumer isn't receiving messages. What would you check?
    Consumer lag suddenly increases. How would you troubleshoot?
    Consumer is processing messages very slowly. Why?
    Consumer keeps restarting. What would you investigate?
    Consumer crashes when processing a particular message. What do you do?
    Consumer receives duplicate messages. Why?
    Messages are processed out of order. Why?
    One partition has huge lag while others don't. Why?
    Adding more consumers doesn't increase throughput. Why?
    Consumer group rebalancing happens frequently. Why?
    Consumer commits offset before processing completes. What can happen?
18. Kafka Producer
    Producer sends messages but consumers don't receive them. How investigate?
    Kafka producer latency increases. What would you check?
    Producer receives timeout errors. Why?
    Messages are being duplicated. What could cause it?
    Producer sends messages to the wrong partition. What would you investigate?
    Kafka broker becomes unavailable. What happens?
    Kafka topic partition becomes overloaded. How would you handle it?
19. Kafka Reliability
    How do you handle duplicate events?
    How do you make a Kafka consumer idempotent?
    Consumer processing fails after DB update but before offset commit. What happens?
    How do you prevent duplicate DB updates?
    Poison message keeps failing. What should happen?
    DLQ starts growing rapidly. How do you investigate?
    Kafka messages are delayed by several minutes. How would you debug?
    Consumer lag is zero but users aren't seeing updates. What else could be wrong?
20. Distributed Transactions
    Order creation succeeds but payment fails. What happens?
    Payment succeeds but order update fails. How do you handle it?
    How would you implement Saga?
    Choreography vs orchestration?
    A Saga compensation fails. What do you do?
    How do you maintain consistency across microservices?
    Why shouldn't we simply use one distributed transaction across all services?
21. Deployment / Production
    API became slow immediately after deployment. What would you do?
    Error rate increases after deployment. How investigate?
    Only the new instances have errors. Why?
    Old instances work but new instances don't. What would you check?
    Deployment causes DB connection exhaustion. Why?
    How would you perform rollback?
    How does blue-green deployment help?
    How does canary deployment help?
    New version causes only 5% of requests to fail. How investigate?
22. Logs / Monitoring / Observability
    CPU, memory and DB metrics look normal but users report slow APIs. What next?
    Error rate increases but application logs show nothing. Why?
    Logs are distributed across 10 services. How trace one request?
    What is a correlation ID?
    How would you identify which microservice caused latency?
    Metrics show high latency but logs don't explain it. What do you do?
    Traces show one downstream call taking 4 seconds. What next?
    Monitoring system itself stops receiving metrics. How investigate?
23. Cascading Failures
    One service goes down and suddenly five other services become unhealthy. Why?
    Downstream latency causes your thread pool to exhaust. Explain the chain.
    Retry + timeout + high traffic causes system-wide failure. Explain.
    Database becomes slow and eventually the entire application becomes unavailable. Explain.
    How do you prevent cascading failures?
24. Full Production Scenarios

These are the highest-value interview questions for you.

API latency increases from 200 ms to 5 seconds. Debug it end-to-end.
Traffic increases 10× and CPU reaches 100%. What do you do?
DB connection pool is exhausted and API latency is increasing. Diagnose it.
Thread pool is exhausted but CPU is only 30%. What could be happening?
Kafka consumer lag continuously increases. Find the root cause.
One downstream service is slow and your entire application is becoming slow. What do you do?
A downstream service starts returning 500 errors and retries cause a traffic explosion. How do you stabilize the system?
After deployment, P99 latency increases but P50 remains normal. Investigate.
Redis goes down and suddenly DB CPU reaches 100%. Explain and fix.
Production is experiencing high latency, high DB connections, high thread usage and increased GC. Walk me through your investigation from beginning to end.



60-Day Microservices Roadmap
Week 1 — Microservices Fundamentals
Day	Topic	      Troubleshooting question
1	API Gateway	     Gateway is returning 502/504 for one service. How do you debug?
2	Service Discovery	Service A cannot find Service B. What would you check?
3	Load Balancing	One instance is overloaded while others are healthy. Why?
4	Inter-service communication	Service A calls B, but B is intermittently slow. How investigate?
5	Timeouts	API calls are hanging for 30 seconds. What could cause it?
6	Retries	After a downstream failure, traffic suddenly doubles. Why?
7	Revision + scenario	API is slow, downstream is slow, and retries are happening. Diagnose it end-to-end.
Week 2 — Resilience
Day	Topic	Troubleshooting question
8	Circuit Breaker	Circuit breaker keeps opening. What could be happening?
9	Exponential Backoff + Jitter	1,000 requests retry at exactly the same time. What happens?
10	Rate Limiting	Traffic suddenly increases 10×. How protect the service?
11	Bulkhead	One dependency causes all application threads to get stuck. How prevent it?
12	Idempotency	Payment API receives the same request twice. How prevent duplicate payment?
13	Distributed failures	Service A is healthy but requests still fail. How investigate dependencies?
14	Revision + scenario	A downstream service is failing and your service is becoming unavailable. Design the protection.
Week 3 — Kafka
Day	Topic	Troubleshooting question
15	Kafka fundamentals	Consumer is not receiving messages. What do you check?
16	Topics & partitions	Why would you increase partitions?
17	Consumer groups	Two consumers are processing the same partition. What's wrong?
18	Offset management	Messages are being processed twice. Why?
19	Consumer lag	Kafka consumer lag keeps increasing. How investigate?
20	Ordering	Messages for the same customer are processed out of order. Why?
21	Revision + scenario	Consumer lag increases rapidly during production traffic. Diagnose it.
Week 4 — Kafka Reliability
Day	Topic	Troubleshooting question
22	At-least-once delivery	Consumer processes the same event twice. How handle it?
23	Idempotent consumer	How would you design a duplicate-safe consumer?
24	Retry topics	Consumer keeps failing on one message. What should happen?
25	Dead Letter Queue	What if DLQ itself starts growing?
26	Producer reliability	Producer sends messages but consumers don't see them. Debug.
27	Kafka transactions	When would you need Kafka transactions?
28	Revision + scenario	Order event is duplicated, delayed, and occasionally out of order. Design a solution.
Week 5 — Database in Microservices
Day	Topic	Troubleshooting question
29	DB connection pool	Connection pool is exhausted. What do you check?
30	Slow queries	API latency increased because DB calls are slow. Debug it.
31	Indexing	Query suddenly became slow after data grew 10×. What do you check?
32	N+1 queries	API makes hundreds of DB queries for one request. How identify/fix?
33	DB locks	Requests are waiting on DB locks. How investigate?
34	Deadlocks	Transactions are frequently deadlocking. What would you do?
35	Revision + scenario	API latency increased and DB CPU + connection pool usage are high. Find the root cause.
Week 6 — Redis & Caching
Day	Topic	Troubleshooting question
36	Why caching	DB is overloaded because of read traffic. How would caching help?
37	Cache-aside	Cache hit ratio suddenly drops. Why?
38	Cache invalidation	Users see stale data after updates. How fix?
39	Cache stampede	10,000 requests hit DB simultaneously after cache expiry. What happened?
40	Redis failure	Redis goes down. What happens to your service?
41	Redis performance	Redis latency suddenly increases. How investigate?
42	Revision + scenario	DB is overloaded because millions of requests miss the cache. Diagnose and fix.
Week 7 — Production Troubleshooting

This is very important for your interviews.

Day	Topic	Troubleshooting question
43	Metrics	API latency suddenly increases. Which metrics do you check first?
44	CPU	CPU reaches 95–100%. How investigate?
45	Memory	JVM memory keeps increasing. What could cause it?
46	GC	GC suddenly increases and API latency rises. Explain/debug.
47	Thread pools	Thread-pool active threads are maxed out. What could cause it?
48	DB pool + threads	Threads are waiting because DB connections are unavailable. Diagnose.
49	Logs	Metrics show errors but don't reveal the cause. How use logs?
50	Revision + scenario	API latency, CPU, thread pool and DB pool all increased. How do you systematically debug?
Week 8 — Observability
Day	Topic	Troubleshooting question
51	Actuator	What production information can Spring Boot Actuator give you?
52	Prometheus	How would you monitor API latency?
53	Grafana	Which dashboards would you create for a microservice?
54	P50/P95/P99	P99 latency increased but P50 is normal. What does that tell you?
55	Distributed tracing	Request crosses 5 services and becomes slow. Find the slow service.
56	OpenTelemetry	How does tracing work across microservices?
57	Correlation ID	One user request generates 10 service calls. How trace it?
58	Production debugging flow	Logs + metrics + traces disagree. How investigate?
59	Full scenario	API latency suddenly jumps from 200 ms → 5 seconds. Debug end-to-end.
60	Mock interview	I'll interview you with 10 microservice production scenarios.



| #      | Module                                   | What to learn                                                                                                                                                                                                    | Depth / Target    | Scenario questions you must handle                                                                                                  |
| ------ | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **1**  | **Microservice Fundamentals**            | Monolith vs microservices, advantages/disadvantages, service boundaries, bounded context, loose coupling, high cohesion, database-per-service, independent deployment, independent scaling, eventual consistency | **Strong**        | How do you split a monolith? How do you decide service boundaries? Why database per service? When should you NOT use microservices? |
| **2**  | **Microservice Communication**           | Synchronous vs asynchronous, REST communication, request-response, event-driven communication, coupling, latency, failure propagation                                                                            | **Strong**        | REST or Kafka? What if downstream service is slow? What if it is unavailable?                                                       |
| **3**  | **REST/API Design**                      | HTTP methods, status codes, idempotency, API versioning, pagination, filtering, timeout, correlation ID, backward compatibility                                                                                  | **Strong**        | Client retries payment request—how prevent duplicate payment? API changed but old clients still exist—what do you do?               |
| **4**  | **API Gateway**                          | Routing, load balancing, authentication, authorization, rate limiting, filtering, SSL termination, CORS, correlation ID                                                                                          | **Medium–Strong** | Gateway is down? Gateway is bottleneck? Where should authentication/authorization happen?                                           |
| **5**  | **Service Discovery**                    | Service registry, registration, discovery, heartbeat, health checks, instance failure, client/server-side discovery                                                                                              | **Medium**        | Service instance dies—how is traffic redirected? How does Order find Payment?                                                       |
| **6**  | **Load Balancing**                       | Client-side vs server-side, round robin, health-aware routing, scaling instances                                                                                                                                 | **Medium**        | One instance is unhealthy but still receives traffic—what do you investigate?                                                       |
| **7**  | **Database per Microservice**            | Data ownership, isolated DB, shared DB problems, cross-service data access, data duplication, eventual consistency                                                                                               | **Strong**        | Order needs Customer data—API call or local copy? Can two services share a DB?                                                      |
| **8**  | **SQL for Microservices**                | JOIN, GROUP BY, HAVING, subqueries, CTE, window functions, aggregate functions                                                                                                                                   | **Strong**        | Query is slow—how do you investigate?                                                                                               |
| **9**  | **Database Indexing**                    | B-tree, index purpose, composite index, cardinality, selectivity, covering index concept, EXPLAIN, execution plans                                                                                               | **Strong**        | Query suddenly became slow. Index exists but isn't used. What do you check?                                                         |
| **10** | **Transactions**                         | ACID, commit, rollback, transaction boundaries, atomicity, consistency                                                                                                                                           | **Very Strong**   | Part of an operation succeeds and another fails—what happens?                                                                       |
| **11** | **Isolation Levels**                     | Read Uncommitted, Read Committed, Repeatable Read, Serializable                                                                                                                                                  | **Very Strong**   | Dirty/non-repeatable/phantom reads. Which isolation level would you choose and why?                                                 |
| **12** | **DB Locks**                             | Shared/exclusive locks, row/table locks, lock contention, lock duration                                                                                                                                          | **Very Strong**   | Two transactions update same row. Why is one waiting?                                                                               |
| **13** | **MVCC** ⭐                               | Multi-version concurrency control, row versions, snapshots/read views, readers vs writers, relationship with isolation                                                                                           | **Very Strong**   | How does DB actually handle concurrent reads/writes? How can one transaction see an older version?                                  |
| **14** | **Deadlocks**                            | Deadlock conditions, lock ordering, detection, rollback, prevention                                                                                                                                              | **Strong**        | Two transactions are waiting forever. How do you identify and fix it?                                                               |
| **15** | **Optimistic/Pessimistic Locking**       | Optimistic locking, versioning, pessimistic locking, `SELECT FOR UPDATE` concept, trade-offs                                                                                                                     | **Very Strong**   | Two users buy last item. Which locking strategy would you choose?                                                                   |
| **16** | **Kafka Fundamentals** ⭐                 | Broker, topic, partition, producer, consumer, consumer group, offset                                                                                                                                             | **Very Strong**   | Explain complete Kafka flow. What happens when producer sends a message?                                                            |
| **17** | **Kafka Partitioning**                   | Partition key, partition selection, ordering, parallelism, hot partition                                                                                                                                         | **Very Strong**   | How guarantee ordering for an order? Why not put everything in one partition?                                                       |
| **18** | **Kafka Consumer Groups**                | Consumer assignment, partition ownership, scaling consumers, consumer failure, rebalancing                                                                                                                       | **Very Strong**   | 6 partitions + 3 consumers? 6 partitions + 10 consumers? Consumer dies?                                                             |
| **19** | **Kafka Offsets**                        | Offset, commit, auto/manual commit, commit failure, replay                                                                                                                                                       | **Very Strong**   | Consumer processed message but crashed before committing—what happens?                                                              |
| **20** | **Kafka Delivery Semantics**             | At-most-once, at-least-once, exactly-once, duplicate processing                                                                                                                                                  | **Very Strong**   | Why can duplicate messages occur? How do you handle them?                                                                           |
| **21** | **Kafka Idempotency** ⭐                  | Idempotent consumer, deduplication, unique constraints, processed-event tracking, idempotent business operations                                                                                                 | **Very Strong**   | Same payment/order event arrives twice. What do you do?                                                                             |
| **22** | **Kafka Failure Handling**               | Retry, retry topics, DLT, poison messages, consumer failure, producer failure, broker failure                                                                                                                    | **Very Strong**   | Consumer repeatedly fails on one message. What happens? What do you do?                                                             |
| **23** | **Kafka Reliability**                    | Replication, leader/follower, ISR, acknowledgements, producer retries, idempotent producer                                                                                                                       | **Strong**        | Broker dies. Is data lost? What determines durability?                                                                              |
| **24** | **Kafka Consumer Lag**                   | What lag means, causes, monitoring, slow consumers, partitions, scaling                                                                                                                                          | **Very Strong**   | Consumer lag keeps increasing. How do you troubleshoot it?                                                                          |
| **25** | **Distributed Transactions** ⭐           | Distributed transaction problem, 2PC concept, why normal DB transaction doesn't span services                                                                                                                    | **Strong**        | Order succeeds, payment succeeds, inventory fails. How do you recover?                                                              |
| **26** | **Saga Pattern** ⭐                       | Saga, choreography, orchestration, compensating transactions, eventual consistency                                                                                                                               | **Very Strong**   | Payment succeeded but order failed. How do you compensate?                                                                          |
| **27** | **Outbox Pattern** ⭐                     | Dual-write problem, transactional outbox, outbox table, event publisher, retries, CDC concept                                                                                                                    | **Very Strong**   | DB succeeds but Kafka fails. How do you guarantee the event isn't lost?                                                             |
| **28** | **Eventual Consistency**                 | Strong vs eventual consistency, stale data, convergence, trade-offs                                                                                                                                              | **Strong**        | Inventory is temporarily stale. Is that acceptable? How do you design around it?                                                    |
| **29** | **Timeouts**                             | Connection timeout, read timeout, request timeout, why timeout is necessary                                                                                                                                      | **Very Strong**   | Downstream takes 2 minutes. Should your service wait?                                                                               |
| **30** | **Retry**                                | Retryable/non-retryable failures, max attempts, exponential backoff, jitter, retry storms                                                                                                                        | **Very Strong**   | Should every 500 response be retried? Can retries make an outage worse?                                                             |
| **31** | **Circuit Breaker** ⭐                    | Closed/open/half-open, failure threshold, recovery testing, fallback                                                                                                                                             | **Very Strong**   | Payment is down. How prevent cascading failure?                                                                                     |
| **32** | **Bulkhead**                             | Resource isolation, limiting concurrent calls, preventing dependency from consuming all resources                                                                                                                | **Strong**        | One slow dependency consumes all threads. What do you do?                                                                           |
| **33** | **Rate Limiting**                        | Request limits, client limits, throttling, protection from traffic spikes                                                                                                                                        | **Strong**        | One client sends 50K requests/minute. What do you do?                                                                               |
| **34** | **Backpressure**                         | Producer faster than consumer, queue growth, consumer capacity, throttling                                                                                                                                       | **Strong**        | Kafka consumer can't keep up with producers. What happens?                                                                          |
| **35** | **Resilience Strategy**                  | Combining timeout + retry + circuit breaker + bulkhead + fallback                                                                                                                                                | **Very Strong**   | Design failure handling for Payment Service. Which mechanisms do you combine and why?                                               |
| **36** | **Redis/Caching**                        | Why caching, cache-aside, TTL, invalidation, stale data, cache hit/miss                                                                                                                                          | **Strong**        | DB is receiving millions of identical reads. What would you do?                                                                     |
| **37** | **Cache Problems**                       | Cache stampede, penetration, avalanche, inconsistency, eviction                                                                                                                                                  | **Strong**        | Thousands of requests miss cache simultaneously. What happens?                                                                      |
| **38** | **Distributed Locking**                  | Why distributed lock, Redis lock concept, expiry, failure scenarios                                                                                                                                              | **Medium**        | Only one service instance should execute a scheduled operation. How?                                                                |
| **39** | **Observability** ⭐                      | Logs, metrics, traces, correlation ID, trace ID, centralized logging                                                                                                                                             | **Very Strong**   | API suddenly becomes slow. How do you identify where the problem is?                                                                |
| **40** | **Metrics**                              | CPU, memory, latency, throughput, error rate, DB connections, Kafka lag, cache hit rate                                                                                                                          | **Very Strong**   | Which metrics do you check when a service is slow?                                                                                  |
| **41** | **Distributed Tracing**                  | Trace, span, trace ID, cross-service request flow                                                                                                                                                                | **Strong**        | Request crosses 5 services—how do you identify which service caused latency?                                                        |
| **42** | **Production Troubleshooting** ⭐⭐⭐       | Incident identification, logs, metrics, traces, dependency isolation, mitigation, RCA, prevention                                                                                                                | **Very Strong**   | API slow, DB slow, Kafka lag, 503, CPU 100%, memory high, service repeatedly restarting                                             |
| **43** | **Microservice Security**                | Service-to-service auth, JWT, OAuth2 concepts, TLS, secrets, authorization, zero-trust concept                                                                                                                   | **Medium–Strong** | How does Service A securely communicate with Service B?                                                                             |
| **44** | **Scaling**                              | Horizontal/vertical scaling, stateless services, bottlenecks, DB scaling, Kafka scaling, cache                                                                                                                   | **Strong**        | Traffic increases 10x. What do you scale?                                                                                           |
| **45** | **Docker**                               | Image, container, Dockerfile, network, environment variables, Compose                                                                                                                                            | **Medium**        | How would you run multiple microservices locally?                                                                                   |
| **46** | **Kubernetes**                           | Pod, Deployment, Service, ConfigMap, Secret, replicas, HPA, readiness/liveness                                                                                                                                   | **Medium**        | Pod dies. Traffic increases. New deployment is failing.                                                                             |
| **47** | **Deployment Strategies**                | Rolling, blue-green, canary, zero downtime, backward compatibility                                                                                                                                               | **Medium**        | How do you deploy a new version without downtime?                                                                                   |
| **48** | **Microservice Patterns**                | API Gateway, Saga, Outbox, Circuit Breaker, Sidecar, CQRS, Strangler, event-driven architecture                                                                                                                  | **Strong**        | Which pattern would you use for a given problem and why?                                                                            |
| **49** | **System Design** ⭐                      | Service decomposition, DB, Kafka, caching, resilience, scaling, consistency, observability                                                                                                                       | **Very Strong**   | Design e-commerce, payment, notification, food delivery systems                                                                     |
| **50** | **Scenario-Based Interview Mastery** ⭐⭐⭐ | Problem breakdown, investigation, mitigation, root cause, permanent fix, trade-offs                                                                                                                              | **Very Strong**   | Completely unfamiliar production scenario                                                                                           |


1. API Latency / Performance
   API normally takes 200 ms but suddenly takes 5 seconds. How do you investigate?
   CPU is normal but API latency increased. What do you check?
   P50 is normal but P99 increased significantly. What could be happening?
   API is slow only during peak traffic. How do you debug?
   Only one endpoint is slow. What do you investigate?
   Latency increased after deployment. How do you identify the cause?
   API latency is high but there are no errors. Why?
   Latency increased across all microservices. Where do you start?
   Service A calls B and C. A is slow. How do you identify whether B or C is responsible?
   Traffic increases 10× and latency increases. How do you investigate?
   CPU, memory and DB look normal but API is slow. What next?
   P99 suddenly spikes for only a few requests. What could cause it?
   API latency increases gradually over several hours. What could cause it?
   Latency is high only for certain customers. What could be different?
   Read APIs are fast but write APIs are slow. Why?
   API is slow only for large payloads. What would you check?
   API became slow without any deployment. What possibilities do you investigate?
   How do you distinguish application latency from downstream latency?
   How do you identify whether DB, Redis or another microservice is responsible?
   What metrics would you check first when investigating API latency?
2. Traffic / Load
   Traffic suddenly increases 10×. What happens?
   CPU reaches 100% during traffic spike. What do you do?
   Requests increase but throughput doesn't increase. Why?
   Traffic returns to normal but latency remains high. Why?
   One service receives much more traffic than others. Why?
   One instance has high CPU while others are idle. Why?
   Load balancer distributes traffic unevenly. How do you investigate?
   How do you identify legitimate traffic vs malicious traffic?
   Flash sale generates millions of requests. How do you handle it?
   What happens when traffic exceeds application capacity?
   How do you calculate/identify service capacity?
   How does horizontal scaling help?
   When would vertical scaling help?
   Why can scaling application instances fail to solve a DB bottleneck?
   What happens if all instances scale simultaneously?
   How would you protect a service from sudden traffic spikes?
3. API Gateway
   Gateway returns 502 for downstream service. Debug it.
   Gateway returns 504. What could cause it?
   Gateway itself becomes slow. What do you check?
   Gateway CPU reaches 100%. Why?
   Service works directly but fails through Gateway. Why?
   Gateway returns 429. What does it indicate?
   Rate limiting blocks legitimate users. How do you investigate?
   Gateway configuration changed and multiple APIs stopped working. What do you check?
   How do you prevent Gateway from becoming a single point of failure?
   Gateway receives huge traffic spike. How do you protect downstream services?
   Where should authentication happen—Gateway or individual services?
   Where should rate limiting happen?
   Gateway timeout vs downstream timeout—what is the difference?
   Gateway is healthy but downstream calls are failing. How do you isolate the problem?
4. Service-to-Service Communication
   Service A cannot communicate with Service B. Troubleshoot.
   Service B is intermittently unavailable. Why?
   A → B works sometimes and times out sometimes. Why?
   Connection refused from B. What does it indicate?
   Connection timeout vs read timeout?
   B health endpoint is UP but requests fail. Why?
   A waits indefinitely for B. What do you check?
   B is slow and causes A to become slow. How do you isolate it?
   B starts returning 500. What should A do?
   B is completely unavailable. How should architecture behave?
   DNS resolution fails between services. How do you troubleshoot?
   Connection reset occurs. What could cause it?
   TLS handshake fails. How do you investigate?
   How do you decide timeout values between services?
   How do you prevent one slow downstream service from consuming all resources?
5. Service Discovery / Eureka
   A cannot discover B. What do you check?
   B is registered in Eureka but requests fail. Why?
   Eureka says instance is UP but it doesn't respond. Why?
   Instances repeatedly register/deregister. Why?
   Eureka contains stale instances. How do you handle them?
   Unhealthy instance continues receiving traffic. Why?
   Eureka itself becomes unavailable. What happens?
   New instances aren't receiving traffic. Why?
   Multiple instances have same service ID. What happens?
   How does service discovery work internally?
   Client-side vs server-side service discovery?
   What happens during Eureka network partition?
6. Timeouts
   Requests fail exactly after 30 seconds. What does that suggest?
   Downstream calls frequently hit read timeout. Investigate.
   Connection timeout suddenly increases. Why?
   Timeout configured as 5 seconds but API takes 20 seconds. Why?
   Should you always increase timeout?
   Multiple services have cascading timeouts. How do you stop it?
   What is connection timeout?
   What is read timeout?
   What is socket timeout?
   How should timeout values be chosen across A → B → C?
   What happens if upstream timeout is shorter than downstream timeout?
   What happens if downstream timeout is longer than upstream timeout?
7. Retries
   Downstream fails and service retries. Traffic doubles. Why?
   What is a retry storm?
   How do you stop a retry storm?
   Three services each retry 3 times. How many requests can reach downstream?
   Retries cause high CPU. What do you investigate?
   When should you NOT retry?
   How does exponential backoff work?
   Why do we need jitter?
   Retry succeeds but duplicate transactions occur. How do you prevent this?
   Should POST requests always be retried?
   Should timeout errors be retried?
   Should 500 errors be retried?
   Should 400 errors be retried?
   Where should retries be implemented—Gateway, service or client?
   What happens when multiple layers perform retries?
   How do you put a maximum limit on retries?
8. Idempotency — Important Missing Area

This is one area I would specifically add to your original list.

What is idempotency?

Why is idempotency important in microservices?
What happens if a payment request is sent twice?
How do you make a POST API idempotent?
What is an idempotency key?
Where do you store idempotency keys?
How long should an idempotency key be retained?
Client times out but server successfully processes the request. Client retries. How do you prevent duplicate processing?
Payment succeeds but response is lost. Client retries. What happens?
How do you implement idempotency using a database?
How do you implement idempotency using Redis?
What happens if two identical requests arrive simultaneously?
How do you make the idempotency check itself thread-safe?
How does a unique DB constraint help with idempotency?
Difference between idempotency and duplicate detection?
Difference between idempotency and exactly-once processing?
How do you make Kafka consumer processing idempotent?
Kafka message is delivered twice. How do you prevent duplicate DB updates?
Consumer updates DB and crashes before committing Kafka offset. What happens?
Consumer commits offset and then DB update fails. What happens?
How do you design an idempotent payment/order API?
How would you handle duplicate events from different producers?
Can GET, PUT and DELETE be idempotent?
Is POST inherently non-idempotent?
How does an idempotency key work across multiple application instances?

This section is extremely important for your interviews.

9. Circuit Breaker
   Circuit breaker suddenly opens. Why?
   Circuit breaker repeatedly opens/closes. Why?
   Circuit breaker is OPEN but downstream is healthy. Why?
   Circuit breaker doesn't open despite downstream failures. Why?
   What happens in HALF_OPEN?
   How does circuit breaker prevent cascading failure?
   What happens when circuit breaker is too aggressive?
   What happens when circuit breaker is too lenient?
   What is failure threshold?
   What is slow-call threshold?
   What should happen when circuit is OPEN?
   What fallback should you provide?
   Can fallback itself cause problems?
10. Rate Limiting
    API receives 100,000 requests/sec. Protect it.
    Rate limiter rejects legitimate users. Debug.
    Implement per-user rate limiting.
    Implement per-IP rate limiting.
    Distributed rate limiting across instances—how?
    Redis goes down while rate limiting. What happens?
    How do you choose rate-limit values?
    Token bucket vs leaky bucket?
    Fixed window vs sliding window?
    Where should rate limiting happen?
    How does Gateway rate limiting protect downstream services?
    How can attackers bypass IP-based rate limiting?
11. Thread Pool
    Active threads reach maximum. Why?
    Thread-pool queue keeps increasing. What does it mean?
    CPU is only 30% but thread pool is exhausted. Why?
    Explain thread-pool exhaustion → API latency.
    How can slow DB queries exhaust application threads?
    How can slow downstream APIs exhaust threads?
    Increasing thread-pool size doesn't fix the problem. Why?
    How do you troubleshoot thread-pool exhaustion?
    What happens if thread pool is too small?
    What happens if thread pool is too large?
    What happens when the executor queue becomes full?
    How does rejection policy affect the application?
    How can GC cause thread-pool exhaustion?
    How can connection-pool exhaustion cause thread-pool exhaustion?
12. JVM / GC / Memory
    JVM CPU reaches 100%. Investigate.
    Heap keeps increasing. Why?
    Frequent GC causes latency. Explain.
    Full GC suddenly increases. Why?
    OutOfMemoryError occurs. Investigate.
    Memory increases slowly for hours. What does it suggest?
    CPU increases after deployment. Determine whether GC is responsible.
    How do you use a heap dump?
    How do you use a thread dump?
    What causes excessive object creation?
    How can a memory leak affect API latency?
    Young GC vs Full GC?
    How does GC stop-the-world affect APIs?
    What metrics indicate GC pressure?
13. Database Connection Pool
    HikariCP pool exhausted. Investigate.
    DB connection wait time increases. Why?
    Pool has 100 connections but requests still wait. Why?
    Increasing pool size doesn't solve it. Why?
    Connections aren't returned. What could cause it?
    DB is healthy but application pool is exhausted. Why?
    Slow queries exhaust connections. Explain.
    Long transactions exhaust connections. Explain.
    How do you decide HikariCP pool size?
    What happens if every application instance has 100 DB connections?
    Why can increasing pool size actually make DB performance worse?
14. Database Performance
    API latency increases because DB queries are slow. Investigate.
    Query fast with 10K rows but slow with 10M. Why?
    How do you use EXPLAIN?
    Query isn't using index. Why?
    Index added but performance doesn't improve. Why?
    DB CPU reaches 100%. Investigate.
    DB I/O is high. Why?
    One query consumes most DB resources. How identify it?
    Query becomes slow without code changes. Why?
    Statistics become stale. What happens?
    How does an index improve performance?
    When can an index hurt performance?
    Composite index—how does column order matter?
    Full table scan occurs. Why?
    DB connection count suddenly increases. Why?
    Database is overloaded. What should application do?
15. N+1 Queries
    One API request generates hundreds of SQL queries. Detect it.
    Why does N+1 happen with JPA/Hibernate?
    How do you fix N+1?
    N+1 appears only after data volume increases. Why?
    How do you identify N+1 in production?
    JOIN FETCH vs EntityGraph?
    Lazy vs eager loading?
    Can changing everything to EAGER solve N+1?
16. DB Locking / Concurrency
    Requests wait on DB locks. Investigate.
    Two transactions update same row. What happens?
    Deadlocks suddenly increase. Why?
    How do you resolve deadlocks?
    Optimistic locking failures increase. Why?
    Pessimistic locking causes performance problems. Why?
    One transaction takes several seconds and others wait. Why?
    How do you prevent unnecessary locking?
    Optimistic vs pessimistic locking?
    Lost update problem?
    Dirty read?
    Non-repeatable read?
    Phantom read?
    How does transaction isolation affect performance?
    How do you handle concurrent payment/order updates?
17. Redis / Caching
    Cache hit ratio suddenly drops. Why?
    Redis latency increases. Investigate.
    Redis goes down. What happens?
    DB CPU reaches 100% after Redis failure. Explain.
    Cache contains stale data. Fix it.
    What is cache stampede?
    Thousands of requests hit DB after cache expiry. Prevent it.
    Redis memory reaches maximum. Investigate.
    Cache works on one instance but not another. Why?
    How do you decide what to cache?
    Cache-aside pattern?
    Write-through vs write-behind?
    Cache invalidation strategies?
    What happens if cached data and DB data become inconsistent?
    How do you prevent cache penetration?
    Cache avalanche vs cache stampede?
18. Kafka Consumer
    Consumer isn't receiving messages. Investigate.
    Consumer lag suddenly increases. Why?
    Consumer processing is slow. Why?
    Consumer keeps restarting. Why?
    Consumer crashes on a particular message. What do you do?
    Consumer receives duplicate messages. Why?
    Messages are out of order. Why?
    One partition has huge lag. Why?
    Adding consumers doesn't increase throughput. Why?
    Consumer group rebalances frequently. Why?
    Offset committed before processing completes. What happens?
    Offset committed after processing. What happens if consumer crashes?
    Consumer is too slow because DB is slow. Explain.
    How do you increase consumer throughput?
    How does partition count affect consumer parallelism?
    Can two consumers in the same group consume the same partition?
19. Kafka Producer
    Producer sends messages but consumers don't receive them. Investigate.
    Producer latency increases. Why?
    Producer timeout occurs. Why?
    Messages are duplicated. Why?
    Producer sends to wrong partition. Investigate.
    Broker becomes unavailable. What happens?
    One partition becomes overloaded. Why?
    How does partitioning work?
    How does key-based partitioning work?
    What happens if the message key changes?
    What are producer acknowledgements?
    acks=0, acks=1, acks=all?
    What is min.insync.replicas?
    What happens when ISR falls below minimum?
    What is replication factor?
    What happens if a broker fails?
20. Kafka Reliability / Exactly Once / Idempotency
    How do you handle duplicate events?
    How do you make consumer idempotent?
    Consumer fails after DB update but before offset commit. What happens?
    How do you prevent duplicate DB updates?
    Poison message keeps failing. What should happen?
    DLQ grows rapidly. Investigate.
    Kafka messages delayed by minutes. Debug.
    Consumer lag is zero but users don't see updates. What else could be wrong?
    At-most-once vs at-least-once?
    What is exactly-once semantics?
    Can you achieve exactly-once across Kafka + MySQL?
    Kafka transaction vs database transaction?
    How does the Outbox Pattern solve this problem?
    What is Inbox Pattern?
    Outbox vs idempotent consumer?
    What happens if DB succeeds but Kafka publishing fails?
    What happens if Kafka publishing succeeds but DB transaction rolls back?
    How do you guarantee an event isn't lost?
    How do you guarantee an event isn't processed twice?
21. Distributed Transactions / Saga
    Order succeeds but payment fails. What happens?
    Payment succeeds but order update fails. How handle it?
    How would you implement Saga?
    Choreography vs orchestration?
    Saga compensation fails. What do you do?
    How maintain consistency across services?
    Why not use one distributed transaction across all services?
    What is eventual consistency?
    How do you handle partial failure?
    How do you retry compensation?
    What if compensation itself is not possible?
    How do you make Saga steps idempotent?
    How do you track Saga state?
    How do you recover a failed Saga?
    What happens if orchestrator goes down?
    What happens if an event is delivered twice during Saga?
    Saga vs 2PC?
22. Deployment / Production
    API becomes slow immediately after deployment. Investigate.
    Error rate increases after deployment.
    Only new instances have errors. Why?
    Old instances work but new instances fail. What check?
    Deployment causes DB connection exhaustion. Why?
    How do you perform rollback?
    How does blue-green deployment help?
    How does canary deployment help?
    New version causes 5% failures. Investigate.
    How do you compare old vs new instances?
    What if database schema changed during deployment?
    How do you perform zero-downtime deployment?
    What happens if deployment succeeds but configuration is wrong?
    What is backward compatibility in microservices?
23. Observability / Monitoring
    CPU, memory and DB metrics look normal but API is slow. What next?
    Error rate increases but logs show nothing. Why?
    Logs distributed across 10 services. Trace one request.
    What is correlation ID?
    How identify which microservice caused latency?
    Metrics show high latency but logs don't explain it. What next?
    Trace shows downstream call taking 4 seconds. What next?
    Monitoring system stops receiving metrics. Investigate.
    Metrics vs logs vs traces?
    What should you monitor for every microservice?
    What are RED metrics?
    What are USE metrics?
    P50 vs P95 vs P99?
    Why can average latency be misleading?
    How would you create an alert for API degradation?
    How do you correlate traffic, latency, errors and resource metrics?
24. Cascading Failures
    One service goes down and five other services become unhealthy. Why?
    Downstream latency exhausts thread pool. Explain the chain.
    Retry + timeout + high traffic causes system failure. Explain.
    DB becomes slow and entire application becomes unavailable. Explain.
    How do you prevent cascading failures?
    How do timeout + retry + circuit breaker work together?
    What happens when every service retries?
    How can connection pools contribute to cascading failure?
    How can thread pools contribute to cascading failure?
    How can cache failure cause cascading failure?
    How can Kafka lag create cascading problems?
    What is bulkhead isolation?
    How would you isolate failures between different downstream dependencies?
25. Important Missing Microservice Design Questions

These are also worth adding to your preparation.

Why microservices instead of monolith?
When should you NOT use microservices?
How do you identify service boundaries?
How do services communicate?
REST vs messaging?
Synchronous vs asynchronous communication?
How do you maintain data consistency?
How do you handle service versioning?
How do you handle backward compatibility?
How do you handle schema changes?
How do you handle authentication between services?
How do you handle authorization?
JWT between microservices—how?
What is mTLS?
How do you secure internal service communication?
How do you prevent duplicate requests?
How do you handle distributed logging?
How do you trace distributed requests?
How do you handle configuration across services?
Centralized configuration vs local configuration?
How do you handle secrets?
How do you handle service failure?
How do you design for high availability?
How do you design for fault tolerance?
How do you scale a microservice?
How do you handle database-per-service?
Why shouldn't multiple services directly share the same DB tables?
How do you migrate from monolith to microservices?
What is eventual consistency?
What is bounded context?
26. Highest-Value End-to-End Interview Scenarios

These are the ones I would make you practice speaking through, rather than just memorizing answers:

API goes from 200 ms → 5 seconds. Debug end-to-end.
P50 normal but P99 suddenly increases.
Traffic increases 10× and CPU reaches 100%.
Thread pool exhausted but CPU is only 30%.
DB connection pool exhausted.
DB is healthy but HikariCP is exhausted.
Slow DB query causes entire API to become slow.
One downstream service becomes slow and brings down your service.
Downstream returns 500 and retries create a retry storm.
Gateway starts returning 502.
Gateway starts returning 504.
Redis goes down and DB CPU reaches 100%.
Kafka consumer lag continuously increases.
Kafka consumer processes duplicate messages.
Kafka consumer updates DB but crashes before committing offset.
Payment succeeds but order update fails.
Order succeeds but payment fails.
Two users simultaneously update the same order.
Deployment increases P99 but P50 remains normal.
Only 5% of requests fail after deployment.
One instance has high CPU while all others are normal.
All microservices suddenly become slow.
CPU, memory, DB and Redis all look normal but users report slowness.
DB becomes slow and eventually the entire system becomes unavailable.
Production has high latency + high DB connections + high thread usage + increased GC. Walk through the entire investigation.
What I recommend for your preparatio





Top 20 Most Important Scenario-Based Microservices Interview Questions
1. A microservice is suddenly down in production. How would you troubleshoot it?
2. One API is responding very slowly. How would you find the root cause?
3. Service A calls Service B, but the request is timing out. What could be causing it?
4. Your application is receiving a sudden spike in traffic and starts failing. What would you do?
5. One downstream service becomes slow. How can it affect other microservices?
6. Service A is calling Service B, but Service B is unavailable. How should Service A handle the failure?
7. Your application works normally, but after deploying a new version, errors suddenly increase. How would you investigate?
8. Users are receiving HTTP 500 errors from your API. How would you debug the issue?
9. Your database becomes slow and all APIs depending on it are affected. What would you check?
10. Your application runs out of database connections. What could cause connection pool exhaustion?
11. A request is processed twice because the client retried it. How would you prevent duplicate processing?

Tests: Idempotency.

12. A message is delivered twice through Kafka. How do you prevent duplicate business operations?

Tests: Idempotent consumer design.

13. Kafka consumer lag is continuously increasing. How would you troubleshoot it?
14. A consumer crashes after processing a Kafka message but before committing the offset. What happens?
15. An order is created successfully, but payment fails. How would you maintain consistency between services?

Tests: Saga pattern / distributed transactions.

16. Two users try to update the same record simultaneously. How would you handle concurrency?

Tests: Optimistic locking, pessimistic locking, transactions.

17. Your API Gateway becomes a bottleneck. How would you identify and solve the problem?
18. One microservice has high CPU and memory usage. How would you investigate?
19. How would you debug an issue that happens only intermittently in production?
20. A production incident occurs and users report that the application is not working. Explain your complete investigation approach.

A strong answer should generally follow:

Understand the impact → Check monitoring/alerts → Identify affected services → Check logs → Check metrics → Trace the request → Identify the dependency/root cause → Fix → Verify → Prevent recurrence

⭐ If you can prepare only 10 first

Start with these:

Service down
Slow API
Timeout between services
Traffic spike
Downstream service failure
HTTP 500 investigation
Database performance issue
Idempotency / duplicate requests
Kafka duplicate messages and consumer lag
Production incident troubleshooting


---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


1. Service-to-Service Communication
   Service A cannot communicate with Service B. How do you troubleshoot?
   Service B is intermittently unavailable. What could cause it?
   Service A → Service B works sometimes but times out sometimes. Why?
   You get Connection Refused when calling Service B. What does it indicate?
   What is the difference between Connection Timeout vs Read Timeout?
   Service B health endpoint is UP, but actual requests fail. Why?
   Service A waits indefinitely for Service B. What do you check?
   Service B is slow, causing Service A to become slow. How do you isolate the problem?
   Service B starts returning HTTP 500 errors. What should Service A do?
   Only one instance of Service B is failing. How do you identify it?
2. API Latency & Performance
   API latency suddenly increases across all microservices. Where do you start?
   One API is slow, but other APIs are fast. How do you investigate?
   P95 latency is high, but average latency looks normal. What does that indicate?
   Latency increases only during peak traffic. Why?
   API latency gradually increases over several hours. What could cause it?
   CPU usage is high and API response time increases. How do you investigate?
   Memory usage continuously increases. What do you check?
   Thread pool is exhausted. What happens and how do you troubleshoot?
   Connection pool is exhausted. What happens?
   A downstream API is slow. How does it affect your service?
3. Database Problems
   Database becomes slow and all dependent APIs are affected. How do you troubleshoot?
   One database query suddenly becomes slow. What do you check?
   Database CPU is high. What could cause it?
   Database connection pool is exhausted. How do you identify and fix it?
   Your application has too many database connections. Why?
   API is slow because of an N+1 query problem. How do you identify it?
   A database query is blocked. What could cause it?
   Deadlocks are occurring. How do you troubleshoot?
   Database latency is intermittent. What could cause it?
   A new deployment causes database performance degradation. What do you check?
4. Failures, Resilience & Cascading Failures
   Service B goes down. What happens to Service A?
   Service B becomes extremely slow. How do you prevent cascading failures?
   Retry logic makes the system worse during an outage. Why?
   A retry storm happens. How do you handle it?
   When should you use exponential backoff?
   Why should retries use jitter?
   Circuit breaker opens frequently. What does it indicate?
   Circuit breaker is open, but Service B has recovered. What happens?
   Bulkhead pattern: when and why would you use it?
   One failing downstream service causes your entire application to fail. How do you prevent this?
5. Kafka & Asynchronous Communication
   Kafka producer sends a message, but the consumer doesn't receive it. How do you troubleshoot?
   Producer retries and sends duplicate messages. How do you handle duplicates?
   Consumer processes the same message twice. Why?
   Consumer crashes after processing a message but before committing the offset. What happens?
   Consumer lag continuously increases. How do you troubleshoot?
   One consumer is slow. How does it affect the consumer group?
   A Kafka broker goes down. What happens?
   Messages are processed out of order. Why?
   How do you guarantee ordering for a specific entity?
   Poison message keeps failing. How do you handle it?
   Consumer cannot process messages because the database is down. What should happen?
   A consumer is restarted. Will it process old messages?
   How do you safely retry failed Kafka messages?
   Producer receives an acknowledgment, but the application crashes afterward. What happens?
   How do you make a Kafka consumer idempotent?
6. Idempotency & Duplicate Requests
   Client sends the same payment request twice. How do you prevent duplicate payment?
   Client times out but the server actually completed the request. Client retries. What happens?
   A job inserts multiple records and crashes halfway. How do you safely retry it?
   Two identical requests arrive simultaneously. How do you handle them?
   When is a database unique constraint enough, and when do you need an idempotency key?
   Your API returns a response, but the client never receives it. The client retries. How do you handle this?
   How do you store and manage idempotency keys?
   What happens if two requests use the same idempotency key with different request bodies?
   How long should you store an idempotency key?
7. Distributed Transactions & Saga
   Order creation involves Payment, Inventory, and Shipping services. How do you handle failure?
   Payment succeeds, but Inventory fails. What happens?
   Inventory is reserved, but Payment fails. What happens?
   A compensation transaction itself fails. What do you do?
   Choreography vs Orchestration Saga — which would you choose?
   How do you make Saga operations idempotent?
   How do you handle duplicate Saga events?
   A service is temporarily unavailable during a Saga. What happens?
   How do you track the state of a distributed transaction?
8. API Gateway & Security
   API Gateway becomes slow. How do you troubleshoot?
   API Gateway goes down. What happens?
   How do you prevent one client from overwhelming your APIs?
   Rate limiting suddenly blocks legitimate users. What do you check?
   Authentication works for some requests but fails for others. Why?
   JWT validation suddenly starts failing. What could cause it?
   How do you handle a slow downstream service at the Gateway?
   Gateway routes traffic to the wrong service. What do you check?
   How do you safely deploy a new API version?
9. Deployment & Scaling
   A new deployment causes errors. What do you do?
   Application works locally but fails in production. How do you troubleshoot?
   One instance has high CPU while others are normal. Why?
   Traffic increases suddenly. How does your system handle it?
   Autoscaling is not happening even though CPU is high. What do you check?
   New version works for some users but not others. Why?
   Deployment causes intermittent failures. Why?
   How do you perform a rollback safely?
   Application starts successfully but fails after a few minutes. What do you check?
10. Monitoring, Logs & Production Debugging
    Users report that the application is slow. Where do you start?
    Error rate suddenly increases. How do you investigate?
    CPU is normal, but the API is slow. What do you check?
    Memory usage is high. What metrics do you check?
    How do you identify which microservice is causing an end-to-end request to be slow?
    Logs show errors, but metrics look normal. What does that indicate?
    Metrics show failures, but logs show no errors. Why could that happen?
    A problem occurs only occasionally. How do you capture and investigate it?
    How do you debug a production issue that you cannot reproduce locally?
    ⭐ Bonus: Harder Interview Scenarios
    Everything looks healthy, but users report failures. How do you investigate?
    Only users from one region experience high latency.
    Only some requests fail, while others succeed.
    System works normally at low traffic but fails under load.
    Database is healthy, services are healthy, but requests still time out.
    A small increase in traffic causes a huge increase in latency. Why?
    After enabling retries, error rate decreased but latency increased significantly. Why?
    Service metrics are normal, but end-to-end latency is high.
    You fixed the immediate issue, but it keeps happening again. What do you do?
    How would you design your microservices so production issues are easier to debug?







Microservices Scenario-Based Interview Questions
1. Service-to-Service Communication
   Service A cannot communicate with Service B. How do you troubleshoot?
   Service B is intermittently unavailable. What could be happening?
   Service A → Service B works sometimes but times out sometimes. Why?
   You get Connection Refused. What does it mean?
   What is the difference between Connection Timeout and Read Timeout?
   Service B's health endpoint is UP, but actual requests are failing. Why?
   Service A waits indefinitely for Service B. How do you handle it?
   Service B becomes slow and causes Service A to become slow. What do you do?
   Service B starts returning HTTP 500 errors. What should Service A do?
   One downstream service is completely down. How do you prevent cascading failure?
   How do you decide timeout values between services?
   When should you use synchronous communication vs asynchronous communication?
   How do you handle partial failure between microservices?
2. API Gateway Scenarios
   One service receives too many requests. How do you protect it?
   A malicious user sends thousands of requests. What do you do?
   How do you implement rate limiting?
   Authentication should happen before requests reach services. Where do you implement it?
   One API should be accessible only to admins. How do you handle it?
   Gateway is becoming slow. How do you troubleshoot it?
   Gateway is down. What happens to all services?
   How do you avoid the API Gateway becoming a single point of failure?
   Different clients need different responses from the same backend. How do you design this?
   How do you route requests to different versions of a service?
   How do you implement request logging in the Gateway?
   How do you add correlation IDs at the Gateway?
   A backend service is slow. Should the Gateway retry automatically?
   How do you configure circuit breaking at the Gateway?
3. Service Discovery & Load Balancing
   A service registers with Eureka but other services cannot find it.
   A service suddenly disappears from service discovery.
   Eureka says a service is available, but requests fail.
   Multiple instances of Service B are running. How does Service A choose one?
   One instance is unhealthy but still receives traffic.
   A new service instance starts. How does it start receiving traffic?
   What happens when service discovery is temporarily unavailable?
   How do you prevent requests from going to unhealthy instances?
   Why might load balancing cause intermittent failures?
4. Resilience & Cascading Failures
   Service B is slow. Service A has 1000 waiting threads. What happens?
   One dependency failure causes the entire system to fail. Why?
   When should you use a Circuit Breaker?
   Circuit Breaker opens frequently. What do you investigate?
   What is the difference between Retry and Circuit Breaker?
   When can retries make a problem worse?
   What is a retry storm?
   How do exponential backoff and jitter help?
   When should you use Bulkhead?
   How do you isolate failures between dependencies?
   What fallback should you return when a dependency is unavailable?
   Can fallback cause incorrect business behavior? Give an example.
   How do you prevent thread pool exhaustion?
   How would you design a resilient service calling 3 downstream services?
5. Performance & Slow API Scenarios
   An API that normally takes 100ms now takes 5 seconds. How do you troubleshoot?
   P95 latency is high, but average latency looks normal. What does that indicate?
   P99 latency suddenly increases. What do you check?
   CPU usage is 95%. How do you investigate?
   Memory usage continuously increases. What could be happening?
   JVM heap is full. What do you check?
   Garbage Collection is taking too long. What do you investigate?
   Thread pool is exhausted. What happens?
   Database connection pool is exhausted. What happens?
   Traffic suddenly increases 10 times. How do you handle it?
   Only one API endpoint is slow. How do you identify the cause?
   All APIs are slow. What common components do you check?
   API latency is high only during peak traffic. Why?
   Requests are queued before processing. Where do you investigate?
6. Observability & Production Debugging

This is especially important for you because you are learning Actuator → Prometheus → Grafana → Tracing.

Users report that the application is slow. What is your debugging approach?
How do you identify which microservice is slow?
How do distributed tracing help?
How do you trace one request across 5 microservices?
What is a correlation ID and why is it useful?
Logs show errors, but metrics look normal. What do you do?
Metrics show high latency, but CPU is normal. What do you check next?
How do you identify whether the problem is application, database, or network?
Which metrics do you check first during an incident?
What is the difference between monitoring and observability?
How do Prometheus and Grafana help in troubleshooting?
What alerts would you configure for a production microservice?
How do you investigate an increase in HTTP 500 errors?
How do you investigate an increase in HTTP 429 errors?
What would you check if error rate increases but traffic remains the same?
7. Database Scenarios
   The database becomes slow and all APIs are affected. How do you troubleshoot?
   Database CPU is high. What could cause it?
   One SQL query suddenly becomes slow. How do you investigate?
   How do indexes improve performance?
   Can too many indexes cause problems?
   Database connection pool is exhausted. How do you debug it?
   What is a connection leak?
   How do you detect connection leaks?
   Two users update the same record simultaneously. What happens?
   How do you handle concurrency?
   What is optimistic locking?
   What is pessimistic locking?
   A transaction is holding locks for too long. What happens?
   Database deadlock occurs. How do you troubleshoot?
   How do you handle database transaction failures?
   Why should each microservice ideally own its database?
8. Distributed Transactions & Saga
   Order Service creates an order, but Payment Service fails. What happens?
   Payment succeeds, but Inventory update fails. How do you handle it?
   How does the Saga pattern solve distributed transaction problems?
   What is a compensating transaction?
   Choreography Saga vs Orchestration Saga?
   What happens if the Saga orchestrator crashes?
   How do you make Saga operations idempotent?
   How do you recover incomplete Sagas?
   Can compensating transactions completely undo everything?
   How do you handle eventual consistency with users?
9. Kafka / Messaging Scenarios
   A Kafka consumer processes the same message twice. What do you do?
   How do you make a consumer idempotent?
   A producer sends a message, but you don't know whether Kafka received it. What do you do?
   A consumer crashes while processing a message.
   When should Kafka offsets be committed?
   Consumer lag continuously increases. How do you troubleshoot?
   One partition has huge traffic while others are idle. Why?
   How do you handle uneven partition distribution?
   A Kafka broker goes down. What happens?
   The leader partition goes down. What happens?
   What happens when all consumers in a consumer group are busy?
   You have 10 consumers but only 3 partitions. What happens?
   How do you retry failed Kafka messages?
   What is a Dead Letter Topic?
   A poison message keeps failing. How do you handle it?
   How do you ensure message ordering?
   Can Kafka guarantee exactly-once processing?
   A consumer processes a message successfully but crashes before committing the offset.
   Database update succeeds but Kafka event publishing fails. How do you solve this?

👉 This last question leads to the Transactional Outbox Pattern, which is a very important interview topic.

10. Deployment & Production Scenarios
    A new deployment causes errors. What do you do?
    How do you rollback a failed deployment?
    How do you deploy without downtime?
    What is Blue-Green Deployment?
    What is Canary Deployment?
    A new version is incompatible with the old version. How do you handle it?
    How do you handle database schema changes during deployment?
    What happens if half the service instances run old code and half run new code?
    How do you safely deploy a breaking API change?
    A deployment succeeds, but the application is unhealthy. What do you check?
11. Kubernetes / Container Scenarios

You can learn these later, but these are common in experienced interviews.

Container keeps restarting. How do you troubleshoot?
Pod is running but application is not accessible.
Pod is in CrashLoopBackOff.
Application works locally but not inside Docker.
Environment variables are missing in production.
Container is using too much memory.
Pod is killed because of memory usage.
How do readiness and liveness probes help?
Service has multiple pods, but traffic doesn't reach some pods.
How do you scale a microservice?
How do you handle sudden traffic spikes?
12. Security Scenarios
    JWT token is valid but user should no longer have access. What do you do?
    JWT token expires during a request.
    How do you handle refresh tokens?
    One microservice should communicate securely with another. How?
    How do you prevent unauthorized service-to-service calls?
    Sensitive data appears in logs. What do you do?
    How do you secure internal APIs?
    How do you rotate secrets without downtime?
    API is vulnerable to brute-force login attempts. How do you protect it?
13. Caching / Redis Scenarios
    Cached data is stale. What do you do?
    Database data changes but Redis still contains old data.
    Redis goes down. What happens?
    How do you prevent cache stampede?
    Cache hit ratio suddenly decreases. What do you investigate?
    When should you not use caching?
    Two users receive inconsistent cached data. Why?
14. Real Production Incident Questions

These are extremely important because interviewers often ask them like this:

Production is down. What is your first step?
Users report intermittent failures. How do you investigate?
Only some users are affected. What do you check?
Errors happen only during peak hours.
Everything works in lower environments but fails in production.
CPU is normal, memory is normal, but the application is slow.
Database is healthy, but APIs are slow.
One microservice is healthy but the complete business flow fails.
Error rate suddenly increases after deployment.
Traffic increases but the system doesn't scale.
One downstream dependency is causing failures everywhere.
Logs are too large to manually search. How do you debug?
You cannot reproduce the production issue locally. What do you do?
The issue happens randomly once every few hours.
⭐ The MOST IMPORTANT Questions to Practice First

Don't try to answer all 175 at once.

Start with these 30 questions:

Service communication
Service A cannot communicate with Service B.
Intermittent failures.
Connection refused vs timeout.
Slow downstream service.
Cascading failure.
Gateway
Too many requests.
Rate limiting.
Gateway slow.
Authentication at Gateway.
Resilience
Circuit Breaker.
Retry storm.
Bulkhead.
Timeout strategy.
Performance
API suddenly becomes slow.
P95/P99 increases.
High CPU.
Memory leak.
Thread pool exhaustion.
Connection pool exhaustion.
Observability
How do you troubleshoot production issues?
How do you identify the slow microservice?
Metrics → logs → traces approach.
Database
Slow database.
Slow query.
Connection leak.
Concurrent update/deadlock.
Kafka
Duplicate message.
Consumer lag.
Consumer failure.
DB update succeeds but event publishing fails.