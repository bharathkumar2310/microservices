Scenario 1 — API suddenly becomes slow

Imagine you're on production support.

Interviewer:
“Users are reporting that the GET /orders/{id} API has suddenly become very slow. It normally takes 200 ms, but now users are seeing 5–10 seconds. How would you investigate this?”

First I will verify whether we still have this issue or this issue was present previously and not there now
Then I would look into all monitoring metric ike traffics, cpu, memory, latency, db latency, threads, gc etc
Based on theese metrics combinations I wil try to narrow down the problem
Once the problem is narrowed down I will look into distribute traicng to find where exactly we have the issue
Once aftre finding where I iwill ook into logs and code to find out why
after finding out wat, where and why I will fix the issue, and then verify it agian whether we have thes issue or not



for db the part to know

monitoring ---> apm/distributed tecing---> Explain/logs/code ---> fix ---> verif__-> monitoring
some scenatios like

index issues, db optimizer issue , n+11 query, pagination, huge db, connection ppooling, transaction issues, new deployments

then know metrics like p95, 99, query latency, connection pool metricses, db cpu, 



select emp_name feom employee e where salary = (select max(salary) from employee e1 where e.dept_id = e1.dept_id) 











---------------------------------------------------------------------------------------------------------------------------------------



A. Service-to-service communication

Service A is unable to connect to Service B. How would you investigate?
Service A is getting "Connection Refused" while calling Service B. What could be the reasons?
Service A is getting a connection timeout while calling Service B. How would you troubleshoot it?
Service A connects to Service B, but the response takes too long and eventually gets a Read Timeout. What could be happening?
Service A → Service B works sometimes but times out sometimes. How would you investigate?
Service B is running, but Service A cannot reach it. What would you check?
Service A can ping Service B's server, but the API call fails. Why?
DNS resolution for Service B is failing. How would you troubleshoot it?
The hostname works from your laptop but doesn't work from Service A. What could be wrong?
Service A receives HTTP 502 from the API Gateway. How would you investigate?
Service A receives HTTP 503 from the Gateway. What could be the cause?
Service A receives HTTP 504 from the Gateway. How would you investigate?
Only one instance of Service B is failing while other instances work. How would you identify the problem?
Requests routed to one particular Service B instance are failing. What could cause this?
Service B is healthy according to its health endpoint, but actual API requests are failing. Why?


B. API suddenly becomes slow
An API that normally takes 200 ms suddenly takes 5 seconds. How would you investigate?
Only one API endpoint is slow while all other endpoints are normal. What would you check?
All APIs in a service suddenly become slow. What could be the possible causes?
Service A is slow, but its CPU and memory look normal. How would you investigate?
Service A is slow only during peak traffic. What could be happening?
The API is fast in development but slow in production. How would you investigate?
An API suddenly starts timing out after a new deployment. What would you check?
Response time increases gradually over several hours. What could cause this?
C. CPU / Memory / JVM
A production service suddenly has 95% CPU utilization. How would you troubleshoot it?
CPU is continuously high even though traffic has not increased. What could be happening?
Memory usage keeps increasing and eventually the service crashes with OutOfMemoryError. How would you investigate?
A service restarts periodically without any code deployment. What would you check?
GC activity suddenly becomes very high. How would you investigate?
One instance has much higher memory usage than other instances. What could be the reason?
The application is not responding, but CPU and memory appear normal. What would you investigate?
The application has many threads and becomes unresponsive. What could be happening?
Thread pool exhaustion occurs in production. How would you troubleshoot it?
D. Database
The database suddenly becomes slow and all APIs depending on it become slow. How would you investigate?
An API is slow because of a database query. How would you identify the problematic query?
Database CPU suddenly reaches 100%. What would you check?
The application cannot obtain a database connection. What could be the reasons?
Database connection pool is exhausted. How would you troubleshoot it?
Connections are increasing continuously and never being released. What could be wrong?
A query that normally takes 100 ms suddenly takes 10 seconds. What would you investigate?
An API works for small data but becomes extremely slow for large data. Why?
An N+1 query problem appears in production. How would you identify and fix it?
Two requests are waiting for each other and database operations are stuck. What could be happening?
A database deadlock occurs in production. How would you investigate it?
The database is available, but the application still gets connection timeout errors. Why?
Only one application instance is unable to connect to the database. What would you check?
E. Kafka / asynchronous communication
Kafka messages are being produced successfully, but consumers are not processing them. How would you investigate?
Consumer lag suddenly increases. What could be the reasons?
Kafka consumer is processing messages very slowly. How would you troubleshoot it?
Messages are being processed twice. Why can this happen?
A Kafka message appears to have been lost. How would you investigate?
A consumer crashes while processing a message. What happens to the message?
Kafka consumer keeps rebalancing. What could cause this?
One consumer instance is much slower than the others. What would you check?
Messages are arriving out of order. Why could this happen?
A downstream service fails after consuming a Kafka message. How would you handle this?
A poison message keeps failing repeatedly. How would you handle it?
How would you investigate a sudden increase in Kafka consumer lag in production?
How would you prevent duplicate processing of Kafka messages?
F. Distributed logging / tracing
A request passes through five microservices and eventually fails. How would you find where it failed?
You see an error in Service A, but the actual failure occurred in Service D. How would you trace it?
How would you trace one user's request across multiple microservices?
You have millions of logs. How would you find the logs belonging to one request?
The API returns 500, but Service A's logs don't show the root cause. What would you do?
Distributed tracing shows one downstream service taking 4 seconds. How would you investigate further?
Logs from different services have different timestamps. How would you correlate them?
A request succeeds in Service A but fails somewhere downstream. How would you identify the exact service?
How would you investigate a production issue when you have logs, metrics and traces available?
G. Resilience
Service B is down. How should Service A behave?
Service B is intermittently failing. Would you use retry? How?
Retries are making the production problem worse. Why?
How would you prevent one failing service from bringing down the entire system?
When would you use a circuit breaker?
Circuit breaker is constantly opening. How would you investigate?
A downstream service takes 30 seconds to respond. How would you protect your service?
How would you handle temporary downstream failures?
When would you use timeout vs retry vs circuit breaker?
What happens if 1000 requests retry simultaneously after a downstream failure?
H. Distributed transactions / Saga
Order creation succeeds but payment fails. How would you maintain consistency?
Payment succeeds but order creation fails. What would you do?
Three microservices participate in one business transaction and the third service fails. How would you handle it?
How would you implement Saga for an e-commerce order?
What happens if a Saga compensation operation itself fails?
How would you make Saga operations idempotent?
When would you choose choreography vs orchestration?
How would you troubleshoot a stuck Saga in production?
I. Deployment / configuration
Everything was working before deployment, but APIs started failing immediately after deployment. How would you investigate?
Only newly deployed instances are failing. What could be wrong?
Old instances work but new instances fail to connect to the database. What would you check?
A configuration value is correct locally but wrong in production. How would you troubleshoot it?
One service instance has different configuration from the others. What could cause this?
A service registers successfully but other services cannot discover it. What would you check?
Health check is failing after deployment even though the application starts successfully. Why?
A deployment causes a sudden increase in 5xx errors. How would you investigate?
J. Scaling / load
Traffic suddenly increases 10×. What happens to your microservices system?
One service cannot handle increased traffic. How would you scale it?
You horizontally scale a service, but performance does not improve. Why?
Traffic is unevenly distributed between service instances. What could be wrong?
Adding more application instances makes the database slower. Why?
Kafka consumers are unable to keep up with increasing traffic. What would you do?
How would you identify the bottleneck in a microservices system under heavy load?
K. Cache
The application is returning stale data from cache. How would you investigate?
Cache suddenly stops working and database traffic increases dramatically. What could happen?
Redis becomes unavailable. How should the application behave?
A cache stampede occurs during high traffic. How would you prevent it?
Cache hit ratio suddenly drops. What would you investigate?
L. Security
A previously working API suddenly returns 401. How would you investigate?
The API returns 403 even though the JWT is valid. Why?
Authentication works at the Gateway but fails at the downstream service. What would you check?
Service-to-service authentication suddenly stops working. What could be wrong?
JWT validation suddenly fails after deployment. How would you troubleshoot it?
M. Complex real-production scenarios
