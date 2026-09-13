# Problem

The API says an order was accepted, but its Kafka event is missing or delayed.
A producer call expresses intent to send.
It is not proof that Kafka stored the record.
With Spring Kafka, `KafkaTemplate.send()` normally returns an asynchronous result.
Success provides broker-acknowledged metadata: topic, partition, and offset.
Failure provides an exception that must be observed.
Logging before the future completes records only intent.
The investigation separates serialization, metadata, connection, authentication, authorization, broker acknowledgment, retry, and application transaction stages.

# Production Situation

The Meridian story continues after the consumer fix.
At 13:20, `order-service` version 5.42 is deployed.
Checkout receives 1,800 requests/min.
HTTP success remains 99.7%.
The producer logs 1,795 `send_intent` events/min.
Only 1,612 send acknowledgments/min appear.
Producer error rate reaches 10.2%.
Request latency remains low because the code does not await or handle the send result.
`orders.v1` LEO grows slower than created-order count.
Missing events affect newly created order IDs.
Broker CPU is 44%; all partition leaders exist.
ISR is healthy at 3 replicas.
Failures contain `TopicAuthorizationException`.
The deployment changed the runtime service account.

# Architecture

```text
Client
  |
  v
order-service
  +--> MySQL order row
  |
  +--> KafkaTemplate.send()
          |
          +--> serialize
          +--> choose partition from key
          +--> metadata lookup
          +--> leader broker
          +--> required acknowledgments
          v
       orders.v1
```

A topic contains partitions.
Each partition leader accepts produce requests.
Followers copy the leader.
With `acks=all`, success waits for acknowledgments required by the in-sync replica rules.
`min.insync.replicas` can reject writes when too few replicas remain.
`acks=1` waits only for the leader.
`acks=0` does not provide a broker acknowledgment.
Idempotent producer mode assigns producer identity and sequence numbers.
It prevents many duplicate appends caused by producer retries within its guarantees.
It does not make database, email, payment, or other external side effects exactly once.

# What I Check FIRST

1. Compare business writes, send intents, send acknowledgments, and send failures.
   WHY: intent and acknowledgment answer different questions.
   LOOK FOR: one acknowledged topic/partition/offset per event expected to publish.
2. Classify the exact producer exception.
   WHY: serialization, auth, timeout, metadata, and broker errors have different paths.
   LOOK FOR: first non-wrapper cause and whether it is retriable.
3. Check affected instances, version, principal, topic, and time.
   WHY: one bad identity or configuration can create a partial failure.
   LOOK FOR: failure isolated to 5.42 pods.
4. Check topic leaders, ISR, broker request errors, and latency.
   WHY: application and cluster failures must be separated.
   LOOK FOR: healthy cluster versus partition-specific distress.
5. Check how the API couples database success to publication.
   WHY: a DB commit followed by an unobserved asynchronous failure loses the notification path.
   LOOK FOR: transactional outbox or an unsafe dual write.

# Step-by-Step Investigation

### Step 1 - Define what "missing" means

* What I check: order ID, stable event ID, creation time, expected topic, and business state.
* Why: a delayed consumer can look like a failed producer.
* Expected: event acknowledgment exists and consumer lag explains delay.
* Bad: no acknowledgment and LEO does not include the event.
* Meaning: investigate the producer side.
* Next: locate send result or exception by event ID.

### Step 2 - Reconcile producer stages

* What I check: event-created, send-intent, send-ack, and send-failed counters.
* Why: every expected event should terminate in acknowledgment or explicit failure.
* Expected: intents equal acks plus final failures over a bounded window.
* Bad: intents exceed both outcomes.
* Meaning: callbacks are not observed, process exits early, or instrumentation is incomplete.
* Next: inspect how the Spring future is handled.

### Step 3 - Inspect effective producer configuration

* What I check: bootstrap servers, client ID, serializers, security protocol, principal, acks, idempotence, retries, delivery timeout.
* Why: environment overrides can change runtime behavior.
* Expected: approved production cluster and service account with write permission.
* Bad: new principal lacks topic write authorization.
* Meaning: broker rejects produce requests before append.
* Next: verify authorization errors in client and broker audit evidence.

### Step 4 - Classify errors by stage

* What I check: `SerializationException`, metadata timeout, auth exceptions, request timeout, and record-too-large failures.
* Why: the exception locates the failed boundary.
* Expected: rare transient retries followed by acknowledgment.
* Bad: deterministic `TopicAuthorizationException`.
* Meaning: retrying cannot repair missing permission and only adds load.
* Next: stop retries for non-retriable authorization failure and correct identity policy.

### Step 5 - Check metadata and leader reachability

* What I check: metadata age, leader mapping, DNS, TCP, TLS, and broker connection metrics.
* Why: producers must discover and reach the leader for the chosen partition.
* Expected: current metadata and successful authenticated connections.
* Bad: repeated metadata refresh timeout for one topic.
* Meaning: wrong topic, DNS/routing trouble, or unavailable leader.
* Next: compare topic description with client logs.

### Step 6 - Check acknowledgment semantics

* What I check: `acks`, `min.insync.replicas`, ISR, request latency, and broker error codes.
* Why: configured durability determines when success is reported.
* Expected: `acks=all`, healthy ISR, and acknowledgment within delivery timeout.
* Bad: `NotEnoughReplicas` while ISR is below policy.
* Meaning: Kafka refuses durability-compromised writes.
* Next: restore replicas rather than weakening durability during an incident.

### Step 7 - Check idempotent retry settings

* What I check: idempotence enabled, compatible acks/retries/in-flight settings, retry rate, and delivery timeout.
* Why: ambiguous network results can cause a retry after the broker appended a record.
* Expected: idempotent sequencing handles broker append retries.
* Bad: custom configuration disables idempotence while retries are high.
* Meaning: duplicates are possible even after eventual producer success.
* Next: restore tested producer durability settings.

### Step 8 - Inspect application result handling

* What I check: whether the future callback records success/failure and propagates required failures.
* Why: asynchronous failure may happen after the HTTP response.
* Expected: success logs acknowledged coordinates; failure enters a durable recovery path.
* Bad: fire-and-forget call followed immediately by HTTP 201.
* Meaning: API success is disconnected from event publication.
* Next: inspect the DB-to-Kafka consistency design.

### Step 9 - Analyze dual-write consistency

* What I check: order DB commit time versus event publication.
* Why: MySQL and Kafka do not share a normal atomic transaction here.
* Expected: order and outbox row commit atomically, then a relay publishes.
* Bad: order commits, process crashes, Kafka send never completes.
* Meaning: an event can be permanently absent.
* Next: implement a transactional outbox and idempotent relay.

### Step 10 - Verify payload and partition selection

* What I check: serialized size, schema version, required headers, key presence, chosen partition.
* Why: serialization happens before broker acknowledgment and keys control ordering scope.
* Expected: valid schema, bounded size, stable order key.
* Bad: null or changed key scatters one order across partitions.
* Meaning: per-order ordering can be lost even when sends succeed.
* Next: correct key contract and compatibility tests.

# Metrics to Check

| Metric | High or low interpretation |
|---|---|
| business records created | baseline events expected |
| send intent rate | application attempts, not broker storage |
| send acknowledgment rate | successful broker-acknowledged appends |
| send final failure rate | events requiring recovery |
| intent outcome gap | missing callback or unfinished sends |
| produce request latency p99 | broker/network/durability wait when high |
| record retry rate | transient failures or broker pressure when high |
| record error rate | final producer failures |
| metadata age/refresh errors | discovery problem when high |
| connection/auth failures | transport or identity issue |
| serialization failures | payload fails before network |
| record size max | risk near broker/client limits |
| buffer available bytes | low means producer backpressure |
| buffer wait time | high means sender cannot drain |
| batch size/compression ratio | efficiency and payload behavior |
| under-replicated partitions | broker durability distress |
| ISR shrink rate | replica stability problem |
| outbox unpublished age | DB-to-Kafka publication delay |

If intent is steady but acknowledgments fall, the API log is not proof of delivery.
If retries spike with stable final success, look for transient broker or network latency.
If non-retriable errors spike, aggressive retry is harmful.
If broker metrics are healthy and failures isolate to one principal, suspect application identity or policy.
If outbox age rises while Kafka acks are normal for other services, inspect the relay.

# Distributed Trace Investigation

The producer span must finish on send result, not when `send()` merely returns a future.
Useful attributes include messaging system, destination, operation, client ID, event ID, partition, offset, and error type.
Secrets and full message payloads are excluded.

```text
traceId=8f330b
POST /orders                               74 ms
  +-- mysql insert order                   18 ms
  +-- kafka publish orders.v1              9 ms ERROR
      error=TopicAuthorizationException
      partition=unassigned offset=unassigned
```

No partition or offset is correct here because no append was acknowledged.
A log claiming `published` before this span ends would be misleading.
For a successful send:

```text
traceId=1ca80d
POST /orders                               91 ms
  +-- mysql transaction                    26 ms
      order + outbox

traceId=relay91
outbox relay
  +-- kafka publish orders.v1              13 ms
      partition=5 offset=740119
      eventId=evt-44192
```

The stable event ID links the originating request, outbox row, producer ack, and consumer.
A missing producer span may mean instrumentation gap, sampling, or code never reached send.
It is not alone proof of message loss.

# Distributed Logs

```text
2026-09-13T13:20:44.601+05:30 INFO service=order-service instance=order-5 version=5.42 traceId=8f330b spanId=81e2 requestId=req-771 eventId=evt-44192 action=send_intent topic=orders.v1 keyHash=019a
2026-09-13T13:20:44.610+05:30 ERROR service=order-service instance=order-5 version=5.42 traceId=8f330b spanId=81e2 requestId=req-771 eventId=evt-44192 action=send_failed topic=orders.v1 error=TopicAuthorizationException retriable=false durationMs=9
2026-09-13T13:21:00.000+05:30 WARN service=order-service instance=order-5 version=5.42 principal=svc-order-v542 metric=unpublished_outbox count=183 oldestAgeSec=40
```

On success I expect:

```text
2026-09-13T13:33:09.100+05:30 INFO service=outbox-relay instance=relay-2 traceId=relay91 spanId=ca72 eventId=evt-44192 action=send_ack topic=orders.v1 partition=5 offset=740119 durationMs=13
```

I correlate event ID across DB, producer, and consumer evidence.
Topic, partition, and offset identify the acknowledged Kafka record.
The intent log alone proves only that application code attempted a send.
One authorization log suggests a cause but fleet-wide principal/error correlation proves scope.

# Commands / Tools

Read-only metadata:

```text
kafka-topics.sh --bootstrap-server kafka-a:9092 --topic orders.v1 --describe
```

This proves leaders, replicas, and ISR at that moment.
It does not prove the application principal can write.

Windows connectivity:

```text
Resolve-DnsName kafka-a
Test-NetConnection kafka-a -Port 9092
```

Linux connectivity:

```text
getent hosts kafka-a
nc -vz kafka-a 9092
```

DNS and TCP success do not prove TLS, SASL, topic authorization, or successful produce.

Spring metrics:

```text
curl.exe -s http://localhost:8080/actuator/prometheus
curl -s --max-time 5 http://localhost:8080/actuator/metrics
```

Access follows production policy.
Kafka UI, broker dashboards, and audit logs help validate principal-level failures.
ACL changes, topic creation, and durability setting changes are governed actions.
They are described and reviewed, not executed as casual diagnostics.

# Root Cause

The deployment changed from `svc-order` to `svc-order-v542`.
The new principal had read metadata permission but not write permission on `orders.v1`.

```text
HTTP request succeeds
    |
order row commits
    |
KafkaTemplate send is started asynchronously
    |
broker rejects new principal
    |
future completes exceptionally
    |
application ignores failure and returns success
    |
order exists without OrderCreated event
```

Healthy broker CPU, leaders, replicas, and ISR eliminated cluster saturation.
Authorization failures isolated to version 5.42 proved the identity mismatch.
The unsafe dual write made the technical failure a business data gap.

# Fix

Immediate mitigation:

* Roll back to the approved service account and application version.
* Stop accepting success if the required publication path is unavailable.
* Use the durable list of unpublished outbox entries to recover affected events.
* Bound recovery by event IDs and reconcile acknowledgments.
* Do not weaken topic ACLs broadly or disable authentication.

Permanent fix:

* Commit the order and outbox row in one MySQL transaction.
* Publish asynchronously from an outbox relay.
* Mark outbox rows published only after broker acknowledgment.
* Handle producer futures and classify retriable versus final errors.
* Enable and test idempotent producer settings.
* Keep stable event IDs so retries are safe downstream.
* Validate service-account authorization before rollout.

# Verification

Before:

* Orders created: 1,800/min.
* Send intents: 1,795/min.
* Send acknowledgments: 1,612/min.
* Final producer errors: 183/min.
* Oldest unpublished outbox row: 11 minutes.

After:

* Acknowledgments match relay attempts minus explicit final failures.
* Authorization errors are zero for 60 minutes.
* Every acknowledgment includes topic, partition, and offset.
* Outbox oldest age falls below 5 seconds.
* Created orders reconcile one-to-one with stable event IDs.
* Consumer effects appear once logically despite safe redelivery.

I sample event IDs end to end.
I verify no broad ACL was introduced.
I also test broker timeout and process crash paths in staging.

# Prevention

* Alert on intent-to-outcome gaps.
* Alert on unpublished outbox age and count.
* Record producer acknowledgments, failures, and error classes.
* Use stable event IDs and schema compatibility checks.
* Prefer transactional outbox over unsafe DB/Kafka dual writes.
* Test service-account permissions in deployment preflight.
* Keep `acks=all` and idempotence under reviewed defaults.
* Monitor ISR and produce request latency.
* Redact credentials and payloads from logs and traces.
* Reconcile business records to published events.

# Interview Answer

### What I would say in an interview

I separate producer intent from broker acknowledgment. A `KafkaTemplate.send()` call and a pre-send log do not prove delivery; I need the asynchronous result with topic, partition, and offset, or an explicit failure. I compare business writes, send intents, acknowledgments, failures, and outbox age, then classify serialization, metadata, auth, broker, timeout, and retry errors. Here a new service account lacked write permission, while the API ignored the failed future after committing MySQL. We rolled back the identity and recovered from a durable outbox. Permanently, I would use an outbox relay, idempotent producer settings, stable event IDs, and alerts on unmatched outcomes.

### Common interviewer traps

* A successful HTTP response is not a Kafka acknowledgment.
* `send()` returning a future is not broker success.
* Idempotent producer mode does not make external side effects exactly once.
* Broadening ACLs is not a safe diagnostic fix.
* Retrying deterministic authorization errors adds load without helping.

### Quick memory flow

Business write -> event ID -> serialize -> metadata -> leader -> ack policy -> acknowledged coordinates -> outbox reconciliation -> consumer effect.

# Interview Follow-up Questions

1. **What proves a Kafka send succeeded?** Successful completion with record metadata containing topic, partition, and offset.
2. **What does `acks=all` mean?** The leader waits for acknowledgments required from the current ISR under broker policy.
3. **Why use idempotence?** It suppresses duplicate appends from supported producer retry sequences.
4. **Does idempotence make an email exactly once?** No; external effects need their own idempotency.
5. **Why use an outbox?** It atomically records business state and publication intent in one database transaction.
6. **What controls ordering?** Kafka orders records within a partition; a stable key keeps related events in that scope.
7. **How do you handle ambiguous timeouts?** Retry under idempotent settings and reconcile by stable event ID and acknowledgment.
8. **Why not set `acks=0` for availability?** It removes broker confirmation and weakens durability evidence.
