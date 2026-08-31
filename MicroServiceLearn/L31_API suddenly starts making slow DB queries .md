API suddenly starts making slow DB queries

Suppose:

    GET /orders/123

Normally:

    Response time = 200 ms

Suddenly:

    Response time = 3–5 seconds

And we suspect the database query became slow.

The important thing is: don't immediately say "add an index." First prove where the slowness is.

1. First confirm the problem

I would first check monitoring/APM.

Look at:

    API latency — P50/P95/P99
    Request rate
    Error rate
    DB query latency
    DB connection pool
    CPU/memory

Number of DB calls

For example:

API P95:
    
    Before → 250 ms
    Now    → 4 seconds


DB query:
    
    Before → 50 ms
    Now    → 3.5 seconds

    Now we have evidence that the DB is actually contributing to the API latency.

Interview answer

    "First I would confirm the latency increase using metrics/APM and determine whether the time is actually being spent in the database rather than assuming the DB is the problem."

2. Identify the exact slow query

This is extremely important.

Suppose the API executes:

    SELECT *
    FROM orders
    WHERE customer_id = 100;

We need to know:

    Which SQL query is slow?
    How long does it take?
    How frequently is it executed?
    Is the same query suddenly slow?
    Are there multiple queries for one API request?

APM/database monitoring can help identify this.

For example:

    GET /orders/123
    |
    +---- SQL 1 → 20 ms
    |
    +---- SQL 2 → 3.2 sec  ← problem
    |
    +---- SQL 3 → 10 ms
    
    Now we focus on SQL 2.

3. Check whether the query itself changed

This is an often-missed point.

Maybe yesterday the application was executing:

    SELECT id, name, price
    FROM products
    WHERE id = ?;

But after a code change:

    SELECT *
    FROM products
    WHERE LOWER(name) = LOWER(?);

The query may have changed because of:

    new code deployment
    ORM/JPA change
    new filtering condition
    changed query parameters
    changed pagination
    new join

So I would check:

    Did a deployment happen?
    Did the SQL query change?
    Did the query parameters change?
4. Run EXPLAIN

Now we investigate how the database executes the query.

Example:

    EXPLAIN
    SELECT *
    FROM orders
    WHERE customer_id = 100;

We want to understand whether the DB is:

    Index lookup
    
    or:
    
    Full table scan

For example, if we see something conceptually like:

    type = ALL
    rows = 5,000,000

that is suspicious.

    It means the DB may be scanning a huge number of rows.

Instead, we would ideally see an indexed lookup such as:

    type = ref
    key = idx_customer_id
    rows = 20
5. Check indexes

Suppose our query is:

    SELECT *
    FROM orders
    WHERE customer_id = 100;

But there is no index:

    customer_id

The DB may have to scan:

    5 million rows

We may add:

    CREATE INDEX idx_orders_customer_id
    ON orders(customer_id);

But don't blindly add indexes.

We first check:

    Does an appropriate index already exist?
    Is the index actually being used?
    Is the query selective?
    Is the column suitable for indexing?
    Are we filtering on multiple columns?
    Is the query applying a function to the indexed column?

For example:

    WHERE LOWER(email) = 'abc@gmail.com'

may prevent a normal index on:

    email

from being used efficiently.

6. Check data growth

This is a very important production scenario.

Imagine:

    6 months ago
    orders table = 100,000 rows
    query = 20 ms
    Today
    orders table = 50 million rows
    query = 3 seconds

The application code hasn't changed.

    The query may have become slower because the data volume increased.

So I would check:

    Table size
    Number of rows
    Index size
    Query execution plan

This is why "it worked fine before" doesn't necessarily mean the query is still efficient.

7. Check database CPU

Suppose:

    DB CPU = 95%

and:

    DB request rate ↑

Then the database may simply be overloaded.

Possible causes:

    More traffic
    More expensive queries
    Missing indexes
    Large scans
    Too many concurrent queries
    Reporting queries
    Batch jobs

So we investigate the workload, not just one SQL query.

8. Check DB connections

This is another common interview follow-up.

Suppose:

    HikariCP:
    active = 10
    maximum = 10
    pending = 50

This tells us application threads are waiting for DB connections.

    The query itself might take:
    
    100 ms


Pool usage high
↓
Why are connections occupied?
↓
┌──────────────┬──────────────┬──────────────┐
↓              ↓              ↓
Slow queries   Long           Too much
transactions   traffic
↓              ↓              ↓
Optimize       Shorten        Reduce/
queries        transaction    scale

but the request experiences:

    2 seconds waiting for connection
    +
    100 ms query

    So the API looks like a slow DB API even though the actual SQL execution isn't the main problem.

You need to distinguish:

    Connection acquisition time
    ↓
    Query execution time
    ↓
    Result processing time

9. Check locking/blocking

        This is another very important production cause.

Suppose:

    Transaction A
    UPDATE orders SET status = 'PAID'
    WHERE id = 123;
    
    holds a lock.
    
    Then:
    
    Transaction B
    SELECT ...

or another update may have to wait depending on the DB/isolation/operation.

You might see:

    Query normally → 50 ms


Now → 5 seconds

    because it is waiting for a lock.

So we check:

    Active transactions
    Locks
    Blocked queries
    Long-running transactions
    Deadlocks

10. Check N+1 queries

Suppose your API requests one order.

You expect:

    1 query

But JPA generates:

    1 query → get orders


    +100 queries → get customers/products/etc.

So:

    Total DB calls = 101

The individual queries might each take only:

    20 ms

but:

    101 × 20 ms ≈ 2 seconds

So the issue isn't necessarily one slow SQL query.

It's too many queries.

This is why I would check:

    "How many DB queries are being executed for a single API request?"

11. Check network/database infrastructure

Suppose:

    SQL execution = 30 ms

but:

    API DB interaction = 2 seconds

Then we need to investigate things like:

    Network latency
    DB server health
    Connection establishment
    Database node problems
    Cloud DB infrastructure

Especially in microservices:

    API
    ↓
    Service
    ↓
    DB connection pool
    ↓
    Network
    ↓
    DB

The DB query isn't necessarily the only place latency can occur.

12. Check recent deployments/configuration changes

I would correlate the incident with:

    Deployment
    Configuration change
    Database migration
    Traffic increase
    Schema change
    Index removal/change
    Application version

For example:

    10:00 → deployment
    10:05 → P95 increases

That's a strong signal.

Or:

        10:00 → traffic increases 5x
        10:02 → DB CPU reaches 95%
        10:03 → query latency increases

Then traffic is likely contributing.



----------------------------------------------------------------------------------------------------------------------------------

| Root cause                | Possible fix                                  |
| ------------------------- | --------------------------------------------- |
| Missing index             | Add appropriate index                         |
| Wrong execution plan      | Query/index/statistics optimization           |
| Huge table                | Partitioning/archival/query optimization      |
| Too many DB calls         | Fix N+1/batch queries                         |
| DB CPU high               | Optimize queries / scale DB / reduce workload |
| Connection pool exhausted | Tune pool and investigate long queries        |
| Lock contention           | Optimize transactions/locking                 |
| Recent bad deployment     | Roll back/fix query                           |
| Traffic spike             | Caching/scaling/rate limiting                 |
| Large result set          | Pagination/projection                         |
| Network latency           | Investigate DB/network infrastructure         |


1. Missing Index :


First check these metrics

| Metric                      | What we're looking for                                               |
| --------------------------- | -------------------------------------------------------------------- |
| **API P95/P99 latency**     | Has API latency increased?                                           |
| **DB query latency**        | Is the SQL query itself taking longer?                               |
| **DB CPU**                  | Is DB CPU unusually high?                                            |
| **DB I/O**                  | Is the DB reading lots of data from disk?                            |
| **Rows examined / scanned** | Is the query scanning huge numbers of rows?                          |
| **DB connections**          | Are connections getting exhausted because queries are taking longer? |


then identify the exact sql query using distributed tracing

then run EXPLAIN

    EXPLAIN
    SELECT *
    FROM orders
    WHERE customer_id = 100;
    
    Suppose MySQL gives us something like:
    
    type = ALL
    key  = NULL
    rows = 5,000,000


then check exiting index and add appropriate index if doing full scan
then again run explain and verify in production



To avoid write overhead whie adding index

4. Batch writes

        If you're inserting thousands of records:

Instead of:

    INSERT
    INSERT
    INSERT
    INSERT
...

    use appropriate batch/bulk operations where supported.
    
    This can reduce overhead from many individual database operations.

5. Reconsider the read/write trade-off

Suppose:

Before index:


    Reads  → 3 seconds
    Writes → 5 ms

After:

    Reads  → 30 ms
    Writes → 8 ms

That's probably a great trade.

But if:

    Reads  → 3 seconds
    Writes → 500 ms

and the system is write-heavy, we need to reconsider the indexing strategy




2. Wrong execution plan


Simple example

You have 10 million orders.

Query:

    SELECT *
    FROM orders
    WHERE customer_id = 100;

There are two possible ways to find the orders.

Road 1 — Use the index
    
    customer_id index
    ↓
    find customer_id = 100
    ↓
    get matching rows
Road 2 — Scan the whole table
    
    orders table
    ↓
    check row 1
    check row 2
    check row 3
    ...
    check 10 million rows

    Obviously, you'd expect the DB to choose Road 1.
    
    But the DB has to estimate which road is cheaper.

Here's where the problem happens

Imagine the DB's statistics say:

    customer_id = 100
    matches approximately 10 rows

So it chooses:

    INDEX → 10 rows

Great.

    But the actual data has changed.

Now:

    customer_id = 100
    matches 5 million rows

The DB's information is outdated.

    It may make a poor decision.
    
    That's one way a bad execution plan can happen.
    
    Another very simple example
    
    Suppose there are 100 customers:
    
    Customer 1 → 100,000 orders
    Customer 2 → 100,000 orders
    ...

Almost every customer has lots of orders.

You ask:

    WHERE customer_id = 1

The index exists.

    But the index would lead to:

        Index
        ↓
        100,000 matching rows
        ↓
        Fetch 100,000 rows
        
        The database may decide:
        
        "That's a huge number of rows. Scanning the table may actually be cheaper."
        
        So it chooses a table scan.
        
        The index existing does NOT guarantee the DB will use it.
        
        So what exactly is a "bad execution plan"?

Suppose the DB chooses:

Full table scan

and it takes:

    5 seconds

But another available approach would have taken:

    200 ms

Then the chosen plan is bad for that query and current data.

Why does this happen?

Remember these 3 main reasons:

1. Statistics are stale
    
       Actual data changed
       ↓
       Statistics still represent old data
       ↓
       Optimizer makes wrong estimate
       ↓
       Chooses poor plan

2. Data distribution is different than expected

For example:

    customer_id = 100

might unexpectedly match 40% of the table.

The optimizer may decide an index isn't worthwhile.

3. Query/index structure isn't optimal

For example, your query might filter and sort:

    WHERE customer_id = 100
    ORDER BY created_at

but your available index only helps with:

    customer_id

The DB may still need to do significant additional work.



3. What does "Huge table" mean?

Suppose your orders table has:

        1 year     → 10 million rows
        3 years    → 100 million rows
        5 years    → 500 million rows

Now you run:

        SELECT *
        FROM orders
        WHERE customer_id = 100;

Even if you have an index, the database may still have a lot of work depending on the query and data distribution.

Or consider:

    SELECT *
    FROM orders
    WHERE order_date >= '2020-01-01';

If your table contains 500 million rows, this query could potentially touch a huge amount of data.

So:

    Huge table
    ↓
    Large amount of data to process
    ↓
    Query becomes expensive
    ↓
    High DB CPU / I/O
    ↓
    High query latency

2. What metrics do we check?

Same investigation pattern.

    Query latency
    Before: 100 ms
    Now:    4 seconds
    Rows examined
    100,000 → 50,000,000
    DB CPU
    40% → 90%
    DB I/O
    50 MB/s → 500 MB/s
    Table size / row count
    orders = 500 million rows

Then use:

    EXPLAIN ...

to see how much data the query is processing.

3. First solution: Query optimization

        Before partitioning or archiving, fix the query if possible.

Suppose someone writes:

    SELECT *
    FROM orders;

Why are we retrieving 500 million rows? 😄

Instead, perhaps:

    SELECT id, customer_id, amount
    FROM orders
    WHERE customer_id = 100
    LIMIT 100;

We reduce:

Amount of data read
+
Amount of data transferred
  +
Amount of data processed

Other query optimizations can include:

    proper indexes
    selecting only required columns
    pagination
    avoiding unnecessary joins
    avoiding unnecessary sorting
    filtering earlier

4. Second solution: Archival

Now imagine your business only actively uses the last 2 years of orders.

But the database has:

    500 million orders


    Active:
    2025-2026 → 100 million
    
    
    Old:
    2020-2024 → 400 million

Why keep all 500 million rows in the main operational table?

We could move old data to an archive.

Conceptually:

    orders
    ↓
    Recent data → Main DB
    Old data    → Archive storage/table
    
    For example:
    
    Main orders table
    2025-2026 → 100M rows
    
    
    Archive
    2020-2024 → 400M rows
    
    Now normal operational queries work against a much smaller dataset.
    
    Important
    
    Archival doesn't mean deleting data.
    
    It means:
    
    Move data that is rarely accessed out of the hot operational dataset while retaining it for historical needs.

5. Third solution: Partitioning

        Partitioning is different from archiving.

        With partitioning, the data can remain within the database/table structure but is divided into smaller partitions.

For example, partition orders by year:

orders


    Partition 2024 → 100M
    Partition 2025 → 150M
    Partition 2026 → 200M

Suppose the query is:

    SELECT *
    FROM orders
    WHERE order_date >= '2026-01-01';

The database can potentially use partition pruning:

500M total rows


    2024 partition ──┐
    2025 partition ──┤── don't need
    2026 partition ──┘── scan this

So instead of examining all partitions, it can focus on the relevant one(s).


    CREATE TABLE orders (
    id BIGINT NOT NULL,
    customer_id BIGINT,
    amount DECIMAL(10,2),
    order_date DATE,
    status VARCHAR(20),
    PRIMARY KEY (id, order_date)
    )
    PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p_future VALUES LESS THAN MAXVALUE
    );


|             | Partitioning             | Sharding                    |
| ----------- | ------------------------ | --------------------------- |
| Where?      | Inside one DB            | Across multiple DB servers  |
| Data split? | Into partitions          | Across databases            |
| Main goal   | Manage/query huge tables | Scale database horizontally |
| Example     | Orders split by year     | Customers split across DBs  |
| Complexity  | Lower                    | Higher                      |







DB CPU

Now imagine the database server itself.

It has CPU cores just like your application server.

For example:

Database Server
CPU: 8 cores
RAM: 32 GB

DB CPU tells us how much of the database server's CPU capacity is being used.

Example:

DB CPU


Before incident: 35%
During incident: 92%
How could this happen?

Suppose this query has no suitable index:

SELECT *
FROM orders
WHERE customer_id = 100;

There are:

10 million orders

The DB may have to examine a huge number of rows.

That requires more CPU work.

So you could see:

DB CPU = 95%
How is it shown?

In monitoring:

Database CPU Utilization


100% |                    ███
90% |                  █████
80% |                ███████
70% |             █████████
60% |
50% | █████
+-------------------------
Before        Incident

Or simply:

CPU utilization: 93%



DB I/O

This one is slightly more confusing.

I/O = Input/Output between the database and storage.

Think:

Database
↓
Reads data from disk/storage
↓
Processes it

Suppose your table contains:

10 million rows

and your query needs to scan a huge portion of the table.

The DB may need to read lots of data from storage.

That creates high disk I/O.

Example

Imagine:

SELECT *
FROM orders
WHERE customer_id = 100;

Without a suitable index:

DB
↓
Read many pages from storage
↓
Check customer_id
↓
Find matching rows

So you might see:

DB I/O


Before:    20 MB/s
Incident:  500 MB/s

Or in a cloud monitoring dashboard you might see metrics such as:

Read IOPS
Write IOPS
Read throughput
Write throughput
Disk latency

The exact names depend on the database/cloud provider.


1. Start with the query

Suppose:

SELECT *
FROM orders
WHERE customer_id = 100;

We run:

EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 100;

You may get something like:

+----+-------+--------+------+------------------+------+---------+------+---------+-------------+
| id | table | type   | key  | possible_keys    | rows | Extra   |
+----+-------+--------+------+------------------+------+---------+------+---------+-------------+
|  1 | orders| ALL    | NULL | idx_customer_id  | 5000000 |       |
+----+-------+--------+------+------------------+------+---------+------+---------+-------------+

Don't try to memorize the whole output.

For now, focus on 4 things:

type
possible_keys
key
rows
2. type — How is MySQL accessing the table?

This is one of the first things I look at.

You might see:

ALL

That means:

Full table scan

Example:

10 million rows
↓
MySQL checks rows
↓
1
2
3
4
...
10,000,000

🚨 This can be a problem for a large table.

Better access types can include:

const
eq_ref
ref
range

You don't need to memorize all of these immediately.

For our current missing-index scenario, remember:

ALL + large table = investigate immediately.

3. possible_keys

This tells us:

What indexes could potentially be useful for this query?

Example:

possible_keys = idx_customer_id

That means MySQL knows:

"There is an index that could potentially help."

But notice something important.

Potentially useful ≠ actually used.

4. key

This tells us:

Which index did MySQL actually choose?

Example:

possible_keys = idx_customer_id
key           = NULL

This is interesting.

It means:

An index could potentially help
↓
But MySQL didn't use it
↓
key = NULL

And if:

type = ALL

then:

Full table scan
+
No index used

That's a strong signal to investigate.

5. rows

This is extremely important.

Suppose:

rows = 5,000,000

MySQL estimates it will examine around 5 million rows.

That's concerning if your query should only return:

20 rows

For example:

Query:


WHERE customer_id = 100


Expected result:
20 orders


EXPLAIN:


rows = 5,000,000

That's a huge amount of work.

Put them together

Imagine:

type           = ALL
possible_keys  = idx_customer_id
key            = NULL
rows           = 5,000,000

We interpret it as:

"There is a potentially useful index, but MySQL isn't using it. It's doing a full table scan and expects to examine around 5 million rows."

Now we investigate why.

6. Compare with a healthier plan

After creating/using an appropriate index:

CREATE INDEX idx_customer_id
ON orders(customer_id);

Run:

EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 100;

You might now see:

type           = ref
possible_keys  = idx_customer_id
key            = idx_customer_id
rows           = 20

Now:

Before:


ALL
↓
key = NULL
↓
5,000,000 rows




After:


ref
↓
key = idx_customer_id
↓
20 rows

That's a much better execution plan.

7. But remember our earlier discussion

What if we see:

key = idx_customer_id

but the query is still slow?

Then we don't say:

"Index exists, so everything is fine."

We continue investigating.

For example:

key  = idx_customer_id
rows = 4,000,000

The index is being used, but it's still examining millions of rows.

Why?

Maybe:

customer_id = 100

matches a huge percentage of the table.

Or maybe the query has:

WHERE customer_id = 100
ORDER BY created_at

and additional work is required.



high CPU



1. What metrics do we check?

Suppose the API suddenly becomes slow.

We see:

API P95       = 3 seconds
DB CPU        = 95%

Now check:

Query latency
DB query P95 = 2.5 sec
Query count
Before = 1,000 queries/sec
Now    = 8,000 queries/sec
Rows examined
Before = 100K rows/sec
Now    = 50M rows/sec
DB I/O
Read I/O ↑
Connections
Active connections = 100/100

So we don't immediately say:

"Scale the DB."

First identify what is consuming the CPU.

2. Find the expensive queries

Suppose database monitoring shows:

Top queries by CPU


Query A → 60% DB CPU
Query B → 20%
Query C → 10%
Others → 10%

Now we investigate Query A.

For example:

SELECT *
FROM orders
WHERE customer_id = 100
ORDER BY created_at DESC;

Run:

EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 100
ORDER BY created_at DESC;

Maybe we discover:

Full table scan
+
Millions of rows examined
+
Large sort

That explains the CPU.

3. Why can a query cause high CPU?

Several possibilities.

A. Full table scan
10 million rows
↓
Check every row
↓
CPU work ↑

Potential solution:

Appropriate index
B. Expensive joins

Example:

SELECT ...
FROM orders o
JOIN customers c ...
JOIN products p ...

If the joins aren't efficient, the DB can perform a lot of processing.

Potential solution:

Optimize query
+
Appropriate indexes
C. Sorting huge amounts of data

Example:

SELECT *
FROM orders
ORDER BY created_at DESC;

If a huge number of rows need to be sorted, CPU can increase.

Potential solutions:

Better filtering
+
Appropriate index
+
Pagination
D. Too many queries

Suppose:

1 API request → 100 DB queries

and traffic increases:

1,000 requests/sec

Now:

100 × 1,000
= 100,000 DB queries/sec

DB CPU can become very high.

Potential solution:

Fix N+1
+
Batch queries
+
Reduce unnecessary DB calls
4. What if the queries are already optimized?

Suppose we investigate and find:

Indexes → good
Queries → good
Execution plans → good
No N+1

But:

DB CPU = 95%
Traffic = 5x normal

Now the problem may simply be too much workload for the current DB capacity.

Then we consider scaling.

5. Scaling the DB

There are two broad directions.

Vertical scaling

Make the DB server bigger:

8 CPU cores
↓
32 CPU cores

More CPU/RAM can handle more workload.

Horizontal scaling

Depending on the database architecture/workload, use things like:

Read replicas
Sharding

For example:

                 Application
                     |
              ┌──────┴──────┐
              ↓             ↓
          Primary       Read Replica
          (writes)       (reads)

If the workload is read-heavy, read replicas can help distribute reads.

6. Reduce workload

Instead of immediately scaling, we can also reduce what we're asking the DB to do.

For example:

Caching

If the same product is requested thousands of times:

Without cache:


1000 requests
↓
1000 DB queries

With cache:

1000 requests
↓
Cache
↓
Only a few DB queries

Other options:

Fix N+1
Batch queries
Optimize expensive queries
Reduce unnecessary polling
Rate-limit excessive traffic




1. What does "pool exhausted" mean?

Suppose all 10 connections are currently being used:

Connection 1 → query running
Connection 2 → query running
Connection 3 → query running
...
Connection 10 → query running

Now request #11 arrives.

It needs a DB connection.

But:

No connection available
↓
Request waits

If the connection isn't returned within the configured timeout:

Connection timeout

and the API can fail or become very slow.

2. What metrics do we look at?

Since you're using HikariCP, these are especially useful.

hikaricp_connections_active

Example:

active = 10

Your application is currently using 10 connections.

hikaricp_connections_max
max = 10

So:

active = 10
max = 10

🚨 Pool is completely occupied.

hikaricp_connections_pending

This is extremely useful.

Suppose:

active  = 10
max     = 10
pending = 50

That means:

10 connections are busy
+
50 threads/requests are waiting for a connection

🚨 Strong indication of pool contention/exhaustion.

Connection acquisition time

If available through your monitoring setup, check how long requests spend waiting to obtain a connection.

For example:

Connection wait = 2 seconds
SQL execution   = 50 ms

This is very different from:

Connection wait = 10 ms
SQL execution   = 2 seconds

In the first case, the pool is the problem, not the query itself.

3. Why does the pool become exhausted?

This is the important question.

Cause 1 — Slow DB queries

Suppose:

Pool = 10 connections

Normally:

Query = 50 ms

Connections return quickly.

But suddenly:

Query = 5 seconds

Now each connection stays occupied for 5 seconds.

Connection 1 → waiting
Connection 2 → waiting
...
Connection 10 → waiting

Pool gets exhausted.

So:

A connection pool problem can actually be caused by a slow query.

That's why the table says:

Connection pool exhausted
↓
Investigate long queries
4. Cause 2 — Too many concurrent requests

Suppose:

Pool = 10

and traffic suddenly increases:

100 requests/sec
↓
many requests need DB
↓
10 connections occupied
↓
remaining requests wait

Even if every query is reasonably fast, the pool can become a bottleneck under high concurrency.

5. Cause 3 — Connection leak

This is more serious.

Suppose application code obtains a connection but doesn't properly release it.

Then:

Connection 1 → never returned
Connection 2 → never returned
...

Eventually:

10 / 10 connections unavailable

and the pool is exhausted.

With Spring/JPA, connection management is normally handled for you, so leaks are less common when the application is using the framework correctly, but they can still occur with incorrect manual JDBC/resource handling or transaction problems.

6. How do we investigate?

Suppose monitoring shows:

hikaricp_connections_max     = 10
hikaricp_connections_active  = 10
hikaricp_connections_pending = 40

We don't immediately increase the pool to 50.

First ask:

Why are all 10 connections busy?

Then check:

1. DB query latency
2. Long-running queries
3. DB CPU
4. DB connections
5. Transaction duration
6. Connection acquisition time
7. Recent traffic increase
7. Example investigation

Suppose we find:

API P95 = 4 sec


Hikari:
active  = 10
max     = 10
pending = 40


DB query:
P95 = 3.5 sec

Now the relationship is:

Slow DB queries
↓
Connections remain occupied
↓
Pool reaches maximum
↓
Other requests wait
↓
API latency increases
↓
Connection timeout

So the real fix may be:

Optimize slow query

rather than:

Increase pool size
8. What does "tune the pool" mean?

Suppose we have:

spring.datasource.hikari.maximum-pool-size=10

We could potentially increase it:

spring.datasource.hikari.maximum-pool-size=20

But only after checking DB capacity.

Because this:

Pool = 10 → 20

doesn't magically create more database capacity.

You could instead turn:

10 concurrent DB queries

into:

20 concurrent DB queries

and overload MySQL.

So pool size must be chosen based on:

Application concurrency
+
DB capacity
+
Query duration
+
Number of application instances
9. Very important in microservices

Suppose you have:

3 application instances

and each has:

maximum pool = 20

Your DB could potentially receive:

3 × 20 = 60 connections

So don't look at only one application's Hikari pool.

Think:

App instance 1 → 20 connections
App instance 2 → 20 connections
App instance 3 → 20 connections
↓
MySQL
60 possible

This becomes very important when scaling horizontally.


Lock contention

        First: what is a lock?

    A database uses locks to control concurrent access to the same data.

    Imagine two requests try to update the same order:

    Request A                    Request B
    |                            |
    | UPDATE order 123           |
    ↓                            |
    DB locks row 123                |
    |                            |
    |                        UPDATE order 123
    |                            ↓
    |                       Has to wait
    ↓
    Commit
    |
    Lock released
    ↓
    B can continue

Request B is waiting for the lock held by A.

That waiting is lock contention.

1. Simple example

Suppose:

-- Transaction A


    BEGIN;
    
    
    UPDATE orders
    SET status = 'PAID'
    WHERE id = 123;

Transaction A has modified/locked the row.

Now another transaction:

-- Transaction B


    BEGIN;
    
    
    UPDATE orders
    SET status = 'CANCELLED'
    WHERE id = 123;

Transaction B wants the same row.

But A hasn't committed yet.

So:

    A → owns lock
    B → waiting

If A takes 5 seconds to commit:

B waits ~5 seconds
2. Why does this make an API slow?

Imagine:

    Request A
    ↓
    UPDATE order 123
    ↓
    holds lock for 5 sec

Meanwhile:

    Request B → waits
    Request C → waits
    Request D → waits

So you might see:

    API latency ↑
    DB query latency ↑
    DB connections active ↑
    Connection pool pending ↑

The SQL itself may normally execute in:

    20 ms

but now:

    Query total time = 5 seconds

because most of that time is waiting for a lock.

3. What metrics do we check?

        For lock contention, the most useful things are:

        Lock wait time

Example:

    Lock wait:
    Normal → 0–5 ms
    Incident → 3 seconds

This tells us queries are waiting for locks.

Blocked queries

You might see:

    Query A → running
    Query B → waiting
    Query C → waiting
    Query D → waiting

The important question becomes:

Who is blocking these queries?

Transaction duration

Suppose:

    Transaction A:
    Started → 10:00:00
    Committed → 10:00:20
    
    It held resources/locks for a long time.
    
    Long transactions are a common contributor to lock contention.
    
    DB query latency
    Before → 30 ms
    Now    → 3 sec
    
    But this alone doesn't tell us it's locking.
    
    We correlate it with lock waits.
    
    DB connections / HikariCP
    
    You might see:
    
    active      = 10
    max         = 10
    pending     = 30
    
    because connections are sitting around waiting for blocked queries to finish.

4. How do we actually investigate?

Suppose:

    API P95 = 4 seconds
    DB query P95 = 3.8 seconds
    DB CPU = 40%

Interesting.

    CPU isn't high.
    
    So why is the DB query taking 4 seconds?
    
    We investigate lock waits.

We discover:

    Transaction A
    ↓
    holds lock on order 123
    ↓
    Transaction B
    ↓
    waiting 3.5 sec

Now we have the root cause.

5. What causes lock contention?
   Cause 1 — Long transactions

Example:

    BEGIN
    ↓
    UPDATE
    ↓
    call another service
    ↓
    do processing
    ↓
    more DB work
    ↓
    COMMIT

The transaction stays open for too long.

Bad pattern:

        DB lock
        ↓
        external API call
        ↓
        wait 3 seconds
        ↓
        commit

The lock may be held while waiting for something unrelated.

Cause 2 — Many requests updating the same row

Imagine:

    100 requests
    ↓
    UPDATE account
    SET balance = ...
    WHERE id = 1

All are competing for the same row.

That's heavy contention.

Cause 3 — Large transactions

Suppose one transaction updates:

1 million rows

and holds locks while doing so.

Other transactions can be blocked.

6. How do we fix it?
   Fix 1 — Keep transactions short

Instead of:

    BEGIN
    ↓
    DB update
    ↓
    external API call
    ↓
    long processing
    ↓
    COMMIT

try to avoid holding a DB transaction while doing unrelated slow work.

Conceptually:

    Do external work
    ↓
    Short DB transaction
    ↓
    Commit
Fix 2 — Optimize the query

If the transaction does:

    UPDATE orders
    SET status = 'PAID'
    WHERE customer_id = 100;

and that updates millions of rows, it can create a lot of locking.

We investigate:

    EXPLAIN
    indexes
    query design
    batch size
Fix 3 — Reduce transaction scope

Instead of:

    One huge transaction
    ↓
    1 million rows
    
    depending on business requirements, process smaller batches.

Fix 4 — Handle concurrency correctly

Sometimes multiple requests legitimately need to modify the same record.

    Then we may use mechanisms such as:
    
    Optimistic locking
    Pessimistic locking
    
    For example, with optimistic locking:
    
    Order version = 5
    
    
    Request A → version 5 → update → version 6
    Request B → version 5 → update fails because version changed
    
    This avoids silently overwriting another transaction's changes.

7. Lock contention vs deadlock

Don't confuse these.

    Lock contention
    A → holds lock
    B → waits

Eventually B can proceed.

    Deadlock
    A → locks Row 1
    A → waits for Row 2
    
    
    B → locks Row 2
    B → waits for Row 1

Now:

    A waits for B
    B waits for A
    
    Neither can proceed.

The DB detects the deadlock and usually aborts one transaction.



1. Optimistic locking

        Imagine two users open the same order.

Initially:

    Order 123
    status = PENDING
    version = 5
    User A

Reads:

    version = 5
    User B

Also reads:

    version = 5

Now A updates:

    UPDATE orders
    SET status = 'PAID',
    version = 6
    WHERE id = 123
    AND version = 5;

A succeeds.

Now B tries:

    UPDATE orders
    SET status = 'CANCELLED',
    version = 6
    WHERE id = 123
    AND version = 5;

But the database now has:

    version = 6

So:

    WHERE version = 5

doesn't match.

B's update affects 0 rows.

We detect:

    "Someone else changed this record."

That's optimistic locking.

2. Why is it called optimistic?

Because we assume:

    "Most of the time, two transactions won't conflict."

    So we don't lock the row when reading it.
    
    We only detect a conflict when updating.

3. JPA example

In Spring Boot/JPA you commonly see:

    @Entity
    class Order {
    
    
        @Id
        private Long id;
    
    
        private String status;
    
    
        @Version
        private Long version;
    }

The:

    @Version
    
    field is used by Hibernate for optimistic locking.
    
    Conceptually Hibernate generates something like:
    
    UPDATE orders
    SET status = ?, version = ?
    WHERE id = ?
    AND version = ?;

If somebody already changed it, the update fails with an optimistic locking exception.

4. When would I use optimistic locking?

Use it when:

    Conflicts are relatively rare

For example:

    Product information
    User profile
    Order details
    Booking information

where multiple users might update the same record, but usually don't simultaneously.

It is also good when you don't want transactions sitting around holding DB locks.

5. Pessimistic locking

Now imagine a different situation.

You have:

    Bank account balance = ₹10,000

    Two requests simultaneously try to withdraw money.

You really don't want both requests modifying the balance at the same time.

So transaction A says:

    "I need this row. Lock it."

Conceptually:

    SELECT *
    FROM account
    WHERE id = 1
    FOR UPDATE;

Now:

    Transaction A
    ↓
    Locks account row
    ↓
    updates balance
    ↓
    COMMIT
    ↓
    releases lock
    
    Transaction B tries to access the same locked row:
    
    Transaction B
    ↓
    FOR UPDATE
    ↓
    WAIT
    
    After A commits:
    
    Lock released
    ↓
    B gets lock
    ↓
    B continues
    
    That's pessimistic locking.

6. Why is it called pessimistic?

Because we assume:

    "A conflict is likely, so I'll lock the data before modifying it."

7. When would I use pessimistic locking?

Use it when:

    Conflicts are frequent and correctness requires serializing access.

Examples:

    Inventory
    
    Suppose:
    
    Stock = 1
    
    Two users try to purchase the last item simultaneously.
    
    You may need strong control over that row.
    
    Financial/account operations
    
    For certain balance-changing operations where concurrent updates need strict serialization.
    
    Highly contended records
    
    If many transactions constantly modify the same row, optimistic retries could become expensive.

8. The biggest difference

|                     | Optimistic                          | Pessimistic                |
| ------------------- | ----------------------------------- | -------------------------- |
| Assumption          | Conflicts are rare                  | Conflicts are likely       |
| Lock while reading? | ❌ Usually no                        | ✅ Yes, where applicable    |
| Conflict handling   | Detect conflict later               | Prevent/serialize conflict |
| Performance         | Usually better under low contention | Can reduce concurrency     |
| Risk                | Updates may fail/retry              | Waiting/blocking           |
| Example             | `@Version`                          | `SELECT ... FOR UPDATE`    |



Scenario

Suppose:

Account balance = ₹10,000

Two requests arrive at almost the same time.

Request A → reads ₹10,000
Request B → reads ₹10,000

A adds ₹1,000:

₹10,000 + ₹1,000 = ₹11,000

B adds ₹500:

₹10,000 + ₹500 = ₹10,500

If both simply write their result:

A → writes ₹11,000
B → writes ₹10,500

Final balance:

₹10,500

But it should have been:

₹11,500

A's update was lost.

That's a lost update.

1. How do you handle concurrent updates?

There isn't one single answer.

You first determine whether multiple requests can modify the same data concurrently and how important the conflict is.

Common approaches are:

Concurrent updates
↓
┌─────┼──────────┐
↓     ↓          ↓
Optimistic   Pessimistic   Atomic DB operation
locking      locking
Approach 1 — Optimistic locking

Use a version field.

Account:


balance = 10000
version = 5

A reads:

balance = 10000
version = 5

B also reads:

balance = 10000
version = 5

A updates:

UPDATE account
SET balance = 11000,
version = 6
WHERE id = 1
AND version = 5;

✅ A succeeds.

B tries:

UPDATE account
SET balance = 10500,
version = 6
WHERE id = 1
AND version = 5;

❌ No row matches because version is now 6.

So B knows:

"Someone modified this record after I read it."

B can then:

reload latest value
↓
recalculate
↓
retry

or return a conflict to the user.

Approach 2 — Pessimistic locking

If conflicts are frequent, you can lock the row.

SELECT *
FROM account
WHERE id = 1
FOR UPDATE;

Now:

Request A
↓
locks row
↓
updates
↓
commit
↓
releases lock


Request B
↓
waits
↓
gets lock
↓
reads latest value
↓
updates

So the updates happen sequentially.

Approach 3 — Atomic database operation

Sometimes you don't even need to read and then write.

Instead of:

READ balance
↓
calculate
↓
WRITE balance

you can do:

UPDATE account
SET balance = balance + 1000
WHERE id = 1;

The database performs the update atomically.

If another request does:

UPDATE account
SET balance = balance + 500
WHERE id = 1;

both increments can be applied safely under the database's concurrency mechanisms.

Final:

₹11,500

This is often a very good solution for simple counter/increment operations.

2. How do you prevent lost updates?

This question is more specific.

You can say:

"I would prevent lost updates using optimistic locking with a version column, pessimistic locking where contention is high, or atomic database operations when the update can be expressed safely as a single SQL operation."

In Spring Boot/JPA

The most common interview answer is:

@Version
private Long version;

For example:

@Entity
class Account {


    @Id
    private Long id;


    private BigDecimal balance;


    @Version
    private Long version;
}

Hibernate uses the version to detect concurrent modifications.

Conceptually:

UPDATE account
SET balance = ?,
version = ?
WHERE id = ?
AND version = ?;

If another transaction already changed it:

updated rows = 0

Hibernate detects the optimistic locking conflict.