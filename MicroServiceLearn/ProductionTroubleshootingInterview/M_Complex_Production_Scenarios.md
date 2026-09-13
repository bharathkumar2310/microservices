# M. Complex Real-Production Microservices Scenarios

## Purpose

The earlier chapters isolate one topic at a time. Real production incidents rarely stay inside one category. A latency incident can involve a deployment, database pool, retries, Kafka backlog, cache misses, and autoscaling at the same time.

This chapter teaches how to combine evidence across domains. Each answer follows:

```text
stabilize
  -> scope
  -> map the path
  -> build a timeline
  -> identify the constrained or failing component
  -> prove the mechanism
  -> correct
  -> verify business and technical recovery
  -> prevent recurrence
```

## Foundational concepts

### Symptom, trigger, root cause, and contributing factor

These terms are not interchangeable:

```text
Symptom:
  Users received 504 responses.

Trigger:
  A deployment increased calls to the pricing database.

Root cause:
  The new query had no supporting index and scanned the full table.

Contributing factors:
  Every application layer retried.
  Pool acquisition had no useful alert.
  The canary did not receive representative traffic.

Mitigation:
  Roll back the deployment and reduce retry attempts.

Permanent correction:
  Optimize/index the query, add a performance regression test,
  use one bounded retry layer, and alert on pool wait and DB p99.
```

### Availability is a business-flow property

A pod can be running, a gateway can return 200 for `/health`, and the business can still be unavailable. Always verify a safe representative transaction through the normal path.

### Capacity problems move

When one bottleneck is relieved, another can become visible:

```text
add application replicas
  -> more DB connections and queries
  -> database saturates
  -> overall latency becomes worse
```

### Distributed correctness survives uncertainty

A timeout means the caller does not know the outcome. It does not necessarily mean the operation failed. Writes, payments, and messages therefore require:

- Stable operation identity.
- Idempotency.
- Durable state transitions.
- Reconciliation.
- Safe retry rules.

---

# Scenario 1 - Latency Jumps, but CPU and Memory Look Normal

## Interview question

> An order API normally responds in 200 ms. Its p99 suddenly becomes 8 seconds, but Service A's CPU and memory are normal. How would you investigate?

## What the evidence means

Normal CPU and memory eliminate only some hypotheses. The service may spend most of its time waiting:

```text
request queue
DB connection pool
HTTP connection pool
database lock/query
downstream service
distributed lock
disk/network I/O
rate limiter
```

Waiting work can produce low CPU while users experience severe latency.

## Step-by-step investigation

### Step 1 - Confirm scope and latency shape

Check:

- p50, p95, p99, and max.
- Request rate and concurrent requests.
- Error/timeout rate.
- Endpoint, method, tenant, payload size, region, and instance.
- First-failure time and recent changes.

Interpretation:

- Only p99 rises: tail issue, one instance, one data shape, occasional lock, cache miss, or dependency outlier.
- All percentiles rise: common queue/dependency/systemic slowdown.
- Latency rises with concurrency but not raw RPS: requests are taking longer and accumulating.

### Step 2 - Follow slow traces

Compare several slow and fast traces:

```text
Fast:
A server span                     200 ms
  DB pool wait                      2 ms
  DB query                         40 ms
  B call                           70 ms

Slow:
A server span                    8.0 s
  DB pool wait                    6.9 s
  DB query                         45 ms
  B call                           80 ms
```

The query is not slow in this example. The service is waiting for a pool connection.

### Step 3 - Inspect saturation, not only utilization

Check:

```text
request queue depth
active/max worker threads
DB pool active/idle/pending/acquire time
HTTP pool leased/idle/pending
downstream concurrency limiter
DB locks/connections
network retransmissions
```

### Step 4 - Explain why the pool is occupied

Possible mechanisms:

- Queries became slower.
- Transactions hold connections during remote calls.
- Connections leak on an exception path.
- Traffic or concurrency increased.
- Pool size differs on one instance.
- Database reduced its connection limit.
- Retries multiply concurrent calls.

Use connection usage duration, transaction traces, leak detection, thread dumps, and DB session data to distinguish them.

### Step 5 - Mitigate safely

- Drain a single bad instance if correlation is proven.
- Roll back a triggering deployment.
- Reduce a retry storm.
- Rate-limit or disable a noncritical expensive path.
- Restore a failing dependency.

Do not blindly enlarge the pool. More connections can overload the database.

### Step 6 - Correct and verify

Fix the holding operation, query, transaction boundary, leak, dependency, or capacity issue. Verify latency percentiles, pool wait, active connections, queue depth, and business success under representative traffic.

## Interview-ready answer

> Normal CPU and memory suggest I should look for waiting and queueing rather than assume the service is healthy. I would compare latency percentiles and slow versus fast traces, then split the request into queue time, pool acquisition, code, database, downstream, and response time. I would inspect worker and connection-pool pending counts, database locks and query time, downstream latency, and per-instance differences. If a DB connection wait is seven seconds while the query is 45 ms, I would determine why connections are held, such as long transactions, leaks, retries, or traffic. I would fix that mechanism and verify p99 and saturation rather than increasing CPU or the timeout.

---

# Scenario 2 - A Deployment Causes Partial 5xx Failures

## Interview question

> A deployment completes successfully, but 30 percent of requests now return 500 or 503 while the remaining requests work. How would you investigate?

## Initial hypothesis

Partial failure after a rollout strongly suggests:

- Only new-version instances fail.
- One zone/node has a problem.
- A route sends some traffic to an incompatible version.
- Configuration or secret injection differs.
- A readiness check is too shallow.

Do not assume 30 percent means exactly 30 percent of pods without checking weights, stickiness, and traffic distribution.

## Step-by-step investigation

### Step 1 - Stop or pause the rollout

If the rollout is still progressing and user impact is rising, pause it through the normal deployment mechanism. Preserve old healthy capacity.

### Step 2 - Identify the error generator

Separate:

- Gateway-generated 503 because no healthy target exists.
- Service-generated 503 due to readiness/overload.
- Application-generated 500 due to an exception.

Capture request ID, route, upstream instance, version, image digest, node, and zone.

### Step 3 - Group errors by version and instance

Build a comparison:

| Version | Requests | Error rate | p99 |
|---|---:|---:|---:|
| old | 7,000 | 0.1% | 250 ms |
| new | 3,000 | 99% | 100 ms |

This gives strong evidence that the new version is defective or incompatible.

### Step 4 - Compare old and new

Check:

```text
image digest and commit
startup arguments and runtime
effective configuration
secret/certificate version
service account/workload identity
resource requests and limits
database schema compatibility
feature flags
downstream URLs
proxy and trust store
```

### Step 5 - Validate readiness and real traffic

The new pod may pass:

```text
/health -> 200
```

while a business route fails because:

- Migration is missing.
- Token scope changed.
- DB permission is absent.
- New configuration key is missing.
- Dependency contract is incompatible.

Use a safe business canary, not only liveness.

### Step 6 - Decide rollback versus roll-forward

Rollback is preferred when:

- Old version is healthy and compatible.
- Error rate is severe.
- Root cause is not immediately and safely correctable.
- No irreversible migration prevents rollback.

Roll forward may be safer when:

- A backward-incompatible migration already completed.
- A small configuration correction is known and tested.
- Rollback would corrupt or misread new data.

### Step 7 - Verify and prevent

Verify healthy target count, per-version error rate, business transaction, queues, and downstream effects. Add:

- Canary analysis by version.
- Backward-compatible expand/migrate/contract schema changes.
- Configuration validation at startup.
- Readiness that reflects traffic acceptance.
- Automated rollback thresholds.

## Interview-ready answer

> Because the incident started with deployment and only part of traffic fails, I would pause the rollout, preserve healthy old capacity, and group errors by image digest, version, instance, node, and zone. I would identify whether 5xx is generated by the gateway or application, compare effective config, secrets, identity, resources, routes, and schema compatibility between old and new instances, and test a real business path because shallow health can pass. I would choose rollback or roll-forward based on data and migration compatibility, then verify per-version recovery and improve canary and readiness gates.

---

# Scenario 3 - A Traffic Spike Causes a Cascading Failure

## Interview question

> Traffic increases ten times. Service B slows down, Service A starts retrying, database CPU reaches 100 percent, and the entire platform becomes unavailable. Explain what happened and how you would respond.

## Failure chain

```text
traffic x10
  -> B queue and latency increase
  -> A requests time out
  -> A retries
  -> effective request rate increases again
  -> B and DB receive more work
  -> pools and threads stay occupied longer
  -> queues grow
  -> latency exceeds more deadlines
  -> more retries
  -> cascading failure
```

This is positive feedback. Retries intended to improve reliability amplify an overload.

## Step-by-step response

### Step 1 - Protect the system

Use existing controls:

- Enforce rate limits and load shedding.
- Disable noncritical expensive work.
- Reduce or disable the harmful retry layer.
- Open a circuit breaker where appropriate.
- Preserve capacity for critical operations.
- Reject excess work quickly instead of letting unbounded queues grow.

### Step 2 - Quantify demand amplification

If original traffic is 10,000 RPS and every failed request makes two retries:

```text
maximum attempts = original requests x 3
                 = 30,000 attempts/second
```

Multiple retrying layers can multiply rather than add attempts:

```text
gateway 3 attempts x A 3 attempts x B 3 attempts
= up to 27 downstream attempts for one original request
```

### Step 3 - Find the first saturated resource

Build a timeline:

```text
traffic rise
queue rise
pool pending rise
DB CPU/latency rise
timeouts rise
retries rise
5xx rise
```

The earliest saturation change is often more useful than the final component that reached 100 percent.

### Step 4 - Recover in dependency order

Reducing traffic may allow the database and B to drain. Scaling A first can make the database worse. Restore from the bottom of the dependency graph:

```text
database/dependency capacity
  -> B processing capacity
  -> A/gateway admitted traffic
```

### Step 5 - Permanent design

- One intentional retry layer where possible.
- Retry only transient and safe/idempotent operations.
- Exponential backoff with jitter.
- Retry budget and total deadline.
- Circuit breaker and bounded concurrency.
- Queue limits and load shedding.
- Autoscaling on leading signals such as queue/concurrency, with dependency limits.
- Capacity tests at and beyond expected peak.
- Caching or work reduction for suitable reads.

## Interview-ready answer

> The platform entered a retry-amplified overload loop. Higher traffic increased queueing and B/DB latency, timeouts triggered retries, and retries increased downstream work until pools and the database saturated. I would first reduce admitted work using existing rate limits, load shedding, circuit breaking, and removal of redundant retry layers, then identify the first saturated component from a timeline rather than only the final 100-percent CPU metric. I would recover dependency capacity before adding more callers. Permanently I would use bounded exponential backoff with jitter, retry budgets, idempotency, bulkheads, queue limits, dependency-aware autoscaling, and peak-load tests.

---

# Scenario 4 - One Availability Zone Has Intermittent Failures

## Interview question

> Requests fail intermittently, but only when Service A in zone 1 calls Service B in zone 2. Same-zone calls work. How would you investigate?

## Likely fault domains

- Cross-zone route or network ACL.
- Firewall/security group source range.
- NAT/SNAT or conntrack on a zone-specific path.
- Packet loss or MTU mismatch.
- Zone-local DNS answer.
- Service-mesh gateway.
- Cross-zone load balancing disabled or misconfigured.
- Zone 2 backend/dependency issue.

## Step-by-step investigation

### Step 1 - Prove the matrix

Measure success, connect time, TLS time, and request time for:

```text
A-zone1 -> B-zone1
A-zone1 -> B-zone2
A-zone2 -> B-zone1
A-zone2 -> B-zone2
```

This separates source-zone, destination-zone, and cross-zone path failures.

### Step 2 - Identify the failing phase

- DNS answer differs: investigate zone-local resolver/discovery.
- Connect timeout: route, drop, ACL, NAT, packet path.
- TLS timeout/reset: proxy, MTU, SNI, certificate, sidecar.
- HTTP timeout: B or dependency in zone 2.

### Step 3 - Compare path evidence

Check:

- Source/destination subnet routes.
- Security group and network ACL in both directions.
- Load-balancer cross-zone behavior.
- NAT and conntrack usage.
- Network retransmissions and packet loss.
- MTU and fragmentation.
- Service-mesh ingress/egress gateways.
- Node-level CNI health.

Use a narrowly filtered, approved packet capture only when needed to show whether SYN, SYN-ACK, TLS records, or application packets disappear.

### Step 4 - Mitigate and fix

Mitigation may temporarily remove an unhealthy zone from routing if healthy capacity remains. Permanent correction must address route/policy/CNI/LB capacity and restore multi-zone resilience. Verify every source-destination combination before re-enabling.

## Interview-ready answer

> I would create a source-zone by destination-zone success matrix and identify whether the failure occurs during DNS, TCP, TLS, or HTTP. If only cross-zone TCP times out, I would compare forward and return routes, security groups, stateless ACLs, NAT/conntrack, load-balancer cross-zone settings, CNI, packet loss, and MTU. If TCP works but HTTP is slow, I would inspect the zone-2 backend and dependencies. I might drain the affected path to mitigate, but I would verify all zone combinations before restoring it.

---

# Scenario 5 - A Payment Is Duplicated After a Timeout

## Interview question

> Service A calls the Payment service. A times out and retries, but the first payment actually succeeded. The customer is charged twice. Why did this happen, and how should the system be designed?

## Why a timeout is ambiguous

```text
A sends charge request
Payment charges card
Payment stores success
response is lost or delayed
A sees timeout
A cannot know whether charge happened
A retries with a new operation
second charge succeeds
```

The error is assuming:

```text
timeout == operation failed
```

## Correct design

### Stable idempotency key

The same logical payment attempt must reuse the same key:

```text
Idempotency-Key: order-123-payment-v1
```

### Durable atomic claim

Payment service stores:

```text
key
request fingerprint
status: PROCESSING/SUCCEEDED/FAILED
result or payment reference
created/updated time
```

A database unique constraint ensures two concurrent duplicates cannot both create independent operations.

### Request-fingerprint validation

If the same key arrives with a different order, amount, or currency, reject it. Returning an old result for a different request is unsafe.

### In-progress handling

A duplicate received while the first request is processing should not start a second charge. It may:

- Wait within a bounded deadline.
- Return a "processing" status and operation ID.
- Allow the caller to query status.

### Provider idempotency

Pass a stable key to the external payment provider if it supports one. Local deduplication alone cannot prevent a duplicate created between the provider call and local commit without a reconciliation strategy.

## Investigation after an actual duplicate

1. Preserve both request IDs, idempotency keys, provider references, timestamps, and amounts.
2. Determine whether the retry reused the same business operation ID.
3. Reconstruct local DB commits and provider responses.
4. Determine which layer retried.
5. Stop further automated duplicate attempts.
6. Reconcile and refund through approved business controls.
7. Search for other affected operations in the same window.

## Interview-ready answer

> A timeout is an unknown outcome, not proof of failure. The first payment committed but its response was lost, and the retry was treated as a new operation. I would require a stable idempotency key for the logical payment, atomically enforce uniqueness, store status and the original result, validate that duplicate requests have the same fingerprint, and pass the key to the provider where supported. Retries must reuse the key. After an incident I would reconcile provider and local records, correct duplicate charges through an audited process, and identify every retrying layer.

---

# Scenario 6 - Kafka Lag, Rebalances, and One Poison Record

## Interview question

> Kafka consumer lag rises quickly. Consumers repeatedly rebalance, and logs show one record failing over and over. How would you stabilize and investigate the system?

## Possible combined mechanism

```text
consumer receives poison record
  -> retries synchronously for a long time
  -> poll loop does not call poll within max.poll.interval
  -> broker removes consumer from group
  -> partitions are reassigned
  -> another consumer receives the same uncommitted record
  -> failure repeats
  -> useful throughput falls and lag rises
```

## Step-by-step investigation

### Step 1 - Quantify lag correctly

For each partition:

```text
lag = log end offset - committed consumer-group offset
```

Inspect:

- Lag by partition, not only total.
- Arrival rate versus processing rate.
- Oldest-record age.
- Consumer count and assigned partitions.
- Rebalance frequency/reason.
- Processing latency and failures.

### Step 2 - Identify partition skew

One hot or blocked partition can dominate total lag. Adding consumers beyond partition count will not increase parallelism for that topic/group.

### Step 3 - Inspect poll and processing timings

Compare:

- `max.poll.interval.ms`.
- Records returned per poll.
- Worst-case batch processing plus retries.
- Heartbeat/session configuration.
- Long GC pauses or blocked consumer thread.

### Step 4 - Isolate poison processing

A bounded retry policy should distinguish:

- Transient failure: retry with backoff.
- Permanent validation/schema/business failure: publish to a dead-letter/quarantine workflow with original metadata and reason.

Do not silently skip a record. Preserve visibility and an auditable replay process.

### Step 5 - Preserve ordering and correctness

Moving past a failed record can violate per-key ordering. Decide explicitly whether:

- The key/partition must pause.
- Later records may proceed.
- The business requires manual correction before replay.

### Step 6 - Verify commit semantics

Ensure offsets are committed only according to the intended processing guarantee. A crash after side effect but before commit can redeliver, so the handler must be idempotent.

### Step 7 - Correct and prevent

- Bound retries and move permanent failures to quarantine/DLT.
- Ensure processing fits poll interval or decouple poll from bounded worker processing safely.
- Reduce batch size when appropriate.
- Fix schema/data/handler cause.
- Use stable group membership/cooperative assignment where suitable.
- Size partitions and consumers for throughput.
- Alert on lag growth rate, oldest age, rebalance rate, and DLT volume.

## Interview-ready answer

> I would inspect lag per partition, arrival versus processing rate, oldest-record age, assignments, and rebalance reasons. A poison record can block processing long enough to exceed `max.poll.interval`, trigger reassignment, and then be redelivered to another consumer, causing a loop. I would bound transient retries, quarantine permanent failures with full metadata and an audited replay path, preserve ordering requirements deliberately, and make side effects idempotent because commit-after-processing still permits redelivery. I would then fix poll/processing timing, partition skew, and the actual handler or data defect.

---

# Scenario 7 - Autoscaling Makes the Database Slower

## Interview question

> API latency rises, so the platform scales from 10 to 50 application instances. Database latency and error rate then become worse. Why?

## Mechanism

If each instance has a DB pool of 30:

```text
10 instances x 30 = up to 300 connections
50 instances x 30 = up to 1,500 connections
```

The database may support neither that connection count nor the resulting query concurrency. More application capacity increases pressure on the shared bottleneck.

Other amplification:

- Cold instances cause cache misses.
- Each instance warms local caches with duplicate reads.
- Health/startup tasks query the database.
- More workers consume queue messages.
- Retries continue during scale-out.

## Step-by-step investigation

1. Plot replicas, total pool capacity, active DB connections, query rate, DB CPU/I/O, locks, and latency on one timeline.
2. Check whether latency began before scaling and accelerated afterward.
3. Calculate total potential connection and query concurrency.
4. Identify expensive queries and cache-miss bursts.
5. Check whether autoscaling uses CPU while the true bottleneck is DB wait.
6. Check scale-down behavior and connection draining.

## Correct design

- Treat database capacity as a global budget.
- Divide a safe connection budget across possible replicas.
- Use a connection proxy/pooler where appropriate, understanding transaction/session constraints.
- Optimize queries and reduce calls.
- Cache suitable reads.
- Rate-limit or bound database concurrency.
- Scale database/read replicas only for workloads they can safely serve.
- Use autoscaling signals that represent useful work and dependency headroom.

## Interview-ready answer

> Horizontal scaling helps only when the application tier is the bottleneck and dependencies have headroom. Increasing replicas multiplied total pool capacity, query concurrency, cold-cache misses, and possibly retries, overwhelming the database. I would correlate replica count with active connections, query rate, DB CPU/I/O/locks, and latency, calculate the global connection budget, and identify expensive work. I would cap DB concurrency, right-size per-instance pools for maximum replicas, optimize/reduce queries, and make autoscaling dependency-aware.

---

# Scenario 8 - Memory Rises for Hours, then One Pod Restarts

## Interview question

> One Java service instance slowly consumes more memory for several hours, GC becomes frequent, latency rises, and Kubernetes eventually restarts the pod. How would you determine whether it is a heap leak, native-memory issue, or container limit?

## First distinction: JVM heap versus process/container memory

```text
JVM heap
metaspace
direct buffers
thread stacks
JIT/code cache
native libraries
memory-mapped files
other process/native allocations
-------------------------------
process RSS / container memory
```

Heap can be stable while RSS rises due to native memory. Conversely, heap occupancy after full GC rising over time suggests retained heap objects.

## Step-by-step investigation

### Step 1 - Determine why it restarted

Inspect:

- Kubernetes termination reason and exit code.
- `OOMKilled` versus liveness restart versus application exit.
- Container memory limit and working set.
- JVM OOM log and heap-dump configuration.
- Node memory pressure/eviction events.

### Step 2 - Compare the bad pod with healthy pods

Compare:

```text
traffic and request types
heap used before/after GC
allocation rate
GC pause/frequency
RSS/container working set
direct buffer usage
metaspace/classes
thread count
image/config/flags
node and uptime
```

### Step 3 - Classify the shape

- Heap after major/full GC rises: likely retained objects/leak or legitimate growing state.
- Heap sawtooths back to stable baseline but RSS rises: native/direct/thread/JVM area.
- Thread count rises with RSS: thread leak and stack memory.
- Direct buffer metrics rise: buffer/connection/resource leak.
- Memory remains below heap max but container is killed: JVM plus native overhead exceeded cgroup limit.

### Step 4 - Capture evidence before restart

Use approved JVM diagnostics:

- GC logs/metrics.
- Class histogram.
- Heap dump for heap OOM or suspected retention.
- Native Memory Tracking if enabled and appropriate.
- Thread dumps and thread count.
- Direct-buffer metrics.

Heap dumps contain sensitive application data and must be secured and handled according to policy.

### Step 5 - Analyze mechanism

Examples:

- Unbounded cache retains customer objects.
- Listener/subscription is never removed.
- `ThreadLocal` retains request data in pooled threads.
- Large responses remain queued.
- Direct buffers are not released.
- New thread per request leaks native stacks.
- Heap sizing leaves no room inside the container for native memory.

### Step 6 - Correct and verify

Fix retention or resource lifecycle, bound caches/queues, close buffers/connections, cap threads, and set container-aware heap/native headroom. Verify memory after full GC and RSS remain stable over several traffic cycles.

## Interview-ready answer

> I would first determine whether Kubernetes reported `OOMKilled`, liveness failure, or an application OOM. Then I would compare JVM heap after GC with process RSS and container working set. Rising post-GC heap suggests retained objects, while stable heap and rising RSS points to direct buffers, thread stacks, metaspace, or native allocations. I would compare the bad pod with healthy pods, capture GC data, class histograms or a secured heap dump, thread counts, direct-memory metrics, and Native Memory Tracking when available. I would fix the retaining object or native-resource lifecycle and leave cgroup headroom rather than only raising the limit.

---

# Scenario 9 - Cache Returns Stale Data After a Successful Update

## Interview question

> The database shows an order as `CANCELLED`, but users continue seeing `ACTIVE` for several minutes through the API. How would you investigate and design the cache correctly?

## Possible paths

```text
write:
API -> DB commit -> cache invalidation/update

read:
API -> local cache -> Redis -> DB
```

Staleness can occur at any cache layer, replica, CDN, or client.

## Step-by-step investigation

### Step 1 - Prove source-of-truth state and timing

Record:

- DB commit time and transaction result.
- Cache key and value/version/TTL.
- Local versus distributed cache.
- Read replica state and replication lag.
- Instance serving the stale response.
- CDN/gateway/client cache headers.

### Step 2 - Inspect write ordering

A dangerous cache-aside sequence:

```text
delete cache
update DB
```

A concurrent reader can repopulate old data before the DB commit.

Common safer sequence:

```text
commit DB
then invalidate cache
```

Even this has failure windows. For example, a reader can load the old database value before the writer commits, the writer can commit and invalidate the cache, and then that reader can repopulate the cache with the old value after the invalidation. The writer can also commit successfully and then crash before invalidation. Stronger requirements may therefore need a transactional outbox/change-data-capture invalidation event, entity-version checks that reject older cache writes, or a carefully designed write-through strategy.

### Step 3 - Check key construction

The write may evict:

```text
order:123
```

while the read uses:

```text
tenant:7:order:123:v2
```

Also inspect caches of lists, aggregates, and negative results.

### Step 4 - Check event delivery

If invalidation is asynchronous:

- Was the event written atomically with the DB change?
- Was it published and consumed?
- Is consumer lag high?
- Did a poison event block the partition?
- Is handling idempotent?

### Step 5 - Correct and prevent

- Define acceptable staleness per business operation.
- Use consistent keys and bounded TTLs.
- Invalidate/update after commit.
- Use outbox/CDC for durable invalidation where needed.
- Include entity version in events and ignore older updates.
- Monitor cache age, invalidation failure, consumer lag, and read-repair.
- Bypass cache for operations requiring read-your-write consistency where justified.

## Interview-ready answer

> I would identify every cache layer and compare the DB commit, invalidation event, cache value/version/TTL, consumer processing, and stale response instance on one timeline. I would check key mismatch, invalidation before commit, failed event publication, consumer lag, read-replica lag, and out-of-order updates. For durable invalidation I would commit an outbox record with the database change, process it idempotently, carry entity versions, and use bounded TTL as a safety net. The correct design depends on the business's allowed staleness.

---

# Scenario 10 - A Saga Is Stuck After Payment Succeeds

## Interview question

> Payment succeeded, inventory reservation failed, and the refund compensation also failed. The order has remained in `COMPENSATING` for two hours. How would you recover it?

## Important principle

Compensation is another distributed business operation. It can time out, fail, be duplicated, or complete without its response being received. A Saga does not provide automatic database rollback.

## Step-by-step investigation

### Step 1 - Reconstruct durable Saga state

Collect:

```text
saga/order ID
current state and version
completed forward steps
commands/events and message IDs
attempt counts
last error and next retry time
payment provider references
inventory state
outbox/inbox records
consumer offsets/DLT entries
```

### Step 2 - Verify real external state

Do not assume the failed refund response means no refund. Query the payment provider with the stable operation/reference ID and reconcile:

- Charged and not refunded.
- Refund succeeded but local state not updated.
- Refund pending.
- Refund rejected permanently.

### Step 3 - Check why workflow stopped

Possible causes:

- Retry scheduler stopped.
- Message is in DLT.
- Orchestrator crashed after publishing but before state update.
- Optimistic-lock conflict rejected transition.
- State-machine guard does not handle a response.
- Duplicate/out-of-order event was ignored incorrectly.
- Compensation exceeded attempts and needs manual review.

### Step 4 - Resume idempotently

Never issue an unkeyed second refund. Reuse the stable compensation operation ID. Acquire state transition atomically using saga version/status, then retry or reconcile.

### Step 5 - Manual intervention

Provide an audited operator action:

- View state and external references.
- Retry a specific idempotent step.
- Mark externally confirmed completion.
- Move to manual business resolution.

Every action needs authorization, reason, actor, timestamp, before/after state, and replay protection.

### Step 6 - Prevent recurrence

- Durable orchestrator/state machine.
- Transactional outbox and consumer inbox/deduplication.
- Explicit retry schedule and terminal/manual states.
- Alerts on state age and compensation failures.
- Reconciliation jobs against external providers.
- Idempotent forward and compensation handlers.

## Interview-ready answer

> I would treat the compensation as a real distributed operation, not an automatic rollback. I would reconstruct the durable saga state and message history, then query the payment provider because a timeout may hide a successful refund. I would determine whether the workflow stopped due to scheduler, DLT, state-transition, optimistic-lock, or missing-event problems. Any resume must reuse the stable refund operation ID and update saga state atomically. If automation cannot resolve it, I would use an authorized, audited manual workflow. Permanently I would add reconciliation and alerts for sagas stuck beyond their expected state age.

---

# Scenario 11 - Intermittent 502 and 504 Come From One Backend

## Interview question

> The gateway returns a mixture of 502 and 504 responses, but only for requests routed to one backend instance. How can one instance produce both statuses?

## Possible mechanism

The same unhealthy instance can fail in different phases:

```text
502:
  instance resets connection
  TLS/protocol response is invalid
  process restarts mid-response

504:
  instance accepts request
  then stalls on DB, lock, GC, or thread pool
  gateway response timeout expires
```

## Step-by-step investigation

1. Map gateway error subreason and request ID to upstream IP.
2. Drain the bad instance while preserving evidence.
3. Compare image, config, node, resources, pools, DNS, and dependencies with a healthy instance.
4. Align gateway logs, B logs, restart/GC events, and thread/pool metrics.
5. For 502s, inspect resets, TLS/protocol, response headers, process exits, and stale connections.
6. For 504s, inspect request queue, blocked threads, pool acquisition, DB/downstream spans, and GC pauses.
7. Check whether one underlying resource event explains both, such as memory pressure causing long GC followed by process restart.

## Example root cause

```text
Unbounded local cache on B3
  -> heap pressure
  -> long full-GC pauses cause 504
  -> container exceeds limit and restarts
  -> in-flight connections reset and cause 502
```

## Interview-ready answer

> The statuses describe different gateway observations, not necessarily different root causes. One instance can stall long enough to cause 504s and then restart or reset connections, causing 502s. I would correlate each gateway subreason with the upstream IP, drain the instance, preserve evidence, and compare it with healthy instances. I would align GC, memory, restart, thread, pool, and dependency events to prove whether one lifecycle or resource problem explains both statuses.

---

# Scenario 12 - Database Is Up, but Applications Cannot Get Connections

## Interview question

> Database monitoring says the database is available, but applications report connection acquisition and connection timeouts. How can both be true?

## Different timeout phases

```text
Application pool acquisition timeout:
  no free connection in the local pool

TCP connect timeout:
  network handshake to database did not complete

TLS/authentication timeout:
  secure/login negotiation stalled

Database query/statement timeout:
  connection exists, but SQL exceeds deadline
```

"Database is up" may mean only that a health monitor connected from another network with another identity.

## Step-by-step investigation

### Step 1 - Read the exact exception

Identify the layer and elapsed duration. Pool-acquisition errors require a different investigation than socket connection timeouts.

### Step 2 - If pool acquisition fails

Check:

- Active, idle, maximum, pending, and acquisition time.
- Connection hold duration.
- Transaction duration.
- Query and lock duration.
- Leaks and close paths.
- Per-instance pool configuration.

### Step 3 - If new DB connections fail

Check:

- DNS and endpoint.
- Network path from the affected app instance.
- Database listener.
- TLS certificate and trust.
- Credentials/secret rotation.
- DB maximum sessions and per-user limit.
- Connection proxy health.
- Failover/primary endpoint transition.

### Step 4 - Compare instances

If one app instance fails, compare its DNS, secret version, trust store, node, route, proxy, config, and pool state. A global DB dashboard can remain green.

### Step 5 - Fix and verify

Correct the exact phase: release leaked/long-held connections, optimize transactions, restore route/auth, correct limits, or repair failover. Do not simply enlarge all pools; calculate total database capacity.

## Interview-ready answer

> Database availability from a monitor does not prove that a particular application can acquire a local pooled connection or establish a new connection with its network and identity. I would classify the exact timeout as pool acquisition, TCP, TLS/login, or statement execution. For pool acquisition I would inspect active/pending connections and hold/transaction/query time. For connect failures I would inspect DNS, route, listener, secret, trust, DB session limits, and failover from the affected instance. I would fix that phase and verify the total connection budget.

---

# Scenario 13 - Security Fails Immediately After Key Rotation

## Interview question

> After an identity-provider signing-key rotation, some services accept JWTs while others return 401. The gateway still authenticates users. How would you investigate?

## Likely mechanism

JWT validators cache the identity provider's JWKS signing keys. Services may differ in:

- JWKS refresh behavior.
- Cache age.
- Network access to the JWKS endpoint.
- Issuer/audience configuration.
- Library version.
- Clock.
- Whether the gateway forwards the original or exchanges the token.

The gateway accepting a token does not prove downstream validators have the new key.

## Step-by-step investigation

### Step 1 - Capture safe token metadata

Do not log the full token. Record safely:

```text
algorithm
kid
issuer
audience
issued-at/expiry
service validation error
```

### Step 2 - Compare accepting and rejecting services

Check:

- Does the rejected token's `kid` exist in current JWKS?
- Can each service resolve and reach the JWKS endpoint?
- When did each cache refresh?
- Do libraries refresh automatically on unknown `kid`?
- Is issuer/audience expected by that service?
- Is system clock synchronized?
- Did the deployment change trust/config?

### Step 3 - Check rotation overlap

A safe rotation generally publishes the new public key before issuing tokens signed with it and retains the old public key until all old tokens expire plus safety margin.

### Step 4 - Restore safely

- Repair JWKS connectivity/refresh.
- Correct issuer/audience configuration.
- Restore required key overlap.
- Roll back a broken validator library/configuration if appropriate.

Do not disable signature validation or accept arbitrary issuers as mitigation.

## Interview-ready answer

> I would safely compare the token `kid`, issuer, audience, and time claims with validator errors and current JWKS, without logging the token. Then I would compare accepting and rejecting services for JWKS reachability, cache refresh behavior, library/config version, and clock. The gateway may have refreshed its cache while downstream services did not. I would restore correct JWKS refresh or key overlap and verify both old unexpired and newly issued tokens, never bypassing signature validation.

---

# Scenario 14 - Observability Is Incomplete During an Incident

## Interview question

> Users report failures across five services, but trace sampling did not retain the failed requests and one service does not propagate trace context. How would you investigate?

## Key principle

Telemetry is evidence, not reality. Missing telemetry does not prove a component was not called.

## Step-by-step investigation

### Step 1 - Start with known boundaries

Use:

- Gateway request ID and access log.
- User-safe business operation ID.
- UTC time window.
- Method/route.
- Source/destination instance.
- Message key/event ID for asynchronous hops.

### Step 2 - Reconstruct a timeline

Search each service for the same request/correlation/business ID. Where propagation breaks, use:

- Parent call timestamp and destination.
- Child access log timestamp and source.
- Stable operation or entity ID.
- Gateway/upstream target logs.
- Database or broker event IDs.

Avoid correlating only by a broad timestamp when traffic is high.

### Step 3 - Use metrics to narrow

Find which service, route, instance, or dependency showed the first rise in errors, latency, queue, or saturation. Then inspect logs in that narrow interval.

### Step 4 - Check logging gaps

Determine whether:

- Logs were dropped due to backpressure.
- The process crashed before flush.
- Log level changed.
- Clock skew misorders entries.
- Sensitive-field filtering removed the correlation field.
- Async context propagation was lost.

### Step 5 - Improve observability

- W3C trace-context propagation across HTTP and messaging.
- Correlation/business IDs where appropriate.
- Tail-based or error-aware sampling.
- Consistent service/instance/route fields.
- UTC and clock synchronization.
- Telemetry pipeline health metrics.
- Avoid high-cardinality metric labels and sensitive baggage.

## Interview-ready answer

> I would not stop because a trace is missing. I would anchor on the gateway request ID, business operation ID, UTC window, route, and upstream target, then reconstruct each synchronous and asynchronous hop from access logs, message IDs, and service logs. Metrics would identify the first component whose errors, latency, or saturation changed. I would account for clock skew and telemetry drops. Permanently I would repair W3C context propagation, use error-aware sampling, and monitor the observability pipeline itself.

---

# Scenario 15 - Full Senior-Level Incident Walkthrough

## Interview question

> At 10:05 UTC, checkout success drops from 99.9 percent to 70 percent. Customers see 504s, Kafka order lag grows, database CPU is 90 percent, and a deployment occurred at 10:00. Walk through your complete response.

## Step 1 - Declare impact and ownership

State:

```text
business flow: checkout
impact: 30% failures and possible delayed/duplicate orders
start time: approximately 10:05 UTC
known signals: 504, Kafka lag, DB CPU
suspected trigger: 10:00 deployment, not yet proven
```

Assign or establish incident coordination, communication, and a decision log according to the organization's process.

## Step 2 - Protect users and correctness

Immediate questions:

- Are payment or order writes completing after the 504?
- Will clients/gateways retry?
- Are operations idempotent?
- Is the Kafka backlog safe and within retention?
- Can the rollout be paused?
- Is old healthy capacity available?

Potential mitigations:

- Pause rollout.
- Route away from proven bad version/instances.
- Roll back if schema/data compatibility permits.
- Rate-limit noncritical checkout features.
- Remove harmful retries.
- Preserve Kafka messages; do not reset offsets casually.

## Step 3 - Build a unified timeline

```text
10:00 deployment starts
10:03 new version receives 10% traffic
10:04 DB query rate rises
10:05 DB p99 and pool wait rise
10:05 checkout p99 crosses gateway timeout
10:06 504 rate rises
10:06 retries increase DB load
10:07 order event processing slows
10:08 Kafka lag grows
```

The order of changes helps distinguish trigger and downstream effects.

## Step 4 - Compare old and new versions

Group:

- Request rate/error/latency.
- Trace spans.
- Query count and query text/fingerprint.
- DB pool use.
- Kafka publish/consume rate.
- Config/feature flag.

Example evidence:

```text
old version: 3 DB queries per checkout
new version: 103 DB queries per checkout
```

This points to an N+1 regression.

## Step 5 - Trace a failed checkout

```text
Gateway total:             5.0 s -> 504
Checkout service:          still running
DB pool wait:              1.8 s
100 item queries:          3.5 s
Order event publish:       not reached before caller timeout
```

Check whether the transaction later commits and event publishes. Use outbox records and idempotency state to determine correctness.

## Step 6 - Decide mitigation

If the new version is proven and rollback-compatible:

1. Roll back.
2. Stop redundant retries.
3. Allow DB and queues to recover.
4. Monitor old version capacity.

Do not scale checkout blindly because it can issue even more queries.

## Step 7 - Recover asynchronous backlog

After DB and checkout stabilize:

- Confirm consumers are healthy.
- Compare processing rate with arrival rate.
- Estimate drain time:

```text
drain time =
  current lag / (processing rate - new arrival rate)
```

- This estimate is valid only when the sustainable processing rate is greater than the new arrival rate.
- If processing equals arrival, the backlog never shrinks.
- If processing is below arrival, lag continues to grow; increase safe processing capacity or reduce admitted work before estimating a drain time.

- Scale consumers only up to partition and downstream capacity.
- Monitor poison records, rebalances, and DLT.
- Reconcile orders/payments that timed out during the incident.

## Step 8 - Verify recovery

Verify:

- Checkout success and p99.
- Gateway 504 rate.
- DB query rate, CPU, locks, and pool wait.
- Kafka lag is decreasing and oldest age recovers.
- No duplicate charges/orders.
- All instances run expected digest/config.
- Synthetic checkout succeeds.

## Step 9 - Permanent correction

- Replace N+1 with bounded bulk query/fetch.
- Add query-count and production-shaped performance tests.
- Improve canary analysis using DB calls/request and p99.
- Use a global retry budget.
- Ensure payment/order idempotency.
- Use transactional outbox for order events.
- Alert on pool wait, DB calls/request, lag growth, and business success.
- Document rollback compatibility for migrations.

## Interview-ready answer

> I would treat this as both an availability and correctness incident. I would pause the rollout, determine whether timed-out checkouts can still commit, and protect idempotency before allowing retries. I would build a common timeline for deployment, per-version errors, DB query rate and pool wait, gateway 504s, retries, and Kafka lag. Then I would compare old and new traces and query counts to prove whether the deployment caused the first bottleneck. If the new version introduced an N+1 query and rollback is compatible, I would roll it back and reduce retry amplification, let the database recover, and then drain Kafka lag within downstream capacity. I would verify checkout success, p99, DB saturation, lag, and duplicate/incomplete operations, then add query-count tests, canary gates, outbox/idempotency, and better saturation alerts.

---

# Additional Advanced Interview Questions

## Question 16 - How do you decide whether to restart a failing service?

Restart only when:

- It is an approved mitigation.
- The instance is not making progress or is unsafe.
- Healthy capacity remains.
- In-flight work and duplicate risk are understood.
- Important evidence has been captured where practical.

A restart is useful for transient corrupted state, deadlock, or resource reclamation, but it can:

- Erase thread/heap/local evidence.
- Create a restart loop.
- Increase cold-cache and dependency load.
- Interrupt non-idempotent work.
- Hide the root cause.

Interview answer:

> I separate mitigation from root cause. I may restart or replace an instance to restore service if it is safe, but first I preserve key evidence, understand in-flight correctness, and ensure capacity. I then investigate why a fresh process changes the condition and prevent recurrence.

## Question 17 - How do you know an incident is fully resolved?

Technical recovery alone is not enough. Verify:

```text
business success and correctness
error rate and latency SLO
all instances/zones/versions
queues and lag draining
pools and resources below saturation
no duplicate or missing operations
no hidden retry traffic
synthetic and real traffic
alerts returned to normal
```

Continue monitoring through an appropriate load cycle and record the evidence.

## Question 18 - What makes a good production hypothesis?

A good hypothesis is specific, mechanistic, and falsifiable:

```text
Bad:
Maybe Kubernetes has a problem.

Good:
Only pods on node N resolve the old database IP because the node-local
DNS cache did not refresh after failover. If true, DNS answers on N will
differ from healthy nodes and direct use of the current endpoint will work.
```

Test the smallest safe observation that can disprove it.

## Question 19 - How do you prioritize when many metrics are red?

Use:

1. Business impact and data risk.
2. Earliest abnormal signal in the timeline.
3. Dependency direction.
4. Saturation and queue propagation.
5. Version/instance correlation.

Many red metrics are consequences. The database, Kafka lag, thread queues, and gateway errors can all result from one bad query plus retries.

## Question 20 - What should a post-incident review produce?

- Clear impact and timeline.
- Root cause with causal mechanism.
- Trigger and contributing factors.
- What helped and delayed detection/recovery.
- Correctness/reconciliation results.
- Action items with owners and due dates.
- Tests, deployment guards, alerts, capacity controls, and runbook improvements.

Avoid blame and vague actions such as "be more careful."

---

# Cross-Domain Cheat Sheet

| Observation | Strong next question |
|---|---|
| CPU normal, latency high | Where are requests waiting or queued? |
| One version fails | What changed in artifact, config, schema, identity, or resources? |
| More replicas make it worse | Which shared dependency is saturated? |
| Timeout on a write | Did the operation complete despite the missing response? |
| Kafka lag rises | Did arrival increase, processing fall, or one partition block? |
| 502 and 504 on one pod | Is it stalling and then resetting/restarting? |
| DB is "up" | Can this instance acquire/connect/authenticate/query? |
| Health is green | Which real business-path components are not checked? |
| Some JWT validators fail | Do JWKS cache, `kid`, issuer, audience, clock, and network differ? |
| No trace exists | Which stable request/business/message IDs can reconstruct the path? |

## Final mental model

```text
Do not ask only:
  "Which component is red?"

Ask:
  "What changed first?"
  "What work is waiting where?"
  "Which resource reached a limit?"
  "Which requests, instances, versions, or data differ?"
  "Could a timeout hide a successful write?"
  "What evidence can disprove my hypothesis?"
  "How will I prove business correctness after recovery?"
```
