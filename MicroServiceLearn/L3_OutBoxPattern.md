# Outbox Pattern - Interview Notes

---

# Table of Contents

1. What is the Outbox Pattern?
2. Why is it Needed?
3. The Dual Write Problem
4. How Outbox Pattern Works
5. Step-by-Step Flow
6. Database Schema
7. Example
8. Failure Scenarios
9. Outbox + Saga Pattern
10. Polling vs CDC
11. Advantages
12. Disadvantages
13. Interview Questions
14. Interview Cheat Sheet

---

# What is the Outbox Pattern?

    The Outbox Pattern is a design pattern used in **microservices** to ensure **reliable event publishing**.

Instead of:

```
    Save Order
    ↓
    
    Publish Kafka Event
```

we do

```
    Save Order
    +
    
    Save Event in Outbox Table
    ↓
    
    Commit Transaction
    ↓
    
    Background Publisher
    
    ↓
    
    Publish Event
```

The event is stored safely inside the database before being sent to Kafka.

---

# Why is it Needed?

Suppose we do

```
Save Order

↓

Publish Kafka
```

What if Kafka is down?

```
Order Saved ✅

Kafka Publish ❌
```

Now

- Order exists
- Inventory never knows
- Payment never knows
- Saga never starts

This is called the **Dual Write Problem**.

---

# Dual Write Problem

We have two separate systems.

```
Database

Kafka
```

Writing to both is **not atomic**.

Possible situations:

```
Database Success

Kafka Failure
```

or

```
Kafka Success

Database Failure
```

Both lead to inconsistent systems.

---

# What Does Outbox Pattern Do?

Instead of writing to Kafka directly:

```
Transaction

Save Order

Save Outbox Event

Commit
```

Later

```
Publisher

↓

Read Outbox

↓

Publish Kafka

↓

Mark as SENT
```

---

# Architecture

```
               Client
                  |
                  v
            Order Service
                  |
        -----------------------
        |                     |
        v                     v
 Orders Table          Outbox Table
                               |
                               |
                     Background Publisher
                               |
                               v
                            Kafka
                               |
         +---------------------+----------------+
         |                                      |
         v                                      v
 Inventory Service                  Payment Service
```

---

# Step-by-Step Flow

## Step 1

Client sends

```
POST /orders
```

---

## Step 2

Start Transaction

```
BEGIN
```

---

## Step 3

Save Order

Orders Table

|OrderId|Status|
|-------|------|
|101|CREATED|

---

## Step 4

Save Event

Outbox Table

|EventId|Event|Status|
|-------|-----|------|
|1|OrderCreated|PENDING|

---

## Step 5

Commit

```
COMMIT
```

Now both are safely stored.

---

## Step 6

Publisher runs every few seconds.

```
SELECT *

FROM Outbox

WHERE Status='PENDING'
```

---

## Step 7

Publish

```
Kafka

Topic

order-events

Message

OrderCreated
```

---

## Step 8

Update

```
Status

PENDING

↓

SENT
```

or delete the row.

---

# Example

Transaction

```
Save Order

Save Outbox Event

Commit
```

Publisher

```
Read Pending

↓

Publish Kafka

↓

Update SENT
```

---

# Failure Scenarios

## Case 1

Database transaction fails

```
Save Order ❌

Rollback
```

Result

```
No Order

No Outbox Event

No Saga
```

This is correct.

---

## Case 2

Database committed

Kafka down

```
Order Saved ✅

Outbox Saved ✅

Kafka ❌
```

Result

```
Outbox Row = PENDING
```

Publisher retries.

Nothing is lost.

---

## Case 3

Service crashes after commit

```
Commit

↓

Crash
```

After restart

```
Read Outbox

↓

Publish
```

Still safe.

---

## Case 4

Publisher crashes

No issue.

Pending rows remain.

Next publisher instance continues.

---

# Outbox + Saga

```
Order

↓

Inventory

↓

Payment
```

Suppose

```
Order Saved

Inventory Reserved

Payment Completed
```

Payment event is first written to

```
Payment Outbox
```

Then published.

If Kafka is down,

```
PaymentCompleted

PENDING
```

Saga simply waits.

When Kafka returns,

Publisher publishes event.

Saga continues.

---

# Important

Outbox **does NOT** solve failed transactions.

Example

```
Payment Transaction Failed

Rollback
```

No Outbox Event exists.

Saga detects this using

- Timeout
- Retry
- Orchestrator

Outbox only guarantees

> **Committed events are never lost.**

---

# Polling vs CDC

## Polling

Publisher queries

```
SELECT *

FROM OUTBOX

WHERE STATUS='PENDING'
```

Advantages

- Easy
- Simple

Disadvantages

- Polling delay
- Extra DB queries

---

## CDC (Debezium)

Instead of polling,

Debezium watches database logs.

```
Database

↓

Transaction Log

↓

Debezium

↓

Kafka
```

Advantages

- Near real-time
- Highly scalable

Disadvantages

- More setup

---

# Advantages

✔ Reliable event publishing

✔ No lost events

✔ Works with Saga

✔ No distributed transaction

✔ Eventual consistency

✔ Retry support

✔ Crash recovery

---

# Disadvantages

✘ Extra Outbox table

✘ Background publisher required

✘ Cleanup needed

✘ Duplicate events possible

Consumers must be idempotent.

---

# Common Interview Questions

## What problem does Outbox solve?

Reliable event publishing after database commit.

---

## What is the Dual Write Problem?

Writing separately to the database and the message broker can leave them inconsistent if one succeeds and the other fails.

---

## Why not publish directly to Kafka?

If Kafka is unavailable after the database commit, the event is lost.

---

## Why is the Outbox row stored in the same transaction?

To ensure that either both the business data and the event are committed, or neither is.

---

## What happens if Kafka is down?

The Outbox row remains `PENDING`.

The publisher retries until it succeeds.

---

## What happens if the service crashes?

The Outbox row is already in the database.

After restart, the publisher resumes processing pending events.

---

## What if the transaction fails?

The transaction rolls back.

No Outbox row is created.

Saga never starts.

---

## Does Outbox guarantee exactly-once delivery?

No.

It guarantees that **committed events are eventually published**.

Consumers should be idempotent because duplicates can occur.

---

## Why is idempotency important?

The publisher may retry after failures, so the same event can be delivered more than once.

---

## Difference Between Saga and Outbox

| Saga | Outbox |
|------|---------|
| Coordinates distributed transactions | Reliably publishes events |
| Uses compensating actions | Prevents event loss |
| Handles business failures | Handles messaging failures |

---

# Interview Cheat Sheet

### Definition

> Outbox Pattern stores events in an Outbox table within the same database transaction as the business data. A background publisher later sends these events to the message broker, ensuring that no committed event is lost.

---

### Flow

```
Client

↓

Order Service

↓

Save Order

+

Save Outbox Event

↓

Commit

↓

Publisher

↓

Kafka

↓

Other Services
```

---

### Remember

✅ Solves Dual Write Problem

✅ Guarantees eventual delivery

✅ Uses background publisher

✅ Works with Saga

✅ Consumers must be idempotent

❌ Does NOT make transactions distributed

❌ Does NOT guarantee exactly-once delivery

❌ Does NOT solve failed local transactions

---

# One-Line Interview Answer

> "The Outbox Pattern solves the dual write problem by storing both business data and the event in the same database transaction. A background publisher reliably publishes pending events to the message broker after the transaction commits, ensuring that committed events are never lost even if the broker is temporarily unavailable."