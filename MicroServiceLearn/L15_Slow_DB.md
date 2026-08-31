“DB query is slow” is one of the most important microservice production scenarios. Let’s learn it the same way as the previous scenarios: symptom → metrics → identify exact cause → fix.

Scenario

Suppose:

Client
↓
API Gateway
↓
Order Service
↓
MySQL

Suddenly users report:

        “Order API is taking 5–10 seconds.”

You check monitoring:

    `API latency ↑
    Traffic → normal
    CPU → normal
    Memory → normal`

Now you suspect the database/query path.

Step 1 — First prove that DB is actually responsible

Don't immediately say "DB is slow."

Go to APM distributed tracing.

Example trace:

    POST /orders
    |
    | 5200 ms
    |
    ├── Order Service     5150 ms
    │
    ├── DB query          4800 ms  ← problem
    │
    └── other processing   350 ms

Now we know:

    Most of the API latency is being spent waiting for the database.
    
    This is much stronger than simply looking at DB CPU.

Step 2 — Find WHICH query is slow

APM/database monitoring should help identify:

    Query:
    SELECT * FROM orders
    WHERE customer_id = ?
    ORDER BY created_at DESC;
    
    
    Execution time:
    4.8 seconds
    
    Now don't stop here.

Ask:

Why is this query taking 4.8 seconds?

There are several possible reasons.

Step 3 — Check the important DB metrics

Think of each metric as answering a specific question.

1. Query latency

        Question:
        
        Are queries actually becoming slower?
        
        Example:
        
        Normal:
        P95 query latency = 50 ms
        
        
        Now:
        P95 = 4.5 sec
        
        This confirms the database/query layer is experiencing latency.

2. DB CPU

Question:

    Is the database spending too much CPU processing queries?

Example:

    DB CPU
    Normal: 40%
    Now:    95%

Possible cause:

Too many queries
+
expensive queries
  +
full table scans

Potential solution:

    optimize query
    add appropriate indexes
    reduce unnecessary queries
    scale DB if workload genuinely increased

3. DB connections

Question:

    Are requests waiting for a database connection?

Example:

    Connection pool:
    Maximum = 100
    Active = 100
    Waiting = 50

This is important.

The query itself may not be slow.

The application might be waiting:

    Request
    ↓
    Wait for DB connection
    ↓
    Connection obtained
    ↓
    Query executes quickly

APM might show:

    Connection acquisition = 3 sec
    Query execution = 50 ms

Then the solution is not query optimization.

You investigate:

    connection pool configuration
    connection leaks
    long-running transactions
    DB capacity
    sudden traffic/concurrency increase

Step 4 — Check slow query characteristics

Suppose you find:

        SELECT *
        FROM orders
        WHERE customer_id = ?
        ORDER BY created_at DESC;

And the table contains:

orders = 100 million rows

Ask:

    Does the DB have an appropriate index?

For example:

    INDEX(customer_id, created_at)

    Without a suitable index, DB may need to examine huge numbers of rows.

Conceptually:

    100 million rows
    ↓
    scan/search
    ↓
    find matching customer
    ↓
    sort
    ↓
    return results

With a suitable index:

    Index
    ↓
    customer_id
    ↓
    created_at
    ↓
    required rows
    
    Much faster.

Step 5 — Use EXPLAIN

This is a very important interview answer.

Run:

    EXPLAIN
    SELECT *
    FROM orders
    WHERE customer_id = ?
    ORDER BY created_at DESC;

You look at things such as:

    which index is being used
    whether a full table scan is occurring
    number of rows examined
    join strategy
    sorting
    temporary tables

For example:

type = ALL
rows = 100,000,000

That is a major warning sign.

It may indicate a full table scan.

Step 6 — Check whether the query recently changed

    This is a production debugging mindset.

Suppose:

    10:00 AM
    Query latency = 50 ms


    10:30 AM
    Query latency = 5 sec

Ask:

What changed around 10:30?

Possibilities:

    New deployment
    Database schema change
    Index removed
    New query introduced
    Traffic pattern changed
    Data volume increased
    DB configuration changed

For example, developer changed:

SELECT id, name
FROM customers
WHERE email = ?;

to:

SELECT *
FROM customers
WHERE LOWER(email) = LOWER(?);

An existing index on email may no longer be used effectively depending on the DB/query design.

So:

Recent deployment + query latency spike = investigate the deployment/query change immediately.

Step 7 — Check locking

Another major scenario:

    Query itself normally = 50 ms


Now = 5 sec

You investigate and discover:

    Transaction A
    ↓
    locks row/table
    
    Transaction B
    ↓
    waiting for lock

So the query isn't computationally slow.

    It is waiting.

You check:

    lock waits
    long-running transactions
    deadlocks
    transaction duration

Example:

    Transaction A
    started: 10:00:00
    still running: 10:05:00

That can cause other queries to wait.

Step 8 — Check DB I/O

Suppose:

    DB CPU = 30%
    DB memory = normal
    DB query latency = high
    Disk I/O = very high

Then the bottleneck could be storage.

For example:

    Query
    ↓
    needs data from disk
    ↓
    high disk latency
    ↓
    query becomes slow

So don't assume:

"CPU is normal, therefore DB is healthy."


Step 10 — Finally decide the solution

The solution depends on what you found.

| Finding                  | Likely solution                          |
| ------------------------ | ---------------------------------------- |
| Missing index            | Add/optimize index                       |
| Full table scan          | Rewrite query/index                      |
| Expensive join           | Optimize query/index/schema              |
| Too many DB connections  | Tune pool / investigate leaks            |
| Connection waiting       | Fix pool/DB capacity                     |
| Lock contention          | Reduce transaction duration/fix locking  |
| Deadlocks                | Fix transaction/order/access pattern     |
| High DB CPU              | Optimize queries or scale                |
| High disk I/O            | Optimize queries / storage / DB capacity |
| Recent bad deployment    | Rollback/fix query                       |
| Huge result set          | Pagination / limit results               |
| Repeated identical reads | Consider caching                         |
| N+1 queries              | Batch/fetch efficiently                  |



You run:

        EXPLAIN
        SELECT *
        FROM orders
        WHERE customer_id = 10;

MySQL might return something like:

    type    possible_keys    key                 rows
    ALL     idx_customer     NULL                10,000,000

Now interpret it:

    type = ALL → full table scan
    possible_keys → indexes that could potentially help
    key = NULL → no index actually chosen
    rows = 10,000,000 → optimizer estimates it may examine ~10M rows
    The interview flow you should remember

If interviewer asks:

"A microservice API became slow because of database queries. How will you troubleshoot?"

Answer in this order:

1. Check API latency in monitoring
   ↓
2. Use APM distributed tracing
   ↓
3. Confirm DB call is consuming most latency
   ↓
4. Identify the exact slow query
   ↓
5. Check query execution time / slow query metrics
   ↓
6. Run EXPLAIN and check indexes/full scans
   ↓
7. Check DB CPU / memory / disk I/O
   ↓
8. Check DB connection pool and connection waits
   ↓
9. Check locks / long transactions / deadlocks
   ↓
10. Check recent deployment/schema/query changes
    ↓
11. Optimize query/index/transaction
    ↓
12. If workload genuinely exceeds capacity → scale DB
    The most important mental model

Don't memorize "slow DB → add index."

Instead think:

Where exactly is the time being spent?

It could be:

API
↓
Application processing
↓
Connection acquisition       ← maybe slow
↓
Query execution               ← maybe slow
↓
Lock waiting                  ← maybe slow
↓
Disk I/O                      ← maybe slow
↓
Result transfer               ← maybe slow

That distinction is what makes your production troubleshooting answer strong.