# Problem

One or more Kafka records repeatedly fail during consumption.
A processing failure can occur before or after listener invocation.
Key or value deserialization may fail before business code runs.
Validation may reject a syntactically valid but unsupported event.
Handler logic may throw.
A database or HTTP dependency may fail transiently.
A permanent business rule may reject the operation.
A poison record is one that repeatedly fails without a likely benefit from immediate retry.
The recovery policy must preserve evidence, avoid partition starvation, and prevent unsafe side effects.

# Production Situation

The Meridian incident reaches its next stage at 19:05.
`order-service` begins sending schema version 4 for `OrderCreated`.
One field changes from integer `warehouseId` to string `warehouseCode`.
Two older `fulfillment-service` pods still use the version 3 deserializer.
Partition 3 reaches offset 760201 and stops making progress.
Its lag rises from 0 to 18,400.
Other 11 partitions remain below 40 lag.
Consumer error rate is 84/min.
Listener-start count on the old pods does not increase for the bad record.
Deserialize-failure count increases.
The default error path seeks back to the same offset.
The record is attempted 600 times in ten minutes.
CPU rises to 67% from logging and repeated deserialization.
No business handler span exists.
Customers whose keys map to partition 3 are delayed.

# Architecture

```text
order-service v5.43
    |
    | schemaVersion=4
    v
orders.v1 partition 3 offset 760201
    |
    v
Spring Kafka consumer
    +--> key deserialize
    +--> value deserialize       X
    +--> validation
    +--> handler
    +--> dependencies
    +--> commit
    |
    +--> retry topics / DLT recovery policy
```

Kafka preserves order within partition 3.
If offset 760201 cannot advance, later records in that partition wait.
Other partitions can continue.
Immediate retry is useful for brief transient failures.
It is harmful for deterministic incompatible bytes.
A retry topic delays and isolates retries using a new Kafka record.
A dead-letter topic, or DLT, holds terminal failures for investigation and governed recovery.
Neither is a trash can.
Both require stable event ID, original coordinates, error class, attempt, and schema metadata.

# What I Check FIRST

1. Locate the first failed stage.
   WHY: no listener log may mean deserialization failed before business code.
   LOOK FOR: fetch succeeded, deserialize failed, handler never started.
2. Scope by topic, partition, offset, schema, producer version, and consumer version.
   WHY: one poison record can block only one ordered partition.
   LOOK FOR: partition 3 and old consumer pods.
3. Classify failure as transient, permanent technical, or business.
   WHY: retry policy depends on whether time can change the outcome.
   LOOK FOR: deterministic schema mismatch versus a temporary timeout.
4. Inspect retry/error-handler behavior and attempt rate.
   WHY: a tight loop can amplify load and logs.
   LOOK FOR: bounded attempts, delay, and terminal routing.
5. Check side effects and commit state before recovery.
   WHY: a failure after partial work can make retry duplicate effects.
   LOOK FOR: idempotent event ID and stage completion evidence.

# Step-by-Step Investigation

### Step 1 - Confirm partition-local blockage

* What I check: lag, committed offset, LEO, oldest age, and consume rate per partition.
* Why: aggregate lag hides a single poison record.
* Expected: all partitions advance.
* Bad: partition 3 committed offset remains 760201 while LEO rises.
* Meaning: the failure at or after that offset blocks ordered progress.
* Next: identify the exact record stage.

### Step 2 - Follow stage counters

* What I check: fetched, deserialized, validated, handler-started, effect-succeeded, and committed counts.
* Why: the first non-advancing transition locates failure.
* Expected: counters remain close.
* Bad: fetched increases but deserialized does not.
* Meaning: failure occurs before listener invocation.
* Next: inspect deserializer error metadata safely.

### Step 3 - Inspect error class and cause

* What I check: root exception, schema version, content type, serializer headers, and target Java type.
* Why: wrapper exceptions can hide the actionable incompatibility.
* Expected: supported schema maps to the configured DTO.
* Bad: integer expected but string token received for warehouse.
* Meaning: incompatible producer/consumer contract.
* Next: compare deployed versions and schema compatibility policy.

### Step 4 - Check record identity and coordinates

* What I check: event ID, original topic, partition, offset, producer timestamp, key hash.
* Why: recovery must identify the exact logical event without logging payloads.
* Expected: stable event ID `evt-55203`.
* Bad: event ID exists only inside bytes that cannot deserialize.
* Meaning: recovery correlation is harder.
* Next: place essential identity and schema headers outside the payload.

### Step 5 - Classify retryability

* What I check: whether the outcome can change without changing bytes or code.
* Why: time helps a 503 but not a deterministic type mismatch.
* Expected: transient network errors use bounded retry.
* Bad: schema mismatch receives immediate infinite retry.
* Meaning: partition starvation and retry amplification.
* Next: stop tight retries and invoke tested terminal recovery.

### Step 6 - Inspect Spring Kafka error configuration

* What I check: `ErrorHandlingDeserializer`, `DefaultErrorHandler`, backoff, exception classifications, recoverer, and ack behavior.
* Why: configuration controls whether poison records loop, pause, or move to DLT.
* Expected: deserialization error is captured and terminally routed after policy.
* Bad: raw deserializer throws outside the recoverable path.
* Meaning: listener container repeatedly seeks without bounded recovery.
* Next: use the tested deserialization-aware error path.

### Step 7 - Check retry-topic design

* What I check: delay tiers, attempt limit, original headers, event ID, and consumer groups.
* Why: retry topics move transient failures away from the main partition.
* Expected: attempts are bounded and delayed.
* Bad: retry topic immediately republishes to itself.
* Meaning: a retry storm is created.
* Next: correct destination and backoff policy.

### Step 8 - Check DLT publication outcome

* What I check: DLT send intent, acknowledgment metadata, and failure callback.
* Why: claiming recovery before DLT acknowledgment can lose evidence.
* Expected: DLT acknowledgment contains topic, partition, and offset.
* Bad: original offset commits while DLT send failed.
* Meaning: poison record may be skipped without recoverable evidence.
* Next: fail safely according to the approved recovery design.

### Step 9 - Inspect partial side effects

* What I check: whether validation, DB, or HTTP effects occurred before the throw.
* Why: retry may repeat completed work.
* Expected: deserialization failure has no business effects.
* Bad: handler wrote DB then failed on notification.
* Meaning: stable event ID and transactional/idempotent boundaries are required.
* Next: reconcile inbox and downstream idempotency.

### Step 10 - Validate compatibility and rollout

* What I check: producer schema change, consumer support matrix, and mixed-version window.
* Why: backward compatibility prevents rolling-deployment breakage.
* Expected: old consumers can read new events during rollout.
* Bad: producer v4 is deployed before all consumers understand it.
* Meaning: release sequencing violated the event contract.
* Next: rollback producer or deploy a backward-compatible payload.

### Step 11 - Plan bounded remediation

* What I check: affected event IDs, count, business priority, corrected consumer version, and replay safety.
* Why: recovery must not duplicate or reorder effects accidentally.
* Expected: finite manifest and idempotent handler.
* Bad: proposed replay is "the entire topic from yesterday."
* Meaning: blast radius is uncontrolled.
* Next: require governance and exact boundaries.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| lag by partition | one rising partition suggests poison record or hot key |
| committed offset unchanged | durable progress is blocked |
| deserialize failure rate | failure before listener |
| validation failure rate | payload readable but invalid |
| handler exception rate | business code entered and failed |
| dependency error rate | external transient/permanent failure |
| attempts per event | retry amplification |
| retry-topic publish rate | deferred recovery traffic |
| retry age | how long transient work has waited |
| DLT publish ack/failure | whether terminal evidence is durable |
| DLT depth | unresolved terminal failures |
| listener start/success | stage progression |
| event age per partition | customer delay |
| log bytes/sec | tight error loop impact |
| rebalance rate | secondary churn |
| duplicate suppression | partial-work retries safely handled |

High deserialize failures with zero handler starts localize before business logic.
High handler failures with normal deserialization localize after conversion.
High DLT rate after deployment indicates compatibility or code regression.
Low DLT depth is not healthy if DLT publication itself fails.
If one partition's age rises while others stay current, investigate ordering blockage there.

# Distributed Trace Investigation

Deserialization can fail before standard listener instrumentation creates a consumer span.
Therefore trace absence is expected in this incident.
The producer span still records acknowledged coordinates and event ID.

```text
traceId=9c210f
order-service publish                         12 ms
  +-- orders.v1 partition=3 offset=760201
      eventId=evt-55203 schemaVersion=4

consumer process span                        MISSING
```

The missing child span has several possible explanations:

* the record was never fetched;
* deserialization failed before span creation;
* context propagation was missing;
* sampling dropped the span;
* the listener never owned the partition.

Stage metrics and error logs select deserialization here.
After recovery:

```text
traceId=recover71
DLT handling
  +-- classify schema_incompatible
  +-- publish orders.v1.DLT                  10 ms
      partition=3 offset=8121
      originalPartition=3 originalOffset=760201
```

The DLT acknowledgment proves durable routing, not business completion.
Replay later creates a new attempt span linked by event ID and original coordinates.

# Distributed Logs

```text
2026-09-13T19:05:02.171+05:30 INFO service=order-service instance=order-8 traceId=9c210f spanId=7aa1 eventId=evt-55203 action=send_ack topic=orders.v1 partition=3 offset=760201 schemaVersion=4
2026-09-13T19:05:02.190+05:30 ERROR service=fulfillment-service instance=fulfill-old-2 traceId=none spanId=none eventId=evt-55203 group=fulfillment-v3 topic=orders.v1 partition=3 offset=760201 stage=deserialize error=MismatchedInputException expected=integer actual=string attempt=1
2026-09-13T19:05:12.208+05:30 ERROR service=fulfillment-service instance=fulfill-old-2 traceId=none spanId=none eventId=evt-55203 group=fulfillment-v3 topic=orders.v1 partition=3 offset=760201 stage=deserialize error=MismatchedInputException attempt=10
2026-09-13T19:06:00.600+05:30 INFO service=fulfillment-service instance=fulfill-old-2 eventId=evt-55203 action=dlt_send_ack topic=orders.v1.DLT partition=3 offset=8121 originalTopic=orders.v1 originalPartition=3 originalOffset=760201 errorClass=schema_incompatible
```

`traceId=none` is explicit rather than a fabricated trace.
The same event ID and original coordinates correlate the producer and error.
Attempt growth proves a tight loop.
The DLT ack proves recovery evidence was stored.
One exception line does not establish fleet impact; partition metrics and version scope do.
Raw record bodies are excluded from logs.

# Commands / Tools

Read-only group and topic inspection:

```text
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --describe
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1 --describe
```

Read-only schema registry query, if the platform uses one:

```text
curl -s --max-time 5 https://schema-registry/subjects/orders-value/versions/latest
```

Windows equivalent:

```text
curl.exe -s --max-time 5 https://schema-registry/subjects/orders-value/versions/latest
```

TLS and authentication options must come from approved secret handling, not command history.
Actuator:

```text
curl.exe -s http://localhost:8080/actuator/prometheus
curl -s --max-time 5 http://localhost:8080/actuator/prometheus
```

These observations do not mutate offsets or messages.
Manual consume can expose production data and create load, so use approved tooling and redaction.
Publishing to DLT, skipping a record, changing offsets, or replaying is a governed action.
This guide describes those actions but does not instruct unapproved execution.

# Root Cause

The producer rollout introduced a backward-incompatible schema while old consumers remained.

```text
producer emits warehouseCode string
    |
old consumer expects warehouseId integer
    |
deserialization fails before listener
    |
error handler seeks to same offset immediately
    |
partition 3 repeats poison record
    |
later records in partition 3 wait
    |
lag, event age, CPU, and log volume rise
```

Broker fetch, leaders, replicas, and ISR were healthy.
The key-specific partition impact followed Kafka's ordering model.
The unbounded retry policy amplified a deterministic compatibility defect.

# Fix

Immediate mitigation:

* Stop the incompatible producer rollout.
* Deploy a consumer that reads both old and new representations.
* Use the tested recovery policy to route the poison record only after DLT acknowledgment.
* Preserve original coordinates, event ID, schema version, and error class.
* Keep affected event IDs for governed replay.
* Do not silently commit and discard.

Permanent fix:

* Enforce backward compatibility in schema CI.
* Use additive evolution and tolerant readers.
* Configure deserialization-aware error handling.
* Classify deterministic schema errors as non-retriable.
* Use bounded retry topics for genuinely transient failures.
* Alert on DLT acknowledgments and failures.
* Make replay idempotent with stable event IDs and inbox constraints.

# Verification

Before:

* Partition 3 lag: 18,400 and rising.
* Deserialize failures: 84/min.
* Attempts for one event: 600 in ten minutes.
* Handler starts for that coordinate: 0.
* Oldest partition-3 event age: 31 minutes.

After:

* Corrected consumers deserialize both schema versions.
* Poison record is durably present in DLT with original lineage.
* Main partition committed offset advances beyond 760201.
* Deserialize error loop stops.
* Partition 3 drains at 1,100/min.
* DLT replay of the bounded event later creates one business effect.
* Duplicate effects remain zero.

I verify DLT success before considering main-partition recovery complete.
I test mixed producer/consumer versions in staging.

# Prevention

* Enforce schema compatibility before publish.
* Include schema version and stable event ID in safe headers.
* Instrument all processing stages.
* Bound retries and classify exceptions.
* Use retry topics for delayed transient recovery.
* Use DLT for terminal evidence with ownership and alerts.
* Preserve original topic, partition, offset, timestamp, and error metadata.
* Test mixed-version rolling deployments.
* Make effects idempotent for retry and replay.
* Maintain a governed DLT runbook.

# Interview Answer

### What I would say in an interview

I first locate the failure stage. If fetch increases but the listener never starts, I inspect deserialization rather than handler code. I scope the exact topic, partition, offset, event ID, schema, and deployed versions, then classify whether time can change the outcome. A deterministic schema mismatch should not be retried in a tight loop because it blocks partition order and amplifies load. Here an incompatible producer field reached old consumers, and the handler repeatedly sought the same offset. We stopped the rollout, deployed a tolerant reader, and durably routed the poison record with full lineage. I verified partition progress, DLT acknowledgment, bounded replay, and one idempotent business effect.

### Common interviewer traps

* No listener log does not mean no fetch; deserialization may fail first.
* Infinite retry is not reliability for deterministic failures.
* A DLT send attempt is not a DLT acknowledgment.
* Skipping an offset can lose business work.
* Replaying without idempotency can repeat effects.

### Quick memory flow

Partition/offset -> first failed stage -> exception class -> retryability -> side effects -> bounded recovery -> DLT ack -> partition progress -> governed replay.

# Interview Follow-up Questions

1. **What is a poison record?** A record that deterministically fails and is unlikely to succeed through immediate retry.
2. **Why can one record grow lag?** Partition order keeps later records behind the repeatedly failing offset.
3. **Why might no trace exist?** Deserialization can fail before consumer span creation.
4. **Retry topic versus DLT?** Retry topics delay transient attempts; DLT stores terminal failures for investigation and recovery.
5. **What metadata belongs in DLT?** Stable event ID, original coordinates, timestamp, schema, attempts, and sanitized error class.
6. **Should you commit after DLT send intent?** No; require the recovery design's durable acknowledgment.
7. **How do you avoid schema incidents?** Backward-compatible evolution, tolerant readers, CI checks, and mixed-version tests.
8. **How is replay made safe?** Bounded selection, preserved IDs, idempotent effects, unique constraints, and verification.
