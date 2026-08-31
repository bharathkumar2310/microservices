| Area                        | Metrics to check                   | What abnormality tells you                                  | What to investigate next                 |
| --------------------------- | ---------------------------------- | ----------------------------------------------------------- | ---------------------------------------- |
| **API / HTTP**              | P50, P95, P99 latency              | Requests are becoming slower; P99 shows worst tail behavior | Identify affected endpoint and trace it  |
|                             | Request rate / RPS                 | Traffic spike may be causing saturation                     | Capacity, autoscaling, thread pool, DB   |
|                             | Error rate                         | Failures may cause retries/timeouts and increase latency    | Logs, downstream, DB, recent deployment  |
|                             | Response status codes              | 4xx/5xx increase can indicate failures                      | Application/downstream causes            |
| **Pod / Instance**          | Requests per pod                   | Uneven traffic distribution                                 | Load balancer, routing, pod capacity     |
|                             | CPU utilization                    | CPU saturation can slow processing                          | Thread dumps, profiling, expensive code  |
|                             | Memory / heap                      | Memory pressure can affect performance                      | GC, heap usage, leaks                    |
|                             | Pod restarts                       | Instability or OOM                                          | Logs, Kubernetes events                  |
|                             | Network I/O                        | Network saturation/abnormal traffic                         | Network/downstream communication         |
| **JVM / GC**                | Heap used                          | High heap usage                                             | Allocation, memory leak                  |
|                             | GC frequency                       | Frequent GC can consume CPU                                 | Object allocation, heap sizing           |
|                             | GC pause time                      | Application pauses during GC                                | GC tuning, allocation rate               |
|                             | Old-gen usage                      | Objects surviving too long                                  | Memory leak / excessive retention        |
| **Thread Pool**             | Active threads                     | High utilization → approaching saturation                   | Thread pool sizing, blocking operations  |
|                             | Pool size                          | Too few threads can cause waiting                           | Configuration and workload               |
|                             | Queue size                         | Requests/tasks waiting for threads                          | Blocking tasks, pool saturation          |
|                             | Rejected tasks                     | Pool is exhausted                                           | Thread pool and application bottleneck   |
| **DB Connection Pool**      | Active connections                 | High usage may indicate DB pressure                         | Query latency, pool size                 |
|                             | Idle connections                   | Shows available capacity                                    | Pool configuration                       |
|                             | Pending/waiting connections        | Requests waiting for DB connection                          | Slow queries, connection leak, pool size |
|                             | Connection acquisition time        | Time spent obtaining connection                             | Pool exhaustion                          |
| **Database**                | Query latency                      | DB operations are slow                                      | Identify slow query                      |
|                             | DB CPU                             | DB may be overloaded                                        | Expensive queries                        |
|                             | DB connections                     | Too many connections                                        | Pool/configuration                       |
|                             | Lock waits                         | Transactions waiting for locks                              | Blocking transactions                    |
|                             | Slow query count                   | Queries exceeding threshold                                 | EXPLAIN/query optimization               |
|                             | Rows examined                      | Query scanning too much data                                | Indexes/query design                     |
|                             | Index usage                        | Missing/poor indexes                                        | EXPLAIN                                  |
| **Downstream APIs**         | Downstream latency                 | Another service is slowing your API                         | Trace downstream call                    |
|                             | Downstream error rate              | Failures can trigger retries/timeouts                       | Logs, resilience mechanisms              |
|                             | Timeout count                      | Calls are exceeding configured timeout                      | Downstream health/timeout configuration  |
|                             | Retry count                        | Same request may execute multiple times                     | Retry configuration/downstream           |
|                             | Circuit-breaker state              | Dependency may be unhealthy                                 | Dependency investigation                 |
| **Network**                 | Network latency                    | Time spent communicating                                    | Network/service location                 |
|                             | Connection errors                  | Communication problems                                      | DNS/network/load balancer                |
|                             | Connection establishment time      | Slow TCP/TLS setup                                          | Connection pooling/network               |
| **Application**             | Method execution time              | Business logic itself is slow                               | Profiling/code                           |
|                             | External call duration             | Time spent calling dependencies                             | Trace dependency                         |
|                             | Lock/wait time                     | Threads blocked waiting                                     | Synchronization/DB locks                 |
|                             | Queue/wait time                    | Work waiting before execution                               | Thread/executor saturation               |
| **Infrastructure**          | Node CPU                           | Multiple pods may compete for CPU                           | Node capacity                            |
|                             | Node memory                        | Resource pressure                                           | Pod eviction/OOM                         |
|                             | Disk I/O                           | Slow disk operations                                        | Logs, DB, storage                        |
|                             | Network utilization                | Infrastructure network saturation                           | Network                                  |
| **Load Balancer / Gateway** | Gateway latency                    | Time spent before reaching service                          | Gateway/routing                          |
|                             | Gateway → service latency          | Network/service communication issue                         | Service/network                          |
|                             | Gateway request rate               | Traffic distribution                                        | Load balancing                           |
|                             | Gateway errors/timeouts            | Gateway or backend issue                                    | Gateway/backend                          |
| **Deployment**              | Latency before vs after deployment | Possible regression                                         | Recent code/config change                |
|                             | Error rate before vs after         | Deployment may have introduced failures                     | Logs/code/config                         |
|                             | Pod version distribution           | Different versions may behave differently                   | Compare versions                         |





Main Causes of CPU High


| Cause                                               | What happens                                                  | Example                                   |
| --------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------------- |
| **1. Expensive business logic**                     | Code performs too much computation                            | Large loops, complex calculations         |
| **2. Infinite / very long loop**                    | CPU keeps executing without finishing                         | `while` loop with incorrect termination   |
| **3. Too many requests**                            | More requests → more CPU work                                 | Traffic spike                             |
| **4. Inefficient algorithm**                        | CPU work grows rapidly with input                             | O(n²) instead of O(n)                     |
| **5. Excessive JSON serialization/deserialization** | CPU spent converting large objects                            | Huge request/response                     |
| **6. Excessive object creation**                    | JVM spends CPU allocating/managing objects                    | Creating thousands of temporary objects   |
| **7. Excessive GC**                                 | CPU spent doing garbage collection                            | Huge allocation rate / heap pressure      |
| **8. Excessive logging**                            | Formatting and writing logs consumes CPU                      | Logging huge objects at high request rate |
| **9. Regex / string processing**                    | Complex string operations consume CPU                         | Expensive regex on large input            |
| **10. Encryption/compression**                      | CPU-heavy operations                                          | Encryption, hashing, compression          |
| **11. Excessive retries**                           | Same work gets executed repeatedly                            | Downstream failure → 3 retries            |
| **12. Thread contention/spinning**                  | Threads repeatedly compete/check instead of doing useful work | Busy-waiting                              |
| **13. Lock contention**                             | Threads repeatedly compete for synchronized resources         | Poorly designed synchronization           |
| **14. Serialization of huge collections**           | CPU spent processing large datasets                           | Returning 100k records                    |
| **15. Bad code/deployment**                         | New release introduces CPU-heavy path                         | New algorithm or loop                     |


                         API LATENCY ↑
                              │
                              ▼
                     ┌─────────────────┐
                     │ 1. TRAFFIC / RPS │
                     └─────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
               TRAFFIC HIGH        TRAFFIC NORMAL
                    │                   │
                    ▼                   ▼
              Check CPU              Check CPU
                    │                   │
          ┌─────────┴─────────┐   ┌─────┴──────────┐
          │                   │   │                │
       CPU HIGH           CPU NORMAL CPU HIGH   CPU NORMAL
          │                   │   │                │
          ▼                   ▼   ▼                ▼
    CPU saturation       WAITING   CPU-consuming   WAITING
    / GC / processing    problem   work            problem
          │                   │   │                │
          └─────────┬─────────┘   └──────┬─────────┘
                    │                    │
                    ▼                    ▼
             CHECK THREAD POOL     CHECK THREAD POOL
                    │                    │
             ┌──────┴──────┐      ┌──────┴──────┐
             │             │      │             │
          SATURATED      NORMAL SATURATED      NORMAL
             │             │      │             │
             ▼             ▼      ▼             ▼
         Queueing      Look at   Queueing    Look at
         / blocking    DB / DS   / blocking  DB / DS




Traffic
↓
CPU
↓
Memory / Heap
↓
GC
↓
Thread Pool
↓
Database
↓
Connection Pools
↓
Downstream Services
↓
Network
↓
Locks / Blocking
↓
Retries
↓
Queues / Kafka
↓
External APIs
↓
Logs
↓
Distributed Tracing


CPU ↑
│
├── 1. Is it application CPU or GC CPU?
│
├── 2. Is it all instances or one instance?
│
├── 3. Did a deployment happen recently?
│
├── 4. Which threads are consuming CPU?
│
└── 5. What operation are those threads performing?