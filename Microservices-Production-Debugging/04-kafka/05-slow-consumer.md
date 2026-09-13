# Problem

The consumer is processing, but each record takes too long and backlog recovery is poor.
A slow consumer differs from a non-consuming consumer.
Offsets continue to advance, just below the required rate.
The bottleneck may be broker fetch, deserialization, an internal queue, handler CPU, database, remote dependency, commit, or retry behavior.
Adding threads helps only when partitions and downstream capacity permit it.
Unbounded concurrency can move lag into memory and overload dependencies.
The goal is sustainable useful throughput, not the highest short benchmark.

# Production Situation

The Meridian team has fixed redelivery safety.
At 17:10, shipment processing gradually slows.
`orders.v1` receives 6,000 events/min.
Group `fulfillment-v3` consumes 4,300/min.
Lag grows 1,700/min.
There are 12 partitions and 12 assigned members.
Ten partitions process about 490/min each.
Partition 7 processes only 85/min and holds 61% of total lag.
Listener p50 is 62 ms, p95 is 920 ms, and p99 is 3.8 seconds.
Kafka fetch p99 is 21 ms.
Deserialization p99 is 3 ms.
Shipment DB connection pool is active 38/40, idle 2, pending 57.
DB acquisition p99 is 740 ms.
The `customerId` key routes one marketplace customer's burst to partition 7.
A fraud API limit of 80 requests/sec adds retry waits.
CPU remains 41%, showing waits rather than compute saturation.

# Architecture

```text
orders.v1
12 partition leaders
    |
    | one assigned consumer lane per partition
    v
Spring Kafka listener
    |
    +--> deserialize
    +--> bounded work queue
    +--> fraud-api
    +--> HikariCP
    +--> shipment-db
    +--> commit
```

Partition ordering means one partition is processed in record order by its assigned lane.
A key determines partition selection.
All events for the same key usually share a partition and preserve per-key order.
A hot key can therefore cap throughput even when other partitions are idle.
Changing keys or partition count can change ordering and mapping semantics.
It is an architectural decision, not an emergency command.
Backpressure means slowing intake or pausing partitions when downstream capacity is full.
It prevents an unbounded local queue from hiding overload.

# What I Check FIRST

1. Compare produce rate, successful consume rate, and lag slope.
   WHY: the arithmetic quantifies the capacity deficit.
   LOOK FOR: `6,000 - 4,300 = 1,700/min`, matching lag growth.
2. Break latency into fetch, deserialize, queue, handler, dependency, and commit.
   WHY: total listener time does not identify the slow stage.
   LOOK FOR: DB acquisition and fraud call dominating.
3. Compare per-partition rates, lag, and keys.
   WHY: one hot partition cannot be split among same-group consumers.
   LOOK FOR: partition 7 owning most backlog.
4. Inspect downstream saturation before scaling consumers.
   WHY: more callers can worsen pools and rate limits.
   LOOK FOR: Hikari pending requests and fraud throttling.
5. Check poll gaps, queue bounds, rebalances, and error retries.
   WHY: slowness can become churn or retry amplification.
   LOOK FOR: stable polling and bounded work in memory.

# Step-by-Step Investigation

### Step 1 - Quantify sustainable throughput

* What I check: input, successful effect, and commit rates over peak and steady windows.
* Why: a short burst benchmark may not represent sustainable dependency capacity.
* Expected: consume capacity exceeds 6,000/min with recovery headroom.
* Bad: long-window output remains 4,300/min.
* Meaning: backlog will grow by about 1,700/min.
* Next: decompose time and partition distribution.

### Step 2 - Check partition distribution

* What I check: lag, record rate, byte rate, and oldest age per partition.
* Why: aggregate averages hide hot partitions.
* Expected: each of 12 partitions receives near 500/min.
* Bad: partition 7 receives a disproportionate keyed burst.
* Meaning: available parallelism is uneven.
* Next: identify key-domain skew using hashed or aggregated labels.

### Step 3 - Verify assignment utilization

* What I check: member-to-partition assignments, idle members, listener concurrency.
* Why: parallelism is the minimum of usable partitions, consumers, and downstream capacity.
* Expected: all 12 partitions have stable owners.
* Bad: more than 12 group members are running.
* Meaning: extra members are idle and add operational churn, not throughput.
* Next: avoid scaling beyond useful assignment count.

### Step 4 - Measure fetch and deserialize

* What I check: fetch latency, bytes/records per fetch, poll rate, deserialization timer and failures.
* Why: this distinguishes Kafka delivery from application processing.
* Expected: fetch p99 21 ms and deserialize p99 3 ms.
* Bad: fetch p99 is seconds across consumers.
* Meaning: investigate broker, network, quotas, or fetch configuration.
* Next: inspect leaders and broker request metrics.

### Step 5 - Measure local queue delay

* What I check: queue capacity, occupancy, enqueue rejects, and record age at handler start.
* Why: fetched records can wait invisibly inside the application.
* Expected: bounded queue under 50% with short wait.
* Bad: queue remains full and age rises.
* Meaning: workers or dependencies cannot drain it.
* Next: pause affected containers/partitions under the designed backpressure policy.

### Step 6 - Measure handler and dependencies

* What I check: handler self-time, fraud latency/status, DB acquisition, query, and transaction time.
* Why: external waits often dominate with low CPU.
* Expected: most records finish under 100 ms.
* Bad: fraud returns 429 and Hikari pending reaches 57.
* Meaning: downstream capacity is exceeded and retries extend service time.
* Next: inspect retry policy and database pool causality.

### Step 7 - Distinguish DB pool wait from slow SQL

* What I check: pool active/idle/pending, acquisition latency, query rate, and query latency.
* Why: waiting for a connection is not the same as executing slow SQL.
* Expected: pending zero and acquisition p99 below 20 ms.
* Bad: active 40/40, pending 57, acquisition 740 ms, query latency normal.
* Meaning: callers cannot enter the database; blindly tuning SQL misses the bottleneck.
* Next: locate long-held or leaked connections and transaction scope.

### Step 8 - Inspect retries and amplification

* What I check: fraud attempts per event, retry delay, jitter, and 429 rate.
* Why: retries consume listener time and increase downstream load.
* Expected: bounded retries with backoff and a terminal recovery route.
* Bad: three immediate retries turn 80 requests/sec into 210 attempts/sec.
* Meaning: retry policy worsens throttling.
* Next: respect retry hints, reduce concurrency, and apply jitter.

### Step 9 - Check poll safety

* What I check: max poll gap, batch size, heartbeat health, and rebalances.
* Why: slow batches can exceed `max.poll.interval.ms`.
* Expected: worst batch completes with margin.
* Bad: queue/handler work approaches the poll limit.
* Meaning: slow consumer may become rebalance storm and duplicate delivery.
* Next: reduce batch size or decouple work with correct commit control.

### Step 10 - Model downstream capacity

* What I check: database transactions/sec and fraud requests/sec safely supported.
* Why: consumer concurrency cannot exceed the narrowest durable dependency.
* Expected: configured concurrency stays below both capacity limits.
* Bad: 12 consumers times per-record calls exceed fraud's 80 requests/sec.
* Meaning: scale-out will worsen throttling.
* Next: batch, cache, rate-limit, or negotiate dependency capacity.

### Step 11 - Test a bounded improvement

* What I check: canary with batched DB writes and fraud cache for eligible records.
* Why: a canary verifies the proposed bottleneck without fleet risk.
* Expected: handler p95 falls and successful throughput rises.
* Bad: no change.
* Meaning: another stage or hot partition remains limiting.
* Next: compare per-stage before/after metrics.

# Metrics to Check

| Metric | High means | Low means |
|---|---|---|
| produce rate | greater incoming demand | traffic decline or producer issue |
| successful consume rate | good if effects are correct | capacity deficit |
| lag slope | worsening backlog | recovery if negative |
| oldest event age | business lateness | fresh processing |
| lag by partition | skew or blocked partition | partition current |
| fetch p99 | broker/network delay | fetch likely healthy |
| deserialize p99/errors | schema or CPU cost | not limiting |
| queue occupancy | backpressure pressure | available local capacity |
| queue wait age | local backlog | prompt handler start |
| handler p95/p99 | slow application path | healthy only with correct output |
| dependency latency/errors | remote bottleneck | look elsewhere |
| Hikari active/max | pool saturation | unused connections |
| Hikari pending | callers blocked for connection | no acquisition queue |
| DB query latency | execution slowness | pool wait may still exist |
| retry attempts/event | amplification | no retry pressure |
| poll gap | rebalance risk | safe cadence |

If CPU is low and pool pending is high, threads are waiting.
If DB query rate falls while requests remain constant, pool blockage can prevent queries from starting.
If fetch stays fast while event age rises, Kafka delivery is not the slow stage.
If only one partition lags, total consumer count is a weak signal.
If throughput rises but errors and duplicates rise too, it is not a valid improvement.

# Distributed Trace Investigation

I select slow and normal events from different partitions.
Messaging attributes include group, topic, partition, offset, event ID, and queue age.

```text
traceId=f901ac partition=7 eventId=evt-hot-77
consumer process                            3,812 ms
  +-- local queue wait                     1,106 ms
  +-- fraud-api attempt 1                    522 ms 429
  +-- retry backoff                        1,000 ms
  +-- fraud-api attempt 2                    418 ms 200
  +-- db connection acquire                  741 ms
  +-- db query                                25 ms
```

```text
traceId=17be21 partition=2 eventId=evt-regular-8
consumer process                               71 ms
  +-- fraud-api                                38 ms
  +-- db acquire                                5 ms
  +-- db query                                 19 ms
```

The trace distinguishes query execution from pool acquisition.
Retry child spans reveal extra calls.
Queue time reveals backpressure before handler code.
Client latency on the fraud span includes network and remote wait.
Server trace, when available, separates remote processing.
A missing fraud server child may indicate propagation or sampling loss, not network failure.
I correlate dependency request ID and event ID in logs.

# Distributed Logs

```text
2026-09-13T17:12:01.010+05:30 INFO service=fulfillment-service instance=fulfill-7 traceId=f901ac spanId=ab11 eventId=evt-hot-77 topic=orders.v1 partition=7 offset=751009 stage=handler_start eventAgeMs=48211 queueWaitMs=1106
2026-09-13T17:12:01.532+05:30 WARN service=fulfillment-service instance=fulfill-7 traceId=f901ac spanId=c901 eventId=evt-hot-77 dependency=fraud-api status=429 attempt=1 latencyMs=522 retryDelayMs=1000
2026-09-13T17:12:03.691+05:30 WARN service=fulfillment-service instance=fulfill-7 traceId=f901ac spanId=db71 eventId=evt-hot-77 dependency=shipment-db stage=connection_acquire latencyMs=741 poolActive=40 poolMax=40 poolPending=57
2026-09-13T17:12:03.837+05:30 INFO service=fulfillment-service instance=fulfill-7 traceId=f901ac spanId=ab11 eventId=evt-hot-77 result=processed totalMs=3827 commitRequested=true
```

Structured stage fields permit aggregation.
The partition field exposes skew.
The DB acquire line does not prove a slow database query.
The 429 line does not alone prove fraud is the only bottleneck.
Metrics across the incident window establish contribution and scale.

# Commands / Tools

```text
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --describe
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1 --describe
```

The first exposes per-partition lag and ownership.
The second exposes partition leaders, replicas, and ISR.

Windows:

```text
curl.exe -s http://localhost:8080/actuator/metrics/hikaricp.connections.pending
curl.exe -s http://localhost:8080/actuator/threaddump
```

Linux:

```text
curl -s --max-time 5 http://localhost:8080/actuator/prometheus
ss -ntp
```

Thread dumps show blocked or waiting stacks at an instant.
They do not provide rate history.
Prometheus supplies fleet trends when labels are bounded.
Database performance views should be queried read-only with approved access.
Increasing partitions, resetting offsets, and pausing production traffic are governed actions.
The commands shown here do not mutate Kafka state.

# Root Cause

Two related capacity constraints caused the slow consumer.

```text
Marketplace customer creates a hot customerId key
    |
61% of backlog lands on partition 7
    |
one ordered lane limits that key
    |
fraud API returns 429
    |
immediate retries increase attempt rate
    |
handlers hold DB connections longer
    |
Hikari reaches 40/40 and callers wait
    |
consume rate falls below produce rate
```

Kafka fetch latency and broker replication were healthy.
Low CPU reflected waiting on fraud and Hikari.
The key skew limited parallelism while retries reduced sustainable downstream capacity.

# Fix

Immediate mitigation:

* Reduce fraud-call concurrency to its safe rate.
* Disable immediate retries and honor bounded backoff with jitter.
* Pause optional enrichment through an approved feature flag.
* Drain backlog at a rate the database and fraud API sustain.
* Do not add consumers beyond partitions or downstream limits.

Permanent fix:

* Batch eligible database writes and shorten transaction scope.
* Cache immutable fraud decisions where policy allows.
* Use bounded queues with pause/resume backpressure.
* Redesign the key only after defining required ordering scope.
* If one customer need not be strictly ordered, shard that key using a documented strategy.
* Capacity-test dependencies and consumer recovery headroom.
* Add retry budgets and rate limiting.

# Verification

Before:

* Produce: 6,000/min.
* Successful consume: 4,300/min.
* Lag slope: +1,700/min.
* Partition 7 share of lag: 61%.
* Handler p99: 3.8 seconds.
* Hikari pending: 57.
* Fraud attempts: 2.6 per affected event.

After:

* Successful consume: 7,400/min during drain.
* Lag slope: -1,400/min.
* Handler p99: 410 ms.
* Hikari pending: 0.
* Fraud attempts: 1.08 per event.
* Oldest event age falls below 15 seconds.
* Error and duplicate-effect rates remain at zero.

I keep the test long enough to prove sustainable, not burst, throughput.
I verify the hot partition drains rather than only the total lag.

# Prevention

* Dashboard each processing stage and partition.
* Alert on event age, lag slope, queue occupancy, and pool pending.
* Capacity-test at peak plus backlog recovery.
* Document key selection and ordering requirements.
* Detect heavy-hitter key categories without exposing customer values.
* Use bounded queues and controlled pause/resume.
* Set dependency concurrency budgets.
* Use limited retries with exponential backoff and jitter.
* Keep DB transactions short.
* Canary performance-affecting changes.

# Interview Answer

### What I would say in an interview

For a slow consumer, I compare produced versus successfully committed work, then break latency into fetch, deserialization, local queue, handler, dependency, and commit stages. I inspect per-partition lag because Kafka parallelism is partition-bound and key skew can make one lane hot. Here fetch was fast and CPU was low, but partition 7 had a marketplace hot key, fraud calls were throttled and retried, and Hikari acquisition queued. More consumers would not split that partition and could overload dependencies. We bounded concurrency, removed immediate retries, shortened DB work, and planned a key strategy consistent with ordering. I verified sustained excess consume capacity, falling per-partition lag, low event age, and unchanged correctness.

### Common interviewer traps

* More consumers do not fix a single hot partition.
* Low CPU can mean blocked threads, not spare end-to-end capacity.
* DB acquisition delay is not SQL execution delay.
* Unbounded queues hide lag in memory.
* Throughput without correctness is not recovery.

### Quick memory flow

Rate math -> partition skew -> assignments -> fetch -> deserialize -> queue -> handler -> dependencies -> commits -> safe backpressure -> sustained verification.

# Interview Follow-up Questions

1. **What limits Kafka parallelism?** Usable partitions, assigned consumers, processing design, and downstream capacity.
2. **What is key skew?** Disproportionate traffic maps to one partition because many events share or collide on keys.
3. **Why not add 30 consumers to 12 partitions?** At most 12 can own partitions in that group.
4. **What is backpressure?** A bounded way to slow or pause intake when processing capacity is full.
5. **How do retries reduce throughput?** They add attempts and hold processing resources while the dependency is already constrained.
6. **Why compare event age and lag?** Lag counts records; age measures customer delay.
7. **When can a key be changed?** Only after reviewing ordering, compatibility, repartitioning, and rollout effects.
8. **How do you prove the fix is sustainable?** Run beyond transient caches and bursts while rates, queues, dependencies, and correctness remain stable.
