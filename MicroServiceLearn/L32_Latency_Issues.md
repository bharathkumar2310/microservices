| Area               | Issue causing API latency      | Why it causes latency                         | What to check                             |
| ------------------ | ------------------------------ | --------------------------------------------- | ----------------------------------------- |
| **Application**    | Inefficient code               | Request takes longer to process               | APM, code execution time, profiler        |
|                    | Expensive computation          | CPU spends longer processing                  | CPU %, APM, CPU profiler                  |
|                    | Large loops / data processing  | More work per request                         | Code, APM, CPU                            |
|                    | Large request/response payload | Serialization/network processing takes longer | Payload size, serialization time          |
|                    | Blocking operations            | Request thread waits                          | APM, thread dump                          |
| **CPU**            | High CPU                       | Threads compete for CPU                       | CPU %, load, process CPU                  |
|                    | CPU-intensive code             | Application spends time calculating           | CPU profiler, thread dump                 |
|                    | CPU throttling                 | Container/process gets limited CPU            | CPU throttling metrics, container metrics |
| **Memory / JVM**   | High memory usage              | JVM spends more time managing memory          | Heap usage, JVM metrics                   |
|                    | Frequent GC                    | Application spends time doing GC              | GC count/frequency, GC pause time         |
|                    | Long GC pauses                 | Requests pause while GC runs                  | GC logs, pause duration                   |
|                    | Memory leak                    | Memory continuously increases                 | Heap usage over time, heap dump           |
| **Threads**        | Thread pool exhausted          | Requests wait for threads                     | Active threads, pool size, queue          |
|                    | Thread contention              | Threads wait for shared resources             | Thread dump, blocked/waiting threads      |
|                    | Deadlock                       | Threads remain blocked                        | Thread dump                               |
|                    | Blocking calls                 | Threads remain occupied waiting               | Thread dump, APM                          |
| **Database**       | Slow SQL                       | API waits for DB                              | SQL execution time, APM                   |
|                    | Missing index                  | DB scans many rows                            | `EXPLAIN`, indexes, rows examined         |
|                    | Wrong index                    | DB chooses inefficient execution plan         | `EXPLAIN`, query plan                     |
|                    | Full table scan                | Large amount of data is scanned               | `EXPLAIN`, rows examined                  |
|                    | N+1 queries                    | One API causes many DB calls                  | SQL logs, APM, query count                |
|                    | Too many DB calls              | Multiple DB round trips                       | APM, SQL logs                             |
|                    | Large result set               | DB/app processes lots of data                 | Rows returned, response size              |
|                    | Bad pagination                 | Large OFFSET scans many rows                  | `EXPLAIN`, query execution time           |
|                    | DB connection pool exhausted   | Request waits for connection                  | HikariCP active/idle/pending, pool size   |
|                    | Connection leak                | Connections aren't returned                   | Active connections, pool metrics, logs    |
|                    | DB CPU saturation              | Queries execute slower                        | DB CPU, active queries                    |
|                    | DB I/O bottleneck              | Queries wait for disk I/O                     | DB I/O metrics                            |
|                    | Lock contention                | Query waits for another transaction           | Locks, waiting transactions               |
|                    | Long transaction               | Locks/resources remain occupied               | Transaction duration, DB locks            |
|                    | Deadlock                       | Transactions are blocked/retried              | DB deadlock logs/metrics                  |
| **Downstream**     | Slow microservice              | Current API waits for downstream              | Distributed tracing, downstream latency   |
|                    | Downstream timeout             | Request waits until timeout                   | Timeout metrics, logs, traces             |
|                    | Downstream unavailable         | Retries/fallbacks add delay                   | Error rate, retry count                   |
|                    | Slow external API              | External dependency delays response           | APM/tracing, external API latency         |
| **Retries**        | Too many retries               | Same operation executes repeatedly            | Retry count, logs                         |
|                    | Large retry timeout            | Request waits longer                          | Timeout configuration, traces             |
|                    | Retry storm                    | Retries overload services                     | Request rate, retry rate, downstream load |
| **Network**        | Network latency                | Request/response takes longer to travel       | Network latency, tracing                  |
|                    | Packet loss                    | Retransmission causes delay                   | Packet loss, network metrics              |
|                    | DNS delay                      | Hostname resolution takes time                | DNS timing, network logs                  |
|                    | TCP/TLS connection setup       | Connection establishment adds latency         | Connection timing, APM                    |
| **Traffic**        | Traffic spike                  | Resources become saturated                    | Requests/sec, CPU, memory                 |
|                    | High concurrency               | Threads/connections become exhausted          | Active requests, thread pool, DB pool     |
|                    | Abnormal traffic               | Unnecessary requests consume resources        | Gateway/access logs, request patterns     |
| **Scaling**        | Too few instances              | Existing instances become overloaded          | Instance count, CPU, request distribution |
|                    | Autoscaling too slow           | Capacity doesn't increase fast enough         | Scaling events, CPU, request rate         |
|                    | Uneven load balancing          | Some instances become overloaded              | Per-instance traffic/latency              |
| **Cache**          | Cache miss                     | Request falls back to DB                      | Cache hit/miss ratio, DB load             |
|                    | Low cache hit rate             | More DB requests                              | Hit ratio, DB query rate                  |
|                    | Cache stampede                 | Many requests hit DB together                 | Cache expiry, DB traffic spike            |
|                    | Cache unavailable              | All traffic goes to DB                        | Cache health, DB load                     |
| **Gateway / LB**   | Gateway delay                  | Request spends time before reaching service   | Gateway latency                           |
|                    | Load balancer delay            | Request takes longer to reach instance        | LB metrics                                |
|                    | Rate limiting                  | Requests are delayed/throttled                | Gateway rate-limit metrics                |
|                    | Uneven routing                 | One instance receives excessive traffic       | Per-instance metrics                      |
| **Deployment**     | New inefficient code           | Processing time increases                     | Deployment comparison, APM                |
|                    | New DB query                   | DB latency increases                          | SQL logs, query metrics                   |
|                    | New downstream call            | Additional waiting                            | Distributed tracing                       |
|                    | Configuration change           | Pool/timeout/cache behavior changes           | Config diff, deployment history           |
| **Infrastructure** | Server resource exhaustion     | Application can't process efficiently         | CPU, memory, disk                         |
|                    | Disk I/O bottleneck            | Operations wait for disk                      | Disk I/O, latency                         |
|                    | Container CPU throttling       | Application gets less CPU                     | Container metrics                         |
|                    | Infrastructure/network issue   | Components communicate slowly                 | Infrastructure + network metrics          |


1. First thing I will notice here is all montitoring metrics and try to find out what exactly the issue might

I would look into no Of traffics, p95,99 latencies, cpu metricses, error metricses threadpool metrices, db latency and db base metrices like db connection, db pool etc
I would compare these metrices issue timeline with normla time line

Say if the traffic metrics is high and latencies are high and error rates are high

---> It can be becuase of high traffic
----> next I will find why the traffic has increased 

    ----> Retry Stroms
   -----> Normal traffic raise ike big billion days
    -----> hacker forcing multiple request
   
So I wil be chceking client based traffic metrics like a particular user has overloaded traffic or evreyone has
If only 1 user has it might be hacking and i will implement rate limiting algo and try to prevent thses issue

If it is form diff people but trying at regualr intervals it might be retry stroms
  ---> will look into distributed tracing where the error comes and fix it and also improe=ve retyr algo withe xponential backoff and jitters

THen i wil chcek threadpool metrics like how many reuest is in pool and how many are active ---> this can help usnarrow down to
genuine traffic not able to handle and i will try to =scale horizontly and verify


Then I woud check db related metrices and try to find wethere we have db ltency and look into those metrices

say i would check query latency metrics if ofund i will go to distributed tracing and look into
exactly which wuery in that endpoint is causing the issue after finding It will check why it has hapeens

---> say it can be querying a large set of data without index 
     +===> I will be using EXPLAIN query and check if index is appropriately used or not 
  if not add appropriate indexes without write overheads

====> If index is there but the query ioprtimizer is ot picking i wil refresh the satistic data and check

====> NExt I would check the no Of requests and no od db calls to ensure we haev n+1 prob

If yes i will fix that using join fetch or entity graph

I will alsioc hekc pod related metrices like ony one pod is reciing high traffi cortheers are not then i will use proper load balancing

If the sql itself is tkaing less time but while converting into result set it is tkaing more tim eI wi implment paginations

if it is tkig more time which means the data might have grown expoentially high
so i will archive hold data ot=r do partiitoning

Check db cpu if high can be because of query optimixzation needed or also more transactioal based locking might happen ansd son