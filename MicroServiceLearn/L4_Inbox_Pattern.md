# Transactional Inbox Pattern - Interview Notes

---

# Table of Contents

1. What is the Transactional Inbox Pattern?
2. Why is it Needed?
3. The Duplicate Message Problem
4. How Inbox Pattern Works
5. Step-by-Step Flow
6. Database Schema
7. Why Inbox + Business Logic Must Be One Transaction
8. Failure Scenarios
9. Kafka Offset Commit
10. Inbox vs Outbox
11. Advantages
12. Disadvantages
13. Interview Questions
14. Interview Cheat Sheet

---

# What is the Transactional Inbox Pattern?

The **Transactional Inbox Pattern** is a **consumer-side reliability pattern** used in event-driven microservices.

Its purpose is to **prevent processing the same event multiple times**.

It does this by storing the **Event ID** in an Inbox table **inside the same database transaction** as the business logic.

---

# Why is it Needed?

Kafka (and many message brokers) provide **At-Least-Once Delivery**.

This means a message can be delivered multiple times.

Example:

```
Kafka

↓

OrderCreated

↓

Inventory Service

↓

Crash before committing offset

↓

Kafka sends OrderCreated again
```

Without protection, Inventory may reserve stock twice.

---

# The Duplicate Message Problem

Suppose inventory has

```
Stock = 10
```

Kafka sends

```
OrderCreated
Quantity = 2
```

Inventory processes

```
10

↓

8
```

Before committing Kafka offset

```
Crash
```

Kafka thinks

```
Message wasn't processed.
```

After restart

```
Kafka sends OrderCreated again
```

Inventory reserves again

```
8

↓

6
```

Only one order existed.

Stock is now incorrect.

---

# How Inbox Pattern Works

Instead of directly processing

```
Receive Event

↓

Business Logic
```

We do

```
Receive Event

↓

Check Inbox

↓

Already Exists?

↓

Yes → Ignore

No → Process

↓

Insert Inbox Row

↓

Commit
```

---

# Architecture

```
                  Kafka

                    |

                    v

           Inventory Service

                    |

        ----------------------------

        |                          |

        v                          v

   Inbox Table             Inventory Table

        |                          |

        -------- One Transaction -----

                    |

                    v

            Commit Kafka Offset
```

---

# Inbox Table

Example

| EventId | EventType | ProcessedAt |
|----------|-----------|-------------|
|abc123|OrderCreated|10:30 AM|

Each processed event has exactly one row.

---

# Step-by-Step Flow

## Step 1

Kafka sends

```
OrderCreated

EventId = abc123
```

---

## Step 2

Consumer starts transaction

```
BEGIN
```

---

## Step 3

Check Inbox

```
SELECT *

FROM inbox

WHERE event_id='abc123'
```

### If Found

```
Already processed.

Skip Business Logic.

Commit Offset.
```

Done.

---

### If Not Found

Continue.

---

## Step 4

Insert Inbox Row

```
Inbox

abc123
```

---

## Step 5

Execute Business Logic

Reserve inventory

```
Stock

10

↓

8
```

---

## Step 6

Commit Database Transaction

```
COMMIT
```

Both

- Inbox Row
- Inventory Update

are committed together.

---

## Step 7

Commit Kafka Offset

Now Kafka knows

```
Message successfully processed.
```

---

# Why Inbox and Business Logic Must Be One Transaction

This is the most important interview concept.

---

## Wrong Example 1

Insert Inbox

```
Inbox Saved ✅
```

Crash

```
Reserve Inventory ❌
```

Kafka sends duplicate.

Consumer checks Inbox.

```
Already Processed
```

Inventory is never updated.

Incorrect.

---

## Wrong Example 2

Reserve Inventory

```
10

↓

8
```

Crash

```
Inbox Not Saved
```

Kafka sends duplicate.

Consumer checks Inbox.

```
Not Found
```

Reserve again

```
8

↓

6
```

Duplicate business operation.

Incorrect.

---

## Correct Solution

Use one transaction.

```
BEGIN

Insert Inbox

Reserve Inventory

COMMIT
```

Only two possibilities exist.

### Success

```
Inbox Saved

Inventory Updated
```

or

### Failure

```
Inbox Rolled Back

Inventory Rolled Back
```

No inconsistent state.

---

# Failure Scenarios

---

## Case 1

Crash before Database Commit

```
BEGIN

Insert Inbox

Reserve Inventory

Crash
```

Transaction rolls back.

Kafka redelivers.

Everything executes again.

Correct.

---

## Case 2

Database Commit Successful

Crash before Kafka Offset Commit

```
Database Commit ✅

Crash

Offset Not Committed
```

Kafka sends duplicate.

Consumer checks Inbox.

```
Event Already Exists
```

Business logic skipped.

Only offset is committed.

No duplicate inventory reservation.

---

## Case 3

Duplicate Message from Kafka

Kafka sends

```
OrderCreated

EventId abc123
```

again.

Inbox contains

```
abc123
```

Skip processing.

---

# Kafka Offset Commit

Correct sequence

```
Receive Event

↓

Start Database Transaction

↓

Insert Inbox Row

↓

Execute Business Logic

↓

Commit Database Transaction

↓

Commit Kafka Offset
```

Never

```
Receive Event

↓

Commit Offset

↓

Business Logic
```

Otherwise a crash causes permanent message loss.

---

# Why Kafka Sends Duplicates

Kafka follows

```
At-Least-Once Delivery
```

If offset isn't committed

Kafka assumes

```
Consumer didn't finish.
```

Therefore

```
Redeliver Message
```

This behavior is expected.

Inbox Pattern makes duplicates harmless.

---

# Inbox vs Outbox

| Outbox | Inbox |
|---------|-------|
| Producer Side | Consumer Side |
| Prevents event loss | Prevents duplicate processing |
| Stores outgoing events | Stores processed incoming event IDs |
| Publisher reads Outbox | Consumer checks Inbox |
| Solves Dual Write Problem | Solves Duplicate Delivery |

---

# Outbox + Inbox Together

```
Order Service

|

| Save Order

| Save Outbox Event

↓

Database Commit

↓

Publisher

↓

Kafka

↓

Inventory Consumer

↓

Check Inbox

↓

Reserve Inventory

↓

Commit Database

↓

Commit Offset
```

Producer reliability

```
Outbox
```

Consumer reliability

```
Inbox
```

Together they provide reliable event-driven communication.

---

# Advantages

✔ Prevents duplicate business execution

✔ Safe with Kafka retries

✔ Handles consumer crashes

✔ Guarantees idempotent processing

✔ Works well with Outbox Pattern

✔ Supports At-Least-Once delivery

---

# Disadvantages

✘ Extra Inbox table

✘ Additional database lookup

✘ Cleanup required

✘ Every event must contain a unique Event ID

---

# Common Interview Questions

## What problem does Inbox Pattern solve?

Duplicate event processing.

---

## Why does Kafka deliver duplicate messages?

Because Kafka provides At-Least-Once delivery.

If offset isn't committed, Kafka retries.

---

## What is stored in the Inbox table?

Usually

- Event ID
- Event Type
- Processed Timestamp

---

## Why must Inbox insert and business logic be in one transaction?

To ensure either both succeed or both fail.

Otherwise inconsistent states occur.

---

## When should Kafka offset be committed?

Only after

- Database transaction commits
- Business logic completes successfully

---

## Can Inbox guarantee Exactly-Once Delivery?

No.

It guarantees

> **Exactly-once processing from the application's perspective.**

Kafka may still deliver duplicates.

The application safely ignores them.

---

## Difference Between Idempotency and Inbox

Idempotency

```
Business logic itself is safe to execute multiple times.
```

Inbox

```
Detects duplicate events before processing.
```

Many systems use both.

---

# Interview Cheat Sheet

### Definition

> The Transactional Inbox Pattern ensures that an incoming event is processed only once from the application's perspective by storing the event ID in an Inbox table within the same database transaction as the business logic. If Kafka redelivers the event, the Inbox table detects it and prevents duplicate processing.

---

### Complete Flow

```
Kafka

↓

Receive Event

↓

Start DB Transaction

↓

Check Inbox

↓

Already Exists?

↓

YES -------------> Skip Business Logic

↓

Commit Offset

NO

↓

Insert Inbox Row

↓

Execute Business Logic

↓

Commit Database Transaction

↓

Commit Kafka Offset
```

---

### Remember

✅ Consumer-side pattern

✅ Solves duplicate processing

✅ Kafka provides At-Least-Once delivery

✅ Inbox + Business Logic = One Transaction

✅ Commit Offset AFTER DB Commit

✅ Commonly paired with Outbox Pattern

---

# One-Line Interview Answer

> "The Transactional Inbox Pattern prevents duplicate event processing by storing each processed event's ID in an Inbox table within the same database transaction as the business update. If Kafka redelivers the same event, the consumer detects the existing Inbox entry, skips the business logic, and safely commits the offset."