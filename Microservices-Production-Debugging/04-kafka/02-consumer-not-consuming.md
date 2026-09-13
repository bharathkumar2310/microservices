# Problem

The topic contains records, yet the Spring Kafka application appears not to consume them.
"Not consuming" has several distinct meanings.
The member may not have joined the intended group.
It may be in the group but own no partitions.
It may own partitions but fetch nothing.
It may fetch records but fail deserialization before the listener.
It may enter the listener and block before producing a visible business effect.
It may process successfully while an observer checks the wrong group or environment.
The investigation follows the record through assignment, fetch, deserialize, queue, handler, dependency, and commit.

# Production Situation

The Meridian incident continues the next morning.
Lag from the promotion was cleared overnight.
A deployment of `fulfillment-service` 3.18 starts at 09:00.
`orders.v1` receives 2,100 records/min.
Group `fulfillment-v3` consume rate drops from 2,100/min to nearly zero.
Lag rises by 2,098/min.
Twelve pods report Spring Boot health `UP`.
The topic has 12 partitions.
The group command shows constant membership changes.
Rebalances increase from 0-1/hour to 46 in 15 minutes.
Listener handler p95 is 4 minutes 52 seconds.
`max.poll.interval.ms` is 300,000 ms.
The deployment introduced a synchronous bulk-pricing call inside the listener.
Customers see orders stuck in "Accepted."

# Architecture

```text
order-service
    |
    | OrderCreated
    v
orders.v1: 12 partitions
    |
    | group fulfillment-v3
    v
12 Spring Kafka pods
    |
    +--> deserialize
    +--> listener queue
    +--> pricing-api
    +--> shipment-db
    +--> offset commit
```

The group coordinator tracks membership and assignments.
Each member heartbeats to show it is alive.
The consumer must also call `poll()` frequently enough.
`session.timeout.ms` bounds missed heartbeats.
`max.poll.interval.ms` bounds time between polls.
With background heartbeats, a process may look alive while processing too long.
If max poll interval is exceeded, the coordinator removes the member.
Its partitions are revoked and assigned again.
That rebalance can make healthy-looking pods perform little useful work.

# What I Check FIRST

1. Check topic, cluster, group ID, and environment configuration.
   WHY: looking at the wrong group or cluster creates a false incident.
   LOOK FOR: the deployed effective values, not only source configuration.
2. Check group state, members, assignments, and repeated rebalances.
   WHY: a consumer without an assignment cannot read records.
   LOOK FOR: stable ownership of all expected partitions.
3. Check listener invocation and stage counters.
   WHY: "no business result" may occur after Kafka already delivered the record.
   LOOK FOR: fetched, deserialized, queued, started, succeeded, failed, committed.
4. Check poll gaps and liveness timers.
   WHY: slow processing can trigger eviction and endless reassignment.
   LOOK FOR: max poll gap close to or above 300 seconds.
5. Check authentication, authorization, deserialization, and offset position.
   WHY: these fail at different points and require different fixes.
   LOOK FOR: explicit broker/client errors and the first failed stage.

# Step-by-Step Investigation

### Step 1 - Prove records exist

* What I check: producer acknowledgments and partition LEO movement.
* Why: producer intent is not evidence that a broker stored the record.
* Expected: acknowledged topic/partition/offset and increasing LEO.
* Bad: only `send requested` logs, with no callback success.
* Meaning: this is likely a producer path issue, not a consumer issue.
* Next: investigate producer metadata, auth, retries, and acknowledgment.

### Step 2 - Verify effective subscription identity

* What I check: bootstrap servers, topic pattern, group ID, client ID, and security principal.
* Why: Spring profiles and environment variables can override expected values.
* Expected: production cluster, `orders.v1`, group `fulfillment-v3`.
* Bad: pods use `fulfillment-v3-canary` or a staging bootstrap server.
* Meaning: monitoring and application observe different streams.
* Next: correct configuration through the normal deployment process.

### Step 3 - Inspect group membership

* What I check: group state, member IDs, hosts, client IDs, and assignments.
* Why: assignment is the right to fetch a partition for this group.
* Expected: `Stable`, 12 members, one partition per member.
* Bad: no members.
* Meaning: listeners did not start, cannot authenticate, or cannot reach brokers.
* Bad: members exist but all assignments are transient.
* Meaning: the group is continuously rebalancing.
* Next: correlate rebalance callbacks with pod and timer logs.

### Step 4 - Check partition availability

* What I check: partition leaders, offline partitions, replicas, and ISR.
* Why: a member cannot fetch from a partition without an available leader.
* Expected: 12 leaders and ISR size 3.
* Bad: specific partitions have no leader.
* Meaning: broker availability is blocking only those partitions.
* Next: engage Kafka operations; do not reset offsets.

### Step 5 - Check poll, heartbeat, and session evidence

* What I check: last poll time, poll duration, heartbeat failures, and max-poll exits.
* Why: heartbeats and polling answer different liveness questions.
* Expected: poll every few seconds and regular heartbeats.
* Bad: `max.poll.interval.ms exceeded by 12,481 ms`.
* Meaning: processing between polls is too long; ownership is revoked.
* Next: identify why the listener took nearly five minutes.

### Step 6 - Locate the stopped stage

* What I check: counters for fetched, deserialized, queued, handler-started, dependency-started, handler-succeeded, commit-succeeded.
* Why: a single "consumed" counter hides the pipeline.
* Expected: counts advance in order with small differences.
* Bad: fetched rises but deserialized does not.
* Meaning: serializer/type/header/schema mismatch.
* Bad: handler-started rises but handler-succeeded stalls.
* Meaning: handler or dependency is blocked.
* Next: inspect exception type, queue age, and child spans.

### Step 7 - Inspect Spring Kafka listener behavior

* What I check: container running state, pause state, concurrency, ack mode, error handler, and thread dump.
* Why: a healthy JVM does not prove listener containers are processing.
* Expected: containers running, not paused, listener threads regularly polling.
* Bad: listener thread waits in the pricing HTTP client.
* Meaning: synchronous dependency time consumes the poll budget.
* Next: inspect pricing timeout, retries, and batch size.

### Step 8 - Check deserialization separately

* What I check: key/value deserializer errors and `ErrorHandlingDeserializer` headers.
* Why: ordinary listener logs may never execute for a poison record.
* Expected: deserialization success rate near 100%.
* Bad: repeated `SerializationException` at one partition/offset.
* Meaning: the same unreadable record can block progress.
* Next: route through the tested recovery policy or DLT; do not skip silently.

### Step 9 - Confirm offset semantics

* What I check: current position, committed offset, LEO, `auto.offset.reset`, and whether this is a new group.
* Why: a new group with `latest` intentionally ignores existing history.
* Expected: committed offsets exist for the established group.
* Bad: no committed offset and `latest`.
* Meaning: consumption begins only after current LEO.
* Next: decide through governance whether historical replay is required.

### Step 10 - Correlate deployment and rebalance loop

* What I check: version 3.18 timing, handler duration, poll gaps, revocations, and redelivery.
* Why: correlation across signals establishes the causal chain.
* Expected: previous version polls well below five minutes.
* Bad: new bulk-pricing call lasts 292-315 seconds.
* Meaning: it consumes nearly all or more than the max poll interval.
* Next: rollback and redesign blocking work.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| LEO rate | proves records are arriving at broker leaders |
| group committed rate | durable progress; zero means no new restart point |
| consumer position | local progress; may move before commit |
| lag by partition | identifies unowned or blocked partitions |
| assigned partition count | zero means that member cannot consume |
| group rebalance rate | high means ownership churn |
| heartbeat failure rate | high suggests broker/network/session trouble |
| max poll gap | near limit means processing threatens membership |
| poll records | zero with assignment points to fetch/offset conditions |
| fetch latency | high points toward broker/network fetch path |
| deserialize failures | high stops records before the listener |
| listener queue age | high means fetched work waits locally |
| handler active time | high identifies application work |
| dependency p99 | high can block the listener |
| commit failure rate | high prevents durable group progress |
| event age | high confirms business staleness |

Low CPU plus zero commit rate suggests waiting or churn, not healthy idleness.
High rebalance rate after deployment suggests lifecycle or poll behavior changed.
High fetch rate with zero handler starts suggests deserialize or local queue failure.
High handler success with zero business effects suggests unsafe accounting or downstream rollback.
Metrics must use the same group, topic, partition, and time range.

# Distributed Trace Investigation

I select an acknowledged event produced after the deployment.
The producer acknowledgment says `partition=4 offset=492011`.
I search for a consumer span with those coordinates and its `eventId`.

```text
traceId=2dd84f
order-service
  +-- kafka produce orders.v1             11 ms
      partition=4 offset=492011

fulfillment consume attempt 1
  +-- pricing-api                    301,842 ms
  X-- processing cancelled after revoke

fulfillment consume attempt 2
  +-- pricing-api                    still running
```

The five-minute child span localizes time to pricing, not Kafka fetch.
The second consumer span shows redelivery after ownership changed.
Consumer spans should include group, client ID, topic, partition, offset, and event ID.
They should not use offset alone as a global identifier because offsets are partition-local.
A missing consumer span may mean no assignment, no fetch, deserialization before instrumentation, sampling, or broken propagation.
I use logs and stage metrics to distinguish those possibilities.
Trace context should be copied from Kafka headers, not parsed from payload text.

# Distributed Logs

```text
2026-09-13T09:03:11.104+05:30 INFO service=fulfillment-service instance=fulfill-a7 version=3.18 group=fulfillment-v3 event=partitions_assigned partitions=4
2026-09-13T09:03:14.008+05:30 INFO service=fulfillment-service instance=fulfill-a7 traceId=2dd84f spanId=cc17 eventId=ord-99114 topic=orders.v1 partition=4 offset=492011 stage=handler_started
2026-09-13T09:08:15.850+05:30 WARN service=fulfillment-service instance=fulfill-a7 group=fulfillment-v3 event=max_poll_exceeded gapMs=301842 limitMs=300000
2026-09-13T09:08:15.861+05:30 INFO service=fulfillment-service instance=fulfill-a7 group=fulfillment-v3 event=partitions_revoked partitions=4
2026-09-13T09:08:16.120+05:30 ERROR service=fulfillment-service instance=fulfill-a7 traceId=2dd84f spanId=cc17 eventId=ord-99114 dependency=pricing-api error=ReadTimeout durationMs=302112
```

The assignment and revocation logs explain why the member repeatedly stops.
The event coordinates connect attempts without exposing payloads.
The dependency duration matches the poll-limit breach.
One max-poll log still is not sufficient proof.
I confirm the pattern across members and compare with the deployment.
Health `UP` only means configured health contributors passed.
It does not prove group stability or business processing.

# Commands / Tools

```text
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --describe
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --state
kafka-consumer-groups.sh --bootstrap-server kafka-a:9092 --group fulfillment-v3 --members --verbose
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1 --describe
```

These are read-only descriptions when supported by the installed Kafka version.
They prove group metadata and broker topic metadata at query time.
They do not prove listener success or downstream effects.

Windows:

```text
curl.exe -s http://localhost:8080/actuator/health
curl.exe -s http://localhost:8080/actuator/threaddump
Test-NetConnection kafka-a -Port 9092
```

Linux:

```text
curl -s --max-time 5 http://localhost:8080/actuator/health
ss -ntp
nc -vz kafka-a 9092
```

TCP success proves a socket can be opened, not Kafka authentication or authorization.
Thread dumps show where Java threads wait at one instant.
Actuator endpoints require approved access controls.
Consuming with an ad hoc group can expose data and change load, so I do not do it casually.
Offset resets and group deletion are governed, mutating actions and are only described here.

# Root Cause

Version 3.18 made a synchronous bulk-pricing request inside the poll thread.

```text
Bulk-pricing request takes 292-315 seconds
    |
poll() is not called within 300 seconds
    |
max.poll.interval.ms is exceeded
    |
member loses partition assignment
    |
group rebalances
    |
same records are assigned and attempted again
    |
little durable progress and growing lag
```

Heartbeats continued for much of the wait, so pods looked live.
Broker health, leaders, replicas, and ISR remained normal.
The root cause was listener work exceeding the poll contract.
The rebalance storm was the mechanism, not an independent Kafka outage.

# Fix

Immediate mitigation:

* Roll back version 3.18.
* Drain one pod at a time so assignments move predictably.
* Disable the optional bulk-pricing enrichment through the approved feature flag.
* Watch that the group becomes stable before scaling.
* Do not merely raise `max.poll.interval.ms` without bounding dependency work.

Permanent fix:

* Move bulk enrichment to a bounded worker stage while maintaining safe poll/commit semantics.
* Set pricing connect/read timeouts and limited retries within the processing budget.
* Reduce `max.poll.records` if a batch cannot complete in the poll interval.
* Use pause/resume and bounded queues for explicit backpressure.
* Emit stage and poll-gap metrics.
* Integration-test slow dependency behavior and partition revocation.

# Verification

Before rollback:

* Consume rate: 2/min.
* Lag slope: +2,098/min.
* Rebalances: 46 in 15 minutes.
* Max poll gap: 315 seconds.
* Oldest event age: 34 minutes.

After rollback:

* Group state remains `Stable` for 45 minutes.
* Rebalances occur only during controlled rollout.
* Max poll gap is 2.4 seconds.
* Consume rate reaches 4,800/min while draining.
* Lag slope becomes negative.
* Oldest event age returns below 10 seconds.
* Pricing timeout and error rates stay within budget.

I also verify no order was skipped and duplicate-safe effects held during redelivery.
Stable membership plus durable commit progress proves more than pod health.

# Prevention

* Alert on zero committed rate while LEO rises.
* Alert on max poll gap at 70% of its limit.
* Track assignments and rebalances per group.
* Instrument every processing stage.
* Keep listener work within a documented processing budget.
* Bound external calls and retry counts.
* Test deserialization failure and poison-record recovery.
* Verify effective group and cluster configuration at startup without logging secrets.
* Use readiness that reflects required listener container state where operationally appropriate.
* Treat offset changes as reviewed production changes.

# Interview Answer

### What I would say in an interview

I do not treat "not consuming" as one failure. I first prove the broker acknowledged records, then verify the exact cluster, topic, and group. I inspect group state and assignments because a live pod with no partition cannot consume. Next I trace the pipeline: fetch, deserialize, queue, handler, dependency, and commit. I compare heartbeats with poll gaps because heartbeats show process liveness while `max.poll.interval.ms` limits processing time between polls. Here a new synchronous pricing call exceeded five minutes, caused revocation and continuous rebalances, and prevented durable progress. We rolled back, bounded dependency time, redesigned the worker flow, and verified stable assignments, falling lag, low event age, and correct effects.

### Common interviewer traps

* `health=UP` does not prove a listener owns partitions.
* Heartbeats do not replace timely polling.
* No listener log can mean deserialization failed before listener code.
* A new group with `latest` may correctly ignore old records.
* Resetting offsets is not a harmless diagnostic.

### Quick memory flow

Broker ack -> correct cluster/topic/group -> members -> assignments -> fetch -> deserialize -> queue -> handler -> dependency -> commit -> business effect.

# Interview Follow-up Questions

1. **What causes a rebalance?** Membership, subscription, partition count, or liveness changes cause assignments to be recalculated.
2. **Session timeout versus max poll interval?** Session timeout covers missed heartbeats; max poll interval covers excessive time between polls.
3. **Can a healthy pod consume nothing?** Yes, it may be idle, unassigned, paused, misconfigured, or stuck before the listener.
4. **Why inspect committed offset?** It is where the group resumes after restart.
5. **What if position moves but commit does not?** Records are fetched locally but durable restart progress is behind.
6. **Why might the listener never see a record?** Deserialization can fail before invocation.
7. **Does increasing max poll interval fix it?** It may hide symptoms; remove or bound slow work first.
8. **How do you prove recovery?** Stable group, assigned partitions, regular polls, successful commits, falling lag, and restored business effects.
