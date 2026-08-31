# Scenario-Based Idempotency Interview Questions — Detailed Answers

## 1. Client retries `POST /orders` after the response is lost

### Scenario

A client sends:

```http
POST /orders
Idempotency-Key: ABC123
```

The server creates the order successfully, but the HTTP response is lost because of a network failure. The client retries.

### Problem

Without idempotency:

```text
Request 1 → create Order 101 → response lost
Request 2 → create Order 102
```

The customer intended one order, but two orders were created.

### Solution

Use an idempotency key supplied by the client.

Store the key and the result of the operation:

```text
IdempotencyKey | Status      | Response
---------------+-------------+----------------
ABC123         | SUCCESS     | Order 101
```

On the first request:

1. Check whether `ABC123` exists.
2. If it does not exist, process the request.
3. Create the order.
4. Store the result against `ABC123`.
5. Return the response.

On retry:

1. Check `ABC123`.
2. Find the existing successful operation.
3. Return the previously stored result.
4. Do not create another order.

### Interview answer

> I would use an idempotency key for the operation. The key would have a unique database constraint. The first request performs the operation and stores the result against that key. If the client retries because the response was lost, the service finds the existing key and returns the previous result instead of executing the business operation again.

---

# 2. Payment succeeds but the client retries

### Scenario

The payment request reaches the payment service.

```text
Client
  ↓
Payment Service
  ↓
Payment Gateway
  ↓
Payment SUCCESS
```

But the response is lost.

The client retries.

### Why this is dangerous

If the payment service simply calls the gateway again:

```text
First request  → ₹1,000 charged
Retry          → ₹1,000 charged again
```

The customer gets charged twice.

### Solution

Use an idempotency key for the payment operation.

```text
Idempotency-Key: PAYMENT-ABC123
```

The payment service should associate that key with the payment operation.

Ideally, the same idempotency key should also be passed to the external payment provider if the provider supports idempotent requests.

```text
Client
   ↓
Payment Service
   ↓
Payment Gateway
```

Both layers should understand the operation identity.

### Important point

A local database idempotency key alone cannot magically undo a payment that already happened externally.

For example:

```text
Payment Gateway → SUCCESS
        ↓
Application crashes
        ↓
DB never records SUCCESS
```

On retry, the application may not know that the payment already happened.

Therefore, for critical external operations, you need mechanisms such as:

- provider-side idempotency
- payment/reference ID
- durable payment state
- reconciliation
- querying the provider before creating another charge

### Interview answer

> For payments, I would use an idempotency key and preferably pass the same operation identifier to the payment provider. The provider should also treat repeated requests with that key as the same payment. I would persist the payment state and use reconciliation for cases where the application crashes after the external payment succeeds but before the local database is updated.
Reconciliation means:

        Periodically comparing your system's records with the payment provider's records and fixing any mismatches.
---

# 3. Why do we need an idempotency key if we already have a unique `order_id`?

### Important distinction

A unique business ID and an idempotency key solve different problems.

Suppose:

```text
order_id = 100
```

has a UNIQUE constraint.

That prevents:

```text
INSERT order_id = 100
INSERT order_id = 100
```

from creating two database rows.

But the second request might still execute business logic.

For example:

```text
Request 1
→ charge payment
→ insert order

Request 2
→ charge payment again
→ insert fails because order_id already exists
```

The unique constraint prevented the duplicate row, but it did **not necessarily prevent the duplicate side effect**.

### Idempotency key

The idempotency key represents:

> "This is the same client operation."

For example:

```text
Idempotency-Key = ABC123
order_id = 100
```

The server can recognize the second request as the same operation before performing the side effect again.

### Interview answer

> A unique business key protects data uniqueness, whereas an idempotency key protects repeated execution of an operation. A unique constraint may prevent duplicate rows, but it doesn't automatically prevent side effects such as duplicate payments, messages, emails, or external API calls.

---

# 4. Two identical requests arrive simultaneously

### Scenario

Two requests arrive at exactly the same time:

```text
Request A ─────┐
               ├── Idempotency-Key = ABC123
Request B ─────┘
```

If both do:

```text
SELECT * FROM idempotency WHERE key = 'ABC123'
```

both might see:

```text
No record
```

Then both execute the operation.

### Correct solution

Use an atomic operation backed by a UNIQUE constraint.

```sql
CREATE UNIQUE INDEX ux_idempotency_key
ON idempotency(idempotency_key);
```

Conceptually:

```text
Request A → INSERT ABC123 → SUCCESS

Request B → INSERT ABC123 → DUPLICATE KEY
```

Now only one request owns the operation.

### But duplicate-key exception alone isn't the entire solution

Request B should not simply return HTTP 500.

It needs to determine the state of the operation.

For example:

```text
ABC123 → IN_PROGRESS
```

Then B can:

- wait/poll for completion,
- return an appropriate "request in progress" response,
- or retrieve the completed result once available.

After A finishes:

```text
ABC123 → SUCCESS
response → Order 101
```

B can return the same result.

### Interview answer

> I would not rely on a check-then-insert because both requests can observe that the key doesn't exist. I would use an atomic insert with a UNIQUE constraint. One request wins and processes the operation. The other detects that the key already exists and then waits for or retrieves the result instead of executing the business operation again.

---

# 5. Application crashes after DB commit but before HTTP response

### Scenario

```text
Client
  ↓
Application
  ↓
DB COMMIT
  ↓
Application crashes
  ↓
HTTP response never reaches client
```

The client assumes the operation failed and retries.

### Without idempotency

```text
First request → Order created
Retry         → Another order created
```

### With idempotency

First request:

```text
ABC123 → SUCCESS → Order 101
```

Retry:

```text
ABC123 exists
        ↓
return Order 101
```

No second order is created.

### Important insight

This is one of the main reasons idempotency is required.

A client cannot distinguish:

```text
operation failed
```

from:

```text
operation succeeded but response was lost
```

Therefore, clients retry.

### Interview answer

> A successful database commit does not guarantee that the client received the HTTP response. Therefore the client can retry. The idempotency record lets the server recognize the retry and return the already-created result.

---

# 6. External API succeeds but local DB update fails

### Scenario

Your service does:

```text
1. Call payment provider
2. Payment succeeds
3. Store SUCCESS in local DB
```

But the application crashes between steps 2 and 3.

```text
Payment Provider
      ↓
SUCCESS
      ↓
Application crashes
      ↓
Local DB not updated
```

### Why a normal transaction isn't enough

A normal database transaction can protect your database.

It cannot automatically roll back a transaction that happened in an external system.

You have two separate systems:

```text
Your DB              Payment Provider
   │                        │
   │                        │
   └──── not one transaction┘
```

### Better design

Use an operation/payment ID.

For example:

```text
payment_reference = PAY123
idempotency_key = ABC123
```

On retry, check whether `PAY123` already succeeded at the provider.

If the provider supports idempotency:

```text
PAY123 → SUCCESS
```

The provider returns the original result rather than charging again.

### Additional protection

Use reconciliation:

```text
Local DB says UNKNOWN
        ↓
Query payment provider
        ↓
Provider says SUCCESS
        ↓
Update local DB to SUCCESS
```

### Interview answer

> A database transaction cannot atomically include an external payment provider. I would use a stable payment reference/idempotency key, provider-side idempotency if supported, durable local states such as PENDING and SUCCESS, and reconciliation to recover ambiguous outcomes.

---

# 7. Kafka consumer processes DB update but crashes before committing offset

### Scenario

```text
Kafka
  ↓
Consumer
  ↓
DB update
  ↓
CRASH
  ↓
Kafka offset not committed
```

After restart, Kafka delivers the message again.

### Why?

Kafka may use at-least-once processing semantics.

The message was successfully processed by the application, but Kafka does not know that because the offset was not committed.

### Result

```text
First delivery:
Order 100 → DB updated

Second delivery:
Order 100 → DB updated again
```

### How to make the consumer idempotent

Give every event a unique event ID.

```text
event_id = EVT123
```

Store processed events:

```text
processed_event
----------------
event_id UNIQUE
```

Processing:

```text
EVT123 not present
      ↓
process event
      ↓
DB update
      ↓
save EVT123
      ↓
commit Kafka offset
```

If EVT123 arrives again:

```text
EVT123 already exists
      ↓
skip business processing
```

### Better transactional approach

If your database and Kafka consumption flow allow the appropriate transaction/outbox pattern, you can make the state changes more robust. But even then, downstream operations should be designed carefully for duplicate delivery.

### Interview answer

> I assume Kafka messages can be delivered more than once. I would make the consumer idempotent using a unique event ID and a processed-event table or an equivalent atomic business constraint. The database update and recording of the processed event should be coordinated so that a retry cannot repeat the business effect.

---

# 8. Duplicate Kafka message

### Scenario

Kafka sends:

```text
OrderCreated
event_id = EVT100
order_id = 500
```

twice.

### Solution 1: Processed-event table

```text
processed_event
----------------
event_id UNIQUE
```

First:

```text
EVT100 → process
```

Second:

```text
EVT100 already processed → skip
```

### Solution 2: Business uniqueness

Suppose the event causes:

```sql
INSERT INTO order_history(order_id, event_type)
VALUES (500, 'CREATED');
```

You could have:

```text
UNIQUE(order_id, event_type)
```

Then the same event cannot create a duplicate history entry.

### Important distinction

A processed-event table is useful when the event itself has a unique identity.

A business constraint is useful when the business operation itself has a natural uniqueness rule.

### Interview answer

> I would prefer a stable event ID and make processing idempotent. Depending on the operation, I can use a processed-event table with a unique event ID or a business-level unique constraint. The key is that duplicate delivery must not create another business side effect.

---

# 9. If Kafka guarantees exactly-once, do we still need idempotency?

### Answer

Do not assume Kafka's exactly-once guarantees make your entire business operation exactly once.

Kafka's guarantees apply to specific Kafka processing and transactional boundaries.

Your application may still call:

```text
Kafka
 ↓
Application
 ↓
Database
 ↓
External REST API
 ↓
Email provider
```

Kafka cannot automatically make every external system exactly once.

### Example

```text
Kafka transaction succeeds
        ↓
External payment API called
        ↓
Application crashes
```

The external payment API may still have received the request.

### Interview answer

> Kafka's exactly-once semantics do not automatically make an entire distributed business workflow exactly once. Database operations, external REST calls, payments, emails, and other side effects still need appropriate idempotency or transactional patterns.

---

# 10. Batch job processes 10,000 records and crashes at 4,000

### Scenario

```text
1 → processed
2 → processed
...
4000 → processed
4001 → processing
4002 → not processed
...
```

The application crashes.

### Simple solution

Checkpoint:

```text
last_processed = 4000
```

Restart:

```text
start from 4001
```

This works well for strictly ordered sequential processing.

### But checkpoint alone is not always sufficient

Suppose processing is parallel:

```text
Worker A → 4001
Worker B → 4002
Worker C → 4003
```

Maybe 4003 finishes first.

If you store:

```text
last_processed = 4003
```

you could incorrectly assume 4001 and 4002 are complete.

### Better approach

Track processing at the record level.

```text
record_id | status
----------+----------
4001      | SUCCESS
4002      | SUCCESS
4003      | SUCCESS
```

Or make the operation itself idempotent so replaying a previously processed record is safe.

### Interview answer

> For a sequential job, a checkpoint can be sufficient. For parallel processing, a single global checkpoint can be unsafe because records can complete out of order. I would use per-record processing state and idempotent operations, potentially combined with checkpoints for efficiency.

---

# 11. Why isn't a single checkpoint enough for parallel jobs?

### Example

```text
Worker 1 → record 100 → slow
Worker 2 → record 101 → success
Worker 3 → record 102 → success
```

If you save:

```text
lastProcessed = 102
```

you might incorrectly skip 100 after restart.

### Correct approach

Track:

```text
100 → FAILED
101 → SUCCESS
102 → SUCCESS
```

On restart:

```text
retry 100
```

This is another reason idempotency is valuable.

Even if you reprocess:

```text
101
102
```

their operations should be safe to repeat.

### Interview answer

> A checkpoint represents progress, but it doesn't necessarily represent every individual record's state when processing is parallel. I would track record-level state or use idempotent operations so replay is safe.

---

# 12. Retry storm

### Scenario

Suppose Service A calls Service B.

B becomes slow.

A retries every failed request three times.

At the same time, clients are also retrying.

```text
100 client requests
        ↓
A
        ↓
300 retries
        ↓
B
```

The extra traffic can make B even slower.

### Idempotency problem

If the operation is not idempotent, retries may also produce duplicate effects.

### Solution

Use:

- idempotency for retryable operations
- exponential backoff
- jitter
- maximum retry attempts
- timeouts
- circuit breakers
- rate limiting
- appropriate retry policies

### Important point

Idempotency does not stop retries.

It makes retries **safe**.

Backoff and circuit breakers control how aggressively retries happen.

### Interview answer

> I would make the operation idempotent so retries are safe, but I would also control the retry behavior with exponential backoff, jitter, retry limits, timeouts, and circuit breakers. Idempotency and retry control solve different problems.

---

# 13. Is GET always idempotent? Is POST always non-idempotent?

### GET

Normally:

```http
GET /orders/100
```

does not change server state.

Calling it multiple times should have the same intended effect.

Therefore GET is generally considered idempotent.

### POST

Normally:

```http
POST /orders
```

creates a new resource.

Calling it twice can create:

```text
Order 101
Order 102
```

Therefore POST is generally non-idempotent.

### But POST can be designed to be idempotent

```http
POST /orders
Idempotency-Key: ABC123
```

Repeated requests with ABC123 can return the same order.

### Interview answer

> HTTP method semantics provide the usual expectation: GET, PUT, and DELETE are generally idempotent, while POST is generally not. However, an application can design a POST operation to be idempotent using an idempotency key.

---

# 14. Why is PUT generally idempotent while POST generally isn't?

### PUT

Example:

```http
PUT /users/10
{
  "name": "Bharath"
}
```

First request:

```text
user 10 → name = Bharath
```

Second request:

```text
user 10 → name = Bharath
```

The final state remains the same.

### POST

```http
POST /users
{
  "name": "Bharath"
}
```

First:

```text
user 101
```

Second:

```text
user 102
```

Therefore POST generally creates a new effect each time.

### Interview answer

> PUT usually specifies the target resource and desired state, so repeating the same PUT results in the same final state. POST generally asks the server to create/process a new subordinate resource or operation, so repeating it can create another effect.

---

# 15. Same idempotency key but different request payload

### Scenario

First request:

```text
Key = ABC123
amount = 1000
```

Later:

```text
Key = ABC123
amount = 5000
```

### Problem

The key identifies one operation.

Reusing it for a different request is ambiguous and potentially dangerous.

### Solution

Store a request hash.

```text
idempotency_key = ABC123
request_hash = SHA256(request)
```

On retry:

```text
same key
+
same hash
→ valid retry
```

Different hash:

```text
same key
+
different hash
→ reject
```

### Interview answer

> I would bind the idempotency key to the original request payload or operation parameters. If the same key is reused with a different payload, I would reject it because the key already represents a different operation.

---

# 16. How long should you retain idempotency keys?

There is no universal number.

It depends on the business operation and how long clients can legitimately retry.

For example:

```text
Short-lived operation → shorter retention
Payment operation      → potentially longer retention
```

### Important consideration

Suppose you delete:

```text
ABC123
```

after 24 hours.

A delayed retry arrives after 48 hours.

The system might treat it as a new request.

Therefore the retention period must match the acceptable retry/duplicate window.

### Interview answer

> The retention period is a business and operational decision. I would retain the key for at least the maximum period during which a legitimate retry could occur, with cleanup based on an expiration policy. For financial operations, I would be especially conservative and rely on the provider's and business domain's requirements.

---

# 17. What if the idempotency record is created but the business transaction fails?

### Scenario

```text
Create idempotency record
        ↓
Create order
        ↓
DB transaction fails
```

Should the key remain?

It depends on the failure.

### Permanent failure

Example:

```text
Insufficient inventory
Invalid request
```

You can store:

```text
FAILED
```

and return the same failure for the same request.

### Transient failure

Example:

```text
Database temporarily unavailable
```

You may want the client to retry the operation.

Therefore the system needs a well-defined state model.

Example:

```text
IN_PROGRESS
SUCCESS
FAILED
```

Potentially also:

```text
EXPIRED
```

### Important transaction design

If the idempotency record and business operation belong to the same local transaction, they can be committed or rolled back together.

For example:

```text
BEGIN
  insert idempotency record
  create order
COMMIT
```

If the transaction rolls back, both changes roll back.

For more complex workflows, separate durable states may be required.

### Interview answer

> I would distinguish permanent business failures from transient technical failures. The idempotency record should represent the operation's state, and transaction boundaries should be designed so that a failed local transaction doesn't leave an incorrect successful result.

---

# 18. Design an idempotency table

A basic table could look like:

```sql
CREATE TABLE idempotency_record (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key VARCHAR(255) NOT NULL,
    request_hash VARCHAR(128) NOT NULL,
    status VARCHAR(30) NOT NULL,
    response_code INT,
    response_body TEXT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP,
    UNIQUE(idempotency_key)
);
```

### Meaning of fields

| Field | Purpose |
|---|---|
| `idempotency_key` | Identifies the client operation |
| `request_hash` | Detects reuse of the key with a different request |
| `status` | Tracks IN_PROGRESS/SUCCESS/FAILED |
| `response_code` | Allows returning the original HTTP result |
| `response_body` | Allows returning the original response |
| `created_at` | Auditing and cleanup |
| `updated_at` | State tracking |
| `expires_at` | Retention/cleanup |

### Important constraint

```sql
UNIQUE(idempotency_key)
```

is critical for concurrency.

---

# 19. What if the idempotency record says IN_PROGRESS?

### Scenario

Request A starts:

```text
ABC123 → IN_PROGRESS
```

Request A is still processing.

Request B arrives with the same key.

### Options

#### Option 1: Wait

Request B waits for A to finish and then returns the same result.

#### Option 2: Return conflict/in-progress

Return an appropriate response indicating that the operation is already being processed.

#### Option 3: Poll

Client can poll the operation status.

### What you should NOT do

Do not blindly execute the operation again.

Otherwise:

```text
A → payment
B → payment
```

could cause duplicate effects.

### Interview answer

> If the key is IN_PROGRESS, I would not execute the operation again. Depending on the API contract, I could wait for the original operation, return an in-progress response, or provide a status endpoint.

---

# 20. Idempotency across multiple microservices

### Scenario

```text
Client
  ↓
Order Service
  ↓
Payment Service
  ↓
Inventory Service
  ↓
Shipping Service
```

The client sends one operation.

### Problem

Even if Order Service is idempotent, downstream services can receive duplicate events.

For example:

```text
Order Service
   ↓
Payment event
   ↓
Payment Service
```

The event may be delivered twice.

### Solution

Give the overall operation a stable identity.

For example:

```text
order_id = ORD100
operation_id = OP123
```

Downstream operations can use that identity.

Each service should make its own processing idempotent.

```text
Payment Service:
OP123 → processed

Inventory Service:
OP123 → processed
```

### Important point

You cannot simply assume:

> "Service A is idempotent, therefore the entire workflow is idempotent."

Each independently executed side effect needs appropriate protection.

### Interview answer

> I would propagate a stable business operation or correlation ID through the workflow. Each service should independently make its own side effects idempotent, because duplicate messages or retries can occur at any service boundary.

---

# 21. Saga event is delivered twice

### Scenario

Saga:

```text
Create Order
    ↓
Reserve Inventory
    ↓
Take Payment
    ↓
Create Shipment
```

Suppose:

```text
InventoryReserved
```

is delivered twice.

### Without idempotency

```text
InventoryReserved
      ↓
reserve inventory
      ↓
same event again
      ↓
reserve inventory again
```

This can incorrectly reserve twice.

### Solution

Use event identity:

```text
event_id = EVT100
```

Store:

```text
EVT100 → processed
```

When the duplicate arrives:

```text
EVT100 already processed
```

Skip the business operation.

### State transition can also help

For example:

```text
PENDING → RESERVED
```

If the state is already:

```text
RESERVED
```

then processing the same event again should not reserve another unit.

### Interview answer

> In a Saga, every consumer should be designed to tolerate duplicate events. I would use a unique event ID with a processed-event record and/or safe state transitions and business constraints. Compensation actions should also be idempotent.

---

# 22. Idempotent state transitions

Suppose:

```text
Order status = PENDING
```

Event:

```text
OrderConfirmed
```

First processing:

```text
PENDING → CONFIRMED
```

Duplicate event:

```text
CONFIRMED → CONFIRMED
```

The second operation has no additional business effect.

### Why this is useful

State transitions can naturally make operations idempotent.

For example:

```text
UPDATE orders
SET status = 'CONFIRMED'
WHERE order_id = 100
  AND status = 'PENDING';
```

After the first update:

```text
PENDING → CONFIRMED
```

The same update again affects zero rows.

### Important caution

State-based idempotency alone may not be enough if the operation has another side effect.

Example:

```text
UPDATE order status
SEND EMAIL
```

The status may be idempotent while the email is sent twice.

Therefore every side effect must be considered.

---

# 23. Transaction and idempotency

Suppose:

```text
BEGIN TRANSACTION

INSERT order
INSERT payment
UPDATE inventory

COMMIT
```

Then the application crashes before returning the response.

The client retries.

### What happens?

The DB transaction is already committed.

Without idempotency:

```text
retry → duplicate business operation
```

With idempotency:

```text
retry
 ↓
find existing operation
 ↓
return previous result
```

### Important relationship

A transaction gives you **atomicity within its transaction boundary**.

Idempotency gives you **safe repeated execution**.

They solve different problems.

### Interview answer

> A transaction protects consistency of the operations inside the transaction, while idempotency protects against repeated requests. I commonly need both: the transaction ensures the business changes commit atomically, and the idempotency mechanism ensures a retry doesn't execute the same operation again.

---

# 24. Does idempotency mean duplicate requests cannot happen?

No.

This is a very important interview distinction.

Idempotency does NOT mean:

```text
same request cannot arrive twice
```

It means:

```text
same operation can be safely executed repeatedly
```

Example:

```http
PUT /users/10
{
    "name": "Bharath"
}
```

Execute once:

```text
name = Bharath
```

Execute ten times:

```text
name = Bharath
```

Final state is the same.

### Important nuance

Mathematically, idempotency means:

```text
f(f(x)) = f(x)
```

In API design, it means repeating the same operation does not create an additional unintended effect.

---

# 25. Complete payment scenario

This is one of the strongest interview scenarios.

### Question

A client sends a payment request. The payment succeeds, but before your service returns the response, the service crashes. The client retries. How do you prevent double payment?

### Step 1: Client sends idempotency key

```http
POST /payments
Idempotency-Key: ABC123
```

The client must reuse the same key when retrying the same operation.

### Step 2: Payment service checks the key

```text
ABC123 exists?
```

If it is already:

```text
SUCCESS
```

return the stored result.

### Step 3: First request creates durable state

Conceptually:

```text
ABC123 → IN_PROGRESS
```

Then process the payment.

### Step 4: Use provider-side idempotency

Send the same stable operation identifier to the payment provider if supported.

```text
Payment Service
      ↓
Provider
      ↓
idempotency key = ABC123
```

The provider should not charge twice for the same key.

### Step 5: Store the result

After successful payment:

```text
ABC123 → SUCCESS
payment_id → PAY100
amount → 1000
```

### Step 6: Application crashes

Suppose:

```text
Provider → SUCCESS
Application → CRASH
```

The local state may be ambiguous.

On retry, do not blindly charge again.

Instead:

```text
Check local state
       ↓
Check provider using payment reference/idempotency key
       ↓
Provider says SUCCESS
       ↓
Update local DB
       ↓
Return PAY100
```

### Step 7: Handle different outcomes

A robust system should distinguish:

```text
IN_PROGRESS
SUCCESS
FAILED
```

For example:

```text
IN_PROGRESS
   ↓
SUCCESS
```

or:

```text
IN_PROGRESS
   ↓
FAILED
```

### Complete flow

```text
                 Client
                   |
                   | Idempotency-Key: ABC123
                   ↓
             Payment Service
                   |
                   | Check key
                   ↓
          +--------------------+
          | Idempotency Record |
          +--------------------+
                   |
             not found
                   |
                   ↓
              IN_PROGRESS
                   |
                   ↓
           Payment Provider
                   |
              payment
                   |
             +-----+-----+
             |           |
          SUCCESS       FAIL
             |           |
             ↓           ↓
        Store result   Store failure
             |
             ↓
          SUCCESS
             |
             ↓
          Response
```

If the client retries:

```text
Client
  ↓
ABC123
  ↓
Payment Service
  ↓
ABC123 already exists
  ↓
Return original result
```

### Strong interview answer

> I would make the payment API idempotent using a client-generated idempotency key and a unique database constraint. The first request creates an IN_PROGRESS operation, executes the payment using the same stable operation identifier with the payment provider if supported, and persists the final result. If the response is lost and the client retries, the service checks the idempotency record and returns the existing result instead of initiating another payment. For the failure window where the external provider succeeded but the local DB wasn't updated, I would use the provider's idempotency mechanism, a stable payment reference, and reconciliation/provider lookup rather than blindly charging again. I would also handle concurrent requests atomically so only one request owns the operation.

---

# 26. Unique constraint exception vs idempotency

A common interviewer follow-up is:

> "If the UNIQUE constraint throws an exception for the second request, isn't that enough?"

### Answer

No.

A UNIQUE constraint prevents duplicate data, but you still need application logic to interpret the duplicate.

For example:

```text
Request A
→ payment succeeds
→ INSERT payment
→ success

Request B
→ payment attempted again
→ INSERT fails due to UNIQUE constraint
```

The payment could already have happened twice.

Therefore the uniqueness check must happen **before or together with the side effect**, and the overall workflow must be designed for retries.

### Key lesson

```text
UNIQUE constraint
       ↓
prevents duplicate database rows

Idempotency
       ↓
prevents repeated business effects
```

They complement each other.

---

# 27. Idempotency key vs business key vs event ID

These are often confused.

| Concept | Meaning | Example |
|---|---|---|
| Idempotency key | Identifies a repeated client operation | `ABC123` |
| Business key | Identifies a business entity | `ORDER100` |
| Event ID | Identifies a particular event/message | `EVT500` |
| Database primary key | Identifies a DB row | `id=12345` |

They can sometimes contain related values, but they represent different concepts.

### Example

```text
Order:
order_id = ORD100

Client request:
idempotency_key = REQ123

Kafka event:
event_id = EVT500
```

Each serves a different purpose.

---

# 28. Idempotency vs exactly-once

These concepts should not be treated as identical.

### Idempotency

Means:

```text
Repeated execution → same intended business effect
```

### Exactly-once processing

Attempts to ensure:

```text
Operation is processed once within a defined system boundary
```

### Real distributed systems

Across:

```text
Service A
Service B
Database
Kafka
Payment provider
Email provider
```

absolute exactly-once behavior is difficult.

Therefore a common practical strategy is:

```text
At-least-once delivery
        +
Idempotent processing
        +
Unique constraints
        +
Transactions
        +
Outbox/inbox patterns where appropriate
```

---

# 29. What should you say if asked "How would you implement idempotency in Spring Boot?"

A typical design:

```text
Controller
   ↓
Idempotency Filter/Interceptor
   ↓
Idempotency Service
   ↓
Database
   ↓
Business Service
```

Request:

```http
Idempotency-Key: ABC123
```

The service:

1. Validate the key.
2. Calculate a request hash.
3. Atomically create the idempotency record.
4. If the key already exists:
    - compare the request hash,
    - if different, reject,
    - if SUCCESS, return stored response,
    - if IN_PROGRESS, wait or return in-progress.
5. If new, execute the business operation.
6. Store the final result.
7. Return the response.

### Database

```text
idempotency_key UNIQUE
```

### Transaction

The business DB changes and idempotency state should be coordinated appropriately.

---

# 30. Most important mental model

Whenever an interviewer gives you a retry scenario, think through these questions:

```text
1. Can the request be repeated?
        ↓
2. What identifies the same operation?
        ↓
3. Where is that identity stored?
        ↓
4. What happens if two requests arrive simultaneously?
        ↓
5. What happens if the application crashes?
        ↓
6. What happens if an external system already succeeded?
        ↓
7. What happens if Kafka redelivers the message?
        ↓
8. What happens if the same key is used with a different payload?
        ↓
9. How long is the key retained?
        ↓
10. Can every side effect safely tolerate repetition?
```

---

# Quick Revision Table

| Scenario | Main solution |
|---|---|
| HTTP response lost | Idempotency key + stored result |
| Payment retry | Provider + application idempotency |
| Same request concurrently | Atomic insert + UNIQUE constraint |
| Crash after DB commit | Stored idempotency result |
| External API succeeds before crash | Provider idempotency + reconciliation |
| Kafka duplicate | Event ID + idempotent consumer |
| Kafka offset not committed | Processed-event/business constraint |
| Batch crash | Checkpoint + per-record idempotency |
| Parallel batch | Record-level state |
| Retry storm | Idempotency + backoff + jitter + retry limits |
| POST duplicate | Idempotency key |
| Same key, different payload | Request hash + reject |
| Key expiration | Retention policy |
| IN_PROGRESS | Wait/poll/reject; never blindly repeat |
| Saga duplicate event | Event ID + idempotent consumer |
| State transition duplicate | Safe state transition |
| Transaction retry | Transaction + idempotency |
| Multiple microservices | Propagated operation ID + idempotent steps |

---

# Final Interview Formula

When answering almost any idempotency scenario, structure your answer like this:

```text
1. Identify the duplicate/retry problem
2. Give the operation a stable identity
3. Store that identity durably
4. Enforce uniqueness atomically
5. Execute the business operation safely
6. Store the result/state
7. Return the same result on retry
8. Handle concurrent requests
9. Handle crashes/ambiguous outcomes
10. Make downstream side effects idempotent too
```

The most important distinction to remember:

> **A UNIQUE constraint prevents duplicate data; idempotency prevents duplicate business effects. A transaction provides atomicity; idempotency makes retries safe.**


    Duplicate API request
    ↓
    Idempotency key
    ↓
    Shared durable storage
    ↓
    Unique constraint
    ↓
    Same key + same payload?
    YES → return previous result
    NO  → reject
    ↓
    Concurrent requests?
    DB uniqueness / atomic claim
    ↓
    Need atomic DB changes?
    Transaction
    ↓
    External side effect?
    Downstream idempotency
    ↓
    Unknown outcome?
    Query + reconciliation
    ↓
    Duplicate event?
    Event ID / business key
    ↓
    Already in final state?
    State-based idempotency
