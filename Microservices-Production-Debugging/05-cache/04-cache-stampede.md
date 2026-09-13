# Problem

A cache stampede, also called a dogpile, occurs when many requests independently rebuild the same missing or expired value.
One miss is normal.
Thousands of concurrent misses for the same expensive key are an incident.
Common triggers are:

* A hot key expires.
* Many keys share the same TTL and expire together.
* A deployment introduces a cold key version.
* Redis restarts or fails over with a cold dataset.
* Invalidation deletes a popular key just before peak traffic.
* A slow refresh outlives the arrival interval, so requests pile up.

The cache is not necessarily slow.
It may answer every miss in 2 ms while the database and application collapse under duplicate rebuild work.
The goal is to coalesce work, spread expiration, refresh safely, and bound fallback.

# Production Situation

At 12:00:00, the daily promotions key expires:

* Endpoint traffic is 8,000 requests/min, about 133 requests/sec.
* The single key `promotions:v7:tenant-42:home` receives 2,400 requests/min.
* Normal hit ratio is 97%.
* Redis GET p99 stays at 3 ms.
* Rebuild query normally takes 650 ms.
* During one 650 ms rebuild window, about 26 same-tenant requests arrive.
* Across 80 tenants whose keys share the noon TTL, 1,900 rebuilds start within two seconds.
* DB calls rise from 240/min to 6,700/min.
* Hikari active reaches 40/40; pending reaches 214.
* DB CPU rises from 34% to 91%.
* API p99 rises from 220 ms to 6.4 s.
* Retry attempts add another 2,100 DB queries/min.
* Redis remains under 30% CPU.

The incident repeats daily because all entries are assigned an absolute expiration at noon.
The symptom is periodic, sharply aligned, and dominated by a few hot namespaces.

# Architecture

```text
                           +--> Redis GET: miss
                           |
Clients -> Gateway -> Promotion Service
                           |
                           +--> per-key single-flight coordinator
                                      |
                         one leader --+--> HikariCP -> MySQL
                         waiters ------+--> shared result
                                      |
                                      +--> Redis SET with TTL+jitter
```

Without single-flight, every waiter queries MySQL.
With it, one in-process leader rebuilds a key while local waiters share the result.
Cross-instance coordination, if required, needs carefully designed leases and fencing; a Redis lock alone is not a universal correctness guarantee.

# What I Check FIRST

1. **Time shape and key concentration.** I compare miss spikes with exact timestamps and top hashed key patterns. A sharp periodic spike suggests synchronized TTL or scheduled invalidation.
2. **Duplicate work per key.** I count concurrent rebuilds and DB queries for the same key. Many rebuilds for one key distinguish a stampede from broad organic misses.
3. **Redis versus fallback latency.** Fast Redis misses plus slow DB/pool spans mean Redis is not the bottleneck.
4. **Expiration, eviction, and invalidation.** I correlate misses with `expired_keys`, `evicted_keys`, delete events, deployments, and key-version changes.
5. **Amplifiers and protections.** I check retries, single-flight leaders/waiters, lock acquisition, stale serving, fallback concurrency, and DB pool saturation.

# Step-by-Step Investigation

### Step 1 - Confirm the incident shape

* **What I check:** Request, hit, miss, DB call, and latency series at one-second or fine-grained resolution.
* **Why:** Five-minute averages can hide a ten-second dogpile.
* **Expected:** Ordinary misses are dispersed.
* **Bad:** Misses and DB calls spike exactly at 12:00:00 every day.
* **Meaning:** Scheduled expiry or invalidation is likely.
* **Next:** Break misses down by key fingerprint and tenant.

### Step 2 - Identify hot keys without exposing data

* **What I check:** Key pattern, namespace, and one-way fingerprint with request count.
* **Why:** A 60% hit ratio across millions of keys differs from one key receiving 40 requests/sec.
* **Expected:** No single key consumes a dangerous share unless specifically designed.
* **Bad:** One promotions key is 30% of endpoint traffic.
* **Meaning:** Expiry of this key creates high concurrent rebuild demand.
* **Next:** Measure rebuild duration and arrival rate.

### Step 3 - Estimate duplicate concurrency

* **What I check:** Request arrival rate for a key multiplied by rebuild seconds.
* **Calculation:** `40 requests/sec * 0.65 sec = about 26 concurrent arrivals`.
* **Why:** This predicts duplicate rebuilds before queueing makes them slower.
* **Bad:** Rebuild latency rises to 3 seconds, producing about 120 arrivals for that one key.
* **Meaning:** The incident has positive feedback: overload slows rebuilds, creating more overlap.
* **Next:** Compare actual leaders and rebuild attempts.

### Step 4 - Prove duplicate rebuilding

* **What I check:** `cache_rebuild_total`, in-flight rebuild gauge, DB query key fingerprint, and trace groups.
* **Why:** A miss spike alone could be many unrelated cold keys.
* **Expected:** At most one rebuild per key per coordination scope.
* **Bad:** 26 simultaneous traces execute the same parameterized query for one key.
* **Meaning:** Requests are not coalesced.
* **Next:** Find whether single-flight is absent, bypassed, or scoped incorrectly.

### Step 5 - Correlate expiration counters

* **What I check:** Delta of `expired_keys` and TTL samples before the event.
* **Why:** Expiration removes keys because their lifetime ended.
* **Expected:** Expirations are distributed across time.
* **Bad:** Thousands of promotion keys reach zero at exactly noon.
* **Meaning:** TTL alignment is the trigger.
* **Next:** Inspect how expiry timestamps are computed.

### Step 6 - Distinguish eviction

* **What I check:** `evicted_keys`, memory/maxmemory, and policy.
* **Why:** Eviction can also create repeated rebuilds, but it follows memory pressure rather than a scheduled clock.
* **Expected:** Evictions remain zero in this incident.
* **Bad:** Evictions rise continuously near maxmemory and hot rebuilt values are removed again.
* **Meaning:** Capacity, cardinality, or policy causes churn.
* **Next:** Fix the working set/policy and still keep rebuild protection.

### Step 7 - Check explicit invalidation and deployment

* **What I check:** `DEL`/unlink application metrics, event-consumer offsets, deployments, and key version.
* **Why:** A broad invalidation or new prefix creates the same cold effect as expiration.
* **Expected:** Invalidation targets changed entities and is rate-bounded.
* **Bad:** A catalog event invalidates every tenant key, or v8 deploys with no warm-up.
* **Meaning:** The trigger is application workflow rather than Redis TTL.
* **Next:** Correct invalidation granularity or rollout.

### Step 8 - Inspect TTL calculation and alignment

* **What I check:** TTL-at-write histogram and a bounded sample of known keys.
* **Why:** Setting all entries to "next noon" aligns expiration.
* **Expected:** Base TTL plus random jitter, for example 30 minutes plus 0-6 minutes.
* **Bad:** Every key has an expiry timestamp of `12:00:00`.
* **Meaning:** Deterministic expiry creates a predictable synchronized wave.
* **Next:** Add bounded jitter consistent with freshness rules.

```text
effective_ttl = base_ttl + random(0, jitter_window)
```

Jitter spreads work; it does not reduce total refresh cost.
Do not add jitter beyond the maximum acceptable staleness.

### Step 9 - Evaluate in-process single-flight

* **What I check:** Leader count, waiter count, key equality, timeout, cancellation, and cleanup.
* **Why:** Single-flight lets one request rebuild while peers await the same future.
* **Expected:** One leader per key per instance; waiters share success or bounded stale fallback.
* **Bad:** The map key omits tenant or entries never remove after failure.
* **Meaning:** Cross-tenant correctness risk or permanent poisoning.
* **Next:** Use full tenant/version key and remove coordination entries in `finally`.

Conceptual Java flow:

```text
value = cache.get(key)
if value exists and is fresh: return value
future = inFlight.computeIfAbsent(key, startOneRefresh)
try: return future within remaining deadline
finally: remove only the completed matching future
```

The leader must have a bounded timeout.
Waiters must not wait beyond their caller deadline.
Failures should not be cached as permanent values.

### Step 10 - Understand single-flight scope

* **What I check:** Number of service replicas and requests per key per replica.
* **Why:** An in-memory coordinator deduplicates only within one JVM.
* **Expected:** With 20 pods, worst case is about 20 rebuilds, often acceptable.
* **Bad:** Even one rebuild per pod overloads the authoritative store.
* **Meaning:** Cross-instance coordination or scheduled refresh may be required.
* **Next:** Compare safer options before adding a distributed lock.

### Step 11 - Consider early refresh

* **What I check:** Refresh-ahead window, key popularity, refresh success, and stale allowance.
* **Why:** A hot key can be refreshed before hard expiry while the old value remains usable.
* **Expected:** One background refresh begins near expiry and ordinary readers receive the current value.
* **Bad:** Every reader independently launches early refresh.
* **Meaning:** Refresh-ahead itself becomes a stampede.
* **Next:** Combine early refresh with single-flight and a concurrency budget.

A probabilistic early-refresh policy spreads refresh starts instead of using one sharp threshold.
It must still honor maximum data age and avoid refreshing cold keys wastefully.

### Step 12 - Use stale-while-revalidate only when correct

* **What I check:** Fresh-until time, stale-until time, data class, and invalidation requirements.
* **Why:** Serving a slightly stale promotion display may be safe while one leader refreshes.
* **Expected:** Stale age is bounded and visible in metrics.
* **Bad:** Permissions, revocations, prices, or inventory are served beyond their safe age.
* **Meaning:** Availability optimization violates correctness.
* **Next:** Fail closed or use an authoritative bounded read for sensitive data.

### Step 13 - Warm deliberately

* **What I check:** Known hot-key list, DB headroom, warm rate, concurrency, and success.
* **Why:** Preloading critical keys before a release or peak avoids first-request latency.
* **Expected:** Warming is prioritized and rate-limited below spare DB capacity.
* **Bad:** A job warms every possible key concurrently.
* **Meaning:** The cure is itself a cache stampede.
* **Next:** Stop the job and warm only measured hot keys with backpressure.

Warming complements, but does not replace, single-flight.
Unexpected keys and recovery paths still need safe miss behavior.

### Step 14 - Bound fallback

* **What I check:** Semaphore/bulkhead permits, queue size, timeout, rejected count, and DB capacity.
* **Why:** A bounded number of misses may use MySQL; excess callers should receive stale data, omission, or fast overload response.
* **Expected:** Fallback concurrency stays under the tested DB budget.
* **Bad:** Every miss occupies a servlet thread and Hikari waiter.
* **Meaning:** Application queueing moves pressure into the pool.
* **Next:** Enforce admission before acquiring DB connections.

Increasing Hikari from 40 to 200 is not a stampede fix.
It can turn an application queue into 200 concurrent DB queries and worsen DB latency.

### Step 15 - Remove retry amplification

* **What I check:** Retry attempts on Redis miss, DB timeout, lock timeout, and rebuild failure.
* **Why:** A cache miss is not an error to retry; parallel retries duplicate work.
* **Expected:** One admitted rebuild has a bounded policy.
* **Bad:** Each of 1,900 rebuilds retries twice.
* **Meaning:** 1,900 requests can become 5,700 or more DB attempts.
* **Next:** Retry only a proven transient idempotent failure with backoff, jitter, and deadline awareness.

### Step 16 - Evaluate distributed locks honestly

* **What I check:** Lease duration, owner identity, renewal, network partitions, pause time, and unlock compare-and-delete behavior.
* **Why:** A fixed lease can expire while a slow rebuild continues.
* **Bad sequence:**

```text
Pod A gets lock with fencing token 105
Pod A pauses for GC beyond lease
Pod B gets token 106 and writes new value
Pod A resumes and writes an older value
```

* **Meaning:** Lease ownership is stale even though A once acquired the lock.
* **Next:** Use fencing tokens accepted by the authoritative write, or make writes version-conditional.

Unlock must compare the unique owner token before delete.
A lock reduces duplicate work but can introduce a new availability dependency.
Do not hold the lock across unrelated network calls.
If duplicate reads are harmless, per-instance single-flight plus DB bulkhead may be simpler and safer.

### Step 17 - Protect invalidation ordering

* **What I check:** Database commit, invalidation event, cache delete/write, and concurrent read ordering.
* **Why:** A reader can repopulate old data after invalidation.
* **Problem sequence:**

```text
Reader loads old DB value
Writer commits new DB value
Writer deletes cache
Reader writes old value into cache
```

* **Meaning:** The cache can remain stale for a full TTL.
* **Next:** Use versioned values, conditional cache writes, short safe TTLs, or an outbox-driven invalidation design.

An outbox stores the domain change and invalidation event in the same database transaction.
A relay publishes the event reliably.
Consumers must be idempotent and compare entity versions so an old event cannot overwrite a newer cache value.
Write-through reduces some windows but still needs partial-failure and ordering design.
Write-behind is risky for authoritative business writes because loss or reordering can corrupt data.

### Step 18 - Check hot-key distribution on Redis

* **What I check:** Per-key application request estimates, shard CPU/network, and value size.
* **Why:** One hot key may overload one cluster shard even when aggregate capacity is free.
* **Expected:** Load is within one shard's capacity and replicated reads meet consistency needs.
* **Bad:** One shard has 90% CPU while others have 20%.
* **Meaning:** Aggregate Redis metrics hide a hot shard/key.
* **Next:** Use local L1 caching for safe immutable data, replicate reads appropriately, or split the data model.

Artificial key salting can distribute reads but complicates invalidation and multiplies memory.
It is not appropriate when strong freshness is required.

### Step 19 - Mitigate the live incident

* Enable bounded stale-while-revalidate for approved promotion display data.
* Admit one rebuild per key per JVM and cap total rebuilds.
* Disable retries on misses and pool timeouts.
* Rate-limit the endpoint or omit optional promotion enrichment.
* Roll back the synchronized TTL deployment if possible.
* Do not flush the cache; that creates a global stampede.
* Do not launch an unlimited warming job.

### Step 20 - Verify the causal hypothesis

* Apply TTL jitter and single-flight to a canary.
* Observe leaders, waiters, DB calls, Hikari pending, and p99 through the former noon boundary.
* Confirm data freshness and tenant isolation.
* Compare canary with unchanged instances at equal traffic.
* Proceed only if duplicate rebuilds and DB pressure disappear.

# Metrics to Check

| Metric | High or changing value means |
|---|---|
| Cache miss rate | Trigger load; segment by key and one-second interval |
| Expirations/sec | Spike aligned with misses suggests synchronized TTL |
| Evictions/sec | Memory-driven churn rather than scheduled expiry |
| Requests per key | Identifies hot keys and rebuild concurrency |
| Rebuild leaders | More than expected indicates coordination failure |
| Rebuild waiters | High shows effective coalescing but may threaten deadlines |
| Rebuild duration p95/p99 | Longer windows allow more arrivals |
| Refresh success/error | Failure can prolong stale service or repeated rebuild |
| Stale served count/age | Availability benefit and correctness risk |
| Fallback permits/queue/rejects | Whether DB work is actually bounded |
| DB calls per cache miss | Above one suggests duplicate work, retries, or N+1 |
| Hikari active/pending | Saturation from fallback |
| DB CPU/query p99 | Authoritative-store impact |
| Redis p99 | Low during misses excludes Redis execution bottleneck |
| API p95/p99/error rate | User-visible consequence |

Useful relationships:

```text
concurrent arrivals for one key ~= requests_per_second_for_key * rebuild_seconds
DB work ~= distinct keys rebuilt * leaders_per_key * queries_per_rebuild * attempts
```

If misses spike with expirations, fix TTL distribution.
If misses spike with a deploy but not expiration, inspect key versions and bypass.
If Redis p99 stays low while pool pending rises, the fallback path is the bottleneck.
If DB query rate falls while Hikari pending rises, requests may be stuck waiting for connections.

# Distributed Trace Investigation

I group traces by cache key fingerprint in a narrow incident window.

```text
traceId=stp-101
Gateway                              6,380 ms
  Promotion Service                 6,340 ms spanId=p101
    Redis GET                           3 ms spanId=r101 result=miss
    singleflight                       0 ms role=leader
    Hikari acquire                  4,910 ms spanId=h101
    MySQL promotion query           1,211 ms spanId=d101
    Redis SET                           5 ms spanId=w101 ttl=1987

traceId=stp-102
Gateway                              6,210 ms
  Promotion Service                 6,170 ms spanId=p102
    Redis GET                           2 ms spanId=r102 result=miss
    singleflight                    6,095 ms role=waiter keyHash=k-77
```

Before the fix, many traces show `role=leader` for the same key.
After the fix, one leader has DB spans and waiters share its result.
Client latency is the gateway's view; server latency is the promotion parent span.
Redis and DB child spans locate dependency time.
Hikari acquisition must be separate from SQL execution.
Repeated sibling DB spans reveal retries.
A missing DB child on a waiter is expected; a missing Redis child may mean local L1 hit, bypass, or missing instrumentation.

# Distributed Logs

```text
2026-09-13T12:00:00.041Z WARN service=promotion-service instance=promo-6c2
traceId=stp-101 spanId=p101 requestId=req-501 endpoint=/home/promotions
tenantHash=t-42 cacheKeyHash=k-77 cacheResult=miss rebuildRole=leader
ttlPolicy=fixed-noon inFlightForKey=26 totalRebuildsInFlight=1900
```

```text
2026-09-13T12:00:05.993Z ERROR service=promotion-service instance=promo-6c2
traceId=stp-101 spanId=h101 endpoint=/home/promotions downstream=mysql
error=SQLTransientConnectionException poolActive=40 poolMax=40 poolPending=214
acquireMs=4910 redisGetMs=3 totalLatencyMs=6340
```

I correlate the trace ID, then compare logs sharing `cacheKeyHash=k-77`.
The first log proves concurrent rebuilding behavior but not why expiry aligned.
TTL-at-write evidence and the noon schedule establish the trigger.
The DB pool error is a downstream consequence, not automatically a database root cause.

# Commands / Tools

```text
Bounded checks for known keys:
redis-cli --tls -h redis.example -p 6379 PTTL "promotions:v7:tenant-42:home"
redis-cli --tls -h redis.example -p 6379 MEMORY USAGE "promotions:v7:tenant-42:home"

Server counters and bounded history:
redis-cli --tls -h redis.example -p 6379 INFO stats
redis-cli --tls -h redis.example -p 6379 INFO memory
redis-cli --tls -h redis.example -p 6379 INFO commandstats
redis-cli --tls -h redis.example -p 6379 SLOWLOG GET 20
```

`PTTL` proves one known key's remaining life, not the distribution.
Application TTL histograms give a safer fleet-wide view.
`INFO stats` counters require rate/delta calculation.
`SLOWLOG` can remain empty during a stampede because Redis answers misses quickly.
Spring Actuator `/actuator/metrics` and `/actuator/prometheus` expose cache, rebuild, pool, and HTTP series.
A controlled `/actuator/threaddump` can show threads waiting on Hikari or a local future.
Avoid `KEYS *`, broad `MONITOR`, `FLUSHDB`, and `FLUSHALL`.
Do not diagnose a hot key by running an expensive full-dataset command during peak traffic.

# Root Cause

The promotion writer assigned every tenant key the same absolute noon expiration.
The service had no single-flight and retried failed rebuild queries.

```text
Aligned noon TTL
  -> 80 hot tenant keys expire together
  -> each Redis GET returns a fast miss
  -> many requests rebuild each key independently
  -> DB calls rise from 240/min to 6,700/min
  -> Hikari reaches 40/40 and pending reaches 214
  -> acquisition and query latency rise
  -> longer rebuilds admit more concurrent requests
  -> retries add work
  -> API p99 reaches 6.4 s
```

This is a feedback loop.
Redis availability and latency were healthy.
The root defect was synchronized TTL plus uncoalesced rebuild behavior; retry policy amplified it.

# Fix

Immediate mitigation:

* Serve promotion data within an approved stale window.
* Enable one rebuild leader per key per JVM.
* Cap total concurrent rebuilds and reject/omit optional enrichment beyond the cap.
* Stop retries on pool-acquisition timeout.
* Roll back fixed-noon expiration logic.
* Avoid cache flush and unlimited warm-up.

Permanent fix:

* Assign a base TTL plus bounded random jitter.
* Add single-flight keyed by version, tenant, and entity.
* Refresh hot values early while an acceptable previous value remains.
* Warm a measured hot-key set at a rate below DB headroom.
* Use an outbox and entity versions for reliable ordered invalidation.
* Add fencing or conditional writes where a cross-instance lease protects authoritative writes.
* Keep DB fallback behind a bulkhead and one end-to-end deadline.

# Verification

| Signal | Before | After |
|---|---:|---:|
| Expirations in peak second | 80 hot keys | 0-4 |
| Leaders for key k-77 | 26 | 1 per JVM, bounded globally |
| DB calls/min | 6,700 | 260 |
| DB calls per rebuilt key | 20-80 | <= number of pods, usually 1 |
| Hikari active | 40/40 | 8/40 |
| Hikari pending | 214 | 0 |
| DB CPU | 91% | 36% |
| API p99 | 6.4 s | 240 ms |
| Error rate | 13% | 0.2% |
| Maximum stale age | Unmeasured | <90 seconds |

I verify through at least two former noon boundaries and a controlled cold-key test.
I compare equal traffic and hot-key distribution.
I validate prices, tenant isolation, version ordering, and maximum stale age, not only latency.
I test leader failure to ensure another refresh can proceed without a permanent stuck entry.

# Prevention

* Alert on per-second miss and expiration spikes.
* Track hot key fingerprints and requests per key without exposing sensitive values.
* Instrument rebuild leaders, waiters, duration, failure, and stale age.
* Add TTL-jitter contract tests and a TTL-at-write histogram.
* Load-test synchronized expiration, Redis restart, and cold deployment.
* Keep fallback concurrency below tested DB headroom.
* Make retries deadline-aware, bounded, and jittered.
* Canary key-version and invalidation changes.
* Use outbox/version checks for ordered cache invalidation.
* Test single-flight cleanup on timeout, cancellation, and exceptions.
* Review distributed-lock assumptions and require fencing for stale-writer safety.
* Maintain an approved stale-data matrix by field and use case.

# Interview Answer

### What I would say in an interview

A cache stampede is many requests rebuilding the same missing value, not simply a low hit ratio. I would correlate fine-grained misses, expirations, hot-key fingerprints, rebuild counts, DB calls, Hikari pending, and traces. In this case Redis answered misses in 3 ms, but all tenant keys expired at noon and dozens of requests rebuilt each key. I would mitigate with bounded stale serving, per-key single-flight, a total rebuild bulkhead, and no retries on pool timeouts. The permanent fix is TTL jitter, early refresh for hot keys, controlled warming, and versioned outbox invalidation. I would verify through the former expiry boundary and check freshness as well as latency.

### Common interviewer traps

* Calling Redis slow because the request is slow.
* Adding only a longer TTL without handling future expiry.
* Running an unlimited warm-up job.
* Assuming one in-memory lock coordinates all replicas.
* Treating a Redis lease as absolute ownership.
* Increasing the DB pool and moving overload downstream.
* Flushing Redis during a miss storm.

### Quick memory flow

```text
Time shape
  -> hot key
  -> expiration/eviction/invalidation
  -> duplicate rebuild proof
  -> single-flight
  -> jitter/early refresh
  -> bounded fallback
  -> ordering/fencing
  -> verify at boundary
```

# Interview Follow-up Questions

### 1. What is TTL jitter?

A bounded random addition or subtraction to base TTL that spreads expirations while staying within freshness limits.

### 2. What is single-flight?

Concurrent requests for the same key share one in-flight refresh result instead of independently rebuilding it.

### 3. Does local single-flight solve a fleet-wide stampede?

It limits work to roughly one rebuild per key per JVM. Cross-instance protection or a strict DB bulkhead may still be needed.

### 4. What is stale-while-revalidate?

Readers receive a bounded older value while one background operation refreshes it. It is suitable only when that staleness is correct.

### 5. Why can Redis `SLOWLOG` be empty?

Redis may return misses quickly; the expensive duplicate work occurs in MySQL and application queues.

### 6. Why are distributed locks insufficient by themselves?

Leases can expire during pauses or partitions. Fencing tokens or version-conditional authoritative writes reject stale owners.

### 7. How does an outbox help invalidation?

It commits the business update and invalidation event atomically, then an idempotent relay publishes it; consumers use versions to reject old events.

### 8. Why not warm every key?

It wastes memory and can overload the database. Warm measured hot keys with bounded concurrency and backpressure.

### 9. How does a stampede become self-reinforcing?

Duplicate rebuilds overload the DB, making rebuilds slower; the longer window admits more duplicate requests and retries.
