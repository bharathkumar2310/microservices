# Problem

Some orders create two shipment reservations even though Kafka contains one logical event.
Kafka consumers commonly provide at-least-once processing.
At-least-once means a record can be delivered again until progress is safely committed.
It does not mean every record is always duplicated.
It means application effects must tolerate redelivery.
The dangerous window is after an external effect succeeds but before the offset commit succeeds.
A crash in that window causes the next member to resume from the older committed offset.
Ordering is guaranteed only within one partition, not across a topic.
Duplicate detection must use a stable event ID, not arrival time or payload equality.

# Production Situation

The Meridian team recovered unpublished events from the outbox.
At 15:40, two fulfillment pods restart during node maintenance.
Group `fulfillment-v3` receives 24,600 records.
Kafka records-delivered count is 24,742, so 142 deliveries are repeats.
Shipment database inserts rise to 24,671.
Seventy-one orders show two reservations.
The consumer uses manual acknowledgment after listener return.
Its handler calls `shipment-api`, then writes a local audit row.
A pod is killed after the shipment API succeeds but before acknowledgment.
The replacement member starts at the committed offset and redelivers.
The shipment API has no idempotency key.
Producer idempotence is enabled and topic inspection finds one Kafka record per event ID.
The duplication therefore occurs at the consumer effect boundary.

# Architecture

```text
outbox-relay
    |
    | eventId=evt-44192, key=order-9021
    v
orders.v1 partition 5 offset 740119
    |
    | group=fulfillment-v3
    v
fulfillment listener
    |
    +--> shipment-api reserve
    |
    +--> local audit
    |
    +--> commit offset 740120
```

Kafka stores the record at one partition and offset.
The group commits the next offset to resume from.
Consumer position may already be 740120 while committed offset remains 740119.
If the process dies then, another member reads offset 740119 again.
The same delivery is not necessarily a second Kafka append.
Producer duplicates, consumer redeliveries, retry-topic copies, and duplicate business effects are separate facts.

# What I Check FIRST

1. Define the duplicate layer.
   WHY: two logs, two deliveries, two Kafka records, and two DB rows are different.
   LOOK FOR: stable event ID and topic/partition/offset for each occurrence.
2. Compare consumer effect time with commit time and restart/rebalance time.
   WHY: the crash-after-effect window is a common at-least-once pattern.
   LOOK FOR: effect success before restart, commit absent, then redelivery.
3. Check producer acknowledgments and records by event ID.
   WHY: this distinguishes duplicate append from consumer redelivery.
   LOOK FOR: one coordinate versus several coordinates.
4. Check idempotency controls at every external effect.
   WHY: producer idempotence cannot protect an HTTP or database side effect.
   LOOK FOR: unique event ID constraint, inbox record, or idempotency key.
5. Check commit mode, error handler, retries, and rebalances.
   WHY: configuration controls when redelivery occurs.
   LOOK FOR: offsets committed only after all required effects succeed.

# Step-by-Step Investigation

### Step 1 - Establish a logical identity

* What I check: immutable `eventId`, business order ID, event type, and schema version.
* Why: Kafka coordinates identify a stored record; event ID identifies logical intent across copies.
* Expected: every retry and replay retains `evt-44192`.
* Bad: a new UUID is generated on every retry.
* Meaning: downstream systems cannot reliably recognize the same logical event.
* Next: correct the event identity contract.

### Step 2 - Count Kafka records

* What I check: acknowledged coordinates and authorized read-only inspection for the event ID.
* Why: two effects do not prove two records were appended.
* Expected: one record at partition 5 offset 740119.
* Bad: same event ID appears at offsets 740119 and 740121.
* Meaning: producer, outbox relay, or replay appended twice.
* Next: inspect producer acknowledgments and outbox publish state.

### Step 3 - Count delivery attempts

* What I check: consumer attempt logs keyed by group, topic, partition, offset, and event ID.
* Why: the same stored record may be delivered repeatedly.
* Expected: one attempt during normal operation.
* Bad: attempts occur on two members around a rebalance.
* Meaning: ownership changed before durable commit.
* Next: align effect, crash, revocation, and commit timestamps.

### Step 4 - Inspect offset chronology

* What I check: position, committed offset, LEO, acknowledgment callback, and commit errors.
* Why: only committed progress survives member replacement.
* Expected: effect success followed by commit of the next offset.
* Bad: effect succeeds at 15:40:12, pod exits at 15:40:13, no commit exists.
* Meaning: redelivery is expected under at-least-once behavior.
* Next: inspect whether the effect is idempotent.

### Step 5 - Inspect commit timing

* What I check: Spring container ack mode and whether async work escapes listener completion.
* Why: committing before required work risks loss; committing after work permits duplicates.
* Expected: commit occurs after durable, required processing.
* Bad: listener launches async work and returns, causing early commit.
* Meaning: a later async failure can lose business processing.
* Bad: effect succeeds but process crashes before commit.
* Meaning: duplicate-safe effect is required.

### Step 6 - Check database idempotency

* What I check: inbox table and unique constraint on `(consumer_name, event_id)`.
* Why: check-then-insert without a constraint races across attempts.
* Expected: second insert conflicts and handler treats it as already processed.
* Bad: only an in-memory cache tracks event IDs.
* Meaning: restart, eviction, or another instance permits duplicates.
* Next: add a durable uniqueness boundary.

### Step 7 - Check HTTP idempotency

* What I check: `Idempotency-Key: evt-44192` and shipment API storage of the result.
* Why: local DB deduplication cannot undo a remote call already repeated.
* Expected: repeated request returns the original reservation.
* Bad: each call creates a new reservation ID.
* Meaning: remote effect is not idempotent.
* Next: add an idempotency contract or redesign ownership of the effect.

### Step 8 - Inspect producer retry duplication

* What I check: producer idempotence, acknowledgments, retries, producer restarts, and outbox state updates.
* Why: an ambiguous acknowledgment can prompt a second publish.
* Expected: idempotent producer suppresses supported retry duplicates.
* Bad: outbox row remains unpublished after ack and another relay republishes it.
* Meaning: logical duplicate append is possible across relay attempts.
* Next: retain stable event ID and make consumer effects idempotent anyway.

### Step 9 - Inspect retry topics and DLT

* What I check: original event ID, original coordinates, attempt number, and destination topic.
* Why: a retry-topic record is a new Kafka coordinate for the same logical event.
* Expected: headers preserve identity and lineage.
* Bad: retry publisher creates a new event ID.
* Meaning: deduplication and trace correlation break.
* Next: fix retry publishing contract.

### Step 10 - Reproduce the crash window

* What I check: controlled test that terminates a consumer after effect success before commit.
* Why: deterministic fault injection validates at-least-once handling.
* Expected: redelivery occurs and unique constraints prevent a second effect.
* Bad: two reservations are created.
* Meaning: idempotency is incomplete.
* Next: close the effect boundary before production rollout.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| unique event IDs produced | logical publication count |
| producer ack count | physical acknowledged appends |
| records delivered | attempts, including redelivery |
| unique events processed | logical consumer work |
| duplicate attempts suppressed | idempotency is actively protecting effects |
| duplicate effects | correctness failure; should be zero |
| commit success/failure | durable group progress |
| position minus committed | in-flight uncommitted window |
| rebalances | high values increase handoff/redelivery opportunities |
| consumer restarts | correlate with repeat attempts |
| handler success latency | long work enlarges crash window |
| retry-topic publish rate | controlled retry volume |
| DLT rate | terminal processing failures |
| unique-constraint conflicts | expected duplicate detection versus abuse |
| downstream idempotency hits | repeated remote requests safely reused |

If delivered count exceeds unique event IDs, redelivery or retry copies exist.
If Kafka has one coordinate but effects are two, the duplication is downstream.
If producer acks show two coordinates with one event ID, publication repeated.
If duplicate-suppressed increases during restarts while duplicate-effects remains zero, controls work.
If commits occur before handler success, low duplicate rate may hide message loss.

# Distributed Trace Investigation

Each delivery attempt receives its own consumer span.
All attempts retain the same event ID.
They may link to the original producer context rather than pretending to be one continuous synchronous call.

```text
eventId=evt-44192
producer span
  +-- orders.v1 partition=5 offset=740119

consumer attempt=1 instance=fulfill-3
  +-- shipment POST                    212 ms success reservation=r-801
  X-- process terminated before commit

consumer attempt=2 instance=fulfill-9
  +-- shipment POST                     18 ms idempotency_hit reservation=r-801
  +-- inbox unique insert                2 ms duplicate
  +-- commit nextOffset=740120            success
```

Before the fix, attempt 2 created `r-802`.
The trace shows where logical duplication became a business duplicate.
Kafka producer idempotence cannot cover the shipment span.
A missing second trace does not disprove redelivery if sampling differs.
Coordinates and structured logs provide deterministic correlation.
Do not put customer data or complete payloads in span attributes.

# Distributed Logs

```text
2026-09-13T15:40:12.104+05:30 INFO service=fulfillment-service instance=fulfill-3 traceId=ac910e spanId=118a eventId=evt-44192 group=fulfillment-v3 topic=orders.v1 partition=5 offset=740119 attempt=1 stage=shipment_success reservationId=r-801
2026-09-13T15:40:13.002+05:30 WARN service=fulfillment-service instance=fulfill-3 event=shutdown reason=node_termination commitCompleted=false
2026-09-13T15:40:17.491+05:30 INFO service=fulfillment-service instance=fulfill-9 traceId=be771a spanId=410c eventId=evt-44192 group=fulfillment-v3 topic=orders.v1 partition=5 offset=740119 attempt=2 stage=handler_started
2026-09-13T15:40:17.702+05:30 ERROR service=fulfillment-service instance=fulfill-9 traceId=be771a spanId=410c eventId=evt-44192 stage=shipment_success reservationId=r-802 duplicateEffect=true
```

The same coordinate proves record redelivery.
Different trace IDs are normal for separate attempts.
The same event ID connects the logical work.
The missing commit between attempts explains why Kafka offered it again.
Logs alone do not prove that Kafka violated delivery guarantees.
I verify group commits, pod termination, and database rows.

# Commands / Tools

```text
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --describe
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1 --describe
```

These read-only commands show offsets and topic metadata.
They do not identify duplicate external effects.

Database read-only reconciliation:

```text
SELECT event_id, COUNT(*) AS reservations
FROM shipment_reservation
WHERE created_at >= :incident_start
GROUP BY event_id
HAVING COUNT(*) > 1;
```

The query finds duplicate rows by stable event ID.
It does not prove which component caused them.

Windows and Linux metrics:

```text
curl.exe -s http://localhost:8080/actuator/prometheus
curl -s --max-time 5 http://localhost:8080/actuator/prometheus
```

Production database queries and message inspection require approved read-only access.
Offset reset, DLT replay, record deletion, and compensating updates are governed actions.
I describe them, bound affected event IDs, obtain approval, and audit outcomes.

# Root Cause

The consumer implemented an at-least-once flow without an idempotent shipment effect.

```text
record delivered at offset 740119
    |
shipment reservation r-801 created
    |
pod terminates before committing 740120
    |
replacement resumes from committed 740119
    |
same record is delivered again
    |
shipment API creates r-802
```

Kafka contained one original record for `evt-44192`.
The producer's idempotence worked as designed.
The duplicate was caused by the unavoidable crash window plus a non-idempotent external API.
At-least-once exposed the application correctness gap.

# Fix

Immediate mitigation:

* Stop the rollout that causes abrupt termination.
* Use a bounded list of duplicate event IDs to block further shipment creation.
* Reconcile duplicate reservations with the shipment owner before compensation.
* Do not skip offsets to hide the symptom.

Permanent fix:

* Send stable event ID as the shipment idempotency key.
* Make shipment API persist key and response under a unique constraint.
* Add a local inbox unique constraint for local state changes.
* Treat duplicate-key conflict as successful prior processing where semantics match.
* Commit only after required durable work succeeds.
* Preserve event ID across retries, DLT, and replay.
* Test crash-after-effect-before-commit explicitly.

# Verification

Before:

* 24,600 unique events.
* 24,742 delivery attempts.
* 142 redeliveries.
* 71 duplicate shipment effects.
* Duplicate suppression count: 0.

After controlled restart:

* 10,000 unique test events.
* 10,137 delivery attempts.
* 137 redeliveries.
* 137 idempotency hits.
* 0 duplicate shipment effects.
* All offsets eventually commit.
* Event age returns to normal.

Redelivery still occurring is acceptable under the chosen semantics.
Zero repeated effects proves idempotency, not an unrealistic absence of retries.
I reconcile database uniqueness, remote reservation IDs, and consumer commits.

# Prevention

* Require stable event IDs in event contracts.
* Use database unique constraints, not only application check-then-act.
* Implement inbox/outbox patterns at ownership boundaries.
* Define idempotency contracts for external APIs.
* Preserve lineage through retry topics and DLT.
* Monitor delivery attempts versus unique effects.
* Test process death, rebalance, commit timeout, and retry paths.
* Use graceful shutdown but never depend on it for correctness.
* Document ordering as partition-scoped.
* Avoid claims of exactly-once external side effects.

# Interview Answer

### What I would say in an interview

I first define which layer is duplicated: Kafka append, consumer delivery, or business effect. I correlate stable event ID with topic, partition, offset, attempts, commits, and downstream rows. Under at-least-once processing, a crash after an external effect but before committing the next offset causes valid redelivery. Here Kafka had one record, but shipment creation had no idempotency key, so the second attempt created another reservation. We fixed the boundary with a stable event ID, a unique inbox constraint, and downstream idempotency, while keeping commits after required work. Verification deliberately caused redelivery and proved zero duplicate effects. I would never claim Kafka alone makes external effects exactly once.

### Common interviewer traps

* Two consumer logs do not prove two Kafka appends.
* Committing before work avoids some duplicates by risking loss.
* In-memory deduplication fails across restart and multiple instances.
* Producer idempotence does not deduplicate HTTP or database effects.
* Global ordering is not guaranteed across partitions.

### Quick memory flow

Stable event ID -> count records -> count attempts -> effect timestamp -> commit timestamp -> crash/rebalance -> idempotency boundary -> verify with fault injection.

# Interview Follow-up Questions

1. **Why does Kafka redeliver?** The group resumes from committed progress when prior processing was not durably committed.
2. **What is at-least-once?** Processing may repeat, so effects must be duplicate-safe.
3. **Why commit after processing?** It avoids claiming progress before required work succeeds, at the cost of possible redelivery.
4. **What is a stable event ID?** An immutable logical operation identifier retained across attempts, retries, and replay.
5. **Why is a unique constraint important?** It makes concurrent deduplication atomic and durable.
6. **What is an inbox?** A durable consumer-side record of processed event IDs, usually written with local effects.
7. **Does Kafka preserve global order?** No, only order within each partition.
8. **How do you test duplicate safety?** Kill the consumer after effect success before commit and verify one logical effect.
