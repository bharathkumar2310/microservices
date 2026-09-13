# Problem

Orders are accepted, but fulfillment events are processed increasingly late.
Kafka consumer lag is the distance between available records and the group's committed progress.
For partition p, record lag is approximately `LEO(p) - committedOffset(p)`.
LEO is the broker's log-end offset: the offset after the newest record.
Committed offset is the next record the group says it should read after a restart.
Consumer position is different: it is the next record in the current member's local fetch flow.
Position can be ahead of committed offset while a batch is still processing.
Lag is not time.
Ten records may mean milliseconds on a busy partition or hours on a quiet one.
Event age, such as `now - event.createdAt`, measures business staleness.
I inspect record lag and event age together.

# Production Situation

This starts a continuing incident at Meridian Shop.
`order-service` publishes `OrderCreated` events to `orders.v1`.
`fulfillment-service` consumes them as group `fulfillment-v3`.
The topic has 12 partitions and replication factor 3.
Normal produce rate is 8,000 records/min.
Normal sustainable consume rate is 10,500 records/min.
After a promotion, produce rate rises to 12,000/min.
Consume rate remains 8,700/min.
Total lag grows by about 3,300/min.
Lag moves from 4,000 to 202,000 in one hour.
Oldest unprocessed event age grows from 8 seconds to 19 minutes.
CPU is 48%, heap is 61%, and GC pause p99 is 31 ms.
All six consumer members are alive.
Business symptom: customers receive shipment estimates late.

# Architecture

```text
Checkout API
    |
    | send OrderCreated(key=customerId)
    v
Kafka cluster: orders.v1
12 partitions, 3 replicas each
    |
    | group=fulfillment-v3, 6 members
    v
Fulfillment Service
    |
    +--> inventory-api
    |
    +--> shipment database
```

A broker stores topic partitions.
A topic is the named stream; a partition is an ordered append-only shard.
Each partition has one leader handling reads and writes.
Followers replicate the leader.
ISR means in-sync replicas sufficiently caught up to be eligible for safe leadership.
A consumer group provides one logical subscription.
Members are the live consumers in that group.
The coordinator assigns each partition to at most one member in that group.
Six members can process at most six partitions concurrently per poll-thread setup here.
Twelve members could use all 12 partitions; a thirteenth would be idle.

# What I Check FIRST

1. Check lag by topic and partition, not only the total.
   WHY: one hot partition and global insufficient capacity require different fixes.
   LOOK FOR: all partitions rising together versus one partition dominating.
2. Compare produce rate with sustainable consume rate over the same interval.
   WHY: if input exceeds output, arithmetic alone explains growing backlog.
   LOOK FOR: `produce - consume` matching the observed lag slope.
3. Check oldest-event age and business completion latency.
   WHY: offset lag does not state how late customers are affected.
   LOOK FOR: age rising with lag and SLO violations.
4. Check assignments, rebalances, member count, and processing latency.
   WHY: unused partitions, churn, or slow handlers reduce effective throughput.
   LOOK FOR: stable ownership and where time is spent.
5. Check broker health, partition leadership, replicas, and ISR.
   WHY: a broker problem can throttle fetches even when application CPU is low.
   LOOK FOR: all leaders available and ISR at expected replication factor.

# Step-by-Step Investigation

### Step 1 - Confirm scope and time

* What I check: incident start, affected group, topic, region, and deployment timeline.
* Why: lag may belong to another group or an intentionally inactive subscriber.
* Tool: Grafana group/topic dashboard and deployment markers.
* Expected: `fulfillment-v3` began rising at 18:05 with promotion traffic.
* Bad: several groups and topics rise at once.
* Meaning: suspect broker, network, or shared downstream capacity.
* Next: compare cluster-wide fetch and broker request metrics.

### Step 2 - Reconcile the lag arithmetic

* What I check: per-minute produced, successfully processed, and committed records.
* Why: a backlog is expected when arrival exceeds sustainable service rate.
* Calculation: `12,000 - 8,700 = 3,300 records/min`.
* Expected: dashboard lag slope is near 3,300/min.
* Bad: lag rises much faster than the rate difference.
* Meaning: commits may be failing, partitions may be replaying, or metrics differ in scope.
* Next: inspect commit failures and member churn.

### Step 3 - Inspect partitions and key distribution

* What I check: LEO, committed offset, lag, consume rate, and key cardinality per partition.
* Why: Kafka parallelism is partition-bound and keys choose partitions.
* Expected: load is reasonably spread over 12 partitions.
* Bad: partition 7 owns 58% of traffic while others are nearly current.
* Meaning: a popular key or poor partitioner creates key skew.
* Next: identify the dominant key class without logging sensitive payloads.
* Important: adding consumers cannot split one partition between group members.

### Step 4 - Inspect group membership and assignments

* What I check: six members, assigned partitions, state, assignment age, and rebalances.
* Why: consumers do useful work only for partitions they currently own.
* Expected: `Stable`, two partitions per member, no repeated revocation.
* Bad: `PreparingRebalance` repeatedly or only four active members.
* Meaning: member crashes, slow polls, rollout churn, or coordinator connectivity.
* Next: inspect heartbeat and `max.poll.interval.ms` evidence.

### Step 5 - Separate fetch from processing

* What I check: poll latency, records per poll, fetch rate, listener queue depth, handler time.
* Why: records pass through fetch, deserialize, queue, handler, dependency, then commit.
* Expected: brokers return data quickly and handler throughput matches input.
* Bad: fetch is fast but handler p95 is 370 ms.
* Meaning: Kafka is delivering; application or downstream work is limiting throughput.
* Next: split handler spans into inventory and database operations.

### Step 6 - Check poll and liveness timers

* What I check: `poll()` cadence, heartbeat rate, `session.timeout.ms`, and `max.poll.interval.ms`.
* Why: heartbeats show process liveness; poll interval limits processing time between polls.
* Expected: poll gaps remain far below max poll interval.
* Bad: poll gap approaches the configured five minutes.
* Meaning: a rebalance risk exists even before obvious member death.
* Next: inspect batch size and blocking calls in the listener.

### Step 7 - Check broker and replication health

* What I check: leader count, offline partitions, under-replicated partitions, ISR shrink, fetch latency.
* Why: consumers fetch from leaders, and replication distress can degrade availability.
* Expected: one leader per partition, no offline partitions, ISR size 3.
* Bad: ISR is 1 for several partitions and fetch p99 jumps.
* Meaning: broker or network pressure is a contributing factor.
* Next: involve the Kafka platform owner with exact partitions and timestamps.

### Step 8 - Test capacity safely

* What I check: per-instance throughput during a controlled canary scale from six to twelve members.
* Why: 12 partitions permit up to 12 active members for this group.
* Expected: consume rate rises above 12,000/min and lag slope turns negative.
* Bad: throughput stays near 8,700/min.
* Meaning: downstream capacity, hot partitions, or a shared lock prevents scaling.
* Next: inspect dependency saturation and partition-level rates.

### Step 9 - Estimate recovery time

* What I check: backlog divided by spare sustained throughput.
* Why: "lag is falling" is not a customer recovery estimate.
* Calculation: with output 15,000/min, spare rate is 3,000/min.
* Calculation: `202,000 / 3,000` is about 68 minutes.
* Bad: event age still rises while record lag falls.
* Meaning: newer high-volume partitions may hide an old blocked partition.
* Next: inspect oldest event age per partition.

# Metrics to Check

| Metric | High or rising means | Low or falling means |
|---|---|---|
| produced records/sec | more offered load | traffic drop or producer trouble |
| consumed records/sec | healthy only if useful processing succeeds | bottleneck, idle assignment, or no input |
| committed records/sec | durable group progress | commit delay or processing stall |
| records lag max | at least one partition is behind | group is close to LEO |
| records lag sum | aggregate backlog | not proof every partition is healthy |
| oldest event age | customer-visible staleness | timely processing |
| listener p95/p99 | slow handler tail | handler likely not limiting |
| poll gap max | rebalance risk near max interval | timely polling |
| rebalance count | assignment churn | stable ownership |
| assigned partitions | actual parallel work | idle or missing members |
| fetch latency | broker/network delay | fetching is probably healthy |
| bytes fetched/sec | data movement volume | empty topic or fetch problem |
| queue depth | application backpressure | queue is drained |
| dependency latency | downstream bottleneck | look elsewhere |
| under-replicated partitions | replica distress | normal replication |

If lag changes immediately after deployment, compare listener concurrency, batch size, and commit mode.
If CPU is low while lag grows, do not conclude there is spare capacity.
Threads may be waiting on inventory, database connections, rate limiters, or locks.
If consumed rate is high but committed rate is low, verify successful effects and commit behavior.
If total lag is flat but event age rises, look for a quiet, stuck partition.

# Distributed Trace Investigation

Kafka traces follow message context rather than one synchronous HTTP call.
The producer injects W3C `traceparent` and a stable `eventId` into Kafka headers.
It creates a producer span before send and records acknowledged topic, partition, and offset.
The consumer extracts context and creates a new consumer/process span.
For long asynchronous delay, keep both `producedAt` and `consumedAt`.

```text
traceId=7ab91c
checkout HTTP span                         82 ms
  |
  +-- kafka produce orders.v1             14 ms
      ack partition=7 offset=881204

queue time                                 17 min 42 sec

fulfillment consume span                  391 ms
  |
  +-- inventory-api                       344 ms
  +-- shipment-db                          31 ms
```

The producer span proves the broker acknowledged a particular record.
The producer's intent log alone does not prove Kafka accepted anything.
Consumer queue time shows business delay even when handler latency is acceptable.
If inventory spans dominate, Kafka fetch is not the primary bottleneck.
Retry spans reveal amplification if one event causes repeated dependency calls.
A missing consumer child span can mean propagation was absent, sampling dropped it, or processing never began.
It does not by itself prove message loss.
I correlate `eventId`, topic, partition, and offset when trace continuity is missing.

# Distributed Logs

```text
2026-09-13T18:05:14.221+05:30 INFO service=order-service instance=order-6c8f traceId=7ab91c spanId=11aa eventId=ord-93841 action=kafka_send_intent topic=orders.v1 keyHash=8e91
2026-09-13T18:05:14.235+05:30 INFO service=order-service instance=order-6c8f traceId=7ab91c spanId=11aa eventId=ord-93841 action=kafka_send_ack topic=orders.v1 partition=7 offset=881204 durationMs=14
2026-09-13T18:22:56.410+05:30 INFO service=fulfillment-service instance=fulfill-2 traceId=7ab91c spanId=92bc eventId=ord-93841 group=fulfillment-v3 topic=orders.v1 partition=7 offset=881204 eventAgeMs=1062175
2026-09-13T18:22:56.801+05:30 INFO service=fulfillment-service instance=fulfill-2 traceId=7ab91c spanId=92bc eventId=ord-93841 result=processed handlerMs=391 commitRequested=true
```

I search by `eventId`, then verify the same topic, partition, and offset.
I distinguish send intent from send acknowledgment.
The acknowledgment identifies what Kafka accepted.
The consume log proves this member received that record.
The processed log proves handler completion, not necessarily offset commit or external correctness.
A separate commit-success metric or callback confirms commit progress.
One slow log is an example, not proof of the population-wide root cause.
I require matching rate, partition, trace, and dependency evidence.
Payloads, credentials, and raw customer identifiers must not be logged.

# Commands / Tools

Read-only group inspection:

```text
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --describe
```

This shows current offset, log-end offset, lag, and ownership.
It does not explain why the handler is slow.

Read-only topic inspection:

```text
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1 --describe
```

This shows partitions, leaders, replicas, and ISR.
It does not prove end-to-end business processing.

Windows application metric query:

```text
curl.exe -s http://localhost:8080/actuator/metrics/spring.kafka.listener
```

Linux application metric query:

```text
curl -s --max-time 5 http://localhost:8080/actuator/prometheus
```

Actuator access must follow production authentication and exposure policy.
Prometheus and Grafana are preferred for fleet-wide history.
Jaeger or Zipkin is used for sampled producer and consumer traces.
CLI commands here are descriptive, read-only examples.
Offset resets, partition changes, or replay commands are governed actions.
I do not run them during diagnosis without approval, a bounded scope, and a rollback plan.

# Root Cause

The promotion raised input above sustainable output.

```text
Produce rate rises to 12,000/min
    |
Consumer capacity remains 8,700/min
    |
Backlog grows 3,300/min
    |
Event age rises
    |
Shipment estimates arrive late
```

Inventory API p95 rose from 120 ms to 340 ms under the same promotion.
Each listener thread spent most time waiting for inventory.
Six members used only six concurrent partition processing lanes.
Broker leaders and ISR were healthy.
No rebalance storm occurred.
Therefore Kafka availability was not the root cause.
The causal root was insufficient sustainable end-to-end consumer capacity.

# Fix

Immediate mitigation:

* Rate-limit nonessential inventory enrichments.
* Scale the group from six to twelve healthy members, matching 12 partitions.
* Confirm inventory has capacity before shifting more load to it.
* Prioritize oldest orders using an approved business process, not an offset hack.
* Avoid blind offset reset because it can skip work or create duplicates.

Permanent fix:

* Batch or cache safe inventory reads.
* Bound dependency timeouts below `max.poll.interval.ms`.
* Use listener concurrency that matches partitions and measured downstream capacity.
* Add backpressure so intake does not create unbounded in-memory queues.
* Revisit the event key only if ordering requirements permit better distribution.
* Increase partitions only through capacity planning because ordering and key mapping change.

# Verification

Before:

* Produce rate: 12,000/min.
* Consume rate: 8,700/min.
* Lag slope: +3,300/min.
* Lag sum: 202,000.
* Oldest event age: 19 minutes.
* Inventory p95: 340 ms.

After:

* Produce rate: 11,800/min.
* Consume rate: 15,100/min while draining.
* Lag slope: about -3,300/min.
* Lag reaches under 2,000 after about 61 minutes.
* Oldest event age returns below 12 seconds.
* No increase in dependency errors, duplicates, or rebalances.

Negative slope proves spare throughput.
Low event age proves business freshness, not merely offset movement.
Stable error and duplicate rates prove the drain did not trade correctness for speed.
After backlog clears, steady consume rate should converge to produce rate.

# Prevention

* Alert on lag slope and oldest-event age, not only an absolute lag threshold.
* Dashboard rates, lag, assignments, rebalances, handler stages, and dependency latency.
* Load-test at peak rate plus recovery headroom.
* Capacity-plan partitions and consumers together.
* Detect partition skew and key-cardinality changes.
* Bound queues and expose queue occupancy.
* Set dependency bulkheads and timeouts from measured budgets.
* Use cooperative rebalancing where supported and tested.
* Record stable event IDs and acknowledged Kafka coordinates.
* Run deployment canaries and compare consume capacity before full rollout.

# Interview Answer

### What I would say in an interview

I first scope lag by group, topic, and partition, then compare input and successful committed output over the same window. If producers send 12,000 records a minute and consumers sustain 8,700, a 3,300-per-minute lag increase is expected. I also check oldest-event age because offset lag is not time. Next I verify assignments, rebalance activity, poll gaps, handler stages, downstream latency, broker leaders, replicas, and ISR. In this case fetch and brokers were healthy, but inventory latency limited six members. We safely scaled to the 12-partition ceiling and reduced dependency cost. I verified negative lag slope, restored event age, stable errors, and no duplicate increase.

### Common interviewer traps

* Adding consumers beyond partition count does not add active partition parallelism.
* Low CPU does not prove spare capacity when threads wait downstream.
* Zero total lag can hide business failure if effects fail after an unsafe commit.
* A send attempt is not an acknowledged Kafka record.
* Offset lag alone does not state event age.

### Quick memory flow

Symptom -> group/topic -> per-partition lag -> rate math -> event age -> assignments -> stages -> broker -> root cause -> safe capacity -> verify correctness.

# Interview Follow-up Questions

1. **What is consumer lag?** For each partition, it is approximately LEO minus the group's committed next offset.
2. **Position versus committed offset?** Position is current in-memory fetch progress; committed offset is restart progress stored for the group.
3. **Why can lag rise with low CPU?** Consumers can wait on a database, API, lock, rate limiter, or queue.
4. **Will more consumers always help?** No; active consumers are capped by partitions and downstream capacity.
5. **How do you convert lag to recovery time?** Divide backlog by sustained consume rate minus current produce rate.
6. **Why check event age?** Equal record lag can represent very different customer delay.
7. **What does ISR tell you?** Which replicas are sufficiently caught up; shrinkage signals replication distress.
8. **Can Kafka guarantee external exactly-once effects?** Not by itself; use idempotent effects, stable IDs, and database constraints or inbox patterns.
