# Problem

Failed events must be retried or replayed without causing a second incident.
Retry handles an attempt that may succeed later.
Replay intentionally processes historical events again.
They have different triggers, controls, and blast radius.
Immediate retries are suitable only for brief transient faults.
Delayed retry topics free the main partition while time passes.
A DLT stores terminal failures and their evidence.
Replay is a governed production change because it creates load and can repeat effects.
Kafka coordinates locate physical records.
Stable event IDs preserve logical identity across retry and replay copies.

# Production Situation

The Meridian team has 1,284 schema-incompatible records in `orders.v1.DLT`.
All corrected consumers now support schema versions 3 and 4.
The affected window is 19:05-19:21.
The manifest contains 1,284 unique event IDs.
Normal traffic is 5,000 events/min.
The group can sustainably process 6,800/min.
The shipment database safely supports 7,000 writes/min.
The fraud API safely supports 100 requests/sec, or 6,000/min.
A naive replay at 4,000/min plus normal traffic would request up to 9,000/min.
That would exceed both consumer recovery headroom and fraud capacity.
All consumer effects now use event ID idempotency.
The inbox table has a unique `(consumer_name, event_id)` constraint.
Before replay, 31 DLT events already have completed business effects from partial earlier attempts.
The plan therefore uses a bounded 800/min rate and full reconciliation.

# Architecture

```text
orders.v1
    |
    +--> transient failure
    |       |
    |       v
    |   orders.v1.retry.1m
    |       |
    |       v
    |   orders.v1.retry.10m
    |
    +--> terminal failure
            |
            v
        orders.v1.DLT
            |
            | governed bounded replay
            v
        orders.v1.replay
            |
            v
        fulfillment-replay group
            |
            +--> inbox uniqueness
            +--> idempotent shipment API
```

A retry copy has a new topic, partition, and offset.
It must retain the original event ID and lineage.
A DLT record is not automatically safe to replay.
The code or dependency must first be corrected.
The effect must be idempotent.
Capacity must include normal traffic plus replay traffic.
Ordering across the original and retry topics is not automatic.
Business rules must state whether delayed records can overtake newer events.

# What I Check FIRST

1. Classify the original failure and prove it is fixed.
   WHY: replaying unchanged failures recreates the DLT loop.
   LOOK FOR: corrected schema support and passing canary.
2. Build an exact manifest of event IDs and original coordinates.
   WHY: "replay yesterday" has an unsafe and unverifiable scope.
   LOOK FOR: 1,284 unique, reconciled events.
3. Verify idempotency and partial prior effects.
   WHY: 31 events already changed business state.
   LOOK FOR: durable inbox uniqueness and downstream idempotency.
4. Calculate combined rate and downstream headroom.
   WHY: replay can overload a healthy live path.
   LOOK FOR: normal 5,000/min plus replay 800/min within every limit.
5. Define pause, rollback, audit, and success criteria.
   WHY: a controlled replay must be stoppable and measurable.
   LOOK FOR: owner, change approval, dashboards, and reconciliation query.

# Step-by-Step Investigation

### Step 1 - Understand retry versus replay

* What I check: whether this is automatic recovery for a current attempt or intentional historical reprocessing.
* Why: operational controls and expected ordering differ.
* Expected: transient failures use bounded retry policy; corrected terminal records use governed replay.
* Bad: all failures are immediately retried forever.
* Meaning: retry classification is absent.
* Next: define transient, permanent technical, and business error classes.

### Step 2 - Prove the original cause is removed

* What I check: corrected version, schema tests, dependency health, and a representative DLT canary.
* Why: replay is not a repair for unchanged code.
* Expected: schema versions 3 and 4 deserialize and process.
* Bad: canary returns the same mismatch.
* Meaning: stop; replay would only repeat failure.
* Next: fix and redeploy before reconsidering.

### Step 3 - Create a bounded manifest

* What I check: unique event ID, original topic, partition, offset, produced time, error class, attempts.
* Why: exact scope makes approval and verification possible.
* Expected: 1,284 rows, no duplicate event IDs unless explained.
* Bad: unbounded time range or unknown count.
* Meaning: blast radius cannot be controlled.
* Next: refine selection with read-only evidence.

### Step 4 - Reconcile existing effects

* What I check: inbox, shipment reservation, audit, and downstream idempotency records.
* Why: a terminal failure may occur after some effects succeeded.
* Expected: 31 partial-success event IDs are known and protected.
* Bad: event IDs are missing from side-effect tables.
* Meaning: safe duplicate recognition is impossible.
* Next: add reconciliation or a business-approved compensation plan.

### Step 5 - Validate event identity and lineage

* What I check: replay retains original event ID while recording a new replay ID or run ID separately.
* Why: generating a new event ID defeats deduplication.
* Expected: `eventId=evt-55203`, `replayRunId=rr-20260913-01`.
* Bad: event ID becomes a random replay UUID.
* Meaning: consumers see a new logical operation.
* Next: correct the replay envelope.

### Step 6 - Review ordering requirements

* What I check: entity key, current entity version, original sequence, and whether stale events may apply.
* Why: replayed old events can arrive after newer events.
* Expected: version checks reject obsolete state transitions safely.
* Bad: old `OrderAccepted` overwrites later `OrderCancelled`.
* Meaning: replay can corrupt current state despite successful processing.
* Next: add optimistic version or state-transition guards.

### Step 7 - Calculate safe rate

* What I check: normal produce rate, sustainable consumer rate, DB capacity, fraud capacity, and safety margin.
* Why: the narrowest dependency sets the replay budget.
* Expected: 5,000 normal plus 800 replay stays under 6,000 fraud calls/min.
* Bad: proposal adds 4,000/min.
* Meaning: combined load would exceed capacity and create new lag.
* Next: lower rate, schedule a safer window, or raise approved capacity.

### Step 8 - Define retry policy

* What I check: exception classifier, max attempts, delays, exponential backoff, jitter, and terminal route.
* Why: retries must give transient faults time without amplification.
* Expected: two or three bounded attempts within the business latency budget.
* Bad: every exception receives the same immediate retry.
* Meaning: permanent failures waste capacity and transient outages receive a retry storm.
* Next: separate non-retriable schema/business errors.

### Step 9 - Define replay isolation

* What I check: replay topic/group, concurrency, rate limit, and live-path dependency budget.
* Why: isolation permits pause without disturbing normal group offsets.
* Expected: dedicated `fulfillment-replay` group and bounded workers.
* Bad: production group offsets are reset backward.
* Meaning: all historical records may be redelivered with unclear scope.
* Next: reject the plan and use a reviewed bounded path.

### Step 10 - Define commit and failure behavior

* What I check: successful effect, inbox transaction, acknowledgment, and re-DLT behavior.
* Why: replay itself still has crash and duplicate windows.
* Expected: idempotent effects complete before durable progress; failures remain observable.
* Bad: replay commits before asynchronous effect completion.
* Meaning: replay can silently lose the intended recovery.
* Next: align commit with durable processing.

### Step 11 - Run a canary batch

* What I check: 10 approved event IDs across schemas, partitions, and partial-effect states.
* Why: small scope tests correctness and capacity.
* Expected: 10 terminal outcomes, including safe idempotency hits.
* Bad: unexpected side effect, lag spike, or DLT repeat.
* Meaning: pause and investigate.
* Next: do not expand until the discrepancy is resolved.

### Step 12 - Monitor full bounded run

* What I check: input rate, success, idempotency hit, failure, event age, live lag, dependency saturation.
* Why: replay success cannot come at the expense of current traffic.
* Expected: live SLO stays healthy and manifest drains predictably.
* Bad: Hikari pending rises or live event age increases.
* Meaning: replay rate is too high.
* Next: pause or lower replay rate under the runbook.

### Step 13 - Reconcile completion

* What I check: every manifest event ends as processed, already processed, rejected by current-state rule, or still failed.
* Why: "messages sent" is not a business outcome.
* Expected: counts sum exactly to 1,284.
* Bad: unexplained gap or duplicate effect.
* Meaning: recovery is incomplete.
* Next: investigate specific event IDs before closing.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| normal produced rate | live demand that retains priority |
| replay publish acknowledgment rate | physical records accepted for replay |
| replay processing success rate | useful recovery outcomes |
| unique replay event count | logical scope progress |
| idempotency hit rate | prior effects safely detected |
| replay failure rate | corrected path still failing |
| retry attempts/event | bounded policy effectiveness |
| retry-topic depth/age | transient backlog |
| DLT depth/oldest age | unresolved terminal work |
| live group lag/age | replay impact on current customers |
| replay group lag | run progress |
| DB pool pending | downstream saturation |
| fraud rate/429 | external capacity breach |
| commit failure | replay durability risk |
| duplicate effects | correctness failure |
| stale-version rejection | ordering/state protection |

High replay acknowledgment with low processing success means publishing is not recovery.
High idempotency hits can be expected for partial prior effects.
High live event age means replay is stealing too much capacity.
Low DLT depth is not enough; all manifest outcomes must reconcile.
A sudden retry increase after replay begins suggests the original fix is incomplete or capacity is exceeded.

# Distributed Trace Investigation

The replay span links to original lineage but is a new processing attempt.
It includes replay run ID and original coordinates.

```text
eventId=evt-55203 replayRunId=rr-20260913-01
replay publisher
  +-- publish orders.v1.replay              9 ms
      partition=2 offset=6101
      original=orders.v1/3/760201

replay consumer
  +-- inbox insert                          3 ms
  +-- shipment-api                         42 ms idempotency_hit
  +-- current-state check                   4 ms
  +-- commit replay offset                  success
```

For an event without prior effect, shipment returns a new single reservation.
For a stale canceled order, the state guard returns `obsolete_event` without overwriting state.
Retry spans must show attempt and delay.
The trace proves the path for sampled events, not completeness.
Manifest reconciliation and metrics prove population outcomes.
A missing child span may reflect sampling or propagation loss.
Original coordinates and stable event ID preserve investigation capability.

# Distributed Logs

```text
2026-09-13T21:00:00.010+05:30 INFO service=replay-controller instance=replay-1 traceId=4bd110 spanId=aa10 replayRunId=rr-20260913-01 eventId=evt-55203 action=replay_send_intent originalTopic=orders.v1 originalPartition=3 originalOffset=760201 destination=orders.v1.replay
2026-09-13T21:00:00.019+05:30 INFO service=replay-controller instance=replay-1 traceId=4bd110 spanId=aa10 replayRunId=rr-20260913-01 eventId=evt-55203 action=replay_send_ack topic=orders.v1.replay partition=2 offset=6101 durationMs=9
2026-09-13T21:00:00.071+05:30 INFO service=fulfillment-replay instance=replay-consumer-2 traceId=91cbee spanId=b702 replayRunId=rr-20260913-01 eventId=evt-55203 originalTopic=orders.v1 originalPartition=3 originalOffset=760201 result=already_processed idempotencyHit=true
2026-09-13T21:01:00.000+05:30 INFO service=replay-controller instance=replay-1 replayRunId=rr-20260913-01 published=800 processed=794 alreadyProcessed=19 obsolete=3 failed=3 inFlight=1 liveEventAgeP99Ms=8200
```

The intent/ack distinction remains important during replay.
The run ID groups operational activity.
The event ID preserves logical identity.
The original coordinates preserve provenance.
An `already_processed` result is successful only after verifying the existing effect.
One successful log is not proof all 1,284 events recovered.

# Commands / Tools

Read-only diagnosis:

```text
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --describe
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-replay --describe
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1.DLT --describe
```

These show group progress and topic metadata.
They do not validate business effects.

Read-only reconciliation example:

```text
SELECT replay_status, COUNT(*)
FROM replay_manifest
WHERE replay_run_id = :run_id
GROUP BY replay_status;
```

Read-only duplicate check:

```text
SELECT event_id, COUNT(*)
FROM shipment_reservation
WHERE event_id IN (:approved_event_ids)
GROUP BY event_id
HAVING COUNT(*) > 1;
```

Windows and Linux monitoring:

```text
curl.exe -s http://localhost:8080/actuator/prometheus
curl -s --max-time 5 http://localhost:8080/actuator/prometheus
```

Actual replay publishing, offset reset, DLT deletion, and compensation mutate production.
They require approval, exact scope, audit logging, rate controls, pause criteria, and owner presence.
No executable mutating Kafka command is provided here.

# Root Cause

The DLT backlog came from the schema incompatibility described in the previous incident.
The recovery risk came from treating replay as simple republishing.

```text
1,284 terminal records require recovery
    |
naive plan proposes 4,000 replay/min
    |
normal 5,000/min plus replay exceeds 6,000/min fraud capacity
    |
fraud returns 429
    |
retries amplify calls
    |
live lag and new DLT records would grow
```

Additionally, 31 records already had partial effects.
Without stable IDs and idempotency, replay would duplicate shipments.
The correct root-cause response combines schema repair, bounded capacity, ordering guards, and effect idempotency.

# Fix

Immediate governed recovery:

* Freeze the exact 1,284-event manifest.
* Prove the corrected consumer with a 10-event canary.
* Preserve event IDs and original coordinates.
* Run isolated replay at no more than 800/min.
* Pause if live event age, DB pending, fraud 429, or replay failure crosses limits.
* Reconcile every terminal outcome.

Permanent fix:

* Classify retryable exceptions explicitly.
* Use delayed retry topics with bounded attempts and jitter.
* Route terminal failures to an owned, monitored DLT.
* Store replay manifests and approvals.
* Make local effects atomic with inbox uniqueness.
* Require downstream idempotency keys.
* Add entity-version checks for stale replay.
* Capacity-test combined live and replay traffic.

# Verification

Before recovery:

* DLT manifest: 1,284 unique events.
* Existing partial effects: 31.
* Oldest failed event age: 1 hour 55 minutes.
* Proposed unsafe combined rate: 9,000/min.

After governed recovery:

* Processed new effects: 1,241.
* Already processed safely: 31.
* Obsolete by current-state rule: 12.
* Remaining failed: 0.
* Total reconciled: `1,241 + 31 + 12 = 1,284`.
* Duplicate effects: 0.
* Live event age p99 remains under 10 seconds.
* DB pending remains 0.
* Fraud 429 rate remains below 0.1%.
* Replay and live group lag return to baseline.

Exact reconciliation proves completeness.
Business state checks prove outcome correctness.
Stable live-path metrics prove recovery did not create collateral damage.

# Prevention

* Enforce backward-compatible schemas.
* Maintain bounded retry classifications and budgets.
* Monitor DLT depth, oldest age, publication failure, and ownership.
* Require stable IDs through original, retry, DLT, and replay.
* Use inbox/outbox and unique constraints.
* Define per-entity ordering and stale-event rules.
* Precalculate downstream capacity for replay.
* Require canary, pause thresholds, and reconciliation.
* Audit who approved and ran each replay.
* Practice recovery in a non-production environment with realistic load.

# Interview Answer

### What I would say in an interview

I distinguish retry from replay. Retries are bounded responses to failures that may clear; replay is an intentional, governed reprocessing of historical events. Before replay, I prove the original defect is fixed, build an exact event manifest, preserve stable IDs and original coordinates, reconcile partial effects, and verify idempotency. Then I calculate normal plus replay load against the narrowest dependency, use an isolated group and a small canary, and define pause criteria. Here 4,000 replay events per minute would have overloaded fraud, so we used 800. We verified all 1,284 outcomes, zero duplicate effects, healthy live event age, and no remaining failures. Kafka alone does not guarantee exactly-once external effects.

### Common interviewer traps

* Replaying an entire time range is not a bounded recovery plan.
* New event IDs defeat deduplication.
* Empty DLT does not prove correct business outcomes.
* Retry without backoff can amplify an outage.
* Resetting production group offsets is not a harmless replay method.

### Quick memory flow

Classify -> fix cause -> bound manifest -> reconcile effects -> preserve identity -> check order -> budget capacity -> canary -> rate-limit -> monitor -> reconcile.

# Interview Follow-up Questions

1. **Retry versus replay?** Retry is policy-driven reattempt of current failure; replay is deliberate historical reprocessing.
2. **Why use retry topics?** They delay transient work and prevent one record from monopolizing the main partition.
3. **What belongs in a DLT record?** Stable ID, original coordinates, timestamps, schema, attempts, and sanitized error context.
4. **Why preserve event ID?** It lets inboxes and downstream systems recognize the same logical operation.
5. **How do you choose replay rate?** Use the smallest headroom across consumers and every dependency, with safety margin.
6. **How do you handle old events after new ones?** Enforce entity version and valid state transitions.
7. **What proves replay completion?** Exact manifest reconciliation to final business outcomes, not publish count.
8. **Can you promise exactly-once external effects?** No; use idempotency, uniqueness, inbox/outbox, and reconciliation.
