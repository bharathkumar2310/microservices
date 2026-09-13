# Production Troubleshooting Study Chapter: Kafka and Asynchronous Communication

## Purpose and safety

This chapter teaches Kafka fundamentals and a production method for diagnosing delivery, lag, duplicates, rebalances, ordering, poison records, and downstream failures.

> Use read-only inspection first. Substitute the sample names with approved environment values and use the cluster's approved authentication method. Consumer offsets, topic configuration, retention, partition counts, and ACLs are production state: do not reset offsets, delete records/topics/groups, increase partitions, or alter configuration during diagnosis without owner approval and a recovery plan. Redact payloads, keys, credentials, and personal data.

Kafka can provide durable transport and strong guarantees within defined boundaries, but **end-to-end exactly-once is not the default**. A consume-process-side-effect pipeline normally has at-least-once behavior unless the complete design, including external effects, implements a stronger protocol. Design consumers to tolerate redelivery.

## Learning goals

After studying this chapter, you should be able to:

1. Explain brokers, topics, partitions, replicas, offsets, leaders, producers, consumer groups, and group assignments.
2. Calculate and interpret lag without confusing it with time delay.
3. Explain at-least-once delivery, commit timing, redelivery windows, and apparent message loss.
4. Diagnose slow consumption, skew, hot partitions, rebalances, polling and heartbeat failures, and broker/producer faults.
5. Design bounded retries, dead-letter handling, idempotency, outbox/inbox patterns, backpressure, and observability.

---

# 1. Architecture and mental models

## 1.1 Record flow

```text
business transaction
   |
producer: serialize -> choose partition -> batch/compress -> send
   |
bootstrap broker -> current partition leader
   |
leader appends record at offset N
   +-> replicas copy log (durability depends on acks/min ISR)
   |
consumer group coordinator assigns partitions
   |
consumer fetches batches -> poll returns records
   |
application processes -> external/database effects
   |
consumer commits "next offset to read"
```

- A **broker** is a Kafka server.
- A **topic** is a named append-only stream divided into partitions.
- A **partition** is an ordered log. Offsets are monotonically increasing positions within that partition, not global message IDs.
- A **leader** serves reads/writes for a partition; replicas provide fault tolerance.
- A **record key** normally determines partition and therefore ordering/affinity.
- A **consumer group** shares partitions: at a given time, one partition is assigned to at most one consumer within that group.
- Different groups consume the same topic independently.

If a topic has 12 partitions and a group has 8 healthy consumers, up to 8 consume concurrently. With 20 consumers, at least 8 are idle. Adding consumers beyond partition count does not add partition parallelism.

## 1.1.1 Beginner walk-through: one order event

Assume checkout creates this logical event:

```text
event_id: evt-9001
key: customer-42
topic: order-created
partition: 3
offset: 815
```

1. The producer serializes the event and hashes `customer-42` to choose partition 3.
2. The producer sends it to partition 3's leader. A successful delivery callback returns partition 3 and offset 815.
3. Replicas copy the record. The exact acknowledgement depends on producer and broker durability settings.
4. Consumer group `billing-v1` has partition 3 assigned to one member, perhaps consumer B.
5. Consumer B polls and receives offset 815 in a batch.
6. Billing charges through an idempotent operation keyed by `evt-9001`.
7. After the charge is durably recorded, the consumer commits 816, meaning "next read begins at 816."

Now consider failure points:

| Failure point | What the system knows | Likely behavior |
|---|---|---|
| Before broker acknowledgement | Send outcome may be unknown | Producer policy may retry |
| After broker write, before callback reaches producer | Record may exist despite producer timeout | Retry can produce another attempt |
| Before consumer side effect | Committed offset remains before 815 | Record is redelivered |
| After side effect, before offset commit | Effect exists, commit does not | Record is redelivered; idempotency matters |
| After offset commit, before non-atomic effect | Group starts after 815 | Effect can be lost |

This example explains why production diagnosis must track both the Kafka identity `(topic, partition, offset)` and a stable business identity such as `event_id`.

## 1.2 Offset positions and lag math

For a partition:

```text
log-start offset <= committed group offset <= log-end offset

record lag = log-end offset - committed group offset
```

The committed offset is usually the **next** record to consume. If log end is 10,000 and committed is 9,400, lag is about 600 records.

Lag is not time:

```text
estimated drain time ~= current lag / (consume rate - produce rate)
```

This is meaningful only when rates are stable and consume rate exceeds produce rate. At 1,000 records lag, the delay could be seconds or hours depending on record rate and processing cost. Track **record age/event-time delay** as well as offset lag. Also distinguish:

- **Position:** next offset the current consumer will fetch/process.
- **Committed offset:** durable group checkpoint used after reassignment/restart.
- **Log end offset (LEO):** next offset on the broker leader.
- **High watermark:** the boundary up to which records have been replicated to the required in-sync replicas and can be exposed to consumers. Transactional `read_committed` consumers can have an additional visibility boundary, the Last Stable Offset (LSO), so do not treat the high watermark and LSO as the same concept.

## 1.3 Delivery semantics and commit timing

### Commit after processing: common at-least-once pattern

```text
poll record N -> perform side effect -> crash before commit N+1
restart at committed N -> process N again
```

No acknowledged work is skipped, but duplicates are possible.

### Commit before processing: at-most-once risk

```text
poll record N -> commit N+1 -> crash before side effect
restart at N+1 -> N is skipped by this group
```

This can create real application-level loss.

### Kafka transactions

Idempotent producers prevent certain duplicate log appends from producer retries. Kafka transactions can atomically write output records and consumed offsets **inside Kafka** when correctly configured, and consumers can use `read_committed`. They do not atomically include a normal HTTP call, email, or arbitrary database update. Do not claim end-to-end exactly-once without naming every boundary and protocol.

## 1.4 Polling, heartbeats, and group membership

Key consumer settings are client/version specific:

- `max.poll.interval.ms`: maximum allowed time between `poll()` calls before the group treats the consumer as failed.
- `session.timeout.ms`: coordinator's failure-detection window when heartbeats stop.
- `heartbeat.interval.ms`: heartbeat frequency for classic group protocols; normally below session timeout.
- `max.poll.records`: maximum records returned per poll; it controls batch work, not fetch bytes alone.
- Fetch min/max wait and partition/total fetch byte limits affect batching and large records.

Some clients heartbeat in a background thread, but that does **not** permit processing longer than `max.poll.interval.ms`. If processing a poll batch takes too long, partitions can be revoked and reassigned while work is incomplete. Static membership and cooperative assignment can reduce avoidable movement but do not fix blocked processing.

## 1.5 Ordering scope

Kafka guarantees append order **within one partition** as observed under the relevant producer/consumer configuration. It does not provide global topic ordering across partitions. Ordering can still appear broken when:

- related records use different or null keys;
- partition count or partitioner changes alter mapping;
- a producer retries under unsafe legacy settings;
- consumers process one partition concurrently and complete out of order;
- retries/DLT replay an older event later;
- observers compare event time rather than log offset;
- multiple producers have no shared sequencing contract.

If order matters per entity, use a stable entity key, preserve sequential handling per key/partition, and make stale events detectable with entity version/sequence.

## 1.6 Retry, DLT, poison, and backpressure model

```text
main topic
   -> consumer
      +-> success: durable effect + commit
      +-> transient failure: bounded retry with backoff/jitter
      +-> persistent/data failure: publish retry/DLT envelope durably, then commit
      +-> overload: pause/bound intake, do not create unbounded memory queues
```

Retry topics can free the main partition, but they relax original ordering and require lifecycle/observability. In-memory sleeps block progress. Infinite immediate retry causes a poison record to pin a partition and hammer dependencies.

## 1.7 Outbox and inbox

### Transactional outbox

```text
database transaction:
  update business row
  insert outbox event
commit once

relay/CDC publishes outbox rows to Kafka
```

This closes the "database committed but publish failed" dual-write gap. The relay can publish duplicates, so consumers remain idempotent.

### Consumer inbox/idempotency

```text
database transaction:
  insert message_id into inbox with UNIQUE constraint
  if newly inserted: apply business change
commit
then commit Kafka offset
```

A duplicate message ID makes the database transaction a no-op. Choose an event ID and idempotency scope/retention consistent with replay needs. For operations such as "set balance to X at version V," state/version checks are safer than repeating "add X."

## 1.8 Glossary

| Term | Meaning |
|---|---|
| ISR | In-sync replicas sufficiently caught up to participate in durability rules |
| `acks` | Producer acknowledgement requirement (`0`, `1`, or `all`) |
| Idempotent producer | Producer mode preventing duplicates from supported retries within Kafka producer guarantees |
| Offset | Position of a record within one partition |
| Lag | Difference between log end and group committed offset |
| Consumer position | Next offset the live consumer plans to fetch/process |
| Rebalance | Group assignment changes among members |
| Tombstone | Key with null value, commonly used for deletion in compacted topics |
| Retention | Time/size policy for log deletion; not a consumer acknowledgement policy |
| Compaction | Retains latest value per key eventually, subject to configuration |
| DLT/DLQ | Dead-letter topic/queue for records requiring separate handling |
| Poison record | Record that predictably fails normal processing |
| Backpressure | Bound or slow intake when processing/downstream capacity is insufficient |
| Idempotency | Repeating an operation produces no additional incorrect effect |
| Outbox/inbox | Durable patterns joining business state with publish/deduplication intent |
| Reprocessing | Deliberate replay from retained records or a retry/DLT stream |

## 1.9 Beginner guide to reading group evidence

Assume a topic has four partitions and group `shipping-v1` reports:

```text
PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  OWNER
0          1000            1000            0    consumer-A
1          450             1450         1000    consumer-B
2          900              900            0    consumer-A
3          700              700            0    consumer-B
```

The group is not uniformly slow. Partition 1 is the only backlog. Adding consumers will not split partition 1; one member owns a partition at a time in the group. Next ask:

- Does partition 1 receive more records or bytes?
- Is it stuck on one offset?
- Does its leader broker have a problem?
- Do its keys map to an expensive tenant?
- Is consumer B slow for partition 3 as well?

If partition 1 moves to consumer A during a normal deployment and remains slow, the problem follows the partition. If every partition on consumer B is slow, inspect B's node, resources, release, and downstream path.

Now consider:

```text
time    LEO    committed    lag
10:00   1000   1000           0
10:01   1600   1300         300
10:02   2200   1600         600
```

Production is about 10 records/s while committed progress is about 5 records/s. Lag grows about 5 records/s. If the oldest unprocessed record is already 20 minutes old, user impact is clearer than the record count alone.

## 1.10 Beginner guide to commits and visible effects

Offsets are group checkpoints, not message deletion. The same Kafka record remains in the partition according to retention, even after many groups consume it.

```text
committed offset 816 means:
"when this group resumes, its next position is normally 816"
```

It does not mean:

- offset 815 was processed correctly;
- an HTTP request or database transaction succeeded;
- every other group processed 815;
- Kafka deleted offset 815;
- no duplicate attempt can occur.

For an external database effect, reason about two durable systems:

```text
Kafka group offset state
Database business/inbox state
```

Without a shared protocol, they cannot be committed in one ordinary atomic transaction. Commit-after-database favors at-least-once:

```text
database effect succeeds
crash before Kafka commit
record returns
durable idempotency prevents second effect
```

Commit-before-database favors at-most-once loss:

```text
Kafka commit succeeds
crash before database effect
group resumes later
record is skipped
```

This is why "duplicates happened" is often a request to improve idempotency, not to move the offset commit earlier.

## 1.11 Worked end-to-end lag diagnosis

At 09:00, `invoice-v2` lag is zero. A rollout begins at 09:05. At 09:08:

```text
produce rate:              400 records/s (unchanged)
successful consume rate:   120 records/s (was 450)
rebalance rate:             12/min (was near zero)
handler p99:                 7s (was 150ms)
downstream DB acquire p99: 6.5s
```

The data flow is:

```text
new release opens a DB transaction per item
 -> DB pool saturates
 -> handler batches exceed max poll interval
 -> members are removed and partitions reassign
 -> unfinished offsets are attempted again
 -> useful consume rate falls
 -> lag and duplicate attempts rise
```

Broker health and producer rate are normal, so a broker scale-up is not supported by evidence. Rolling back is a safe reversible mitigation. The root fix batches database work, bounds processing concurrency to pool capacity, and sets the poll batch from measured worst-case duration. Recovery is not "lag stopped growing"; successful consume rate must exceed production, oldest-event age must fall, and duplicate effects must remain prevented by the inbox.

---

# 2. Core metrics, tools, and generic workflow

## 2.1 Core metrics

### Producer

- send rate, error rate by exception, retry rate, record-error rate;
- request latency, request timeout, throttle time;
- batch size, compression ratio, record queue time, buffer available/exhaustion;
- record size, serialization failures;
- acknowledgements, delivery callback failures, producer fencing/transaction errors.

`send()` returning a future is not proof of broker acknowledgement. Observe/await the delivery result according to the application's durability contract.

### Broker/topic/partition

- broker availability, controller events, leader changes;
- offline partitions, under-replicated partitions, ISR shrink/expand;
- produce/fetch request latency and errors;
- bytes/records in/out, disk utilization/latency, network and request-handler idle;
- per-partition ingress, size, and leader distribution;
- retention/compaction and log-start offsets.

### Consumer/group

- records consumed/s and processing success/failure rate;
- per-partition committed offset, LEO, record lag and lag trend;
- oldest unprocessed event age/business delay;
- poll rate/idle ratio, batch size, processing p50/p95/p99;
- commit latency/failures, fetch latency/errors;
- heartbeat/coordinator failures, member count, assignment, rebalance count/duration;
- retry/DLT publish and consume rate, poison count, duplicate/idempotency conflict count;
- downstream latency/errors, local worker queue depth, CPU/memory/GC.

Fleet-average lag hides a hot partition. Always retain topic, partition, group, cluster, region, client version, and instance dimensions, but do not use message IDs or keys as metric labels.

## 2.2 Read-only command examples

Exact script paths and flags vary by Kafka distribution/security.

```bash
# Group state, members, assignment, current offsets, log end, and lag
kafka-consumer-groups.sh \
  --bootstrap-server <broker:9092> \
  --group <group-id> \
  --describe

# Topic partition/leader/replica/ISR metadata
kafka-topics.sh \
  --bootstrap-server <broker:9092> \
  --topic <topic> \
  --describe

# Configuration inspection (read-only describe)
kafka-configs.sh \
  --bootstrap-server <broker:9092> \
  --entity-type topics \
  --entity-name <topic> \
  --describe
```

Interpret group output:

```text
CURRENT-OFFSET  LOG-END-OFFSET  LAG
9400            10000           600
```

A blank current offset can mean the group has never committed that partition. A negative or unavailable value can be a transient/command/version condition; verify broker/client telemetry. Never paste secrets into shell history. Use approved config files and access controls.

## 2.3 Generic production workflow

1. **Define impact and time.** Which business events, topic, group, partitions, region, and interval?
2. **Clarify "produced/processed/lost."** Application attempted send, broker acknowledged, record exists, fetched, handler succeeded, offset committed, and side effect visible are separate facts.
3. **Inspect group and partition state.** Members, assignments, offsets, LEO, lag trend, event age, hot partitions, and rebalances.
4. **Follow one safe correlation identity.** Event ID, key hash, topic-partition-offset, producer trace, consumer trace, idempotency record, and downstream request; avoid exposing payload.
5. **Compare demand with capacity.** Produce rate versus successful consume rate, processing duration, batch size, partition count, and available consumers.
6. **Check each layer.** Producer callbacks/errors; broker leaders/ISR/disk/throttle; network/auth; consumer poll/heartbeat/commit; handler/downstream.
7. **Correlate changes.** Deployment, schema, traffic/key distribution, partitions, ACL/certificates, broker event, downstream incident.
8. **Mitigate without destroying evidence.** Throttle producers, scale only if partitions/capacity allow, pause bad workload, protect downstream, route poison records via approved flow.
9. **Correct and replay safely.** Fix code/config/data, prove idempotency, then use an approved bounded replay plan.
10. **Verify and prevent.** Lag trend falls, event age recovers, success and side effects reconcile, duplicates remain harmless, and alerts/runbooks/tests are added.

---

# 3. Original interview questions

## 1. Kafka messages are being produced successfully, but consumers are not processing them. How would you investigate?

**Precise meaning.** "Produced successfully" must mean the broker acknowledged the record, not merely that application code called `send`. "Not processing" could mean no fetch, handler failure, commit/visibility issue, or wrong consumer.

**Where it can happen.** Producer topic/cluster, broker partition leader, ACL/TLS/network, group assignment, subscription, offset position, deserializer, poll loop, paused partition, handler, or downstream.

**Causal mechanisms.** Wrong environment/topic/group; producer callback failed; no active members; no partition assignment; ACL/auth failure; consumer starts at latest after earlier records; partitions paused; deserialization error before handler logs; handler thread blocked; rebalance loop; `read_committed` waits behind an open transaction; record exceeds fetch settings; DLT/retry route.

**Ordered debugging.**

1. Capture event ID/key hash, expected cluster/topic, producer timestamp, and acknowledged topic-partition-offset.
2. Verify topic metadata and that LEO advanced on that partition.
3. Describe the intended group: state, members, assignment, committed offset, LEO, lag.
4. Check consumer deployment/readiness, subscription pattern, group ID, client logs, ACL/TLS/DNS, and deserialization errors.
5. Inspect poll rate, assigned/paused partitions, processing threads, commit errors, and rebalance events.
6. Trace from fetch to handler, idempotency store, retry/DLT, and downstream side effect.
7. Compare a healthy partition/instance and recent config/schema/deployment changes.

### Beginner expansion: follow custody of one record

"Produced" and "processed" are several separate facts:

```text
producer called send
  -> broker acknowledged topic/partition/offset
  -> consumer fetched bytes
  -> deserializer created an object
  -> handler ran
  -> durable business effect succeeded
  -> offset commit succeeded
```

- A fire-and-forget producer can log success before the future/callback later fails.
- A consumer with the wrong group ID may process records, but not as the expected application group.
- A deserialization exception can happen before the normal handler and its first log statement.
- A partition can be assigned but paused by the framework after an error.
- A handler may succeed but its offset commit fail, causing later redelivery rather than absence.
- A record in an open Kafka transaction is not visible to a `read_committed` consumer until commit.

### Why the diagnostic order matters

1. An acknowledged offset gives a precise address; without it, first investigate the producer.
2. LEO proves the partition received data and distinguishes a wrong cluster/topic.
3. Group state reveals whether anyone owns the partition and how far the group has progressed.
4. Runtime checks explain why ownership or decoding is absent.
5. Poll, pause, commit, and rebalance evidence locates the consumer lifecycle stage.
6. End-to-end tracing distinguishes handler execution from durable business outcome.
7. Healthy comparisons expose a partition-, schema-, instance-, or release-specific issue.

### Evidence interpretation

| Evidence | Interpretation | Next action |
|---|---|---|
| No delivery acknowledgement; producer errors | Record not proven in Kafka | Fix producer path |
| Acknowledged offset, but inspecting different cluster/topic | Routing/config mismatch | Correct environment |
| Group has no active members | Consumer deployment/membership failure | Inspect replicas/auth/crashes |
| Member owns partition; committed offset is before record | Backlog, pause, decode, or handler stall | Poll/error/lag investigation |
| Committed offset is after record; no effect | Filter, early commit, dedupe, DLT, or effect visibility | Trace application custody |
| Deserialization counter rises; handler count does not | Failure before handler | Schema/error-handler path |

### Worked mini-example

The producer callback proves `orders-6@1205`. The expected group shows partition 6 assigned and committed offset 1204, but normal handler logs are empty. A deserialization-error metric rose after schema version 9 deployed. The message is not missing and adding consumers cannot help; a compatible reader or governed quarantine path is required.

**Tools, queries, metrics, interpretation.** Producer delivery callback; `kafka-consumer-groups --describe`; topic describe; consumer `records-lag-max`, fetch/poll/commit metrics; broker auth logs; schema-registry metrics. No group members means deployment/membership; member assigned with increasing lag means it cannot keep up/process; committed offset beyond the record means it may already have been handled or skipped.

**Immediate mitigation.** Restore consumer replicas/config/credentials, roll back, unpause through approved application controls, isolate bad schema records, or scale up to partition count if downstream capacity exists.

**Root-cause correction.** Correct topic/group/subscription and startup offset policy, make producer acknowledgement explicit, handle deserialization safely, bound processing, and repair membership/auth.

**Prevention and alerts.** Synthetic canary event, producer acknowledgement/error alerts, no-active-member and event-age alerts, schema compatibility checks, and config/environment validation.

**Common traps.** Looking in the wrong group; assuming lag zero means processing; changing offsets before locating the record; logging sensitive payload; adding consumers beyond partitions.

**Interview-ready answer.** "I prove the producer acknowledgement and topic-partition-offset, verify LEO, then inspect the intended group's members, assignment, committed offset, lag, poll and errors. I trace the record through deserialization, handler, idempotency, retry/DLT, commit, and downstream effect."

## 2. Consumer lag suddenly increases. What could be the reasons?

**Precise meaning.** Committed offsets are advancing more slowly than log ends. It may reflect healthy bursts, slower processing, no commits, or assignment disruption.

**Where it can happen.** Producer demand, partition distribution, broker fetch path, consumer poll/handler/commit, local resources, or downstream.

**Causal mechanisms.** Produce-rate spike; slower database/API; poison retries; hot partition/key skew; fewer consumers; rebalances; long GC/CPU throttling; large messages/batches; broker disk/network throttle; commit failures; deployment; processing completed but commits delayed.

**Ordered debugging.**

1. Graph per-partition lag and oldest-event age; note onset and slope.
2. Compare produce rate `P`, successful consume rate `C`, and estimate whether `C>P`.
3. Inspect group members, assignments, rebalance count/duration, and recent deployments.
4. Compare handler latency/error/retry, downstream latency, poll interval, batch size, CPU/memory/GC.
5. Check hot partitions, key distribution, record size/schema, broker fetch latency/throttle/ISR/disk.
6. Check committed offset versus live consumer position to distinguish processing from commit delay.
7. Verify mitigation by negative lag slope and falling event age, not only more replicas.

### Beginner expansion: how backlog is created

Let producers append 500 records/s while consumers successfully finish 400 records/s:

```text
lag growth = 500 - 400 = 100 records/s
after 10 minutes: about 60,000 additional records
```

If consumers are improved to 800 records/s while production remains 500:

```text
drain rate = 800 - 500 = 300 records/s
rough drain time for 60,000 records = 200 seconds
```

Real traffic varies, so this is an estimate, not a promise.

- A traffic spike raises LEO faster.
- Slow downstream calls reduce completed records/s.
- Rebalances create periods with little or no progress.
- A hot key can overload one partition while other consumers are idle.
- Processing may finish while commits lag, making committed-offset lag temporarily look worse.
- Large records may keep record count modest while byte and CPU load explode.

### Why the diagnostic order matters

1. Per-partition lag plus age establishes user impact and detects skew.
2. Rate math tells whether the problem is excess input or reduced output.
3. Membership and rebalance checks explain missing processing capacity.
4. Handler/downstream/resource metrics locate the reduced output.
5. Partition and payload checks find uneven work hidden by totals.
6. Position-versus-commit separates true processing backlog from checkpoint delay.
7. Negative slope and falling age prove recovery.

### Evidence interpretation

| Pattern | Likely cause |
|---|---|
| Every partition grows after input rate triples | Demand spike |
| One partition grows; peers are current | Hot key, poison record, or leader issue |
| Lag stair-steps during frequent assignments | Rebalance instability |
| Handler p99 matches downstream p99 | Downstream bottleneck |
| Live position advances but committed offset does not | Commit policy/failure |
| Record lag flat but oldest-event age grows | Very low-rate or blocked old record; inspect age directly |

### Worked mini-example

Fleet lag reaches 200,000, but 190,000 belongs to partition 4. That partition receives all records with a null/constant key from one producer version. Adding ten consumers leaves partition 4 with one owner. The mitigation throttles that producer; the root fix restores a distributed stable key and plans ordering-safe migration.

**Tools, queries, metrics, interpretation.** Per-partition LEO/current offset, records produced/consumed, processing timers, group describe, broker and downstream dashboards. `d(lag)/dt > 0` means intake exceeds committed progress; if only one partition grows, skew or a record-specific block is likely.

**Immediate mitigation.** Protect downstream, throttle noncritical producers, increase consumers only up to useful partitions/capacity, reduce safe batch work, roll back, or route a poison record through approved retry/DLT handling.

**Root-cause correction.** Optimize/batch processing, scale partitions and consumers through planned design, correct key skew, fix downstream capacity, tune poll settings to measured processing, and bound retries.

**Prevention and alerts.** Alert on lag slope and event age per partition, capacity-test peak, monitor rebalance/processing/downstream saturation, and forecast drain time.

**Common traps.** Lag is not seconds; average lag hides hot partitions; offset resets hide backlog and can lose/replay work; scaling consumers can overload downstream.

**Interview-ready answer.** "I compare per-partition lag slope and event age with produce and successful consume rates. Then I check group membership/rebalances, handler and downstream latency, retries, resource saturation, skew, and broker fetch health. Recovery means consume capacity exceeds production and age drains."

## 3. Kafka consumer is processing messages very slowly. How would you troubleshoot it?

**Precise meaning.** Measure fetch wait, queue wait, handler service time, downstream time, and commit time; "consumer duration" often combines them.

**Where it can happen.** Broker fetch, deserialization, consumer loop, worker pool, CPU/GC, database/API, retries, offset commit, or logging.

**Causal mechanisms.** Slow per-record I/O; serial calls that can be safely batched; connection pool exhaustion; excessive logging; large payload/deserialization; lock contention; GC; too many records per poll; tiny inefficient batches; remote rate limits; per-message transaction; thread pool queue.

**Ordered debugging.**

1. Trace representative records and split fetch, deserialize, queue, business logic, downstream, and commit.
2. Compare records/s, batch size, handler percentile, errors/retries, and lag by partition.
3. Profile CPU/allocation only with approved low-overhead tools; inspect threads/pools/GC.
4. Inspect database/API latency and concurrency limits.
5. Check poll cadence, `max.poll.records`, fetch sizes, record sizes, and `max.poll.interval.ms`.
6. Determine safe concurrency: preserve per-key order and cap downstream load.
7. Test changes with realistic payload, skew, failure, and rebalance.

### Beginner expansion: separate waiting from work

One record's elapsed processing can be decomposed:

```text
fetch wait + local queue wait + deserialize
+ business CPU + database/API wait + retry delay + commit
```

- If fetch wait is high but producers are active, inspect broker/network/fetch configuration.
- If local queue wait is high, the consumer accepts faster than workers complete and needs backpressure.
- If CPU is high during deserialization, payload size/compression/schema mapping may dominate.
- If database acquisition time is high, Kafka is healthy but the handler's pool is saturated.
- If retries sleep on the poll thread, one failure can stop a whole assigned partition.
- If a poll returns 1,000 records and each needs one second, the batch cannot fit a short poll interval.

### Why the diagnostic order matters

1. Stage tracing prevents optimizing the wrong layer.
2. Rate, batch, error, and lag measurements quantify the bottleneck.
3. Profile/thread/resource evidence distinguishes CPU from waiting.
4. Dependency evidence often explains a slow handler better than Kafka settings.
5. Poll-budget analysis connects processing design to group stability.
6. Concurrency is chosen only after ordering and dependency safety are known.
7. Failure/rebalance testing validates behavior beyond a happy-path benchmark.

### Evidence interpretation

| Evidence | Meaning |
|---|---|
| Fetch latency high, handler fast | Broker/network/fetch path |
| Queue depth rises, workers all busy | Processing capacity/backpressure |
| Handler CPU profile dominated by JSON mapping | Serialization/payload |
| Database pool pending tracks handler p99 | Database dependency |
| Processing time exceeds max poll interval | Rebalance risk |
| More worker threads raise downstream 429/timeouts | Dependency is the true limit |

### Worked mini-example

The consumer handles 40 records/s while input is 100/s. Traces show 20 ms of local work and a 200-ms database call for every record. Eight consumers already own all eight partitions, and the database pool is saturated. Adding consumers would worsen it. A bulk upsert reduces 100 calls to one call per batch and raises safe throughput without breaking per-key order.

**Tools, queries, metrics, interpretation.** OpenTelemetry trace from consume to dependencies, JFR/profiler, thread dumps, pool metrics, consumer poll/fetch/commit metrics. High poll idle can mean broker starvation; low poll rate with long handler means application processing; high local queue means intake exceeds workers.

**Immediate mitigation.** Scale within partition/downstream limits, throttle producers, pause noncritical work, reduce batch size if poll interval is exceeded, or batch downstream operations if semantics permit.

**Root-cause correction.** Optimize and batch I/O, remove N+1 calls, use bounded concurrency by key/partition, improve serialization, right-size pools, and redesign expensive work asynchronously.

**Prevention and alerts.** Stage timers, per-record size and handler SLO, queue-depth/backpressure alerts, capacity tests, and poll-budget checks.

**Common traps.** Unbounded worker threads break ordering and memory; larger batches can cause rebalances; committing faster does not make effects faster; adding consumers past partition count does nothing.

**Interview-ready answer.** "I decompose fetch, deserialization, queue, handler, downstream, and commit time, then correlate processing percentiles with poll cadence, resources, retries, and per-partition lag. I optimize the dominant stage and add only bounded, ordering-safe parallelism."

## 4. Messages are being processed twice. Why can this happen?

**Precise meaning.** Distinguish duplicate records in Kafka from redelivery of one offset and duplicate external effects.

**Where it can happen.** Producer retry, outbox relay, consumer crash/rebalance, manual replay, retry topic, offset commit, or downstream API timeout.

**Causal mechanisms.** Consumer performs effect then crashes before commit; rebalance revokes during work; async commit fails; producer retries without idempotence; two business events share meaning; relay republishes; downstream succeeded but response was lost; offset reset/replay.

**Ordered debugging.**

1. Compare stable event ID, topic-partition-offset, producer ID/metadata, group, and attempt.
2. If offsets differ, investigate producer/relay/business duplication; if offset is same, investigate redelivery/commit.
3. Correlate handler success, downstream result, commit acknowledgement, crash/rebalance, and retries.
4. Inspect idempotency decision and its atomicity/retention.
5. Reproduce the crash windows before and after side effect.

### Beginner expansion: two identities answer two questions

The Kafka address `(topic, partition, offset)` identifies one log record. The business `event_id` identifies one logical event across republishing and retry topics.

- Same offset handled twice means Kafka redelivery, often because effect succeeded before offset commit.
- Same event ID at two offsets means producer, relay, retry, or replay published more than one record.
- Different event IDs with the same business command may be an upstream business-duplication bug.
- A downstream timeout is ambiguous: the charge may have succeeded even though the response was lost.
- An async offset commit may be requested but fail during rebalance.
- An in-memory dedupe set disappears on restart and does not coordinate multiple instances.

### Why the diagnostic order matters

1. Stable IDs and offsets classify redelivery versus republish immediately.
2. Offset comparison selects producer-side or consumer-side investigation.
3. Effect and commit ordering identifies the exact duplicate window.
4. Atomicity determines whether dedupe itself has a race.
5. Crash injection proves behavior at boundaries that normal tests rarely hit.

### Evidence interpretation

| Observation | Interpretation |
|---|---|
| Same offset, first attempt effect success, no commit | Expected at-least-once redelivery |
| Different offsets, same event ID | Republish/relay/retry duplicate |
| Commit succeeded before effect began | At-most-once loss risk, not duplicate protection |
| Two consumers both pass "does ID exist?" | Non-atomic check-then-act race |
| Unique inbox conflict, one effect only | Idempotency working |
| Downstream shows same idempotency key twice, one result | Downstream idempotency working |

### Worked mini-example

Consumer charges card for `evt-7`, then the pod loses power before committing offset 91. Another member receives offset 91 and tries again. A unique inbox row and payment idempotency key return the original charge result, then offset 92 commits. There were two processing attempts but one business effect, which is the practical goal.

**Tools, queries, metrics, interpretation.** Structured metadata logs, producer delivery metrics, group commit errors, rebalance logs, inbox table unique conflicts, downstream idempotency logs. Same event ID with different offsets suggests republish; same offset processed twice is normal at-least-once redelivery.

**Immediate mitigation.** Stop harmful side effects, enable/use downstream idempotency key if supported, quarantine affected event class, and reconcile state before replay.

**Root-cause correction.** Idempotent consumer/inbox, atomic state transition plus dedupe record, transactional outbox, producer idempotence, and synchronous/verified commit strategy appropriate to processing.

**Prevention and alerts.** Duplicate-attempt and dedupe-hit metrics, chaos tests at every crash window, globally stable event IDs, replay runbooks, and idempotency retention policy.

**Common traps.** Claiming exactly-once because Kafka transactions are enabled; deduping only in memory; committing before effect to suppress duplicates creates loss; event IDs generated anew on retry defeat dedupe.

**Interview-ready answer.** "At-least-once consumers can repeat an effect when they crash after processing but before committing. I distinguish same-offset redelivery from different-offset republish, trace effect and commit outcomes, and make the business operation idempotent with durable atomic deduplication."

## 5. A Kafka message appears to have been lost. How would you investigate?

**Precise meaning.** "Lost" can mean never acknowledged, written elsewhere, expired, compacted, skipped by offset, filtered, failed deserialization, processed but invisible, or delayed.

**Where it can happen.** Producer application/buffer, broker durability/retention, wrong cluster/topic/partition, consumer group offsets, transaction isolation, handler/retry/DLT, or downstream read model.

**Causal mechanisms.** Fire-and-forget send; producer process exits before flush/callback; `acks`/ISR durability not matching requirement; retention elapsed; compaction superseded a key; transaction aborted; consumer began at latest; commit-before-process; offset reset; wrong group; filter/dedupe; DLT; downstream write failed; observability gap.

**Ordered debugging.**

1. Define event identity, creation time, expected cluster/topic/group/side effect, and retention window.
2. Find producer intent and delivery acknowledgement with topic-partition-offset or explicit error.
3. Check topic metadata/config, partition LEO/log-start, leader/ISR and broker incident history.
4. Determine whether the offset is within retained range and transaction visibility rules.
5. Inspect intended group's committed offset and assignment history: before, at, or after record?
6. Trace deserialization, filters, handler attempts, dedupe, retry/DLT, commit, and downstream state.
7. Reconcile source-of-truth/outbox records with Kafka and consumer inbox/effects.

### Beginner expansion: "lost" is a custody question

Do not begin by changing offsets. Build a chain of evidence:

```text
business event/outbox row
 -> producer acknowledgement
 -> retained Kafka offset
 -> group position
 -> consumer attempt
 -> retry/DLT or durable effect
```

- The application can log "publishing" before the broker accepts anything.
- The producer can write to a similarly named topic in a test cluster.
- Retention can remove an old offset before a stopped group returns.
- Compaction can later remove an older value for a key, while a tombstone represents deletion.
- `auto.offset.reset=latest` can start a new group after existing records.
- A filter or dedupe decision can intentionally skip handler work.
- A consumer can commit early and crash before a non-atomic effect.
- The effect may exist under a different read-model delay, making observability look like loss.

### Why the diagnostic order matters

1. Identity and retention window define what can still be proven.
2. Acknowledgement separates producer intent from Kafka custody.
3. Log boundaries reveal whether the address is still retained.
4. Offset position tells whether the group has not reached or has passed the record.
5. Application tracing explains what happened after fetch.
6. Reconciliation finds gaps even when logs are incomplete.

### Evidence interpretation

| Evidence | Conclusion |
|---|---|
| Outbox row exists, no publish acknowledgement | Relay/producer gap |
| Acknowledged cluster/topic differs from expected | Misrouting |
| Offset is below current log-start | Retention removed it |
| Group commit remains before record | Delayed, not lost |
| Group commit passed record; DLT contains ID | Quarantined, not lost |
| Group commit passed; no attempt/effect; early commit enabled | Application-level skip/loss risk |
| Inbox/effect exists but UI does not | Read-model/cache/visibility issue |

### Worked mini-example

Support reports missing `evt-123`. The outbox says published, and the callback records partition 2 offset 700. Group commit is 900, suggesting it passed. Search by event ID finds the record in the DLT with a schema error. The event was not lost; alerting and DLT visibility were missing. After deploying compatibility, a bounded idempotent replay completes it.

**Tools, queries, metrics, interpretation.** Producer callback logs, broker/topic describe, group describe, approved metadata-only record lookup keyed by ID, outbox/inbox queries, audit trail. If no broker acknowledgement exists, do not call it broker loss. If committed offset passed it with no effect, inspect early commit/filter/dedupe.

**Immediate mitigation.** Preserve retention/evidence through approved controls, stop unsafe offset advancement, repair consumer, and replay only bounded identified events after idempotency review.

**Root-cause correction.** Await/record acknowledgements, transactional outbox, suitable `acks`/replication policy, commit after durable processing, correct retention, audit retry/DLT, and end-to-end reconciliation.

**Prevention and alerts.** Outbox-to-Kafka-to-inbox reconciliation, age/absence SLOs, producer callback errors, DLT alerts, schema validation, and documented retention/replay.

**Common traps.** Resetting offsets first; searching payloads unsafely; confusing compaction with immediate deletion; assuming successful application log means acknowledgement; ignoring wrong environment.

**Interview-ready answer.** "I build a custody chain: producer intent and broker acknowledgement, retained partition offset, group position, handler/dedupe/retry/DLT, and durable side effect. That distinguishes true durability failure from wrong routing, retention, skip, filtering, or observability."

## 6. A consumer crashes while processing a message. What happens to the message?

**Precise meaning.** Outcome depends on whether the consumed offset was committed and whether the side effect became durable before the crash.

**Where it can happen.** Poll-to-process-to-commit window, rebalance, external side effect, retry framework, or Kafka transaction.

**Causal mechanisms and outcomes.**

| Before crash | After restart/reassignment |
|---|---|
| Not processed, offset not committed | Record is normally redelivered |
| Effect committed, offset not committed | Record redelivered; duplicate effect unless idempotent |
| Offset committed, effect not durable | Record skipped by group; application-level loss |
| Kafka transaction aborted | Transactional Kafka outputs/offsets are not visible to `read_committed`; input is retried |
| Effect and offset atomically covered by a valid protocol | Protocol-specific recovery applies |

**Ordered debugging.**

1. Identify partition/offset, last committed offset, consumer position, and crash time.
2. Check whether handler and external transaction committed.
3. Check sync/async offset commit result and group reassignment.
4. Inspect idempotency/inbox and downstream state before replay.
5. Restart through normal group membership and verify the expected retry.

### Beginner expansion: enumerate the crash windows

Kafka retains the record independently of a consumer process. The important state is the last committed group offset and any external effect.

```text
poll 815
 -> apply database update
 -> commit database
 -> commit Kafka offset 816
```

Crashing before the database commit normally leaves no effect and no offset advance. Crashing after database commit but before offset commit creates a duplicate-attempt window. Committing offset 816 before database commit reverses the risk: a crash can make the group skip 815 permanently.

Kafka does not roll back an ordinary external database because a consumer process dies. Kafka transactions cover Kafka records and offsets only when the whole Kafka transaction protocol is used correctly.

### Why the diagnostic order matters

1. Offset and commit establish where reassigned consumption resumes.
2. External audit state determines whether retry is harmless or ambiguous.
3. Commit callback and rebalance evidence tell whether a requested checkpoint became durable.
4. Idempotency inspection protects against a second harmful effect.
5. Normal group restart preserves coordinator semantics; manual offset movement would alter evidence.

### Evidence interpretation

| State before crash | Expected recovery |
|---|---|
| Effect absent, offset not committed | Redeliver and process |
| Effect present, offset not committed | Redeliver; dedupe returns prior result |
| Offset committed, effect absent | Group skips; reconcile/recover explicitly |
| Kafka transaction aborted | Kafka outputs/offsets hidden from `read_committed`; retry |
| Commit outcome unknown | Treat as redelivery-capable and rely on idempotency |

### Worked mini-example

Offset 50 creates invoice `evt-50`. The database committed the invoice, but the pod was killed before the synchronous offset commit returned. The replacement starts at 50. Its inbox unique constraint detects `evt-50`, performs no second insert, and commits 51. This is correct at-least-once recovery, not an exactly-once transport claim.

**Tools, queries, metrics, interpretation.** Group offset, commit callbacks/errors, application transaction/audit logs, inbox key, crash/rebalance logs. A success log before database commit is not proof of effect.

**Immediate mitigation.** Restore consumer with idempotency enabled; quarantine uncertain harmful operations; reconcile the specific offset before manual replay.

**Root-cause correction.** Commit only after durable processing, atomically dedupe with business effects, use outbox/inbox, and classify/retry failures.

**Prevention and alerts.** Crash-window tests, commit/effect tracing, graceful shutdown that stops intake and completes bounded in-flight work, and duplicate-safe APIs.

**Common traps.** Assuming Kafka deletes messages when read; committing in `finally`; manually moving offsets; believing graceful shutdown covers process kill/node loss.

**Interview-ready answer.** "If the offset was not committed, Kafka normally redelivers after reassignment. If the effect happened first, that creates a duplicate window; if the offset committed first, it creates a loss window. I use commit-after-processing plus durable idempotency."

## 7. Kafka consumer keeps rebalancing. What could cause this?

**Precise meaning.** Group membership or subscription metadata repeatedly changes, causing partition revocation/assignment and interrupted progress.

**Where it can happen.** Consumer poll loop, heartbeat/coordinator path, orchestration, group coordinator/brokers, DNS/auth, topic metadata, or deployment.

**Causal mechanisms.** Processing exceeds `max.poll.interval.ms`; heartbeat/session timeout due to GC/CPU/network; crash loop; aggressive autoscaling; rolling deployments; unstable instance IDs; coordinator errors; auth expiry; subscription topic changes; manual assignment misuse; mixed incompatible client/protocol behavior.

**Ordered debugging.**

1. Build a rebalance timeline with reason, generation, member IDs, joins/leaves, duration, and deployment/pod events.
2. Check time between polls and worst-case batch processing against `max.poll.interval.ms`.
3. Check heartbeat failures, session timeout, GC pauses, CPU throttling, network, coordinator logs.
4. Check pod restarts/readiness, autoscaler churn, credential rotation, and client versions/assignors.
5. Inspect handler blocking, `max.poll.records`, shutdown behavior, and partition revoke callbacks.
6. Change one setting only after proving the violated timing relationship.

### Beginner expansion: two clocks, two failure classes

The group asks two different questions:

1. **Is the member alive?** Heartbeats and session timeout answer this.
2. **Is the application still polling and making progress?** The max poll interval answers this.

A client may heartbeat successfully on a background thread while its handler spends 10 minutes processing a batch. If `max.poll.interval.ms` is 5 minutes, the member can still be removed. Conversely, a long stop-the-world pause or broken network can stop heartbeats and trigger session timeout even if application logic is normally fast.

Other triggers include pod restarts, autoscaling every minute, expired credentials, coordinator movement, topic subscription metadata changes, and duplicate static member IDs.

### Why the diagnostic order matters

1. A reasoned timeline separates expected deployment rebalances from instability.
2. Poll-gap comparison proves application progress violations.
3. Heartbeat/GC/network evidence proves liveness violations.
4. Orchestrator and credential evidence catches membership churn outside handler code.
5. Client/protocol review detects incompatible or unstable membership configuration.
6. One measured setting change avoids masking a crash with very slow failure detection.

### Evidence interpretation

| Log/metric clue | Likely cause |
|---|---|
| "max poll interval exceeded" | Batch/handler blocks poll |
| Heartbeat/session timeout plus long GC pause | Runtime liveness pause |
| Join/leave matches pod rollout | Deployment churn |
| Rebalances match autoscaler oscillation | Unstable scaling policy |
| Authentication errors precede leave | Credential/TLS path |
| "fenced instance ID" | Duplicate static member identity |
| Coordinator errors across many groups | Broker/coordinator incident |

### Worked mini-example

`max.poll.records=500`; a rare event takes two seconds, so a worst-case batch can take 1,000 seconds. Max poll interval is 300 seconds. Logs show max-poll eviction, and reassignment causes duplicate attempts. Reducing batch size to 50 is immediate mitigation; the durable fix bounds per-record work and sends long jobs to a separate workflow.

**Tools, queries, metrics, interpretation.** Consumer rebalance/heartbeat/poll metrics, logs, Kubernetes events, GC/CPU, broker coordinator metrics. `max.poll.interval` removal points to application processing; session timeout points to heartbeat/liveness path.

**Immediate mitigation.** Roll back crash-looping deployment, stabilize replica count, reduce poll batch, restore coordinator/network/auth, and bound handler work. Static membership may reduce brief restart churn but must use unique stable IDs.

**Root-cause correction.** Poll reliably, hand off only to a bounded architecture that safely tracks completion, tune timeouts to measured worst case, fix GC/resources/network, and use graceful cooperative deployments where supported.

**Prevention and alerts.** Rebalance rate/duration and poll-interval alerts, soak tests with worst messages, deployment disruption budgets, unique member identity validation, and GC monitoring.

**Common traps.** Increasing every timeout; static IDs duplicated across replicas; processing asynchronously then committing offsets for unfinished work; ignoring rebalance callbacks.

**Interview-ready answer.** "I correlate each rebalance reason with poll gaps, heartbeat/session health, GC/CPU/network, restarts, autoscaling, and deployments. Long processing points to max-poll violations; missed heartbeats point to liveness. I fix the cause before tuning."

## 8. One consumer instance is much slower than the others. What would you check?

**Precise meaning.** Determine whether the instance itself is slow or its assigned partitions/messages are more expensive.

**Where it can happen.** Partition assignment/key skew, node/zone, CPU/memory/GC, network, local disk, downstream shard, config/version, cache, or poison record.

**Causal mechanisms.** Hot partition; larger records; expensive tenant/key; one node throttled/noisy; cold cache; different image/config; connection pool issue; zone latency; repeated retries; uneven number of partitions; sticky assignment history.

**Ordered debugging.**

1. Compare per-instance assignment, per-partition ingress/lag, record size/type, and processing time.
2. Normalize by records and bytes; decide workload skew versus machine slowness.
3. Compare version/config, node/zone, CPU throttling, GC, memory, network, pools, and downstream latency.
4. Trace representative records from a hot and normal partition without exposing payload.
5. Observe whether slowness follows the partition after a normal rebalance or stays with the instance/node.
6. Inspect retry/poison and key distribution.

### Beginner expansion: instance or assignment?

A consumer is not given equal *records*; it is given partitions. Two consumers can each own three partitions while one receives ten times the traffic or larger events.

- A hot partition makes its owner look slow.
- A single poison offset can pin only the owner of that partition.
- One node may be CPU-throttled or have slow network to the database.
- One instance may run a different release, trust bundle, or pool configuration.
- One availability zone may add 30 ms to every downstream call.
- A cold cache can cause a temporary outlier after restart.

The most useful experiment is observational: after a planned/normal reassignment, does the symptom follow the partition or remain with the host? Do not force repeated rebalances solely as a test during an incident.

### Why the diagnostic order matters

1. Assignment and partition workload must be controlled before comparing machines.
2. Per-record/per-byte normalization makes the comparison fair.
3. Runtime and node evidence tests the instance hypothesis.
4. Traces explain whether a particular event class costs more.
5. Symptom movement is strong partition-versus-host evidence.
6. Retry and key inspection identifies deterministic hot work.

### Evidence interpretation

| Observation | Interpretation |
|---|---|
| Slow instance owns the only hot partition | Assignment skew |
| Hot lag follows partition to a new owner | Partition/key/data issue |
| Every partition assigned to node X slows | Node/zone/config issue |
| CPU throttling only on one pod | Resource limit/noisy node |
| Same bytes but handler p99 differs by version | Code/config regression |
| Repeated failures at one offset | Poison record |

### Worked mini-example

Consumer C appears five times slower. It owns partition 7, which contains image-analysis events averaging 5 MB while other partitions average 20 KB. CPU and records/s alone mislead. Bytes/s and event-type timing show workload skew. The fix separates expensive event classes and revises partitioning/capacity, rather than repeatedly replacing C.

**Tools, queries, metrics, interpretation.** Metrics labeled by instance/topic/partition, assignment logs, Kubernetes/node metrics, traces. If the hot lag follows partition, fix key/workload skew; if it stays on node, fix instance infrastructure/config.

**Immediate mitigation.** Replace/drain a faulty instance or node, isolate poison input, add capacity if useful, or throttle hot-key producers/downstream safely.

**Root-cause correction.** Better stable key distribution, hot-key sharding with ordering trade-off, equal config/resources, downstream shard capacity, and bounded record complexity.

**Prevention and alerts.** Outlier dashboards, per-partition rate/size/age, config hashes, node-zone labels, skew tests, and maximum record limits.

**Common traps.** Averaging across consumers; assuming equal partition count means equal work; rebalancing repeatedly; changing keys without understanding ordering and migration.

**Interview-ready answer.** "I compare assignments and per-partition work first. Then I normalize by records/bytes and compare node, version, resources, pools, retries, and downstream latency. If slowness follows a partition it is skew; if it stays with the host it is instance-local."

## 9. Messages are arriving out of order. Why could this happen?

**Precise meaning.** Define order by event sequence, event time, broker offset, processing start, completion, or visible side effect. Kafka promises only partition order.

**Where it can happen.** Producer key/partitioner, multiple producers, topic repartitioning, retries, consumer concurrency, retry/DLT, or downstream observation.

**Causal mechanisms.** Related events use different partitions; null/random key; partition count changed; async processing completes later offset first; failed earlier event is retried after later one; clocks differ; multiple producers race; unsafe producer retry settings; replay overlaps live traffic.

**Ordered debugging.**

1. Capture entity ID, event sequence/version, event time, topic-partition-offset, producer, attempt, and completion time.
2. Determine whether alleged inversion is within one partition by offset.
3. Verify stable serialized key and partitioner/version/partition-count history.
4. Inspect producer settings/errors and concurrent producer sources.
5. Inspect consumer worker concurrency, commit tracking, retry/DLT and replay.
6. Define the business ordering contract and test it under failure.

### Beginner expansion: define the clock and sequence

Consider two events for account 42:

```text
version 10 -> address A
version 11 -> address B
```

If both use key `account-42`, the same partition receives version 10 before 11 when the producer appends them in that order. But a worker pool can start both and finish version 11 first. If version 10 fails and is retried tomorrow, it may overwrite newer state unless the database conditionally applies only a newer version.

Event timestamps are not a reliable ordering proof. Producer clocks differ, events can be created earlier but published later, and retries preserve old event time. Partition offsets are the broker order for one partition.

### Why the diagnostic order matters

1. Capturing every sequence dimension prevents arguing about different definitions of "arrived."
2. Same-partition verification tests Kafka's actual ordering scope.
3. Key and partition history reveal whether related records were colocated.
4. Producer evidence checks append order and retry behavior.
5. Consumer/retry evidence checks completion and replay order.
6. The business contract determines whether serialization or version rejection is required.

### Evidence interpretation

| Evidence | Meaning |
|---|---|
| Events are in different partitions | No cross-partition order guarantee |
| Same partition offsets are 10 then 11 | Kafka log order is correct |
| Side effects finish 11 then 10 | Consumer/downstream concurrency reordered completion |
| Version 10 returns from retry topic after 11 | Retry path relaxed order |
| One key maps differently after partition increase | Partition-count migration effect |
| Database rejects version 10 after 11 | Version guard prevents stale state |

### Worked mini-example

Offsets 100 and 101 are read in order, but handlers run concurrently. Offset 101 finishes first; offset 100 later overwrites the profile. The fix is not a broker setting. The consumer serializes work per account or uses `UPDATE ... WHERE current_version < incoming_version`, making stale completion harmless.

**Tools, queries, metrics, interpretation.** Metadata logs, producer partition callback, topic history, consumer traces. Different partitions means no Kafka global-order violation. Same partition offsets fetched in order but effects reversed means consumer/downstream concurrency.

**Immediate mitigation.** Serialize affected entity processing, pause conflicting replay, route key consistently, and make consumers reject/defer stale versions where business-safe.

**Root-cause correction.** Stable entity key, per-key/partition sequencing, versioned events and conditional state updates, ordering-aware retry design, and controlled repartition migration.

**Prevention and alerts.** Include event ID/entity/version/occurred-at, detect sequence regressions, contract-test partitioning, and document that timestamps are not offsets.

**Common traps.** Demanding global order while scaling partitions; increasing partition count silently changes key mapping for common partitioners; sorting by clock; letting one failed record block forever without policy.

**Interview-ready answer.** "I define what 'order' means and compare topic-partition-offset. Kafka orders only within a partition. I verify stable keys and partition history, then inspect producer retries, consumer parallel completion, and retry/replay paths, using entity versions to protect state."

## 10. A downstream service fails after consuming a Kafka message. How would you handle this?

**Precise meaning.** Decide whether failure is transient, permanent, ambiguous, or overload, while keeping offset and side-effect semantics correct.

**Where it can happen.** HTTP/database dependency, consumer transaction, retry scheduler/topic, circuit breaker, DLT, or compensation workflow.

**Causal mechanisms.** Timeout, 429/5xx, network outage, invalid request, auth/config error, downstream saturation, or success with lost response. Blind retries can amplify the outage and duplicate effects.

**Ordered debugging and handling.**

1. Classify error and idempotency: retryable status/exception, permanent validation, or ambiguous result.
2. Keep event ID and attempt metadata; inspect downstream before retrying ambiguous operations.
3. Retry only transient failures with exponential backoff, jitter, cap, and total age/attempt budget.
4. Use downstream idempotency key or local inbox/state machine.
5. Apply circuit breaker/concurrency limit and backpressure; do not flood recovery.
6. After budget, publish an approved retry/DLT envelope durably before committing input.
7. Alert, repair, replay in bounded batches, and reconcile outcomes.

### Beginner expansion: classify before retrying

Not every failure deserves the same response:

- **Transient:** HTTP 503, connection reset, or short rate limit may recover.
- **Permanent for this record:** validation 400 or unsupported schema will not improve through immediate retry.
- **Configuration/systemic:** expired credential affects every record and should open an incident, not create millions of DLT entries.
- **Ambiguous:** a timeout after sending a payment request does not reveal whether payment succeeded.
- **Overload:** retrying immediately increases load and lengthens recovery.

The offset should advance only after either the durable business result exists or durable ownership has moved to a retry/DLT mechanism. A retry held only in process memory disappears on crash.

### Why the diagnostic order matters

1. Error classification prevents harmful or pointless retries.
2. Stable attempt metadata and status checks resolve ambiguous outcomes.
3. Backoff and budget limit amplification.
4. Idempotency makes repeated attempts safe.
5. Circuit/concurrency controls protect an already failing dependency.
6. Confirmed retry/DLT publication prevents a gap before input commit.
7. Bounded replay avoids a second outage after recovery.

### Evidence interpretation

| Result | Handling direction |
|---|---|
| 429 with `Retry-After` | Scheduled bounded retry and lower concurrency |
| 503/timeouts across all records | Circuit/backpressure; dependency incident |
| 400 for one event version | Permanent validation/schema path |
| Timeout, downstream lookup finds completed operation | Reuse result; do not repeat effect |
| Retry/DLT publish failed | Do not commit input yet; preserve ownership |
| DLT rate spikes after deployment | Likely code/schema/config regression |

### Worked mini-example

Shipping API times out after accepting order `evt-44`. A blind retry creates two shipments. With an idempotency key, the consumer first queries/retries using `evt-44`; the service returns the original shipment. Retry is capped and jittered, and the input offset commits only after that durable result is stored.

**Tools, queries, metrics, interpretation.** Dependency traces, response status, retry attempt/age, circuit state, DLT rate, downstream saturation, idempotency audit. A timeout is ambiguous: it does not prove the downstream did nothing.

**Immediate mitigation.** Reduce concurrency, open circuit/load-shed, throttle intake, extend scheduled backoff, or quarantine affected operation. Avoid infinite tight retry.

**Root-cause correction.** Idempotent downstream contract, durable retry/DLT, realistic timeout budget, capacity and bulk API improvement, or saga/compensation for multi-step work.

**Prevention and alerts.** Dependency SLO, retry amplification ratio, DLT age/count, circuit alerts, chaos tests, and replay runbook.

**Common traps.** Commit then rely on an in-memory retry; treat all 4xx/5xx alike; retry at consumer, client, proxy, and service simultaneously; keep a database transaction open during backoff.

**Interview-ready answer.** "I classify transient, permanent, and ambiguous failures. I use a stable idempotency key, bounded jittered retries and backpressure; after the retry budget I durably route to retry/DLT before committing, alert, fix, and replay safely."

## 11. A poison message keeps failing repeatedly. How would you handle it?

**Precise meaning.** A specific record predictably fails because of data/schema/business logic rather than a transient dependency.

**Where it can happen.** Deserializer before handler, schema evolution, validation, business invariant, record-size limit, or buggy code path.

**Causal mechanisms.** Unknown schema/version, malformed bytes, incompatible enum, missing required field, unexpected null, invalid business state, deterministic exception, or excessively expensive payload.

**Ordered debugging and handling.**

1. Identify topic-partition-offset, event/schema ID, exception class, attempt count, and first failure; protect payload.
2. Decide whether deserialization can be captured as bytes/headers by a safe error handler.
3. Confirm deterministic versus dependency failure.
4. After a small bounded retry budget, publish the original record/envelope plus non-sensitive error metadata to a governed DLT.
5. Confirm DLT publish acknowledgement, then advance/commit main input according to framework semantics.
6. Alert owner, diagnose in controlled access, fix producer/schema/consumer, and replay through a validated idempotent path.

### Beginner expansion: why one record can stop a partition

Partitions are ordered logs. A simple consumer often handles offset N before committing N+1. If N always throws and the framework seeks back to N, later offsets in that partition cannot make durable progress.

- Invalid JSON fails during deserialization before business error handling.
- A new enum value breaks an older consumer.
- A required field is absent because producer and consumer schema versions are incompatible.
- A valid but extremely large record exhausts memory or exceeds fetch/handler limits.
- A deterministic null dereference fails every time.
- A business invariant, such as a missing customer, may need a defined reject/repair path.

Quarantining allows later records to proceed but changes strict order. That trade-off must be explicit, especially when later events depend on the poison event.

### Why the diagnostic order matters

1. Exact offset and exception establish that the same record repeats.
2. Deserialization handling must work before an object exists.
3. Deterministic classification prevents sending a temporary outage to DLT.
4. A retry budget avoids infinite partition pinning.
5. DLT acknowledgement preserves custody before the main offset advances.
6. Controlled diagnosis and replay protect sensitive data and ordering.

### Evidence interpretation

| Evidence | Meaning |
|---|---|
| Same offset, same exception, many attempts | Poison record |
| Different offsets all fail after auth expiry | Systemic configuration, not individual poison |
| Handler never starts; deserialize errors rise | Pre-handler schema/bytes failure |
| DLT acknowledgement absent | Main offset must not advance yet |
| Lag grows only behind one offset | Partition pinned |
| Replayed record fails identically | Cause not fixed or replay path differs |

### Worked mini-example

Offset 340 contains enum value `SUSPENDED`, unknown to consumer v1. It fails before handler logging and partition lag grows. Error handling captures safe metadata and original bytes to the governed DLT, confirms publication, then commits 341. Consumer v2 adds compatible handling; replay keeps the original event ID and is rate-limited.

**Tools, queries, metrics, interpretation.** Error-type counters, retry attempt/age, DLT delivery callback, schema registry compatibility/history, partition lag. One partition pinned at one offset with repeated identical exception is classic poison behavior.

**Immediate mitigation.** Quarantine through established DLT, disable only the failing optional path, or deploy a compatible reader. Do not dump sensitive payload to logs.

**Root-cause correction.** Schema compatibility, tolerant versioned deserialization, validation contract, producer fix, explicit unknown-field policy, and size limits.

**Prevention and alerts.** Contract/schema tests, DLT count/age alert, replay tooling with approval/idempotency, poison test fixtures, ownership and retention.

**Common traps.** Infinite retries; silently skip; DLT without monitoring; committing before confirming DLT write; DLT replay back into the same failure loop; storing secrets indefinitely.

**Interview-ready answer.** "I identify the exact offset and deterministic error, bound retries, durably quarantine it with safe metadata, and only then advance the main flow. I fix schema/data/code, test idempotency, and replay through a monitored governed path."

## 12. How would you investigate a sudden increase in Kafka consumer lag in production?

**Precise meaning.** This asks for an incident procedure, emphasizing onset, partition-level evidence, capacity math, and safe response.

**Where it can happen.** Any stage from broker ingress through group membership and processing to commit; downstream and rollout changes are frequent causes.

**Causal mechanisms.** A production spike is often caused by rate/size/key distribution changes, deployment/regression, downstream incident, rebalance storm, broker throttle/storage event, poison retry, lost replicas, GC/CPU throttling, or commit failures.

**Ordered debugging.**

1. Record onset, topic/group, SLO impact, per-partition lag and oldest-event age.
2. Compare incident versus baseline producer records/bytes and successful consumer records/s.
3. Calculate lag slope and whether current capacity can drain; avoid promising a drain time when rates are unstable.
4. Check member/assignment changes, restarts, rebalances, poll gaps, heartbeat and commits.
5. Compare processing stages, errors/retries/DLT, downstream latency, CPU/memory/GC/pools.
6. Identify hot partitions and changes in key, size, schema, or expensive event type.
7. Check broker leaders, ISR, disk/network/request latency, throttling and controller events.
8. Correlate deploy/config/ACL/certificate/partition changes and mitigate the proven constraint.
9. Verify lag slope becomes negative and business event age returns to SLO without error/duplicate growth.

### Beginner expansion: an incident timeline

This question resembles question 2, but an interview expects an operational sequence. Start with a timestamp and preserve state before changing consumer offsets or restarting everything.

Example timeline:

```text
14:00 deployment begins
14:03 rebalances increase
14:05 handler p99 rises from 100 ms to 4 s
14:07 committed-offset lag begins rising
14:10 oldest-event age breaches 5 minutes
```

The order suggests a consumer regression rather than an initial broker failure. A different timeline, where broker fetch latency and ISR problems begin before consumer slowdown, would point toward the broker/storage path.

### Why the diagnostic order matters

1. Onset and partition scope prevent lifetime-counter and fleet-average mistakes.
2. Input/output comparison quantifies backlog creation.
3. Drain math informs mitigation urgency and recovery expectations.
4. Membership and commit state identify lost effective capacity.
5. Stage/dependency/resource evidence locates processing regression.
6. Key, size, and schema checks expose data-dependent hotspots.
7. Broker checks prevent incorrectly blaming application code.
8. Change correlation produces a reversible, testable action.
9. Age, slope, errors, and duplicates together prove safe recovery.

### Evidence interpretation

| Incident result | Branch |
|---|---|
| Produce rate jumped, consume capacity unchanged | Demand/capacity response |
| Consume rate fell exactly at deployment | Rollback/canary comparison |
| No active members | Restore deployment/auth/group membership |
| Rebalance duration occupies much of interval | Stabilize group and poll/liveness |
| One partition pinned at one offset | Poison/skew/partition leader |
| Broker fetch/throttle/disk rises across groups | Broker/platform incident |
| Lag falls but downstream errors soar | Catch-up overload; reduce concurrency |

### Worked mini-example

Lag jumps after a consumer rollout. Broker health and input rate are normal. Poll gaps exceed five minutes because the release makes one sequential HTTP request per record; max poll evictions cause repeated work. Rolling back stabilizes assignments and makes lag slope negative. The corrected release batches calls and has a worst-case poll-budget test.

**Tools, queries, metrics, interpretation.** Group/topic describe, per-partition metrics, APM traces, broker dashboards, orchestration events. Stable LEO with frozen committed offset points to consumer/commit; rapidly rising LEO points to demand; one partition points to skew/poison.

**Immediate mitigation.** Roll back, restore members, protect/scale downstream, scale consumers to useful partitions, throttle intake, or isolate poison traffic. Preserve offset evidence.

**Root-cause correction.** Fix regression/skew/poll/retry/downstream/broker capacity and document tested drain strategy.

**Prevention and alerts.** Multi-window alert on event age plus lag slope, per-partition dashboard, capacity headroom, deployment canary, and backlog game day.

**Common traps.** Treating this as only "add consumers"; offset reset; ignoring event age; declaring recovery when lag plateaus instead of drains; overloading dependencies during catch-up.

**Interview-ready answer.** "I timestamp the onset and compare per-partition lag slope and event age with produce and consume rates. I inspect membership/rebalances, processing/downstream, skew, commits and broker health, mitigate the proven bottleneck, and verify a safe negative lag slope."

## 13. How would you prevent duplicate processing of Kafka messages?

**Precise meaning.** Prevent duplicate **business effects**, since redelivery and republishing can still occur in distributed systems.

**Where it can happen.** Producer, relay, consumer offset window, database, downstream API, retry/DLT, or replay.

**Causal mechanisms.** At-least-once redelivery, ambiguous responses, producer retry/relay duplicate, concurrent handlers, dedupe expiry, non-atomic check-then-act, and deliberate replay.

**Ordered design and verification.**

1. Assign a stable immutable event/idempotency ID at business-event creation; preserve it across retries and relays.
2. Define dedupe scope and lifetime longer than possible replay/redelivery.
3. At consumer, use a database unique constraint/inbox and business update in one transaction.
4. Prefer naturally idempotent/versioned state transitions; pass the key to downstream services.
5. Commit input only after durable effect/dedupe; handle commit failure as expected redelivery.
6. Use transactional outbox to avoid database-plus-publish dual writes.
7. If Kafka-only consume-transform-produce, consider Kafka transactions/read-committed, while stating the boundary.
8. Chaos-test crashes at every step and concurrent delivery of the same ID.

### Beginner expansion: idempotency is a business property

Some operations are naturally idempotent:

```text
set order status to SHIPPED at version 8
```

Repeating "set to SHIPPED" can be harmless. Other operations are not:

```text
increment balance by 10
send an email
charge a card
```

For these, use a stable event ID and durable record of the outcome. A safe relational pattern is:

```text
BEGIN;
INSERT INTO inbox(event_id) VALUES ('evt-9'); -- UNIQUE
UPDATE account SET ...;                       -- only if insert succeeded
COMMIT;
```

Both statements must share one transaction. A separate "does ID exist?" query followed by an effect has a race: two consumers can both see absence and both act.

### Why the diagnostic/design order matters

1. Stable IDs define what counts as the same logical operation.
2. Scope/lifetime prevents an old replay from escaping dedupe.
3. Atomic inbox plus state change closes concurrency and crash gaps.
4. Naturally idempotent/versioned state reduces special-case storage.
5. Commit-after-effect accepts redelivery without accepting duplicate effects.
6. Outbox closes the source database/publish gap.
7. Kafka transactions are selected only for Kafka-contained boundaries.
8. Crash/concurrency tests prove the design under real failure windows.

### Evidence interpretation

| Test result | Meaning |
|---|---|
| Two simultaneous attempts create one inbox row/effect | Atomic dedupe works |
| Dedupe row exists but effect is absent | Inbox and effect were not atomic |
| Duplicate arrives after dedupe expiry and acts again | Retention too short for replay window |
| Producer idempotence enabled but consumer repeats effect | Wrong boundary; consumer idempotency still needed |
| Kafka output atomic, HTTP side effect duplicates | External effect is outside Kafka transaction |
| Dedupe hit rate rises during rebalance, effects stay single | Expected resilience |

### Worked mini-example

Two group generations briefly attempt `evt-pay-77` around a rebalance. Both insert the inbox key; the database unique constraint permits one. That transaction records payment intent and an outbox result. The loser reads the stored outcome and commits its offset. The design tolerates redelivery without claiming that Kafka alone delivered exactly once.

**Tools, queries, metrics, interpretation.** Unique-key conflict/dedupe-hit rate, event ID/offset audit, commit errors, outbox relay attempts, downstream idempotency record. Dedupe hit is often healthy evidence of resilience, not necessarily an incident.

**Immediate mitigation.** Gate harmful operations behind idempotency, stop replay, reconcile duplicate effects, and quarantine non-idempotent event types.

**Root-cause correction.** Atomic inbox/business transaction, conditional version update, downstream idempotency, stable IDs, and outbox. Avoid distributed check-then-act.

**Prevention and alerts.** Contract requires event IDs, retention policy, concurrency/crash tests, replay approval, duplicate-effect reconciliation, and metrics.

**Common traps.** In-memory set; dedupe check and effect in separate transactions; using partition-offset as universal event identity after republish; saying `enable.idempotence` makes consumers exactly-once; committing first.

**Interview-ready answer.** "I do not promise no redelivery; I prevent duplicate effects. A stable event ID flows end to end, and the consumer atomically records it with the business update under a unique constraint, then commits the offset. Outbox/inbox and downstream idempotency close other boundaries."

---

# 4. Important additional interview questions

## A1. What producer settings determine durability, and what does an acknowledgement prove?

**Answer.** `acks=all` asks the leader to wait for the current in-sync durability requirement; `min.insync.replicas` defines how many ISR replicas must acknowledge for writes under that policy. Replication factor determines potential copies. Idempotence protects supported retry sequences from duplicate appends and imposes compatible settings in modern clients. A successful callback proves acceptance according to configuration at that time, not that a consumer processed it or that an external side effect occurred. Monitor ISR, under-replication, delivery errors, timeouts, retries, and broker durability configuration. Choose settings from loss/availability requirements and test broker failure.

## A2. When should partitions be increased?

**Answer.** Increase partitions when measured throughput/parallelism cannot be met by existing partitions and brokers/downstream have capacity, after reviewing key distribution, ordering, storage, controller, and client effects. Existing records are not automatically redistributed. Common key-to-partition mapping changes when partition count changes, so future records for a key may land on a different partition, weakening continuity of per-key order during migration. More partitions also cost metadata, files, replication, and rebalance time. Treat this as an architecture change, not an incident reflex.

## A3. How do you size a consumer?

**Answer.** Measure per-record/batch service time by event class and downstream limits. Required stable capacity satisfies:

```text
total sustainable consume rate > peak produce rate
```

Useful consumers cannot exceed assigned partitions for a normal group. Include failure headroom and catch-up capacity, but bound concurrency below database/API limits. Set `max.poll.records` so worst-case batch processing remains comfortably inside `max.poll.interval.ms`, considering retries and pauses. Validate record bytes/fetch limits, heap, GC, commit frequency, rebalance, and skew under production-like traffic.

## A4. How do schema changes cause incidents?

**Answer.** Producers and consumers deploy independently. Removing/renaming fields, incompatible type changes, enum additions, changed defaults, or subject-strategy mistakes can break serialization or semantics. Use versioned schemas and registry compatibility appropriate to rollout direction, test old producer/new consumer and new producer/old consumer, prefer additive compatible changes, tolerate unknown fields where safe, and monitor serialization/deserialization by schema ID. Compatibility tooling cannot prove business-semantic compatibility.

## A5. How should a DLT replay be performed?

**Answer.** First fix and deploy the cause. Define a bounded set by topic/partition/offset, error class, time, schema, and tenant; review sensitive data and retention. Prove the target path is idempotent and decide ordering relative to live traffic. Rate-limit replay below downstream headroom, preserve original event ID and provenance, emit a new replay attempt ID, monitor success/failure/duplicates/lag, and stop on thresholds. Reconcile business results. Never blindly dump the entire DLT back into the main topic.

## A6. What does `auto.offset.reset` do?

**Answer.** It applies when a group has no valid committed offset for a partition, for example a new group or an offset removed by retention. `earliest` begins at the log start, `latest` begins near current end, and some clients support error/other policies. It does not normally reposition a group that has a valid commit, and it is not a duplicate-prevention mechanism. Choose it from replay/loss requirements and alert when it is invoked unexpectedly.

---

# 5. Decision trees

## 5.1 "Produced but not processed"

```text
Broker delivery callback acknowledged?
|
+-- No -> producer serialization, buffer, auth/network, timeout, broker error
|
+-- Yes: have topic-partition-offset
    |
    +-- Record offset outside retained log?
    |   +-- retention/compaction/aborted transaction investigation
    |
    +-- Intended group has active assigned member?
    |   +-- No -> deployment, wrong group/topic, auth, rebalance
    |
    +-- Committed offset <= record offset?
    |   +-- Yes -> lag, pause, deserialization, poison, slow handler
    |   +-- No  -> filter/dedupe/commit-before-effect/retry-DLT/downstream visibility
```

## 5.2 Lag

```text
Per-partition lag rising?
|
+-- All/many partitions
|   +-- produce rate increased?
|   +-- consumer count fell/rebalances?
|   +-- shared downstream or broker slow?
|   +-- deployment/resource/commit regression?
|
+-- One/few partitions
    +-- hot key/rate or larger/expensive records?
    +-- poison record/retry loop?
    +-- partition leader/broker fault?

After mitigation:
consume rate > produce rate AND event age falling?
  yes -> draining safely
  no  -> bottleneck remains or downstream is protected by a deliberate cap
```

## 5.3 Failure disposition

```text
Handler failure
|
+-- Permanent validation/schema/business error
|   -> bounded/no retry -> durable DLT/quarantine -> alert -> fix -> governed replay
|
+-- Transient dependency failure
|   -> idempotent? bounded exponential retry + jitter + total budget
|   -> exhausted -> retry topic/DLT
|
+-- Ambiguous outcome (timeout/reset)
|   -> query idempotency/status before retry; same idempotency key
|
+-- Capacity overload
    -> backpressure/concurrency limit/circuit breaker; avoid retry storm
```

## 5.4 Duplicate or loss

```text
Same stable event ID observed twice?
|
+-- Same topic-partition-offset
|   -> redelivery: crash/rebalance/commit failure/manual replay
|
+-- Different offsets
|   -> producer/outbox relay/business republish
|
+-- Duplicate external effect only
    -> non-atomic/expired/missing idempotency or ambiguous downstream retry

Expected effect absent?
|
+-- no broker acknowledgement -> producer path
+-- record not retained/visible -> retention, compaction, transaction
+-- committed offset passed it -> early commit, filter, dedupe, DLT, effect failure
+-- committed offset before it -> backlog/processing delay
```

---

# 6. Cheat sheets

## 6.1 Incident evidence card

```text
Timestamp/window and business impact:
Cluster/topic/group:
Event ID and key hash (not payload):
Topic-partition-offset and schema ID:
Producer acknowledgement/error/latency:
Partition leader/replicas/ISR/log start/log end:
Group state/members/assignments:
Committed offset/live position/per-partition lag:
Oldest unprocessed event age:
Produce rate/bytes versus successful consume rate:
Poll/batch/processing/commit percentiles:
Rebalance reason/rate/duration:
Retry/DLT/dedupe/downstream metrics:
Consumer version/config/node/zone:
Recent deploy/schema/ACL/certificate/partition changes:
```

## 6.2 Symptom-to-first-evidence

| Symptom | First evidence |
|---|---|
| Produced, not consumed | Delivery callback and acknowledged topic-partition-offset |
| Lag rising | Per-partition lag slope, event age, produce versus consume rate |
| Rebalance loop | Rebalance reason plus poll gaps, heartbeat, GC and restarts |
| One slow instance | Assignment/workload versus host/config comparison |
| Duplicate effect | Event ID, offset(s), effect commit and offset commit timeline |
| Apparent loss | Producer-to-broker-to-group-to-effect custody chain |
| Out of order | Entity sequence and topic-partition-offset |
| Poison record | Fixed partition/offset and repeated deterministic exception |
| Downstream outage | Error class, ambiguity, idempotency, retry amplification |

## 6.3 Settings reasoning card

| Setting/concept | Reason about it this way |
|---|---|
| Partition count | Upper bound on active consumers in one group; more has ordering/operational cost |
| `max.poll.records` | Worst-case batch work must fit safely within max poll interval |
| `max.poll.interval.ms` | Processing progress contract, not heartbeat timeout |
| Session/heartbeat | Liveness detection; tune from network/GC behavior and client protocol |
| Offset commit | Checkpoint after durable effect for at-least-once; can redeliver |
| `auto.offset.reset` | Behavior only when no valid committed offset exists |
| Producer `acks`/ISR | Broker acknowledgement durability/availability trade-off |
| Retry | Bounded by attempts and total age, with jitter and idempotency |
| DLT | Governed exception stream requiring alerts, ownership, retention, and replay |

## 6.4 Durable rules

1. "Send called" is not "broker acknowledged"; "offset committed" is not "business effect correct."
2. Preserve topic, partition, offset, event ID, attempt, schema, and trace identity.
3. Lag is records; pair it with rate, slope, and event age.
4. Diagnose per partition before fleet averages.
5. One group member cannot share one partition concurrently under normal assignment.
6. Kafka ordering is partition-scoped; consumer completion can still reorder effects.
7. Commit-after-effect gives at-least-once and requires idempotency.
8. Bounded retry, backoff, DLT, and replay are one lifecycle, not separate features.
9. Outbox closes database-to-Kafka dual-write gaps; inbox/idempotency protects effects.
10. Kafka-only transactions do not make arbitrary external systems exactly-once.
11. Scale consumers only when partitions and downstream capacity make it useful.
12. Never reset offsets as a diagnostic shortcut; preserve and understand the evidence first.
