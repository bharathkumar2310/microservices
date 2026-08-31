API/HTTP SCENARIOS


| Metric / Signal           | What you see                  | Possible problem                                             | Next thing to check                       | Typical fix                                                 |
| ------------------------- | ----------------------------- | ------------------------------------------------------------ | ----------------------------------------- | ----------------------------------------------------------- |
| **Request rate (RPS)**    | Sudden increase               | Traffic spike / overload                                     | CPU, threads, DB pool, downstream traffic | Autoscaling, rate limiting, caching, capacity increase      |
| **Latency (p50/p95/p99)** | Latency increased             | Slow DB, downstream API, CPU, GC, thread pool                | Distributed trace                         | Fix slow component                                          |
| **5xx error rate**        | 500/502/503/504 increased     | Application exception, dependency failure, timeout, overload | Logs + traces                             | Fix root cause; timeout/circuit breaker if dependency issue |
| **4xx error rate**        | 400/401/403/404 increased     | Client/auth/routing problem                                  | Logs + request details                    | Fix validation/auth/routing/client                          |
| **502**                   | Gateway receives bad response | Backend unavailable/crashed/bad response                     | Gateway + backend logs                    | Fix backend/network/config                                  |
| **503**                   | Service unavailable           | Service overloaded/down/no healthy instance                  | Service health + load balancer            | Restore service/autoscaling                                 |
| **504**                   | Gateway timeout               | Backend taking too long                                      | Trace backend latency                     | Optimize backend / timeout appropriately                    |
| **Request timeout count** | Increasing                    | Slow dependency / thread starvation                          | Trace + thread pool                       | Optimize dependency / timeout / bulkhead                    |
| **Active requests**       | Continuously increasing       | Requests aren't completing                                   | Thread pool + downstream latency          | Find blocking/slow operation                                |
| **Concurrent requests**   | Very high                     | Service saturation                                           | CPU, threads, DB connections              | Scale / optimize / limit concurrency                        |


Latency scenario

| Finding                    | What it can mean                | Next step                        | Fix                                    |
| -------------------------- | ------------------------------- | -------------------------------- | -------------------------------------- |
| p50 ↑                      | General performance degradation | Trace representative requests    | Find common bottleneck                 |
| p95 ↑ but p50 normal       | Some requests are slow          | Examine slow traces              | Investigate specific path/query        |
| p99 ↑ dramatically         | Tail latency problem            | Look for DB/downstream/GC spikes | Fix worst-case operation               |
| Service A latency ↑        | A itself may be slow            | Check A's CPU/thread/DB          | Fix A                                  |
| Service A waits for B      | Downstream dependency slow      | Trace B                          | Fix B or add timeout/circuit breaker   |
| DB span dominates          | Database bottleneck             | Slow query/DB metrics            | Query optimization/indexing            |
| External API dominates     | Dependency slow                 | External-call metrics/logs       | Timeout, retry policy, cache, fallback |
| No obvious dependency slow | Thread/CPU/GC issue             | Thread dump/CPU/GC               | Optimize/scale                         |


CPU Scenario

| CPU finding             | Possible problem            | What to check next            | Fix                              |
| ----------------------- | --------------------------- | ----------------------------- | -------------------------------- |
| CPU suddenly 90–100%    | Traffic spike               | RPS + request rate            | Scale horizontally               |
| CPU high + latency high | CPU saturation              | Thread/CPU profiling          | Optimize expensive code          |
| CPU high + GC high      | Excessive object allocation | GC metrics + heap             | Reduce allocations / tune memory |
| CPU high + RPS normal   | Code regression             | Recent deployment + profiling | Optimize code/rollback           |
| One pod CPU high        | Uneven load                 | Load balancing                | Fix distribution                 |
| All pods CPU high       | Genuine capacity problem    | Traffic vs capacity           | Scale                            |
| CPU low + latency high  | CPU isn't bottleneck        | Trace + DB + threads          | Investigate I/O/dependencies     |
| CPU constantly high     | Sustained saturation        | Autoscaling configuration     | Scale capacity                   |


Memory/JVM Scenario

| Metric                 | Abnormal finding        | Possible problem           | Next step                    | Fix                                  |
| ---------------------- | ----------------------- | -------------------------- | ---------------------------- | ------------------------------------ |
| Heap usage             | Continuously increasing | Memory leak                | Heap dump                    | Fix retained objects                 |
| Heap usage             | Sudden spikes           | Large allocations          | GC + allocation profiling    | Reduce allocation                    |
| GC frequency           | Very high               | Too much garbage           | GC logs                      | Optimize allocations                 |
| GC pause               | Long pauses             | Heap/GC pressure           | GC analysis                  | Tune JVM / reduce allocation         |
| OOM errors             | JVM crashes             | Heap exhausted             | Heap dump + logs             | Fix leak / increase appropriate heap |
| Non-heap/direct memory | High                    | Native/direct buffer issue | JVM/native metrics           | Fix resource usage                   |
| Pod memory             | Near container limit    | Container OOMKill          | Kubernetes/container metrics | Fix memory usage / right-size        |


DataBase scenarios

| DB Metric / Signal     | What you see | Possible problem                  | Next step                        | Typical fix                        |
| ---------------------- | ------------ | --------------------------------- | -------------------------------- | ---------------------------------- |
| DB CPU                 | Very high    | Expensive queries                 | Identify top queries             | Optimize SQL/index                 |
| Query latency          | Increased    | Slow query                        | `EXPLAIN` / query plan           | Index/query optimization           |
| Connections            | Near max     | Pool exhaustion                   | Check active/pending connections | Fix slow queries/leaks/pool sizing |
| Pending DB connections | Increasing   | Threads waiting for DB            | Find long-running queries        | Optimize queries/transactions      |
| Active connections     | High         | Too many concurrent DB operations | Query + transaction analysis     | Reduce concurrency/optimize        |
| Lock waits             | Increasing   | Transactions blocking each other  | Find blocking transactions       | Shorten transactions/fix locking   |
| Deadlocks              | Increasing   | Concurrent transactions conflict  | DB deadlock logs                 | Correct transaction/order/indexing |
| Disk I/O               | High         | Heavy reads/writes                | Query analysis                   | Index/query/schema optimization    |
| Rows scanned           | Huge         | Full/table scan                   | `EXPLAIN`                        | Add/adjust index                   |
| Slow queries           | Increasing   | Bad SQL/index/query plan          | Query analysis                   | Optimize                           |
| Transaction duration   | High         | Long-running transaction          | Application + DB                 | Shorten transaction                |


Connection Pool Scenario


| Metric             | Finding    | Meaning                         | Next investigation              | Fix                                        |
| ------------------ | ---------- | ------------------------------- | ------------------------------- | ------------------------------------------ |
| Active connections | Near max   | Pool heavily used               | Slow queries/transactions       | Optimize DB work                           |
| Idle connections   | 0          | No spare connections            | Active + pending                | Increase capacity only if DB can handle it |
| Pending threads    | Increasing | Requests waiting for connection | Why connections aren't returned | Fix slow queries/leaks                     |
| Connection timeout | Increasing | Couldn't obtain connection      | DB latency/pool exhaustion      | Fix root cause                             |
| Pool max           | Too small  | Artificial bottleneck           | Compare demand vs DB capacity   | Tune pool carefully                        |
| Pool max           | Very large | Can overload DB                 | DB max connections/CPU          | Reduce pool size                           |


n+1 Query Scebario

| Finding                                                     | Meaning                                | Next step                    | Fix                                 |
| ----------------------------------------------------------- | -------------------------------------- | ---------------------------- | ----------------------------------- |
| One API request generates hundreds/thousands of SQL queries | N+1 likely                             | Trace + SQL logs             | Fetch join/entity graph/batching    |
| DB latency high                                             | Many queries                           | Count queries/request        | Reduce query count                  |
| CPU/DB normal but API slow                                  | Excessive round trips                  | Inspect ORM-generated SQL    | Optimize JPA fetching               |
| `1 + N` pattern                                             | One parent query + one query per child | Examine relationship mapping | `JOIN FETCH`, EntityGraph, batching |


Kafka Scenario

| Metric                | Finding       | Possible problem              | Next step                 | Fix                           |
| --------------------- | ------------- | ----------------------------- | ------------------------- | ----------------------------- |
| Consumer lag          | Increasing    | Consumer slower than producer | Consumer throughput       | Scale consumers/optimize      |
| Consumer lag          | Sudden spike  | Consumer stopped/slow         | Consumer logs             | Fix consumer                  |
| Consumer throughput   | Low           | Processing slow               | Trace/code                | Optimize                      |
| Producer rate         | Suddenly high | Traffic spike                 | Compare consumer capacity | Scale consumers               |
| Consumer errors       | Increasing    | Processing failure            | Logs                      | Fix exception/data issue      |
| Rebalances            | Frequent      | Consumer instability          | Consumer logs/config      | Fix consumer lifecycle/config |
| Partition utilization | Uneven        | Hot partition/key             | Partition distribution    | Improve key strategy          |
| Consumer CPU          | High          | Processing expensive          | Profiling                 | Optimize/scale                |
| Consumer DB latency   | High          | DB bottleneck                 | DB metrics                | Optimize DB                   |





Thread Pool Scenario

| Finding                 | Possible problem  | Next check          | Fix                           |
| ----------------------- | ----------------- | ------------------- | ----------------------------- |
| Active threads near max | Thread saturation | Thread pool metrics | Increase carefully / optimize |
| Queue size increasing   | Tasks waiting     | Thread dump         | Find slow/blocking operation  |
| Threads blocked         | Lock/I/O          | Thread dump         | Remove blocking               |
| Threads waiting on DB   | DB bottleneck     | Hikari metrics      | Fix DB                        |
| Threads waiting on HTTP | Downstream slow   | Trace               | Timeout/circuit breaker       |
| Many threads + CPU low  | Blocking I/O      | Thread dump         | Async/non-blocking/optimize   |
| Many threads + CPU high | CPU-bound         | Profiling           | Optimize/scale                |


Down Stream Scenario

Service A → Service B → Service C

| Finding                   | Meaning                        | Next step             | Fix                              |
| ------------------------- | ------------------------------ | --------------------- | -------------------------------- |
| A latency ↑               | Something inside A/dependency  | Trace                 | Find slow span                   |
| B latency ↑               | B is slow                      | Check B metrics       | Fix B                            |
| B errors ↑                | B failing                      | B logs                | Fix B                            |
| A timeout calling B       | B slow/unavailable             | B + network metrics   | Timeout/circuit breaker/fix B    |
| Retry count ↑             | Dependency unstable            | Logs/traces           | Fix dependency + control retries |
| All requests to B fail    | B unavailable                  | Health/service status | Restore B                        |
| Only some B requests slow | Specific query/path/data issue | Trace slow requests   | Optimize path                    |


Retry / CB scenario


| Finding                         | Possible problem                | Next step                   | Fix                                 |
| ------------------------------- | ------------------------------- | --------------------------- | ----------------------------------- |
| Retry count ↑                   | Dependency failures/timeouts    | Check downstream            | Fix dependency                      |
| Retry count ↑ + traffic ↑       | Retry storm                     | Calculate effective traffic | Reduce retries/backoff              |
| Circuit OPEN                    | Dependency consistently failing | Check dependency            | Restore dependency                  |
| Circuit repeatedly opens/closes | Dependency unstable             | Logs/metrics                | Stabilize dependency                |
| Requests multiply               | Excessive retries               | Inspect retry configuration | Exponential backoff + limit retries |
| Timeout too high                | Threads held too long           | Thread pool                 | Tune timeout                        |


trafiic/Load Scenario

| Finding                        | Possible problem    | Next check               | Fix               |
| ------------------------------ | ------------------- | ------------------------ | ----------------- |
| RPS ↑ + CPU ↑                  | Traffic overload    | Autoscaling              | Scale             |
| RPS ↑ + DB connections ↑       | DB pressure         | DB CPU/query latency     | Optimize/scale DB |
| RPS ↑ + latency ↑              | Capacity reached    | CPU/threads/DB           | Scale/optimize    |
| RPS normal + latency ↑         | Internal regression | Trace                    | Find bottleneck   |
| One instance gets more traffic | Load imbalance      | Load balancer            | Fix routing       |
| Sudden traffic spike           | Burst               | Rate limiter/autoscaling | Rate limit/scale  |


Gateway Scenario

| Metric             | Finding | Possible issue                | Next step            | Fix                           |
| ------------------ | ------- | ----------------------------- | -------------------- | ----------------------------- |
| Gateway latency    | High    | Gateway processing/downstream | Trace                | Optimize                      |
| Gateway CPU        | High    | Gateway overloaded            | Traffic              | Scale                         |
| 5xx                | High    | Backend/dependency            | Logs + traces        | Fix backend                   |
| 429                | High    | Rate limit reached            | Check client traffic | Tune rate limit appropriately |
| 502                | High    | Bad backend response          | Backend logs         | Fix backend                   |
| 504                | High    | Backend timeout               | Trace                | Fix slow backend              |
| Active connections | High    | Connection saturation         | Connection metrics   | Tune/scale                    |


Deployment Scenario

| Finding                                                 | What it suggests                | Next step               | Fix                |
| ------------------------------------------------------- | ------------------------------- | ----------------------- | ------------------ |
| Latency suddenly increases immediately after deployment | Regression                      | Compare old/new version | Rollback/fix       |
| 500s increase after deployment                          | Code/config regression          | Logs + traces           | Rollback/fix       |
| CPU increases after deployment                          | Inefficient new code            | Profiling               | Optimize           |
| DB queries increase after deployment                    | ORM/code change                 | SQL metrics             | Fix query/fetching |
| Memory increases after deployment                       | Possible leak/allocation change | Heap/GC                 | Fix                |
| Only new pods affected                                  | New version issue               | Compare versions        | Rollback           |


| First metric you notice         | Main possibilities                    | Then look at                 | Typical fixes                |
| ------------------------------- | ------------------------------------- | ---------------------------- | ---------------------------- |
| **Traffic ↑**                   | Load/traffic spike                    | CPU, threads, DB, downstream | Scale, rate limit, cache     |
| **Latency ↑**                   | DB/downstream/CPU/threads/GC          | Distributed trace            | Fix bottleneck               |
| **5xx ↑**                       | Exception/timeout/overload/dependency | Trace + logs                 | Fix root cause               |
| **CPU ↑**                       | High traffic/expensive code           | Profiling + RPS              | Optimize/scale               |
| **Memory ↑**                    | Leak/large allocation                 | GC/heap dump                 | Fix leak/allocation          |
| **GC ↑**                        | Excessive allocation                  | GC/heap                      | Optimize/tune                |
| **DB latency ↑**                | Bad query/index/locks/load            | Slow queries + `EXPLAIN`     | Optimize SQL/index           |
| **DB connections ↑**            | Slow queries/long transactions        | Hikari + DB                  | Fix DB work                  |
| **DB pending ↑**                | Pool exhausted                        | Active connections + queries | Fix bottleneck               |
| **Lock waits ↑**                | Transaction contention                | Blocking transactions        | Shorten/fix transactions     |
| **Kafka lag ↑**                 | Consumer too slow/failing             | Consumer metrics/logs        | Scale/optimize               |
| **Thread queue ↑**              | Thread starvation/blocking            | Thread dump                  | Remove blocking/scale        |
| **Downstream latency ↑**        | Dependency slow                       | Trace + dependency metrics   | Fix dependency/timeout       |
| **Retry ↑**                     | Dependency instability                | Logs + downstream            | Backoff/limit/fix dependency |
| **Circuit OPEN**                | Dependency unhealthy                  | Dependency metrics           | Restore/fallback             |
| **429 ↑**                       | Rate limiting                         | Gateway traffic              | Tune limit/client behavior   |
| **502/503/504 ↑**               | Backend unavailable/timeout           | Gateway + trace              | Fix backend/network          |
| **N+1 detected**                | Excessive DB queries                  | SQL trace                    | Fetch join/batching          |
| **Post-deployment degradation** | Regression                            | Version comparison           | Rollback/fix                 |


And this is the exact flow you should use in interviews

When interviewer says:

"API latency suddenly increased. How will you troubleshoot?"

Don't give them 20 random things.

Say:

1. Metrics

"First I check request rate, p50/p95/p99 latency, error rate, CPU, memory, thread pool and DB connection-pool metrics to determine whether the issue is traffic, resource saturation or errors."

2. Trace

"Then I use distributed tracing to identify which span is contributing most to the latency — application code, database, or downstream service."

3. Logs

"Once I identify the problematic component, I check its logs for exceptions, timeouts, connection failures or other errors."

4. Root cause

"Then I inspect the relevant code, SQL, configuration or dependency to determine the root cause."

5. Fix + verification

"After applying the fix, I deploy it, monitor the same metrics and traces, and verify that latency and error rates have returned to normal."

That same skeleton works for 80–90% of production microservice troubleshooting questions.

The part that changes from scenario to scenario is which metrics you focus on and what abnormal metric value means. That's what we should master next.