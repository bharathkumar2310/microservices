# Production Troubleshooting Study Chapter: Distributed Transactions and Saga

## Purpose

This chapter explains how several independently deployed services complete one business operation without pretending that their databases share one atomic transaction. It starts with local transactions and eventual consistency, then develops production-safe Saga design, diagnosis, compensation, reconciliation, and manual recovery.

The running example is an e-commerce order involving Order, Inventory, Payment, and Shipping services. The principles also apply to subscriptions, travel bookings, account provisioning, and other multi-service workflows.

> Safety: Prefer read-only evidence and approved application operations. Never repair a Saga by directly editing several databases or replaying arbitrary production messages. Preserve evidence, use a tested idempotent command or reconciliation tool, require authorization for financial or customer-visible actions, and record every manual decision.

## Learning goals

After studying this chapter, you should be able to:

1. Distinguish a database-local transaction from a distributed business transaction.
2. State Saga invariants and identify which temporary inconsistent states are allowed.
3. Explain why compensation is a new business operation rather than a technical rollback.
4. Model compensable, pivot, and retriable transactions in a durable state machine.
5. Compare choreography and orchestration without treating either as automatically reliable.
6. Design idempotent commands, consumers, outbox/inbox handling, and concurrency control.
7. Bound retries and timeouts without converting uncertainty into duplicate side effects.
8. Recover from failed compensation, duplicate or out-of-order events, and stuck Sagas.
9. Operate reconciliation and manual intervention with auditability and least privilege.
10. Give concise, evidence-based interview answers while retaining production depth.

---

# 1. Foundational mental model

## 1.1 One business transaction, many local transactions

A local ACID transaction commits changes atomically inside one transactional resource, normally one database:

```text
BEGIN
  INSERT order
  INSERT outbox_event
COMMIT
```

Both rows become durable, or neither does. That guarantee does not automatically include another service's database, message broker, payment provider, or email system. A network call inside the transaction does not extend ACID across those systems.

A distributed business transaction instead consists of local transactions connected by durable messages or commands:

```text
Order local commit
  -> Inventory local commit
  -> Payment provider operation and Payment local commit
  -> Shipping local commit
```

There can be intervals when Order says `PENDING_PAYMENT`, Inventory says `RESERVED`, and Payment has not completed. This is eventual consistency: replicas of the business truth may temporarily differ, but the protocol is designed to converge to an allowed terminal state.

Eventual consistency does not mean "anything is acceptable eventually." A correct design defines:

- Safety invariants that must never be violated.
- Temporary states that are visible and how callers interpret them.
- A liveness goal that every Saga eventually reaches a terminal state or an explicitly owned manual state.
- Reconciliation that detects when normal progress stops.

## 1.2 Saga definition

A Saga is a sequence of local transactions. After each local commit, the workflow records progress durably and triggers the next step. If a later step cannot complete, compensating transactions semantically counteract earlier committed work where business rules permit.

```text
Forward path:
create pending order -> reserve stock -> authorize payment -> confirm order

Compensation path:
cancel payment authorization -> release stock -> cancel order
```

There is no global database lock and no rewind of time. Other actors may have observed the intermediate states. Compensation must therefore be an explicit, authorized, idempotent business operation.

## 1.3 Compensation is not rollback

A database rollback makes uncommitted writes invisible. A compensation happens after a local commit and creates more durable history.

Example:

```text
10:00 Payment authorization succeeds.
10:01 Customer receives a bank notification.
10:02 Inventory confirmation fails.
10:03 System voids the authorization.
```

The void does not erase the authorization, the notification, fees, exchange-rate effects, or audit history. It changes the business state from authorized to voided. A captured payment may require a refund, which can take days and may fail or incur a fee. Some effects, such as an already sent email or executed shipment, cannot be perfectly undone.

Compensation design asks:

- Is an inverse operation legally and technically available?
- Is it a void, refund, release, cancellation, credit, or human process?
- What if the original result is unknown?
- What if the compensation is repeated?
- What if the business has crossed a no-return point?

## 1.4 Compensable, pivot, and retriable transactions

Classify steps before choosing their order:

- A **compensable transaction** can be semantically undone, such as reserving then releasing stock.
- A **pivot transaction** is the point of no return after which backward compensation is not practical or permitted, such as handing a parcel to a carrier or issuing a nonrefundable entitlement.
- A **retriable transaction** is expected to succeed eventually if retried safely, such as writing a final status to an available local database.

A useful Saga shape is:

```text
compensable steps -> pivot -> retriable completion steps
```

Place irreversible effects as late as practical. Before the pivot, failure can move backward through compensation. After the pivot, design for roll-forward rather than pretending a rollback remains possible.

The classification is business-specific. Payment authorization is often compensable by voiding; payment capture may be a pivot in one domain and compensable by refund in another.

## 1.5 Durable state machine

Do not keep Saga progress only in memory, a request thread, or an expiring queue message. Persist a state machine with versioned transitions:

```text
STARTED
  -> STOCK_RESERVE_REQUESTED
  -> STOCK_RESERVED
  -> PAYMENT_REQUESTED
  -> PAYMENT_AUTHORIZED
  -> COMPLETED

Failure:
PAYMENT_DECLINED
  -> STOCK_RELEASE_REQUESTED
  -> STOCK_RELEASED
  -> CANCELLED

Uncertain/manual:
PAYMENT_OUTCOME_UNKNOWN
COMPENSATION_RETRYING
MANUAL_REVIEW
```

Each transition should record Saga ID, business ID, current state, state version, triggering message ID, timestamps, attempt count, reason code, and correlation/trace information. Legal transitions are explicit. A conditional update prevents two workers from applying incompatible transitions:

```text
UPDATE saga
SET state = 'PAYMENT_REQUESTED', version = 8
WHERE saga_id = 'saga-20260913-0042'
  AND state = 'STOCK_RESERVED'
  AND version = 7;
```

One affected row means this worker won. Zero means the state changed or the input is stale; reread rather than forcing the update.

## 1.6 Idempotency, concurrency, and semantic locks

An operation is idempotent when repeating the same logical request has the same business effect as applying it once. Transport delivery is commonly at least once, so duplicates are expected.

Use a stable operation key, not a new key per retry:

```text
idempotency key = saga ID + logical step name
example = saga-20260913-0042:authorize-payment
```

The receiving service atomically stores the key and result with its local business change. A duplicate returns the stored result. Merely checking then acting is racy:

```text
unsafe: SELECT key -> call provider -> INSERT key
safe:   claim unique key atomically -> perform provider operation with the same key
        -> persist outcome, with explicit recovery for an in-progress claim
```

Idempotency prevents duplicate application of the same command; it does not resolve conflicting commands. `reserve` and `release`, or two different Sagas reserving the last unit, require concurrency rules:

- Unique constraints.
- Compare-and-set state/version updates.
- Per-aggregate sequencing.
- Optimistic or pessimistic local locking.
- A business **semantic lock**, such as `PENDING` order status or a stock reservation with an owner and expiry.

A semantic lock prevents conflicting business actions while a Saga is incomplete without holding a database lock across services.

## 1.7 Outbox, inbox, and delivery reality

The dual-write problem occurs when a service commits its database change and separately publishes a message:

```text
database commit succeeds
process crashes before publish
```

or:

```text
publish succeeds
database commit fails
```

The transactional outbox solves the first boundary by writing business state and an outbox row in one local transaction. A relay publishes committed outbox rows. Because the relay can crash after publish but before marking the row sent, duplicates remain possible.

An inbox/deduplication table lets a consumer atomically record a message ID with its local effect. The combination provides atomic local state plus at-least-once message processing, not magical end-to-end exactly-once effects.

Ordering is also scoped. A broker may preserve order only within a partition. Partition by the aggregate or Saga key when order matters, include sequence numbers, reject or park stale transitions, and buffer only with a bounded policy. Never assume wall-clock timestamps establish global order.

## 1.8 Timeouts, retries, and uncertain outcomes

A timeout means the caller stopped waiting; it does not prove the remote operation failed. Four outcomes matter:

```text
request never arrived
request arrived and failed
request succeeded but response was lost
request is still running
```

Before retrying a side effect, query by idempotency key or use a provider API that accepts that key. Retry transient failures with an overall deadline, bounded attempts, exponential backoff, jitter, and a retry budget. Permanent business failures such as insufficient funds should not be retried as infrastructure failures.

Timeout ownership should be explicit. A scheduler may mark an overdue state, but it must not race a late success into an impossible transition. State/version checks and late-result handling are protocol requirements.

## 1.9 Choreography and orchestration

In **choreography**, services react to events and emit new events:

```text
OrderCreated -> InventoryReserved -> PaymentAuthorized -> OrderConfirmed
```

It reduces central control and can fit simple, stable flows, but the workflow becomes distributed across consumers. Cycles, hidden coupling, ambiguous ownership, and difficult global observability grow with complexity.

In **orchestration**, a durable orchestrator owns the Saga state and sends commands:

```text
orchestrator -> ReserveInventory
InventoryReserved -> orchestrator
orchestrator -> AuthorizePayment
```

It centralizes flow visibility, timeouts, and compensation policy. It can become a bottleneck or a "god service" if it absorbs domain rules, and it must be highly available. The orchestrator stores workflow policy; each service still owns its data and validates commands.

Both styles need durable messaging, idempotency, observability, and reconciliation. Choice of topology does not remove failure semantics.

## 1.10 Glossary

| Term | Meaning |
|---|---|
| ACID | Atomicity, consistency, isolation, and durability within a transaction boundary |
| Aggregate | Data changed as one consistency unit inside a service |
| Saga | Local transactions coordinated as one distributed business workflow |
| Compensation | A new business action that semantically counteracts an earlier committed action |
| Eventual consistency | Allowed temporary divergence with a designed path to convergence |
| Invariant | Business rule that must hold in all allowed states |
| Orchestration | A durable coordinator directs Saga steps |
| Choreography | Participants react to events without one flow controller |
| Outbox | Messages persisted atomically with producing service state |
| Inbox | Received message identities persisted atomically with consumer effects |
| Idempotency key | Stable identity for one logical operation across retries |
| Semantic lock | Business state or reservation that prevents conflicting operations |
| Pivot | Point after which backward compensation is not feasible |
| Tombstone | Durable marker that an entity/event was cancelled or deleted |
| DLQ | Dead-letter queue for messages that cannot proceed under normal policy |
| Reconciliation | Comparison of durable facts to detect and repair nonconverged workflows |
| Poison message | Input that repeatedly fails until isolated or corrected |
| Fencing token | Monotonic token used to reject work from stale owners |

---

# 2. Core evidence and metrics

## 2.1 Evidence hierarchy

Build a timeline from durable facts rather than one log line:

1. Business records: order, reservation, payment, shipment.
2. Saga state and transition history.
3. Inbox/outbox rows and relay status.
4. Broker metadata: topic, partition, offset, key, delivery attempts, consumer lag.
5. Provider records queried by idempotency key.
6. Application metrics, traces, and structured logs.
7. Deployment/configuration changes and infrastructure events.

Logs can be sampled, delayed, duplicated, or lost. The authoritative state usually lives in service-owned records and the external provider. Correlation IDs connect evidence; they are not proof by themselves.

## 2.2 Minimum structured fields

For every step, capture non-sensitive identifiers:

```text
saga_id, saga_type, business_id
step, prior_state, new_state, state_version
command_id, message_id, causation_id, correlation_id
participant, outcome, reason_code
attempt, next_attempt_at
created_at, started_at, completed_at
artifact_version, instance_id, trace_id
```

Do not put card data, secrets, or unrestricted personal information in logs, message headers, idempotency keys, or dashboards.

## 2.3 Operational metrics

| Metric | Interpretation |
|---|---|
| Saga started/completed/failed rate | Flow health by Saga type and version |
| In-progress age histogram | Tail age reveals stalled workflows |
| Count by state and state age | Identifies the exact accumulating transition |
| Step duration and error rate | Locates slow/failing participant |
| Retry attempts and retry-exhausted rate | Distinguishes transient recovery from retry storms |
| Compensation started/completed/failed | Shows business recovery health |
| Manual-review count and oldest age | Exposes operational debt and customer risk |
| Outbox unpublished age/count | Detects relay or broker publication failure |
| Inbox duplicate/conflict count | Shows redelivery and protocol defects |
| Consumer lag and DLQ rate | Detects consumption failure, but lag alone is not a Saga diagnosis |
| Invalid/stale transition count | Detects duplicate, out-of-order, or concurrency bugs |
| Unknown external-outcome count | Highlights timeout ambiguity requiring status lookup |
| Reconciliation mismatch count | Measures failure to converge |

Alert on rates and age against business deadlines, not just queue length. Ten five-minute-old Sagas may be worse than a thousand Sagas that normally finish in one second; one stuck high-value settlement may be critical.

## 2.4 State transition evidence

An append-only transition history should answer:

```text
What was believed before?
Which input caused the decision?
Which guarded state/version changed?
Which local transaction committed?
Which output was durably scheduled?
Who or what retried, compensated, or intervened?
```

Audit entries should be tamper-evident or access-controlled according to risk. Store reason codes and actor identity for manual changes. Keep business audit retention separate from high-volume debug-log retention.

---

# 3. Generic investigation and design workflow

## 3.1 Production investigation workflow

1. **Define impact and freeze unsafe automation.** Identify affected Saga type, time window, tenants, financial exposure, and whether new work should be paused. This prevents an unbounded incident while preserving healthy unrelated traffic.
2. **Capture one representative identity.** Use a Saga ID and business ID from an actual failed request. Aggregates show scope; one traceable case reveals the protocol path.
3. **Read current authoritative states.** Query each owner through approved read APIs or read-only access. This establishes facts before retries change them.
4. **Reconstruct a UTC timeline.** Join transition history, inbox/outbox, broker metadata, provider lookup, traces, and deployments using IDs. Ordering by one host's log timestamp is unsafe when clocks or ingestion differ.
5. **Find the last proven durable transition.** Separate "command sent" from "participant committed" and "reply consumed." The gap selects the next evidence source.
6. **Classify the failure.** Business rejection, transient technical failure, permanent technical failure, uncertain outcome, duplicate, out-of-order input, illegal transition, or operator hold require different actions.
7. **Check retry and ownership state.** Inspect attempt count, next retry, leases/fencing tokens, consumer assignment, DLQ, and scheduler health. This distinguishes waiting by policy from genuinely stuck work.
8. **Choose an idempotent recovery.** Resume, redrive, query-and-record, compensate, roll forward, or route to manual review. Simulate preconditions and expected state first.
9. **Verify convergence.** Confirm all participant states, final event publication, customer-visible status, and financial/inventory totals. A successful command response is not sufficient.
10. **Preserve and correct.** Save IDs and timelines, fix the mechanism, add a replay/concurrency test, and alert on the earliest durable symptom.

## 3.2 Design workflow

For each workflow:

1. Write business invariants and legal temporary states.
2. Assign one owner to every piece of data.
3. Draw forward and compensation transitions, including unknown and manual states.
4. Classify each step as compensable, pivot, or retriable.
5. Define command/event schemas, stable identities, and compatibility policy.
6. Define atomic local boundaries using state plus outbox/inbox.
7. Define idempotency and conflicting-command concurrency behavior.
8. Set deadlines, retry classifications, backoff, and late-result policy.
9. Define compensation failure escalation and authorization.
10. Add reconciliation, audit retention, dashboards, alerts, and runbooks.
11. Test crash points before/after every commit and publish.
12. Test duplicates, reordering, long delay, participant outage, and operator replay.

---

# 4. Original interview questions

# 1. Order creation succeeds but payment fails. How would you maintain consistency?

## Meaning and invariants/risks

"Order creation succeeds" should normally mean a pending order was committed, not that a completed order was promised. The invariant is that an unpaid order must not become fulfillable. Reserved inventory must either remain valid under a bounded policy or be released. The customer must see an honest state such as `PAYMENT_FAILED` or `CANCELLED`, not a false success.

Risks include inventory leakage, shipment without payment, repeated charges, contradictory customer messages, and a late payment success racing cancellation.

## Failure locations

- Payment is legitimately declined.
- Payment service cannot acquire a connection or is unavailable.
- Provider receives the authorization but the response is lost.
- Payment commits locally but its result event is not published.
- Result is published but Order consumer is down, lagging, or rejects the schema.
- Timeout scheduler starts compensation while a success is in flight.
- Inventory release fails after payment failure.

## Detailed causal mechanisms with plain-language example

Suppose order `O-731` is `PENDING_PAYMENT` and stock is reserved. The provider authorizes the card, but the response packet is lost. Treating the timeout as "payment failed" and submitting a second authorization with a new key can charge twice. Cancelling immediately can also conflict with the late success.

The correct state is initially `PAYMENT_OUTCOME_UNKNOWN`. Payment queries the provider using the original idempotency key. A confirmed decline follows compensation; a confirmed authorization follows the forward path if the order is still eligible, or voids/refunds under an explicit late-success rule.

## Ordered investigation/design reasoning and why each step matters

1. Read Order's state/version and transition history to learn whether it was merely created or already confirmed.
2. Read Payment's operation by Saga step key; this distinguishes no attempt, in-progress, decline, authorization, and unknown.
3. Query the provider by the same merchant/idempotency reference because local absence does not prove remote absence.
4. Inspect Inventory reservation status and expiry so recovery does not sell released stock.
5. Inspect outbox/inbox and broker offsets to locate a missing notification without repeating the side effect.
6. Check timeout and late-result transitions for a race; legal guarded transitions decide whether to continue or compensate.
7. Choose decline compensation only after resolving uncertainty, because a timeout is not a decline.

## Evidence, state transitions, tools, and result interpretation

Expected decline path:

```text
PENDING_PAYMENT -> PAYMENT_DECLINED
PAYMENT_DECLINED -> STOCK_RELEASE_REQUESTED
STOCK_RELEASE_REQUESTED -> STOCK_RELEASED
STOCK_RELEASED -> CANCELLED
```

Use approved order/payment read APIs, Saga history, outbox/inbox queries, broker consumer-group lag, distributed traces, and provider lookup. An outbox row marked pending with no broker record points to the relay. A broker record with no inbox row points to routing or consumer progress. An inbox row plus unchanged order state points to transaction failure or an illegal transition.

## Immediate mitigation/recovery

Pause new payment attempts for the affected cohort if duplicate-charge risk exists. Resolve unknown provider outcomes first. Redrive only the missing idempotent message or resume the specific guarded transition. Release stock and cancel the pending order for a confirmed permanent decline. If money was captured, use an authorized void/refund workflow and communicate its actual status.

## Permanent correction/design

Create orders as pending, persist state plus outbox atomically, use provider idempotency keys, model unknown and late-success states, and make stock release idempotent. Define expiration and customer notification policy. Never call payment and then mark Order complete in an unprotected dual write.

## Prevention and alerts

Alert on old `PENDING_PAYMENT`, unknown outcomes, unpublished outbox age, duplicate provider operations, failed releases, and divergence between paid and fulfillable states. Chaos-test response loss after provider success.

## Common mistakes

- Deleting the order and losing audit history.
- Assuming timeout means failure.
- Retrying with a new payment key.
- Marking the order confirmed before payment evidence.
- Treating a refund as instantaneous rollback.
- Releasing inventory without handling late payment success.

## Concise interview-ready answer

I create the order in a nonfulfillable pending state and run payment as an idempotent Saga step. A confirmed decline transitions the Saga through idempotent stock release and order cancellation. A timeout becomes `PAYMENT_OUTCOME_UNKNOWN`, not failure; I query the provider by the original key before retrying or compensating. State/version guards handle late results, while outbox/inbox records, reconciliation, age alerts, and an auditable refund or manual path ensure convergence.

---

# 2. Payment succeeds but order creation fails. What would you do?

## Meaning and invariants/risks

This ordering is dangerous because money exists without a durable business aggregate to own it. The primary design invariant is usually "a durable pending order and operation identity exist before payment." If payment nevertheless succeeds, every charge must be associated with a recoverable business reference and must end in a valid order or an auditable void/refund.

## Failure locations

- A client pays before calling Order.
- Order database commit fails after Payment is called.
- Order commit succeeds but its response is lost, so the caller believes creation failed.
- Payment event arrives before OrderCreated because of cross-topic ordering.
- Schema validation or unique-key conflict rejects the order.
- A legacy flow has no shared correlation/idempotency key.

## Detailed causal mechanisms with plain-language example

A checkout calls Payment, gets authorization `P-91`, then Order's database is unavailable. There is no row to attach fulfillment to. Repeatedly trying to insert an order may be correct if the full order intent was durably captured elsewhere; inventing missing line items from logs is not.

If only authorization occurred, voiding may be safer. If capture occurred and a trusted checkout-intent record has all validated data, a guarded roll-forward can create the order using the original business key. The choice is a business rule, not an engineer's ad hoc production decision.

## Ordered investigation/design reasoning and why each step matters

1. Query Order by business/idempotency key, not only by a returned ID; a lost response may hide a successful commit.
2. Query Payment and provider for authorization versus capture, amount, currency, and operation reference; remedy depends on actual financial state.
3. Find a durable checkout intent or request record and validate ownership, price version, and inventory; logs are not authoritative order input.
4. Inspect outbox/event timing to determine whether Order exists but downstream observation failed.
5. Apply the documented policy: create from trusted intent, void authorization, refund capture, or manual review.
6. Verify ledger, customer status, and duplicate prevention after recovery.

## Evidence, state transitions, tools, and result interpretation

Safer normal flow:

```text
CHECKOUT_ACCEPTED -> ORDER_PENDING_COMMITTED -> PAYMENT_REQUESTED
PAYMENT_AUTHORIZED -> ORDER_CONFIRMED
```

Exceptional flow:

```text
PAYMENT_FOUND_WITHOUT_ORDER -> ORPHAN_PAYMENT_REVIEW
ORPHAN_PAYMENT_REVIEW -> ORDER_CREATED_FROM_INTENT
or
ORPHAN_PAYMENT_REVIEW -> VOID_REQUESTED -> VOIDED
or
ORPHAN_PAYMENT_REVIEW -> REFUND_REQUESTED -> REFUNDED
```

Use unique business keys, payment-provider references, durable checkout intent, payment ledger, and reconciliation reports. A unique-key conflict may mean a concurrent successful create; reread before compensating.

## Immediate mitigation/recovery

Stop automatic captures if orphan payments are increasing. Do not issue both an order creation and refund concurrently. Claim each exception using a version or work lease. Prefer void over refund where valid and approved. Put ambiguous/high-value cases into a restricted manual queue with complete evidence and customer communication.

## Permanent correction/design

Commit a pending order or durable checkout intent before payment. Carry the same operation key to Payment and provider. Make order creation idempotent on checkout key. Use state plus outbox in one transaction and reconcile provider transactions against orders.

## Prevention and alerts

Alert on payment records lacking a valid order after a short grace period, duplicate merchant references, and refund/void failures. Test crashes before order commit, after order commit, after provider success, and before response delivery.

## Common mistakes

- Assuming "API returned error" means no order row exists.
- Creating an order from incomplete logs.
- Refunding an authorization that should be voided.
- Running roll-forward and compensation simultaneously.
- Losing the original payment-to-checkout reference.

## Concise interview-ready answer

First I query Order by the checkout idempotency key because a failed response may hide a committed order, then query the provider for the exact payment state. The durable business policy chooses one guarded path: create the order from a complete trusted checkout intent, void an authorization, refund a capture, or send it to manual review. I prevent recurrence by committing a pending order before payment, using one stable key end to end, and reconciling orphan payments.

---

# 3. Three microservices participate in one business transaction and the third service fails. How would you handle it?

## Meaning and invariants/risks

The first two services may already have committed. The response depends on whether the third failure is permanent, transient, or unknown and whether earlier steps are compensable. The invariant is not "all databases change at the same instant"; it is that only legal intermediate states occur and the workflow converges under an explicit forward or compensation policy.

## Failure locations

- Third command was never published.
- It is queued behind lag or routed incorrectly.
- Third service rejects a business rule.
- Third service commits but crashes before replying.
- Its reply is lost or cannot be consumed.
- Compensation in service two or one fails.
- Concurrent user cancellation conflicts with the Saga.

## Detailed causal mechanisms with plain-language example

Account service creates a pending account, Entitlement reserves a license, and Provisioning times out. If provisioning is safely retriable and the license remains valid, retrying forward is sensible. If Provisioning rejects an unsupported region, release the license and cancel the pending account. If provisioning may have succeeded remotely, first query its operation key; blindly compensating could leave a live resource attached to a cancelled account.

## Ordered investigation/design reasoning and why each step matters

1. Enumerate committed local states for all three participants; call attempts are not commits.
2. Locate the command/reply through outbox, broker, inbox, and transition history to identify the broken boundary.
3. Classify the third outcome as business failure, transient, permanent technical, or unknown.
4. Check step classification and pivot position; this determines backward compensation versus forward retry.
5. Check semantic-lock validity and concurrent state/version before acting.
6. Retry with the original operation key only within policy, or launch compensations in reverse dependency order.
7. Verify every compensation and terminal aggregate state; route exhausted recovery to manual ownership.

## Evidence, state transitions, tools, and result interpretation

```text
S1_DONE -> S2_DONE -> S3_REQUESTED

transient:
S3_REQUESTED -> S3_RETRYING -> S3_DONE -> COMPLETED

permanent before pivot:
S3_FAILED -> C2_REQUESTED -> C2_DONE
          -> C1_REQUESTED -> C1_DONE -> CANCELLED

unknown:
S3_REQUESTED -> S3_OUTCOME_UNKNOWN -> QUERY_RESULT
```

One participant's log `processed=true` is weaker than its committed business row plus inbox record. A DLQ entry shows normal consumption stopped, not whether the side effect happened before failure.

## Immediate mitigation/recovery

Throttle or pause the affected Saga type if the third service is broadly unavailable and backlog threatens expiry. Extend reservations only through approved bounded rules. Resolve unknown outcomes before compensation. Redrive a corrected poison message only after fixing its cause. Compensate reverse dependencies and monitor each result.

## Permanent correction/design

Persist a durable state machine; make each step and compensation idempotent; use outbox/inbox; classify errors; guard transitions; define pivot and retry limits; include reconciliation and manual states. If the third dependency is predictably unavailable, consider decoupled acceptance with honest pending status rather than a long synchronous request.

## Prevention and alerts

Alert on state age, step error rate, retry exhaustion, DLQ, expired semantic locks, and compensation failures. Test each crash window and a third-service outage under realistic backlog.

## Common mistakes

- Automatically compensating every timeout.
- Retrying permanent business rejection.
- Compensating in arbitrary order.
- Holding a database transaction open across all calls.
- Declaring success because an event was published.

## Concise interview-ready answer

I inspect the durable state of all three services and classify the third outcome. If it is transient and after a pivot or otherwise safely retriable, I retry forward with the same idempotency key and bounded backoff. If it is a confirmed permanent failure before the pivot, I compensate earlier committed steps in reverse dependency order. Unknown outcomes require status lookup first. Durable transitions, outbox/inbox, version guards, reconciliation, and manual escalation make that process operable.

---

# 4. How would you implement Saga for an e-commerce order?

## Meaning and invariants/risks

Implementation begins with business invariants:

- An order is fulfillable only when required stock and payment conditions hold.
- Available stock never becomes negative.
- One checkout operation causes at most one logical payment authorization/capture.
- Reservation, authorization, shipment, cancellation, and refund remain auditable.
- Every nonterminal order has an owner, deadline, and recovery path.

Temporary `PENDING` states are part of the API contract. A 202 response with an order-status resource is often more truthful than holding a request open until every participant finishes.

## Failure locations

Every boundary can fail: local commit, outbox relay, broker delivery, consumer transaction, provider call, reply publication, timeout scheduler, compensation, schema evolution, or concurrent customer action.

## Detailed causal mechanisms with plain-language example

An orchestrated flow can be:

```text
1. Order creates PENDING order and Saga row.
2. Orchestrator commands Inventory to reserve.
3. Inventory commits reservation plus reply outbox.
4. Orchestrator commands Payment to authorize.
5. Payment queries/calls provider with stable key and records outcome.
6. Orchestrator confirms order.
7. A later fulfillment Saga captures payment and ships under its own pivot rules.
```

Separating authorization from capture may reduce refund exposure, but business and provider rules decide. If payment is declined, Inventory releases. If authorization succeeds after cancellation, Payment follows the late-success void policy.

## Ordered investigation/design reasoning and why each step matters

1. Define aggregates and owners: Order owns order status, Inventory owns quantity/reservation, Payment owns payment ledger, Shipping owns shipment.
2. Draw legal forward, compensation, timeout, unknown, and manual transitions so no behavior is invented during an incident.
3. Order steps by reversibility and place the pivot late.
4. Define stable Saga, command, message, and provider keys to make retries recognizable.
5. Put state change and outbox write in each local transaction.
6. Put inbox claim and business effect in each consumer transaction where feasible.
7. Guard state versions and define per-order sequence handling for concurrency/reordering.
8. Add timeout scheduling, bounded retry classification, reconciliation, and operator controls.
9. Version message schemas additively and support mixed participant versions.
10. Test all crash points and business exceptions before launch.

## Evidence, state transitions, tools, and result interpretation

```text
PENDING
 -> RESERVING_STOCK
 -> STOCK_RESERVED
 -> AUTHORIZING_PAYMENT
 -> PAYMENT_AUTHORIZED
 -> CONFIRMED

decline:
AUTHORIZING_PAYMENT -> PAYMENT_DECLINED
 -> RELEASING_STOCK -> CANCELLED

uncertain:
AUTHORIZING_PAYMENT -> PAYMENT_UNKNOWN
 -> PAYMENT_AUTHORIZED or PAYMENT_DECLINED or MANUAL_REVIEW
```

Expose an operator view assembled from Saga history, participant status lookups, and broker evidence. Do not let the view mutate records by default. Trace spans should link command publish, consume, provider call, and reply, but durable state remains authoritative.

## Immediate mitigation/recovery

Feature-disable only the affected checkout path if needed, preserve browsing and existing-order status. Apply bounded backpressure rather than accepting unlimited pending work. Resume specific states with idempotent commands. Void/refund and release only from approved transitions. Use manual review for legal or financial ambiguity.

## Permanent correction/design

Use a durable orchestrator when the flow has several branches, deadlines, and compensation policies; choreography can remain suitable for simple notifications. Keep domain logic in participant services. Implement outbox relay monitoring, inbox uniqueness, provider key lookup, semantic reservation expiry, and a reconciliation worker.

## Prevention and alerts

Create SLOs for completion time and compensation time. Alert by oldest state age, transition-specific accumulation, outbox age, unknown payment count, reservation expiry risk, and manual backlog. Run periodic invariant checks such as "confirmed order without payment authorization."

## Common mistakes

- Treating the synchronous HTTP call chain as the Saga.
- Keeping orchestrator state only in memory.
- Making the orchestrator write participant databases.
- Publishing without an outbox.
- Omitting unknown and manual states.
- Using one generic retry rule for business decline and network failure.

## Concise interview-ready answer

I define Order, Inventory, Payment, and Shipping as separate owners and persist an order Saga state machine. A pending order commands idempotent stock reservation, then payment authorization, then guarded confirmation; confirmed permanent failure compensates in reverse order, while unknown outcomes are queried. Every local effect is atomic with outbox/inbox records, transitions use versions, irreversible work is late, and age metrics, reconciliation, schema compatibility, and audited manual recovery make it production-ready.

---

# 5. What happens if a Saga compensation operation itself fails?

## Meaning and invariants/risks

Compensation failure is expected in distributed systems and must be a first-class state, not an unhandled exception. The Saga is not cancelled until compensation is durably confirmed. Risks include held inventory, unreturned money, conflicting customer status, endless retry storms, and silent manual backlog.

## Failure locations

- Compensation request is not published or delivered.
- Participant is unavailable or rejects the command.
- Compensation committed but acknowledgement was lost.
- Original action never occurred, or its result remains unknown.
- Compensation is no longer legal, such as refund window expiry.
- A concurrent forward action changed the resource.
- Provider compensation is asynchronous or partially completed.

## Detailed causal mechanisms with plain-language example

Payment capture succeeded, shipping failed, and refund returns HTTP 503. The refund may not have reached the provider, may be pending, or may have completed before the response was lost. Retrying with the same refund key is safe only if the provider honors it. Marking the order `REFUNDED` on request submission lies to the customer; the state should be `REFUND_REQUESTED` or `REFUND_PENDING` until confirmed.

## Ordered investigation/design reasoning and why each step matters

1. Keep the exact compensation state and original-operation reference; without them, recovery can duplicate or target the wrong effect.
2. Classify failure and query remote status for ambiguous outcomes.
3. Reread current aggregate/version to ensure compensation is still legal.
4. Retry only transient failures with the same key, bounded backoff, jitter, and ownership lease.
5. If normal retries exhaust, open an auditable exception with amount, risk, deadline, and runbook.
6. Apply an alternative approved action, such as credit rather than expired provider refund, only under business authorization.
7. Verify all downstream state and close the exception explicitly.

## Evidence, state transitions, tools, and result interpretation

```text
REFUND_REQUESTED
 -> REFUND_PENDING
 -> REFUNDED

technical failure:
REFUND_REQUESTED -> COMPENSATION_RETRYING
 -> REFUND_REQUESTED or MANUAL_REVIEW

permanent failure:
REFUND_REQUESTED -> COMPENSATION_BLOCKED
 -> APPROVED_ALTERNATIVE_REMEDY -> COMPENSATED
```

Track attempts and next-attempt time. A rising retry count with unchanged provider status indicates persistent failure. A provider success plus local pending status indicates acknowledgement/recording repair, not another refund.

## Immediate mitigation/recovery

Stop duplicate automated attempts if idempotency is uncertain. Query status, then safely resume or record confirmed completion. Protect expiring refund windows and customer deadlines with prioritized escalation. Do not delete DLQ entries until their business effect and replay result are verified.

## Permanent correction/design

Model compensation as a durable sub-workflow with idempotency, status lookup, deadlines, alternate remedies, and manual escalation. Separate "requested" from "completed." Give operators a least-privilege action that invokes validated domain commands rather than direct SQL.

## Prevention and alerts

Alert immediately on compensation failure rate, oldest compensation age, retry exhaustion, provider-window expiry, and growing manual queue. Exercise provider outage and lost-response scenarios.

## Common mistakes

- Marking the whole Saga cancelled when compensation was only requested.
- Infinite immediate retries.
- Generating a new refund key per attempt.
- Assuming compensation cannot fail.
- Hiding failure in a DLQ without ownership.
- Editing the ledger to look refunded.

## Concise interview-ready answer

A failed compensation leaves the Saga in a durable compensation-pending or blocked state; it is not complete. I resolve ambiguous remote outcomes, then retry transient failures with the same key, bounded backoff, and guarded ownership. Exhausted or permanently illegal compensation creates an audited manual exception and approved alternate remedy. Alerts on age, retry exhaustion, and financial deadlines plus reconciliation ensure the exception cannot disappear.

---

# 6. How would you make Saga operations idempotent?

## Meaning and invariants/risks

Idempotency maps repeated delivery of one logical operation to one business effect and a stable result. It is required for commands and compensations because publishers, brokers, consumers, schedulers, clients, and operators can all retry. Risks include duplicate charges, double releases, repeated emails, and a false belief that broker "exactly once" covers external systems.

## Failure locations

- Producer creates a new operation ID on every retry.
- Consumer checks a key then crashes before recording it.
- Business write commits but deduplication record does not.
- Provider call succeeds before local result is stored.
- Same key is reused for different payloads.
- Deduplication expires before redelivery.
- Out-of-order opposite commands are each individually idempotent but conflict.

## Detailed causal mechanisms with plain-language example

Two workers receive `ReserveStock(order-44)` concurrently. A read-before-write dedupe check finds no row in both workers, and both decrement stock. A unique inbox key and a transaction that claims it with the reservation makes one succeed. The loser returns the recorded result.

For an external charge, send `order-44:authorize:v1` to the provider. If the process crashes after provider success, a recovery query using that key discovers the authorization and records it instead of charging again.

## Ordered investigation/design reasoning and why each step matters

1. Define the logical operation boundary; "HTTP request" is often too narrow.
2. Generate a stable key at the workflow owner and propagate it unchanged across retries.
3. Bind key to operation type, target, and payload hash so accidental conflicting reuse is rejected.
4. Enforce uniqueness atomically with the local effect or a durable in-progress claim.
5. Persist terminal result so duplicates receive consistent behavior.
6. For external effects, use provider idempotency/status lookup and recover in-progress claims.
7. Add aggregate version/sequence rules because idempotency alone does not order reserve and release.
8. Retain keys for the maximum replay/audit horizon.

## Evidence, state transitions, tools, and result interpretation

```text
RECEIVED -> IN_PROGRESS -> SUCCEEDED
                      `-> FAILED_PERMANENT
                      `-> OUTCOME_UNKNOWN
```

Unique constraint violations are normal duplicate signals if the stored payload matches. A matching key with a different payload is a protocol conflict and should not return the old success silently. Metrics should distinguish duplicate-same, duplicate-in-progress, and key-conflict.

## Immediate mitigation/recovery

If duplicates are causing side effects, pause the affected consumer partition or operation, preserving the backlog. Determine which effects actually committed. Reconcile external providers and correct through business compensation. Do not bulk-delete duplicate messages as a substitute for fixing state.

## Permanent correction/design

Implement inbox and business mutation in one local transaction, outbox for resulting events, unique business keys, state/version guards, and provider keys. Define behavior for duplicate in-progress requests and stale commands. Make compensations idempotent independently from forward operations.

## Prevention and alerts

Test concurrent duplicate delivery, crash after side effect, crash before result persistence, delayed redelivery, and key/payload conflict. Alert on conflict rate and unusual duplicate spikes; ordinary duplicate-same counts can be informational.

## Common mistakes

- Random UUID per retry.
- In-memory cache as the only dedupe store.
- Check-then-act without a unique constraint.
- Claiming HTTP `PUT` is automatically business-idempotent.
- Treating idempotency as mutual exclusion or ordering.
- Expiring payment keys too early.

## Concise interview-ready answer

I assign a stable key per logical Saga step and propagate it through retries and the external provider. The participant atomically records that key with its local business effect, stores the result, and rejects key reuse with a different payload. In-progress and unknown outcomes have recovery rules. Outbox/inbox handles delivery, while unique constraints, aggregate versions, and sequence checks separately handle concurrency and ordering.

---

# 7. When would you choose choreography vs orchestration?

## Meaning and invariants/risks

This is a tradeoff about ownership and visibility, not synchronous versus asynchronous communication. Choreography distributes reaction logic among participants; orchestration centralizes workflow control in a durable coordinator. Both can be event-driven and both can fail.

## Failure locations

Choreography risks hidden cycles, accidental subscribers, incompatible event semantics, distributed timeout policy, and no obvious owner for stuck work. Orchestration risks coordinator outage, a throughput hotspot, overcentralized domain logic, and command coupling. Both risk outbox gaps, duplicate handling defects, and schema incompatibility.

## Detailed causal mechanisms with plain-language example

A two-step flow where `UserRegistered` causes an independent welcome email is natural choreography; email failure should not cancel registration. A checkout with reserve, payment, fraud review, deadlines, compensation, and manual review benefits from orchestration because the business flow has branches and one accountable owner.

If eight services each emit events that trigger one another, understanding why a refund occurred may require reconstructing an implicit graph. An orchestrator transition history makes that policy explicit, but a poorly designed orchestrator that directly calculates stock and writes every database violates service ownership.

## Ordered investigation/design reasoning and why each step matters

1. Count participants, branches, timeouts, compensations, and manual states; complexity drives need for explicit control.
2. Decide whether there is a natural workflow owner; lack of ownership is an operational risk.
3. Assess coupling stability and event reuse; facts useful to independent consumers favor events.
4. Assess end-to-end visibility and audit requirements.
5. Evaluate scale and availability of a coordinator and team ability to operate it.
6. Keep participant domain validation local regardless of choice.
7. Consider a hybrid: orchestrated core transaction plus choreographed notifications/analytics.

## Evidence, state transitions, tools, and result interpretation

For choreography, maintain a documented event-flow graph, causation IDs, consumer ownership, schema registry, and end-to-end age metrics. For orchestration, inspect workflow state, command/reply history, scheduler, and shard/lease health. In either case, prove atomic publication and idempotent consumption.

## Immediate mitigation/recovery

During an incident, identify the current workflow owner. In choreography, trace causation across topics and stop only the faulty consumer/flow if possible. In orchestration, fail over or resume durable coordinator workers without bypassing state guards. Do not switch architecture during the incident.

## Permanent correction/design

Choose choreography for a small, loosely coupled reaction graph with independent consequences. Choose orchestration for multi-step business workflows with ordering, deadlines, compensations, or strong audit/operational needs. Use hybrid boundaries and avoid shared databases.

## Prevention and alerts

Review event graphs for cycles and unowned events. Load-test orchestrator partitions and failover. Alert on end-to-end Saga age in addition to component health.

## Common mistakes

- "Choreography has no coupling."
- "Orchestration is a distributed monolith by definition."
- Using events as imperative commands with unclear owner.
- Putting all domain decisions in the orchestrator.
- Choosing only because a framework is fashionable.

## Concise interview-ready answer

I choose based on workflow complexity and ownership. Simple, stable, independent reactions fit choreography. A transaction with several ordered steps, deadlines, compensation, and manual states usually needs a durable orchestrator for explicit policy and observability. Participants still own domain rules, and both options require outbox/inbox, idempotency, schema compatibility, and reconciliation. A common design is orchestrated checkout with choreographed side effects such as notifications.

---

# 8. How would you troubleshoot a stuck Saga in production?

## Meaning and invariants/risks

A Saga is stuck when it remains nonterminal beyond its expected state-specific deadline and normal retry policy is not making progress. It may instead be intentionally waiting for a timer or human decision, so age must be interpreted with state and policy. Risks include expiring reservations, customer uncertainty, financial exposure, and unsafe mass replay.

## Failure locations

- Orchestrator worker, scheduler, or lease is unhealthy.
- Outbox relay is stopped.
- Broker topic/ACL/routing is wrong, partition is unavailable, or consumer lag is high.
- Consumer repeatedly fails and message is in DLQ.
- Participant committed but reply is absent.
- Illegal/stale transition rejects a late result.
- Retry `next_attempt_at` is wrong due to clock/unit/config error.
- Semantic lock expired.
- Schema/deployment mismatch affects only one version.
- Manual-review queue has no owner.

## Detailed causal mechanisms with plain-language example

Saga `S-42` remains `PAYMENT_REQUESTED`. The orchestrator outbox says published, the broker has the command, Payment inbox and authorization are committed, but Payment's reply outbox is unpublished because the relay lost its broker credential. Retrying authorization is wrong. Repairing the relay and publishing the existing reply advances the Saga safely.

## Ordered investigation/design reasoning and why each step matters

1. Scope count, oldest age, first occurrence, state, tenant, partition, and version to separate one poison case from systemic failure.
2. Select a representative Saga and capture state/version, deadlines, attempts, and IDs before changing it.
3. Find the last proven local commit in transition history.
4. Walk the next boundary in order: producer outbox, broker record, consumer assignment/lag, inbox, participant business row, reply outbox, return broker record, orchestrator inbox.
5. Query external providers for unknown side effects.
6. Check scheduler/lease/fencing state and illegal-transition metrics for concurrency or late results.
7. Correlate the start with deployment, configuration, certificate, schema, broker, or dependency changes.
8. Pick the narrowest idempotent recovery and test one case.
9. Verify end-to-end convergence before bounded batch recovery.

## Evidence, state transitions, tools, and result interpretation

Example safe Kubernetes evidence:

```text
kubectl -n shop-prod get deploy,pods
kubectl -n shop-prod describe pod payment-worker-6f947c8c7b-k9m2q
kubectl -n shop-prod logs payment-worker-6f947c8c7b-k9m2q --since=20m
```

Use read-only application admin views, Saga SQL replicas, broker consumer-group tools, tracing, provider lookup, deployment history, and time-synchronized logs. Redact sensitive payloads.

Interpretation chain:

| Last proof | Next focus |
|---|---|
| State committed, no producer outbox | Atomicity/design defect |
| Outbox pending | Relay, credentials, broker connectivity |
| Broker has message, no inbox | Routing, consumer assignment, lag, deserialization |
| Inbox and business row exist, no reply outbox | Consumer transaction/design defect |
| Reply published, no coordinator inbox | Return routing, lag, schema |
| Coordinator inbox exists, state unchanged | Transition guard, duplicate/stale input, transaction failure |

## Immediate mitigation/recovery

Pause only affected intake if backlog or exposure is growing. Restore relay/consumer/scheduler health. Quarantine poison messages. Query unknown external outcomes. Resume one Saga through a guarded idempotent command, then process a bounded cohort with rate limits and monitoring. Escalate expired or ambiguous cases to manual review.

## Permanent correction/design

Add state-specific deadlines, heartbeat/lease monitoring, outbox age metrics, transition history, reconciliation sweeps, DLQ ownership, self-service read-only diagnostics, and restricted recovery commands. Correct schemas/configuration and write a regression test at the exact failed boundary.

## Prevention and alerts

Alert on oldest nonterminal age by state, no-progress rate, unpublished outbox age, consumer lag, DLQ growth, invalid transitions, retry scheduler drift, compensation age, and unowned manual cases. Include runbooks that distinguish replay from re-execution.

## Common mistakes

- Restarting everything before collecting state.
- Replaying from the first step instead of the missing transition.
- Looking only at application logs.
- Treating DLQ removal as recovery.
- Bulk SQL-updating state to `COMPLETED`.
- Ignoring a late success after compensation.

## Concise interview-ready answer

I scope the affected states and choose one Saga, then reconstruct its durable path from Saga transition through producer outbox, broker, consumer inbox, participant state, reply outbox, and coordinator inbox. I query external providers for ambiguous effects and inspect retries, leases, DLQ, versions, and deployments. I recover one case with the narrowest guarded idempotent action, verify all participant invariants, then perform bounded reconciliation and fix the failed boundary plus its age alert.

---

# 5. Cross-cutting production patterns

## 5.1 Duplicate and out-of-order events

Design consumers around facts, not delivery assumptions:

```text
if message ID already completed:
    return stored outcome
if aggregate sequence is less than current sequence:
    classify as stale and acknowledge according to policy
if aggregate sequence > expected next sequence:
    park briefly or request missing state; do not wait forever
if transition is legal at current version:
    apply atomically
else:
    record conflict for reconciliation
```

Sequence must be per aggregate or workflow. A global sequence can become a bottleneck. When facts are commutative, such as independent counters with proper semantics, strict ordering may be unnecessary. When `PaymentAuthorized` and `OrderCancelled` race, a documented late-result transition is required.

## 5.2 Semantic lock and reservation lifecycle

A reservation is a durable business record:

```text
reservation_id
owner_saga_id
resource and quantity
state: ACTIVE, CONSUMED, RELEASED, EXPIRED
expires_at
version
```

Expiry is itself a transition, not silent deletion. A late consume after expiry must fail or enter revalidation. Clock source, grace period, renewal authorization, and reconciliation should be explicit. Avoid leases so short that normal pauses expire them and so long that failures hoard capacity.

## 5.3 Reconciliation architecture

Reconciliation is a correctness control, not just incident cleanup:

1. Select bounded nonterminal or mismatched candidates.
2. Read authoritative participant and provider states.
3. Derive the one legal next action from versioned policy.
4. Claim the case with a lease/fencing token.
5. Invoke the normal idempotent domain command.
6. Verify convergence and append an audit decision.
7. Rate-limit and expose backlog/age.

Avoid a reconciler that writes every service database. It should call owner-approved operations and be safe if two reconcilers overlap.

## 5.4 Manual intervention

Manual review is a designed terminal-like holding state with:

- Reason and severity.
- Complete non-sensitive evidence links.
- Permitted actions and preconditions.
- Dual approval for high-risk financial actions where required.
- Actor identity, timestamp, before/after state, and reason.
- Customer communication status.
- Deadline and escalation owner.
- Reconciliation verification after action.

Operators should never need to infer SQL updates from a wiki. A recovery API should validate current state/version, enforce authorization, use the original operation key, produce an audit record, and support dry-run output.

## 5.5 Auditability and privacy

Audit the decision chain, not secret payloads. Payment records can retain provider references, amount, currency, operation type, state, and actor without storing card credentials. Restrict access by role, encrypt sensitive fields, set retention by regulatory/business need, and log access to manual controls.

---

# 6. Decision trees

## 6.1 A Saga step timed out

```text
Step timed out
|
+-- Does the step have an external or committed side effect?
|   |
|   +-- No proof it was attempted -> safely submit with original key
|   |
|   `-- It may have been attempted
|       |
|       +-- Status lookup available -> query by original key
|       |   |
|       |   +-- success -> record/continue if transition legal
|       |   +-- confirmed failure -> retry or compensate by classification
|       |   `-- unknown -> bounded retry of lookup, then manual review
|       |
|       `-- No status lookup
|           -> do not blind-retry non-idempotent effect
|           -> isolate and follow approved ambiguity procedure
|
`-- After outcome is known
    +-- before pivot and permanent failure -> compensate
    `-- after pivot or retriable requirement -> roll forward
```

## 6.2 A compensation failed

```text
Compensation failed
|
+-- Duplicate-safe and transient?
|   `-- retry with same key, backoff, jitter, deadline
|
+-- Outcome unknown?
|   `-- query target/provider; record confirmed state
|
+-- Permanent or business-illegal?
|   `-- COMPENSATION_BLOCKED -> approved alternate remedy/manual review
|
`-- Confirmed complete?
    `-- verify all invariants -> close Saga and audit exception
```

## 6.3 Retry, compensate, or manual review

| Condition | Preferred direction |
|---|---|
| Transient error, safe idempotent step, deadline remains | Retry forward |
| Confirmed permanent failure before pivot | Compensate backward |
| Failure after pivot with retriable completion | Roll forward |
| Unknown non-idempotent external result | Query; manual if unresolved |
| Compensation permanently unavailable | Approved alternative/manual |
| Conflicting concurrent business action | Reevaluate current state; do not force old plan |

---

# 7. Useful extra interview questions

## 7.1 Can a Saga provide strong consistency?

It can enforce carefully chosen invariants using local atomicity, reservations, uniqueness, and state guards, but it does not provide global serializable isolation across independent databases. Intermediate states are observable. If a requirement truly needs one atomic invariant, reconsider service/data boundaries or use a single transactional owner.

## 7.2 Does Kafka exactly-once remove the need for idempotency?

No. Broker transactions can atomically consume and produce within supported broker boundaries, but database writes, HTTP calls, payment providers, emails, and operator replays remain outside. End-to-end business idempotency and reconciliation are still necessary.

## 7.3 Should compensation always run in reverse order?

Reverse dependency order is a strong default because later effects often depend on earlier ones. Independent compensations may run in parallel, but only when business dependencies, rate limits, and failure isolation permit it. The order must be explicit.

## 7.4 How do you evolve Saga event schemas?

Use additive changes, defaults for absent fields, tolerant readers, schema compatibility checks, versioned business semantics when meaning changes, and mixed-version tests. Retain consumers for in-flight old events or migrate them with a controlled, auditable process.

## 7.5 How do you prevent two workers from owning one Saga?

Use conditional state/version updates or a lease with a monotonic fencing token. A lease alone is unsafe if an old worker resumes after expiry; participants must reject stale tokens or the state transition must be guarded.

## 7.6 When is two-phase commit preferable?

When all required resources support it, latency and availability tradeoffs are acceptable, operational tooling is mature, and the consistency boundary truly cannot be redesigned. In autonomous microservices and external providers it is often unavailable or harmful, but it should be evaluated rather than rejected by slogan.

## 7.7 What should a Saga API return?

Often `202 Accepted` with a durable operation/order ID and status URL. The status distinguishes pending, completed, failed, compensation pending, and manual review. If the business operation can reliably complete within the caller deadline, a synchronous facade may wait, but internal durability cannot depend on that connection.

## 7.8 How do you test a Saga?

Test invariant properties and every transition. Inject crashes before and after local commit, outbox publish, consumer commit, provider response, and compensation response. Deliver duplicates and reordering, run concurrent cancel/complete, advance timers, deploy mixed schemas, and verify reconciliation plus audit output.

---

# 8. Cheat sheets

## 8.1 Design checklist

```text
[ ] Business invariants and allowed temporary states written
[ ] Data and workflow owner named
[ ] Forward, failure, unknown, late, compensation, manual states drawn
[ ] Compensable, pivot, and retriable steps classified
[ ] Stable operation and message identities defined
[ ] Local state plus outbox/inbox atomic
[ ] Idempotency result and payload-conflict behavior defined
[ ] Concurrency/version/sequence policy defined
[ ] Timeouts, deadlines, retry budget, and jitter defined
[ ] External outcome lookup available
[ ] Compensation failure and alternate remedy designed
[ ] Reconciliation, audit, operator authorization designed
[ ] Mixed-version and crash-point tests present
[ ] State-age and invariant alerts owned
```

## 8.2 Incident checklist

```text
Scope -> representative Saga -> current durable states -> UTC timeline
-> last proven commit -> next missing boundary -> outcome classification
-> retry/lease/DLQ/version checks -> narrow idempotent recovery
-> verify all invariants -> bounded cohort -> permanent fix and alert
```

## 8.3 Evidence interpretation

| Evidence | Proves | Does not prove |
|---|---|---|
| HTTP timeout | Caller stopped waiting | Remote failure |
| Command published | Broker accepted a record | Consumer business effect |
| Inbox row | Consumer claimed/processed per its transaction design | External provider success |
| Provider lookup success | Provider recognizes the operation | Order state advanced |
| Compensation requested | Recovery started | Compensation completed |
| DLQ entry | Normal handling stopped | Original side effect did not occur |
| Final Saga state | Coordinator believes workflow ended | Every participant invariant, unless reconciled |

## 8.4 One-minute interview framework

```text
1. State invariants and legal intermediate state.
2. Identify local transaction boundaries and owners.
3. Resolve confirmed versus unknown outcomes.
4. Choose retry forward, compensation backward, or manual path around pivot.
5. Require stable keys, guarded transitions, outbox/inbox, and durable state.
6. Explain evidence, mitigation, reconciliation, alerts, and permanent test.
```

## Final principle

A production Saga is not a chain of retries. It is a durable business protocol that makes uncertainty explicit, protects invariants with local atomicity and concurrency rules, treats compensation as real work, and provides evidence and authorized recovery until every case converges.
