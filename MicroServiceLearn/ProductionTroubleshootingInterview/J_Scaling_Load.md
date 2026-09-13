# Production Troubleshooting Study Chapter: Scaling and Load

## Purpose

This chapter explains how demand becomes concurrent work, how finite resources saturate, and how to scale a microservices system without moving or amplifying the failure. It starts with throughput, latency, Little's Law, and queues, then develops production workflows for application replicas, databases, Kafka, load balancers, autoscaling, backpressure, and realistic load tests.

Safety comes first: make reversible changes, preserve evidence, respect dependency capacity, and prefer graceful load shedding over an uncontrolled collapse. Do not run an unapproved load test against production.

## Learning goals

After studying this chapter, you should be able to:

1. Relate throughput, concurrency, latency, utilization, saturation, and queue growth.
2. Use Little's Law without treating it as a capacity formula.
3. Distinguish vertical scaling, horizontal scaling, and optimization.
4. Explain why adding replicas may move a bottleneck to a database, broker, network, or shared pool.
5. Diagnose autoscaling lag, uneven traffic, hot partitions, pool exhaustion, and retry amplification.
6. Design stateless services, bounded queues, backpressure, rate limits, and overload protection.
7. Test capacity without coordinated omission or an unrealistic workload.
8. turn incident evidence into a capacity model, scaling policy, alert, and permanent design change.

---

# 1. Mental model

## 1.1 Demand, completions, and in-flight work

- **Offered load** is work callers try to submit, including requests later rejected or timed out.
- **Throughput** is completed useful work per unit time, such as successful requests per second.
- **Concurrency** is work currently in progress or waiting inside the measured boundary.
- **Latency** is elapsed time for one operation. Always state the boundary and percentile.
- **Capacity** is the highest sustainable useful throughput that still meets latency and error objectives.
- **Saturation** begins when a constrained resource cannot accept more work without queueing, rejection, or loss.

Little's Law applies to a stable system over a consistent boundary:

```text
average concurrency = average throughput * average time in system
L = lambda * W

Example:
200 requests/second * 0.250 seconds = 50 requests in flight
200 requests/second * 4 seconds = 800 requests in flight
```

The second case may consume 16 times as many sockets, request objects, threads, and downstream connections even though arrival rate did not change. Little's Law describes observed averages; it does not say the system can safely support the calculated concurrency, identify the bottleneck, or model bursty and unstable periods.

## 1.2 The bottleneck controls throughput

A request path is a series of finite resources:

```text
client -> DNS/TLS -> load balancer -> application worker
       -> DB pool -> database CPU/I/O/locks
       -> cache or downstream service
       -> Kafka producer/broker/consumer
```

The resource with the lowest effective service capacity constrains the path. Scaling that resource can increase throughput until another resource becomes limiting. This is **bottleneck movement**, not failure of scaling.

Useful clues:

| Observation | Likely interpretation |
|---|---|
| Arrival and completion rates rise; latency remains stable | Headroom remains |
| Arrival rises; completion flattens; queue and latency rise | Saturation |
| CPU rises linearly with throughput | CPU demand tracks work |
| CPU is low; pool wait is high | Waiting on a finite dependency or lock |
| More pods; per-pod load falls; total throughput unchanged | Shared bottleneck |
| More pods; downstream attempts rise faster than successes | Retry or fan-out amplification |
| One shard/partition is full while fleet average is low | Skewed bottleneck |

Fleet averages can hide one hot pod, zone, tenant, key, partition, or route. Capacity must be assessed at the smallest place where work can queue.

## 1.3 Vertical and horizontal scaling

**Vertical scaling** gives one instance more CPU, memory, I/O, or a larger machine. It is useful for single-threaded phases, large heaps, per-instance caches, and fast emergency headroom. Limits include machine size, restart/migration, larger failure blast radius, and no automatic redundancy.

**Horizontal scaling** adds instances or workers. It improves capacity only when work can be divided and dependencies have headroom. It needs:

- Stateless request processing, or externalized/partitioned state.
- Traffic distribution that reaches new instances.
- Independent enough resources rather than one shared lock or leader.
- Correct connection and thread budgets across the whole fleet.
- Idempotency or deduplication where retries and redelivery occur.

Optimization reduces resource demand per unit of useful work. It may be safer and cheaper than scaling, but an incident may require immediate capacity first.

## 1.4 Statelessness is a scaling property

A stateless replica does not require a later request to reach the same process. Session state, workflow ownership, files, locks, and cursor state must be in an appropriate shared or partitioned store, carried in a validated request, or routed deliberately.

"No local database" does not prove statelessness. In-memory sessions, local caches that affect correctness, scheduled jobs, WebSocket ownership, and singleton leaders are state. Sticky sessions can preserve behavior but cause uneven load and make failover harder.

## 1.5 Pools multiply across replicas

Pool size is a per-instance number, but dependencies see the fleet total:

```text
20 pods * 30 DB connections = up to 600 DB connections
40 pods * 30 DB connections = up to 1,200 DB connections
```

Connections, consumer fetches, HTTP sockets, thread stacks, caches, and retry budgets all multiply. A bigger application fleet can overwhelm a database with connection scheduling, lock contention, buffer churn, duplicate work, and less effective caching.

## 1.6 Autoscaling is delayed feedback

An autoscaler observes a signal, waits for collection and evaluation, decides, schedules instances, starts processes, passes readiness, warms caches/connections/JIT, and only then serves traffic.

```text
response delay =
metric delay + evaluation delay + scheduling delay +
image/startup delay + readiness delay + warm-up delay
```

CPU is useful for CPU-bound work, but weak for I/O-bound saturation. Better signals can include in-flight work per ready replica, queue age, backlog per consumer, pool wait, or a carefully designed concurrency metric. Scaling on latency alone can react after queues are already dangerous. Use minimum capacity, predictive/scheduled scaling for known peaks, safe stabilization windows, and dependency-aware maximums.

## 1.7 Backpressure prevents unbounded work

Backpressure makes overload visible to producers instead of hiding it in memory:

- Bounded queues reject or delay excess work.
- Rate limits enforce per-tenant or global budgets.
- Admission control reserves capacity for critical operations.
- Load shedding returns a quick explicit response rather than timing out later.
- Retry budgets, exponential backoff, and jitter prevent synchronized retries.
- Deadlines and cancellation stop useless work.
- Kafka producer throttling or paused consumption controls downstream pressure.

Failing a bounded fraction quickly can preserve useful throughput. An unbounded queue converts overload into high latency, memory exhaustion, and broad failure.

## 1.8 Glossary

| Term | Meaning |
|---|---|
| RPS | Requests per second; specify offered, attempted, completed, or successful |
| Service demand | Resource time needed for one completed unit of work |
| Utilization | Fraction of a resource's available capacity in use |
| Saturation | Demand at or beyond useful capacity, visible as queueing/rejection |
| Headroom | Capacity between normal peak and a safe operating limit |
| Little's Law | Average in-system work equals throughput times average time in system |
| Bottleneck | Resource currently limiting end-to-end useful throughput |
| Backlog | Accepted work not yet completed |
| Backpressure | Feedback that slows or rejects producers when consumers cannot keep up |
| Load shedding | Deliberate rejection of excess or low-priority work |
| Bulkhead | Resource isolation that limits one workload's blast radius |
| Retry amplification | One logical request generates multiple physical attempts |
| Warm-up | Startup cost before an instance reaches representative performance |
| Hot partition | One partition receives disproportionate work |
| Coordinated omission | Load generator pauses arrivals during stalls and underreports latency |
| Open-loop test | Sends arrivals independently of response completion |
| Closed-loop test | Each virtual user usually waits for a response before sending the next request |

---

# 2. Metrics and evidence

## 2.1 Minimum evidence set

| Signal | Why it matters | Explicit interpretation |
|---|---|---|
| Offered, accepted, completed, successful rate | Separates demand from useful output | Offered rises while completed flattens means a limit or loss |
| p50/p95/p99 latency histogram | Shows typical and tail wait | Rising tail with stable service time suggests queueing or a subset |
| In-flight work | Connects rate and elapsed time | Rising with flat throughput means accumulating work |
| Queue depth and oldest age | Measures pending work | Age is often more actionable than count across variable job sizes |
| Rejections, 429, 503, timeouts, cancellations | Shows overload behavior | Quick bounded rejection differs from late timeout |
| CPU, throttling, run queue | Finds compute constraints | High run queue/throttling plus flat throughput supports CPU saturation |
| Memory, allocation, GC, restarts | Finds concurrency cost | Rising in-flight work can drive memory pressure without a leak |
| Thread/event-loop queue | Finds execution admission wait | Busy=max plus queue growth indicates a worker limit |
| DB/HTTP pool active, max, pending, wait | Finds finite client resources | Pending and wait rising at max proves pool contention, not its cause |
| Dependency latency/errors/rate | Finds moved bottleneck and amplification | Compare attempts per inbound request and dependency headroom |
| Replica desired/ready/serving | Separates scale decision from capacity | Desired rising without ready means provisioning/startup lag |
| Per-instance request share | Finds imbalance | Compare counts normalized by ready time and capacity |
| Kafka lag and oldest record age | Measures consumer debt | Growing lag while input exceeds output means consumers are behind |
| Broker/partition throughput and throttles | Finds broker or partition limit | A hot partition can hide behind normal cluster average |

Break down safely by route, status, version, zone, pod, tenant class, and partition. Avoid unbounded user IDs or request IDs as metric labels; use traces or controlled logs for high-cardinality detail.

## 2.2 Evidence hierarchy

1. **Timeline:** exact onset, duration, peak, and recovery.
2. **Change and demand overlays:** deployments, config, scaling events, traffic, batch jobs, dependency incidents.
3. **End-to-end RED:** rate, errors, duration at every service boundary.
4. **Resource USE:** utilization, saturation, errors for CPU, memory, pools, disks, network, DB, and broker.
5. **Distributed traces:** compare healthy and slow paths, including queue/pool spans and retries.
6. **Runtime evidence:** profiles, thread dumps, query plans, broker diagnostics, and load-balancer statistics.

Correlation narrows hypotheses. A cause needs a mechanism and evidence at the constrained resource.

---

# 3. Generic scaling and load workflow

## Step 1 - Define the workload and impact

Record the first bad time, route/event type, offered and successful rate, p50/p95/p99, error classes, in-flight work, tenants/regions, versions, and recent changes. This prevents comparing different traffic shapes or investigating outside the incident window.

## Step 2 - Stabilize without multiplying load

Stop a harmful rollout, cap retries, shed low-priority traffic, enforce rate limits, pause nonessential producers, or add proven safe capacity. Preserve dashboards, traces, scaling events, and profiles. Do not blindly enlarge every pool or timeout.

## Step 3 - Draw the request or event path

List each admission point, queue, worker, pool, dependency, shard, and asynchronous handoff. Mark where offered rate, completion rate, latency, queue age, utilization, and rejection can be measured. This turns "the system is overloaded" into testable stages.

## Step 4 - Find the first place arrivals exceed completions

Move from edge to dependency. The first sustained queue growth or rejection usually identifies where pressure starts. A downstream symptom may be caused by excess attempts upstream, so compare logical operations with physical calls.

## Step 5 - Classify the constraint

- CPU: run queue/throttling and CPU per request.
- Worker or event loop: busy, queued, blocked stacks, loop lag.
- Pool: active=max, pending, acquisition time, hold time.
- Database: query/lock/I/O/CPU/connection evidence.
- Kafka: per-partition input/output, lag age, rebalance and processing time.
- Network: connection errors, RTT, retransmits, bandwidth.
- Skew: per-pod, key, tenant, shard, or partition distribution.
- External quota: explicit throttle responses and quota telemetry.

## Step 6 - Test one mechanism

Use a controlled canary, profile, query plan, queue sample, or representative load test. Predict the result before changing anything. For example: "If DB pool wait is caused by slow query X, reducing X execution time should lower connection hold time, pending borrowers, and API p99 without increasing pool size."

## Step 7 - Mitigate, verify, and watch movement

After a change, verify useful throughput, latency, errors, queues, dependency utilization, and cost. Watch for a new bottleneck. Roll back if the predicted signals do not improve.

## Step 8 - Make it durable

Fix service demand or partitioning, set explicit budgets and bounds, model capacity, update autoscaling, add overload tests, document safe operating limits, and alert before the SLO is exhausted.

---

# 4. Original scaling/load questions

## 1. Traffic suddenly increases 10×. What happens to your microservices system?

### Meaning and what it does not prove

This asks how a distributed system behaves as offered load exceeds one or more capacities. A 10x traffic graph does not prove every service receives 10x work: caching, routing, fan-out, retries, filtering, and asynchronous buffering transform load. High CPU alone does not prove CPU is the first bottleneck.

### Issue locations

Edge rate limits and load balancers; TLS and connection limits; gateway workers; application CPU, memory, threads, and pools; cache; databases; downstream APIs and quotas; Kafka brokers, partitions, and consumers; network and observability pipelines.

### Causal mechanisms and example

Below capacity, completions roughly track arrivals. Near saturation, a finite resource becomes busy, wait grows nonlinearly, callers retain more concurrent requests, and timeouts trigger retries. Those retries add work. Queues consume memory, health probes time out, instances restart, caches cool, and the failure cascades.

Example: traffic rises from 500 to 5,000 RPS. The gateway distributes it, but each request makes three DB calls. The DB can sustain 4,000 calls/s, so connection hold time rises. Application DB pools fill, in-flight requests grow by Little's Law, callers retry, and the DB sees more than 15,000 attempts/s. Adding application pods adds connections but not DB execution capacity.

### Ordered investigation and why

1. Confirm offered versus accepted and successful rate; this separates demand from dashboard artifacts and rejection.
2. Scope impact by route, tenant, region, version, and pod; one expensive route may dominate.
3. Compare latency, errors, in-flight work, queue age, and completions; this identifies unstable accumulation.
4. Walk the path to the first saturated queue/resource; downstream distress alone may be secondary.
5. Measure retries and fan-out per logical request; amplification can exceed the original 10x.
6. Inspect autoscaler desired, pending, ready, and serving times; requested replicas are not capacity.
7. Verify DB, cache, broker, and external quota headroom before scaling the application.

### Tools, metrics, evidence, and interpretation

Use gateway/LB request counters, service histograms, traces, Kubernetes replica/events data, CPU throttling, pool pending time, DB active sessions/locks, Redis latency/evictions, and Kafka lag age. Arrival up 10x, completion flat, and one queue's age rising identifies a capacity boundary. CPU at 40 percent does not exclude a full DB pool. Desired replicas at 30 but ready at 10 identifies provisioning lag.

### Immediate mitigation

Apply fair rate limits, shed optional work, disable nonessential fan-out, cap retries, protect critical tenants with bulkheads, pause batch traffic, and scale only a demonstrated parallel bottleneck within dependency budgets. Pre-warmed standby capacity may help.

### Permanent fix/design

Create a demand and dependency capacity model, optimize expensive paths, use bounded queues and deadlines, partition hot workloads, make services horizontally scalable, reserve headroom, pre-scale predictable peaks, and negotiate quotas.

### Prevention and alerts

Load-test peak plus failure scenarios. Alert on error-budget burn, queue age, rejections, in-flight work, pool wait, retry ratio, dependency saturation, autoscaling lag, and ready capacity versus forecast demand.

### Common mistakes

- Looking only at average CPU or average latency.
- Treating accepted or completed RPS as offered traffic.
- Scaling all tiers without a connection and dependency budget.
- Increasing timeouts and queues, which retains more doomed work.
- Letting every layer retry.

### Interview-ready answer

I would expect one finite resource to saturate first, after which queueing, latency, concurrency, timeouts, and retries can cascade. I would compare offered and completed rates, find the first growing queue or saturated resource across the path, control retries and excess traffic, then scale or optimize that proven constraint while watching dependency capacity and bottleneck movement.

## 2. One service cannot handle increased traffic. How would you scale it?

### Meaning and what it does not prove

The task is to choose a scaling method from evidence. "Cannot handle" must be defined as SLO failure, saturation, queue growth, or rejection. It does not prove more replicas are correct; a shared database, lock, leader, partition, quota, or inefficient algorithm may dominate.

### Issue locations

Instance CPU/memory, event loop, workers, local state, session routing, DB/HTTP pools, singleton jobs, shared locks, load balancer registration, startup/warm-up, and dependencies.

### Causal mechanisms and example

If each stateless pod can sustainably complete 200 RPS, five similarly loaded pods give roughly 1,000 RPS before shared constraints. A CPU-bound service may scale horizontally or vertically. A service blocked on one database query will mostly create more waiting calls.

Example: image conversion is CPU-bound at one core per worker. Adding CPU and workers vertically helps quickly; independent pods improve capacity and resilience. In contrast, an order service with a global serializable transaction cannot gain linear throughput from pods until contention is redesigned.

### Ordered investigation and why

1. Define the SLO and measure offered/completed rate, service demand, and saturation.
2. Identify the constrained resource; scaling the wrong dimension wastes capacity.
3. Check statelessness, affinity, singleton behavior, and partitionability.
4. Calculate fleet-wide DB connections, downstream calls, broker traffic, and quota at the proposed scale.
5. Benchmark one instance at representative data and traffic; derive safe per-instance capacity.
6. Choose vertical, horizontal, partitioned, or optimization changes and canary them.
7. Confirm load reaches new capacity and that the bottleneck does not move dangerously.

### Tools, metrics, evidence, and interpretation

Use CPU profiles, run queue/throttling, worker and pool metrics, traces, LB target counts, HPA events, DB wait events, and load tests. Throughput increasing proportionally while per-pod latency stays stable supports horizontal scaling. Replica count rising with unchanged total throughput and unchanged shared dependency ceiling refutes it.

### Immediate mitigation

Scale a proven safe resource, reduce optional work, cache safe reads, throttle expensive tenants, pause batch operations, or allocate a larger instance. Keep queues bounded and verify readiness before routing traffic.

### Permanent fix/design

Externalize correctness-critical state, partition work, remove serial sections, right-size pools, optimize service demand, build autoscaling around a leading saturation signal, and define min/max capacity from dependency budgets.

### Prevention and alerts

Maintain per-instance capacity tests and scaling runbooks. Alert on saturation, queue age, ready replicas, scaling ceiling, provisioning failures, and dependency utilization.

### Common mistakes

- Assuming Kubernetes replica count equals serving capacity.
- Scaling from CPU when the service waits on I/O.
- ignoring startup, JIT, connection, and cache warm-up.
- copying a large per-pod pool across many replicas.
- claiming linear scaling without measuring it.

### Interview-ready answer

I first prove the constrained resource and per-instance sustainable capacity. For parallel stateless work I add replicas; for per-instance CPU or memory limits I may scale vertically; for a hot shard or serial section I partition or redesign it. Before scaling I budget shared connections and dependency load, then verify useful throughput, latency, distribution, and the new bottleneck.

## 3. You horizontally scale a service, but performance does not improve. Why?

### Meaning and what it does not prove

More replicas without improved useful throughput means either traffic did not use them, the application cannot parallelize, another resource limits it, or added overhead canceled the gain. It does not by itself mean horizontal scaling never works.

### Issue locations

Load balancer discovery/readiness, sticky sessions, clients with long-lived connections, service mesh, hot keys/shards, global locks, leader-only work, DB/cache/broker, per-node limits, external quota, and autoscaler warm-up.

### Causal mechanisms and example

Amdahl's Law says a serial fraction limits parallel speedup. Shared bottlenecks impose another ceiling. New pods may receive no traffic because clients reuse a few HTTP/2 connections or session affinity pins users.

Example: ten pods become twenty, but all execute the same locked account update. DB lock wait doubles and completed transactions remain 800/s. Application CPU falls, yet p99 rises.

### Ordered investigation and why

1. Verify new replicas are ready and serving; desired/running is insufficient.
2. Compare per-pod request counts, connection counts, version, and latency; this exposes routing skew.
3. Compare total offered, completed, and successful throughput; latency alone can mislead.
4. Inspect app worker/pool saturation and serial profiles.
5. Inspect shared dependencies and attempts per request; find the moved ceiling.
6. Check data/partition skew and external quotas.
7. Run a controlled scale curve at 1, 2, 4, and 8 replicas; shape reveals diminishing returns.

### Tools, metrics, evidence, and interpretation

Use endpoint discovery, LB target statistics, per-pod counters, connection telemetry, traces, lock profiles, DB wait events, Kafka per-partition metrics, and a controlled load test. Equal traffic with flat throughput and rising DB wait points to DB contention. Idle new pods point to routing or affinity. CPU falling per pod while throughput is fixed indicates a non-CPU shared limit.

### Immediate mitigation

Undo replicas if they worsen dependency load, rebalance or recycle client connections carefully, route around unhealthy targets, cap concurrency, throttle hot workloads, and mitigate the actual dependency.

### Permanent fix/design

Remove serial/global coordination, partition data and keys, design clients for discovery and connection rotation, enforce fleet-wide concurrency budgets, and test scaling efficiency.

### Prevention and alerts

Track throughput per ready replica, coefficient of load variation, dependency wait, connection totals, and scale efficiency. Alert when replicas rise but completed throughput does not.

### Common mistakes

- Measuring pod count rather than serving requests.
- Ignoring sticky sessions and long-lived connections.
- increasing DB pools to make application wait disappear.
- expecting linear scale indefinitely.

### Interview-ready answer

I would verify that ready replicas actually receive balanced traffic, then compare total successful throughput with per-pod saturation. If traffic is balanced, I look for serial work, hot partitions, shared DB/cache/broker limits, quotas, and fleet-wide pool growth. A scale curve and dependency evidence distinguish routing failure from a moved bottleneck.

## 4. Traffic is unevenly distributed between service instances. What could be wrong?

### Meaning and what it does not prove

Uneven distribution means normalized work per healthy instance differs materially. Raw request count does not prove imbalance unless adjusted for ready time, instance capacity, route cost, and long-lived work.

### Issue locations

LB algorithm and health, endpoint propagation, readiness, zone locality, sticky cookies, source-IP hashing, consistent hashing, HTTP keep-alive/2/connection pooling, gRPC streams, service mesh, DNS caching, client-side discovery, hot keys, and heterogeneous instance resources.

### Causal mechanisms and example

Round-robin usually balances new connections, not individual requests. With ten long-lived HTTP/2 connections created when only two pods existed, adding eight pods may leave most streams on the old pods. Hashing can also map a high-volume tenant to one instance.

### Ordered investigation and why

1. Define imbalance using request count, concurrent work, CPU, and weighted cost per serving minute.
2. Verify health/readiness and endpoint membership from both control plane and clients.
3. Determine whether balancing occurs per request, connection, source, session, or key.
4. Inspect connection age/count, affinity, DNS TTL, locality, and mesh policy.
5. Split by route, tenant class, and key/partition to find workload skew.
6. Compare instance versions, CPU limits, throttling, and warm-up.
7. Change one routing factor in a canary and verify redistribution without dropping sessions.

### Tools, metrics, evidence, and interpretation

Use LB target request/connection metrics, service endpoint lists, mesh proxy stats, per-pod route histograms, and connection-age data. Equal connection counts but very different request counts can be multiplexing or heavy clients. A pod absent from client endpoint views is discovery propagation, not random chance.

### Immediate mitigation

Drain a hot target safely, remove unhealthy endpoints, correct readiness, add connection rotation where supported, disable unintended affinity through a reviewed change, or isolate a hot tenant. Do not terminate stateful sessions blindly.

### Permanent fix/design

Choose an algorithm matching request cost, support connection rebalancing, shard high-volume tenants, avoid correctness dependence on affinity, use topology policies intentionally, and normalize capacity weights for heterogeneous instances.

### Prevention and alerts

Alert on per-pod load coefficient of variation, hot target saturation, endpoint divergence, and old connections after scale-out. Test scale-out with realistic persistent connections.

### Common mistakes

- Comparing counts without ready duration.
- Assuming round-robin means per-request fairness.
- restarting pods until the graph looks even.
- removing stickiness before externalizing session state.

### Interview-ready answer

I normalize work by serving time and capacity, then verify endpoint health and ask what unit the balancer distributes: requests, connections, sessions, sources, or keys. I inspect long-lived connections, affinity, DNS/discovery, zone policy, and hot tenants. I mitigate safely, then fix routing and state design and alert on per-instance skew.

## 5. Adding more application instances makes the database slower. Why?

### Meaning and what it does not prove

This is usually fleet concurrency exceeding database capacity or increasing contention. It does not prove the database needs a larger machine; inefficient queries, retry amplification, locks, or an oversized aggregate connection budget may be the cause.

### Issue locations

Per-pod DB pool settings, transaction scope, query plans, indexes, lock hot spots, DB connection/process limits, CPU, I/O, buffer cache, replicas, proxies, network, and application retries.

### Causal mechanisms and example

More pods multiply concurrent queries. Past the DB's useful concurrency, tasks compete for CPU, buffers, I/O, locks, and connection scheduling. Each query takes longer, holds its connection longer, and increases application pool occupancy. Positive feedback follows.

Example: 10 pods with 20 connections allow 200 sessions; scaling to 40 allows 800. The DB has 16 cores and a hot row. Context switching and row-lock queues grow, transaction latency quadruples, and timeouts trigger duplicate retries.

### Ordered investigation and why

1. Correlate replica and total connection changes with DB latency; establish timing.
2. Compare logical requests, SQL calls, and retries; detect amplification.
3. Inspect pool active/pending/hold time per pod and fleet total.
4. Use DB wait categories, CPU, I/O, locks, query latency, and plans to identify the mechanism.
5. Rank queries by total time, not only slowest single execution.
6. Check transaction duration and idle-in-transaction sessions.
7. Model a safe global concurrency budget and test it gradually.

### Tools, metrics, evidence, and interpretation

Use pool metrics, DB activity/wait views, slow-query or query-store statistics, lock graphs, execution plans, CPU/I/O telemetry, and traces. Connections rising with lock wait and flat transactions/s supports contention. High pool pending with low DB activity may instead indicate leaked or long-held connections.

### Immediate mitigation

Cap aggregate DB concurrency, reduce pod pool sizes deliberately, disable duplicate retries, throttle expensive routes, shorten or cancel noncritical work, and roll back excessive scaling. Kill sessions only through approved DB procedure with impact understood.

### Permanent fix/design

Optimize queries/indexes, shorten transactions, eliminate hot-row coordination, cache safe reads, batch writes, use replicas only for appropriate consistency needs, and enforce a fleet-wide DB concurrency budget through a proxy or adaptive limiter.

### Prevention and alerts

Alert on total connections, lock wait, query total time, pool wait, retries, and DB saturation. Include scale-out in load tests and document the maximum safe replica/pool combination.

### Common mistakes

- increasing both replicas and per-pod pool size.
- treating connection count as throughput.
- moving reads to replicas without considering lag and read-after-write.
- optimizing one slow query while ignoring a frequent query's total load.

### Interview-ready answer

Application scale multiplies DB connections and calls. Once useful DB concurrency is exceeded, CPU, I/O, locks, and cache contention make every query slower and hold connections longer. I correlate replica and connection growth, inspect DB waits and top total-time queries, cap global concurrency and retries, then fix queries, transactions, hot data, and pool budgets.

## 6. Kafka consumers are unable to keep up with increasing traffic. What would you do?

### Meaning and what it does not prove

Consumers are behind when production rate persistently exceeds completion rate, visible as growing lag or record age. Lag count alone does not prove impact: record cost varies, compacted topics and burst recovery differ, and a stopped producer can make lag flat while old records remain.

### Issue locations

Producer rate and batching, broker and network, topic partition count/skew, consumer group membership/rebalances, fetch settings, poll loop, deserialization, processing, downstream DB/API, offset commits, retries, poison records, and GC.

### Causal mechanisms and example

Within one consumer group, a partition is assigned to at most one consumer at a time. Consumer instances beyond partition count are idle. A hot partition limits throughput to one partition consumer. Slow synchronous processing can exceed the poll interval, cause rebalances, replay, and more work.

Example: a topic has six partitions and twelve consumers. One customer produces 60 percent of events to one keyed partition. That partition grows lag while six consumers are idle. Adding consumers cannot divide that partition.

### Ordered investigation and why

1. Measure input rate, completed processing rate, lag count, and oldest age by partition.
2. Determine whether lag is global or skewed; this decides scale versus repartition.
3. Check active members, assignments, rebalances, poll interval violations, and commit failures.
4. Measure consume, deserialize, business processing, downstream, retry, and commit time.
5. Inspect broker throttle, disk/network, fetch latency, and under-replicated partitions.
6. Validate processing semantics, ordering, idempotency, and poison-record handling before parallelizing.
7. Estimate drain time: backlog divided by spare completion rate.

### Tools, metrics, evidence, and interpretation

Use consumer-group describe output, per-partition broker metrics, client fetch/poll/commit metrics, traces, downstream pool data, and profiles. Rising lag only on one partition proves skew. Frequent rebalances plus max-poll breaches implicate processing/poll design. Broker throttle affects multiple groups and producers.

### Immediate mitigation

Scale consumers only up to useful partitions, pause noncritical producers, rate-limit intake, isolate poison records to a controlled dead-letter path, reduce safe processing cost, or temporarily increase downstream capacity. Preserve ordering and idempotency guarantees.

### Permanent fix/design

Choose partition keys for evenness and ordering needs, increase partitions with compatibility review, decouple polling from bounded processing safely, batch where valid, make handlers idempotent, add retry topics with backoff, and capacity-plan broker plus downstreams.

### Prevention and alerts

Alert on oldest-record age, lag growth rate, rebalance rate, processing p99, commit failures, partition skew, broker throttle, and projected time to SLO breach. Load-test replay and downstream degradation.

### Common mistakes

- Adding consumers beyond partition count.
- looking only at total lag.
- increasing `max.poll.interval` to hide slow work.
- committing before durable processing without accepting loss semantics.
- retrying poison records in a tight loop.

### Interview-ready answer

I compare production and completion rates plus oldest lag by partition, then check assignment, skew, rebalances, processing stages, downstream waits, and broker health. I scale only within partition parallelism, control producers and retries, and protect ordering and idempotency. Long term I fix partitioning, handler cost, bounded concurrency, retry design, and capacity alerts.

## 7. How would you identify the bottleneck in a microservices system under heavy load?

### Meaning and what it does not prove

The bottleneck is the resource whose constrained capacity currently limits useful end-to-end throughput. The hottest-looking metric is not necessarily it: 100 percent cache CPU may be harmless if requests are not waiting, while a low-CPU service can be blocked on a pool.

### Issue locations

Every queue and finite resource from edge through services, runtime, pools, cache, DB, broker, network, and external providers, including individual shards and partitions.

### Causal mechanisms and example

At the bottleneck, utilization approaches useful capacity, its queue or rejection rises, and total throughput stops scaling. Upstream queues are consequences; downstream resources may be underfed. Removing it moves the knee of the load curve until another resource limits.

Example: API p99 rises at 1,200 RPS. App CPU is 55 percent, DB CPU is 65 percent, but HTTP client pool pending time rises from 0 to 900 ms and downstream completions flatten. Increasing the client pool briefly increases downstream overload; the real boundary is downstream capacity.

### Ordered investigation and why

1. Define useful throughput and the violated SLO.
2. Align all evidence to the same incident window and workload.
3. Draw the path and mark queues, rates, latency, utilization, and errors.
4. Find the first queue whose arrival exceeds departure or first hard rejection.
5. Decompose its service time versus wait time with traces and resource metrics.
6. Check per-instance, per-shard, and per-tenant distributions to avoid averages.
7. Form a falsifiable hypothesis and make a controlled change.
8. Verify the predicted improvement and watch where the bottleneck moves.

### Tools, metrics, evidence, and interpretation

Use RED/USE dashboards, distributed traces, continuous profiles, thread dumps, pool metrics, DB query/wait analysis, Kafka partition metrics, LB data, and controlled load curves. A utilization plateau alone is not proof. Queue growth plus flat departures and a matching wait mechanism is strong evidence. If reducing service demand by 30 percent moves maximum throughput by roughly the predicted amount, the hypothesis gains support.

### Immediate mitigation

Reduce arrivals or expensive work, cap retries, shed safely, add validated capacity to the actual constraint, and isolate critical traffic. Keep evidence and watch downstreams.

### Permanent fix/design

Reduce demand per operation, partition the constraint, set end-to-end concurrency budgets, design backpressure, improve observability at every queue, and maintain capacity tests.

### Prevention and alerts

Use SLO burn plus leading saturation alerts, queue age, completion/arrival divergence, pool wait, retry amplification, and per-shard skew. Rehearse overload and dependency failure.

### Common mistakes

- Selecting the highest percentage metric.
- using average latency and fleet averages.
- profiling only after restarting the bad instance.
- changing several limits at once.
- declaring success when one queue moved elsewhere.

### Interview-ready answer

I define useful throughput and the SLO, map every queue and finite resource, and find the first place arrivals exceed completions or rejections begin. I decompose wait from service time using traces and resource evidence, check skew, then test one causal hypothesis. After mitigation I verify throughput and latency and look for bottleneck movement.

---

# 5. High-value additional questions

## 8. Why can an autoscaler react too late or make overload worse?

### Meaning and what it does not prove

Autoscaling is delayed feedback. A late response does not prove the threshold alone is wrong; metric delay, pending capacity, readiness, warm-up, traffic distribution, and dependency ceilings all matter.

### Issue locations

Metric pipeline, HPA/KEDA policy, cluster capacity, scheduler, image pull, startup/readiness, JVM/cache/connection warm-up, LB registration, scale-down policy, and dependencies.

### Causal mechanisms and example

A two-minute CPU window detects a burst after queues form. Pods need three more minutes to start and warm. Meanwhile retries amplify demand. New pods open hundreds of DB connections and worsen the constrained database. Scale-down can then remove capacity while backlog still exists.

### Ordered investigation and why

1. Reconstruct signal, decision, desired, scheduled, ready, serving, and recovered timestamps.
2. Validate that the signal leads the SLO failure and represents the bottleneck.
3. Check pending reasons and warm-up behavior.
4. Verify new pods receive traffic and dependencies have budget.
5. Review stabilization, min/max, target, and scale velocity against burst shape.
6. replay a controlled burst and dependency slowdown.

### Tools, metrics, evidence, and interpretation

Use autoscaler events, metric timestamps, scheduler events, readiness duration, per-pod request counts, startup profiles, and DB connection totals. Desired replicas rising before SLO breach but pending pods means supply lag; CPU staying low while queue age rises means the signal is wrong.

### Immediate mitigation

Raise safe minimum capacity or pre-scale a known event, free cluster capacity, shed load, and cap dependency concurrency. Avoid an unreviewed maximum increase.

### Permanent fix/design

Use a leading workload/saturation signal, short safe windows, warm pools, predictive scaling, tuned startup/readiness, and dependency-aware maximums.

### Prevention and alerts

Alert on time from desired to serving, pending pods, max-replica saturation, backlog per ready replica, and new-replica load share.

### Common mistakes

- scaling on a lagging average.
- counting unready replicas as capacity.
- aggressive scale-down with backlog.
- omitting dependency load from scale policy.

### Interview-ready answer

I treat autoscaling as delayed feedback and reconstruct each delay from metric observation to serving capacity. I verify the signal predicts the actual bottleneck, check pending and warm-up, traffic uptake, and dependency budget, then tune minimums, signal, velocity, and stabilization and test realistic bursts.

## 9. How do you design and interpret a trustworthy load test?

### Meaning and what it does not prove

A load test estimates behavior for a defined workload and environment. Passing one synthetic test does not prove production capacity for different data, cache state, dependencies, failures, or arrival patterns.

### Issue locations

Generator capacity and network, workload model, arrival process, data cardinality, authentication, payloads, cache state, test environment, dependency stubs, observability, and result aggregation.

### Causal mechanisms and example

In a closed-loop test, each virtual user waits for a response. When the server stalls, the generator sends fewer requests; omitted scheduled arrivals are never timed, so reported latency looks better. An open-loop or corrected-latency design preserves the intended arrival schedule and exposes queueing.

Example: 100 users wait for each response. At a 10-second stall, request rate collapses from 1,000 to 10 RPS and p99 appears moderate. Production callers continue at 1,000 RPS and create a large queue.

### Ordered investigation and why

1. Define SLO, offered-load profile, duration, and success criteria.
2. Reproduce route mix, payload/data distributions, think time, connections, identity, and cache conditions.
3. Choose open-loop arrivals or record intended schedule to correct coordinated omission.
4. prove the generator and network exceed target capacity.
5. Ramp to locate the knee, then hold long enough for GC, pools, autoscaling, and leaks.
6. inject dependency latency/failure and retry behavior.
7. capture server-side offered/completed rate, histograms, queues, and resources.
8. compare multiple runs and state limitations.

### Tools, metrics, evidence, and interpretation

Use a generator with constant-arrival support and HDR-style histograms, synchronized clocks, server metrics, traces, and profiles. Generator CPU saturation invalidates the test. Throughput plateau plus queue growth marks a capacity knee. Report non-2xx and timeouts, not only completed latency.

### Immediate mitigation

If a pre-release test fails, stop rollout and lower safe limits. During an approved resilience exercise, abort at guardrails to protect shared systems.

### Permanent fix/design

Version workload models, seed realistic non-sensitive data, automate repeatable tests, preserve baselines, include soak/burst/failure/recovery phases, and test overload controls.

### Prevention and alerts

Gate releases on agreed regression budgets and test generator health. Monitor environment drift and dependency stub assumptions.

### Common mistakes

- Coordinated omission.
- warming the cache when production starts cold, or vice versa.
- testing one endpoint and tiny data.
- using average client latency only.
- load-testing production without approval and guardrails.

### Interview-ready answer

I define the workload and SLO first, reproduce production route, data, connection, and cache behavior, and use an arrival model that does not hide stalls through coordinated omission. I verify generator headroom, ramp and soak, inject dependency failures, and interpret client results with server queues, resources, errors, and profiles.

## 10. How do backpressure and rate limiting differ, and where should they be applied?

### Meaning and what it does not prove

Backpressure is feedback from constrained consumers; rate limiting enforces an admission budget. A 429 proves a policy decision, not necessarily system saturation. A full queue proves demand exceeded that stage temporarily, not why service slowed.

### Issue locations

Gateway, per-service admission, worker queues, HTTP clients, DB concurrency limiters, Kafka producers/consumers, batch schedulers, and tenant boundaries.

### Causal mechanisms and example

Without bounds, a caller can occupy all workers while waiting on a slow dependency. With a bulkhead of 50 downstream operations and a queue of 100, excess requests fail quickly. A per-tenant token bucket prevents one tenant consuming all 150 positions.

### Ordered investigation and why

1. Identify scarce resources and their safe concurrency.
2. classify traffic priority, fairness, idempotency, and retry behavior.
3. place the limit before expensive work and near the protected resource.
4. bound queues and define explicit rejection responses and retry hints.
5. test steady overload, bursts, and recovery.
6. verify upstream respects the signal and does not retry aggressively.

### Tools, metrics, evidence, and interpretation

Track admitted/rejected rate by policy, queue age, limiter occupancy, useful throughput, and retry attempts. Stable useful throughput with bounded latency under overload shows protection works. Rising upstream attempts after 429 shows clients ignore backoff.

### Immediate mitigation

Tighten a reviewed concurrency limit, shed low-priority work, communicate retry-after guidance, and cap client retries.

### Permanent fix/design

Use hierarchical tenant/global budgets, bulkheads, deadlines, cancellation, adaptive concurrency where tested, and asynchronous queues with explicit retention and dead-letter policy.

### Prevention and alerts

Alert on sustained rejection, fairness violations, queue age, retry ratio, and limiter saturation; exercise recovery so rejected traffic does not return simultaneously.

### Common mistakes

- placing the limit after expensive work.
- unbounded "buffering."
- one global limit that starves critical traffic.
- returning retryable errors without jitter guidance.

### Interview-ready answer

Rate limiting controls admission; backpressure communicates downstream capacity. I place bounded concurrency and queues before the scarce resource, apply fair tenant and priority budgets, return explicit quick rejection, and ensure callers use bounded jittered retries. I verify that useful throughput stays stable and recovery is not a retry wave.

---

# 6. Decision trees

## 6.1 Latency rises under load

```text
Is offered rate actually higher?
|-- No -> check deployment, dependency, data shape, skew, and telemetry.
`-- Yes
    |
    Is successful completion rate still rising?
    |-- Yes -> capacity remains; inspect tail latency and approaching saturation.
    `-- No
        |
        Find first growing queue or rejection.
        |-- CPU/run queue/throttling -> optimize or add compute capacity.
        |-- worker/event-loop -> remove blocking, bound work, right-size.
        |-- pool pending -> inspect resource hold time and dependency.
        |-- DB waits -> fix query/transaction/contention; cap global concurrency.
        |-- Kafka lag -> inspect partitions, processing, rebalances, broker.
        `-- external quota -> shed, cache safely, batch, or negotiate capacity.
```

## 6.2 More replicas do not help

```text
Are new replicas ready and serving?
|-- No -> fix scheduling, startup, readiness, registration, or warm-up.
`-- Yes
    |
    Is traffic balanced by cost?
    |-- No -> inspect connections, affinity, discovery, locality, hot keys.
    `-- Yes
        |
        Is per-pod app saturation lower?
        |-- No -> inspect local serial work or per-node/shared resource.
        `-- Yes
            |
            Is total useful throughput flat?
            |-- Yes -> shared dependency, partition, quota, or serial limit.
            `-- No -> scaling helps; quantify efficiency and new safe ceiling.
```

## 6.3 Kafka lag grows

```text
Is lag age/count growing on all partitions?
|-- One/few -> hot key, slow partition, poison record, or broker leader issue.
`-- Most/all
    |
    Are active consumers fewer than useful partitions?
    |-- Yes -> scale consumers and verify assignments.
    `-- No
        |
        Processing time high?
        |-- downstream wait -> protect/fix dependency and bound concurrency.
        |-- CPU/deserialization -> optimize, batch, add partition parallelism.
        |-- rebalances -> fix poll/liveness/session behavior.
        `-- broker throttle/I/O -> address broker/network capacity.
```

---

# 7. Final cheat sheet

## Core equations and checks

```text
Little's Law: average in-flight = throughput * average latency
Fleet DB connection ceiling = replicas * per-replica pool maximum
Backlog growth rate = arrival rate - completion rate
Approximate drain time = backlog / (completion rate - new arrival rate)
Scale efficiency = throughput gain / resource or replica gain
```

Use only when units and measurement boundaries match. Drain time requires positive spare capacity and roughly stable rates.

## Fast evidence sequence

1. Time, scope, change, and SLO impact.
2. Offered, accepted, completed, and successful rate.
3. p50/p95/p99, in-flight work, queue depth, and oldest age.
4. First saturated resource or rejection from edge to dependency.
5. Retries/fan-out and per-pod/shard/partition skew.
6. Desired versus ready versus actually serving capacity.
7. Controlled mitigation, predicted result, and bottleneck movement.

## Production rules

- Scale the proven constraint, not the loudest dashboard.
- Budget pools and connections across the fleet.
- Keep queues and retries bounded.
- Reject excess work before spending scarce resources.
- Treat ready, warm, and receiving traffic as separate states.
- Averages hide hot instances and partitions.
- Test sustained, burst, skew, dependency-failure, and recovery load.
- Report offered load and all failures; avoid coordinated omission.
- App scaling can overload dependencies; verify their headroom first.
