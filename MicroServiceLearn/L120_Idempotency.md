Why is idempotency important in microservices?

    Idempotency is important in microservices because the same request/message may be processed more than once, and we want repeated processing to produce the same final result as processing it once.

1. Why can the same request happen multiple times?

        Duplicates are common in distributed systems:
        
        Client retries because of a timeout.
        API Gateway retries a failed request.
        Service A retries calling Service B.
        Kafka consumer processes a message, crashes, and receives it again.
        Network failure occurs after the operation succeeds but before the response reaches the caller.

The classic solution: Idempotency Key

        The client generates a unique key:

    POST /payments
    Idempotency-Key: abc123

The payment service stores:

    abc123 → PAYMENT_SUCCESS

When the same request comes again:

    POST /payments
    Idempotency-Key: abc123
    
    The service checks:
    
    Does abc123 already exist?
    |
    YES
    |
    Return previous result

It doesn't execute the payment again.


1. Idempotency key

For an API:

    POST /payments

    Idempotency-Key: abc123

Database:

    abc123 → SUCCESS

Retry:

    abc123 → already processed → return previous result
2. Unique constraint

        You don't necessarily need an explicit idempotency key.

Suppose business rule says:

    An order can have only one payment.

Database:

    UNIQUE(order_id)

Requests:

    Payment(orderId=101, ₹500)
    Payment(orderId=101, ₹500)

The database allows only one.

    This is particularly useful as a final safety net even if you already use an idempotency key.

3. State-based idempotency

Suppose:

Order:

    CREATED → PAID → SHIPPED

Consumer receives:

    ShipOrder(101)

First message:

    CREATED
    ↓
    SHIPPED

Duplicate message:

    SHIPPED
    ↓
    Already SHIPPED → don't execute again

So the current state itself prevents the duplicate operation.


How do you make a POST API idempotent?


    "POST is normally non-idempotent, so if I need to make it idempotent, 
    
    I would give each logical operation an idempotency identifier and persist the result. 
    On a retry, the service detects that the same operation was already processed and returns the previous result instead of performing the business operation again. 
    I would also use a database unique constraint and transaction to handle concurrent duplicate requests."

Where and how long you store idompotent keys

    "Typically server-side, often in a database for durable operations such as payments. 
    We can also use Redis when low-latency deduplication is needed and its durability characteristics are acceptable.
    We store the key along with the processing status and often the original result or a reference to it. 
    A unique constraint helps prevent concurrent duplicate processing."
    "There is no fixed duration. I retain it for at least the maximum period in which the same logical operation can be retried or replayed, with some safety margin. 
    The duration depends on the API retry policy, asynchronous message-retention/replay window, and the business risk of duplicate processing."

      First, I would determine whether the records are independent or need to be treated atomically. If partial completion is acceptable, I don't need one transaction for the entire job. I would process the records in smaller batches or chunks.
      
      I would maintain a durable checkpoint indicating the last successfully processed position or batch. If the job crashes, I resume from the last successful checkpoint rather than starting from the beginning.
      
      However, the checkpoint only tells me where to resume; it doesn't prove that records from a previously committed batch haven't already been inserted. So I also need a stable identifier for each source record, such as a source order ID, and enforce a unique constraint on that identifier in the database.
      
      For example, if batch 2 commits but the job crashes before updating the checkpoint, the retry may process batch 2 again. The unique constraint prevents those records from being inserted twice.
      
      If the records don't have a stable identifier, I would first establish an appropriate business key or have the source/job generate a stable record or operation ID. Without some stable identity, the system cannot reliably determine whether a retried record is the same record.
      
      If the records must succeed or fail together, I would instead process the appropriate batch inside a database transaction. If anything fails before commit, the batch is rolled back. For a large job, I would use smaller transactional batches rather than one huge transaction.
      
      So the main mechanisms are: checkpointing to know where to resume, stable business identifiers and unique constraints to prevent duplicates, and transactions when atomicity is actually required."

1. Using Redis
    
       Request
       ↓
       Check Redis for idempotency key
       ↓
       Exists?
       ┌─┴─┐
       YES  NO
       ↓    ↓
       Return  Store key
       result    ↓
       Process
       ↓
       Store result

Example:

    ABC123 → SUCCESS

On retry:

    ABC123 exists
    ↓
    Don't process again
    ↓
    Return SUCCESS

Typically you'd use an atomic operation such as:

    SET ABC123 PROCESSING NX EX 300
    
    The NX is important so two concurrent requests don't both acquire the key.

Limitation: Redis data can expire or be lost depending on its configuration, so the retention/durability requirements matter.

2. Using Database

    Create an idempotency table:

idempotency_key | status  | result
----------------|---------|--------
ABC123          | SUCCESS | payment-5001

And enforce:

    UNIQUE(idempotency_key)

Flow:

    Request
    ↓
    Check DB
    ↓
    Key exists?
    ┌─┴─┐
    YES  NO
    ↓    ↓
    Return  Process
    result    ↓
    Save key + result

If the same request comes again:

    ABC123 already exists
    ↓
    Don't process again
    ↓
    Return previous result

Main difference

|                   | Redis                    | DB                  |
| ----------------- | ------------------------ | ------------------- |
| Speed             | Very fast                | Slower than Redis   |
| Durability        | Depends on configuration | Stronger            |
| Storage           | In-memory primarily      | Persistent          |
| Typical use       | Fast deduplication       | Durable correctness |
| Unique constraint | Not DB constraint        | **Yes**             |


"A job inserts multiple records. If the job is stopped halfway, how do you handle retry?"


      "First, I would decide whether the records should be treated atomically.
      If they should all succeed or fail together, I would process them as a batch inside a database transaction. 
      The job decides the batch size, while the service/repository performs the inserts within the transaction. 
      If the job fails halfway through that transaction, the transaction is rolled back, so none of that batch's inserts remain.

      For a large job, I wouldn't put the entire job in one transaction. 
      I'd divide it into smaller batches or chunks. Each successful batch is committed and a checkpoint is maintained. 
      If the job fails during a later batch, I can resume from the last successful checkpoint.

      For retries, I also need duplicate protection. If the records have a natural unique business key,
     I can enforce a database unique constraint. If they don't have any common identifier, 
     the job can generate a batch/operation ID or unique record IDs so that a retried operation can be identified.
      This prevents duplicate inserts if the previous transaction actually committed but the job crashed before recording the checkpoint.



What happens if two identical requests arrive simultaneously?
How do you make the idempotency check itself thread-safe?

The problem

      Suppose:
      
      Request A ──┐
      ├──→ Payment Service
      Request B ──┘

Both have:

      Idempotency-Key = ABC123

If you do this:

      Request A: GET ABC123 → NOT FOUND
      Request B: GET ABC123 → NOT FOUND

      Request A → process payment
      Request B → process payment

💥 Customer gets charged twice.

The problem is that:

      CHECK
      
      and
      
      INSERT

were separate operations.

Solution 1: Database unique constraint

Create:

      CREATE UNIQUE INDEX
      ON idempotency_records(idempotency_key);

Then:

      Request A                    Request B
      |                            |
      ↓                            ↓
      Check key                     Check key
      |                            |
      ↓                            ↓
      Not found                     Not found
      |                            |
      ↓                            ↓
      INSERT ABC123                INSERT ABC123
      |                            |
      ↓                            ↓
      SUCCESS                      UNIQUE VIOLATION
      |                            |
      ↓                            ↓
      Process operation             Don't process

      The database guarantees that only one request can successfully create the record.

This is usually the strongest approach when using a DB.


How does an idempotency key work across multiple application instances?

This is very important.

      Suppose you have:

                 Load Balancer
                /      |      \
               ↓       ↓       ↓
             App-1   App-2   App-3
               \       |       /
                \      |      /
                   Database

Client sends:

      Idempotency-Key: ABC123

First request happens to reach App-1:

      ABC123
      ↓
      App-1
      ↓
      process payment

Retry might reach App-2:

      ABC123
      ↓
      App-2

If the key were stored only in App-1's memory, App-2 wouldn't know about it:

      App-1 memory:
      ABC123 → SUCCESS
      
      App-2 memory:
      ABC123 → ????

App-2 could process the payment again. ❌

Therefore the idempotency state must be shared

Use a shared store:

                 Load Balancer
                /      |      \
               ↓       ↓       ↓
             App-1   App-2   App-3
                \      |      /
                 \     |     /
                  Shared Redis
                       +
                      DB

For example:

      Redis:
      
      ABC123 → SUCCESS

Now:

      Request 1 → App-1
      ↓
      Redis ABC123
      ↓
      process
      ↓
      SUCCESS

Retry:

      Request 2 → App-2
      ↓
      Redis ABC123
      ↓
      SUCCESS
      ↓
      Don't process again

It doesn't matter which application instance receives the request.


How do you design an idempotent payment/order API?



      Client
      ↓
      POST /payments
      Idempotency-Key: ABC123
      ↓
      Payment Service
      ↓
      Check idempotency record
      ↓
      Already processed?
      ┌───────┴────────┐
      YES               NO
      ↓                 ↓
      Return            Create
      existing          PROCESSING record
      result              ↓
      Process payment
      ↓
      Save result
      ↓
      SUCCESS



How would you handle duplicate events from different producers?


Different producers represent the same business operation
      
      Producer A → A123 → Order 101
      Producer B → B456 → Order 101

Use a business idempotency key, such as:

      orderId + eventType
      
      provided that combination is actually unique for your business operation.

Interview answer

      "For an idempotent payment or order API, I would require the client to send an idempotency key and store its processing state and result in a shared store. I would atomically claim the key so concurrent requests cannot both process it, and use a database unique constraint as durable duplicate protection. On retries, I return the previously stored result instead of executing the operation again."
      
      "For duplicate events, I would put a unique event ID in each event and have the consumer maintain processed-event state with a unique constraint. If different producers can generate different event IDs for the same business operation, event ID alone isn't enough; I'd use a domain-level idempotency/business key such as orderId plus event type, depending on the business semantics."




What happens if the service crashes after the business operation succeeds but before saving the idempotency result?

Suppose:

      POST /payments
      Idempotency-Key: ABC123

The service does:

      1. Receive ABC123
         2. Check idempotency record → not found
         3. Charge ₹1000
         4. Payment succeeds ✅
         5. 💥 Service crashes
         6. Idempotency result was NOT saved

Now the database/payment system says:

      Payment = SUCCESS

but the idempotency store says:

      ABC123 = nothing

The client doesn't know what happened and retries:

      POST /payments
      Idempotency-Key: ABC123

If your implementation simply says:

      ABC123 not found
      ↓
      charge again

💥 Customer gets charged twice.

So how do we prevent this?

The key is:

      The business operation and the idempotency record need a reliable relationship.

For a payment that is stored in your own DB, a common solution is to put the idempotency key on the payment record itself and enforce uniqueness.

For example:
      
      payments
      --------------------------------
      payment_id
      idempotency_key
      order_id
      amount
      status
      
      UNIQUE(idempotency_key)

Then:

      ABC123 → Payment 5001

      If the service crashes after the DB transaction commits, the retry can query:
      
      WHERE idempotency_key = 'ABC123'

and discover:

      Payment 5001 → SUCCESS

So it returns the existing payment rather than creating another one.

But what if the business operation is an external payment provider?

This is harder.

Suppose:

      Your Payment Service
      ↓
      External Payment Provider
      ↓
      ₹1000 charged
      ↓
      💥 Your service crashes

Your DB might not yet know that the payment succeeded.

On retry, you cannot safely say:

"Let's charge again."

      because the first charge may have succeeded.

      You need the same idempotency key passed to the external payment provider, if the provider supports idempotency.

      Your Service
      |
      | ABC123
      ↓
      Payment Provider

Retry:

      Your Service
      |
      | ABC123 again
      ↓
      Payment Provider
      |
      ↓
      Returns existing payment

Therefore the provider also doesn't charge twice.
the provider mostly wil support some identification techniques


Use a provider transaction/payment intent you can query
Instead of blindly charging again:

      ABC123 → PROCESSING

After recovery:

      ABC123 → PROCESSING
      ↓
      Query provider
      ↓
      Did payment happen?
      /       \
      YES        NO
      |          |
      SUCCESS     retry

For example:

      Payment Service
      ↓
      "What's the status of payment ABC123?"
      ↓
      Payment Provider
      ↓
      SUCCESS

Then you update your DB to:

      ABC123 → SUCCESS


What happens when the idempotency record is stuck in PROCESSING?


Option 1: Wait

      If the original request is probably still running:

      ABC123 → PROCESSING
      ↓
      Wait/poll
      ↓
      SUCCESS
      ↓
      Return result

Option 2: Timeout / lease

Give PROCESSING a timeout:

      ABC123 → PROCESSING
      |
      5 minutes
      ↓
      EXPIRED

If it has been processing longer than the allowed period, another request/job can take over.

      But don't blindly retry, especially for payments, because the original operation might have succeeded just before the service crashed.

What if the same idempotency key arrives with a different request payload?
How do you guarantee atomicity between the idempotency record and business operation?



1. Same idempotency key with a different payload

Suppose the first request is:

      POST /payments
      Idempotency-Key: ABC123
      
      {
      "orderId": 101,
      "amount": 1000
      }

It succeeds:

      ABC123 → SUCCESS

Now someone sends:

      POST /payments
      Idempotency-Key: ABC123
      
      {
      "orderId": 101,
      "amount": 5000
      }

You must not treat this as a normal retry.

      The idempotency key represents the same logical operation, so the payload should be the same.

Solution: store a request fingerprint/hash

When the first request arrives:

      Idempotency-Key = ABC123
      
      Payload:
      orderId=101
      amount=1000
      
              ↓
      
      Hash(payload)
      ↓
      
      HASH1
      
      Store:
      
      key       | request_hash | status
      ABC123    | HASH1        | SUCCESS

On retry:

      ABC123
      amount=1000
      ↓
      HASH1
      ↓
      matches
      ↓
      return previous result

But if:

         ABC123
         amount=5000
         ↓
         HASH2
         
         then:
         
         HASH2 ≠ HASH1
         ↓
         REJECT

Typically return something like:

      409 Conflict

with a message such as:

      Idempotency key was already used with a different request.

Why is this necessary?

Otherwise:

      ABC123 + ₹1000
      ↓
      Payment SUCCESS
      
      ABC123 + ₹5000
      ↓
      "Already processed"

You would incorrectly associate two different operations with the same key.

So:

One idempotency key must represent one logical operation.

2. How do you guarantee atomicity between idempotency record and business operation?

This depends on whether they're in the same database.

            Same database → use one DB transaction
      
      For example:
      
      BEGIN TRANSACTION
      
      1. Create/claim idempotency record
         2. Create payment/order
         3. Update idempotency record → SUCCESS
      
      COMMIT
      
      If anything fails:
      
      ROLLBACK
      
      Everything rolls back together.

For example:

      Idempotency record ❌
      Payment record      ❌
      
      or, on success:
      
      Idempotency record ✅
      Payment record      ✅
      
      That's true atomicity because both operations participate in the same DB transaction.

But what if Redis is the idempotency store?

This is where you need to be careful.

Suppose:

      Redis
      ↓
      idempotency key
      
      DB
      ↓
      payment

You cannot do:

Redis transaction
+
DB transaction

      and assume they are one atomic transaction.

For example:

Redis → SUCCESS ✅
DB → 💥 failed

Now Redis says success while DB says failure.

Or:

      DB → SUCCESS ✅
      Redis → 💥 failed

Now the retry might not find the idempotency result.

Therefore, for strong atomicity:

Prefer:

      Idempotency table
      +
      Business table
      ↓
      SAME DATABASE
      ↓
      ONE TRANSACTION

For example:

BEGIN

      INSERT idempotency(key, request_hash, status)
      VALUES ('ABC123', 'HASH1', 'PROCESSING');
      
      INSERT payment(...);
      
      UPDATE idempotency
      SET status = 'SUCCESS';
      
      COMMIT
What if you still want Redis?

That's fine, but understand its role.

You can use:

      Redis
      ↓
      Fast lookup/cache

and:

      DB
      ↓
      Durable source of truth

      The DB transaction provides the correctness guarantee; Redis is used for performance/concurrency coordination.

Interview answer

For the first question:

      "An idempotency key represents one logical operation, so I would store a hash/fingerprint of the original request along with the key. If the same key arrives with the same payload, I return the existing result. If the payload differs, I reject the request because the client is reusing the key for a different operation."

For the second question:

      "If the idempotency record and business record are in the same database, I put both operations in the same database transaction, so they commit or roll back atomically. If the idempotency state is stored in Redis while the business data is in a DB, they cannot share a normal local transaction, so I would treat the DB as the durable source of truth and use Redis primarily for fast lookup or coordination."

The mental model

      Same key + same payload
      ↓
      RETRY
      ↓
      Return existing result
      
      
      Same key + different payload
      ↓
      INVALID
      ↓
      REJECT


      Idempotency + business record
      ↓
      Same DB transaction
      ↓
      Atomic