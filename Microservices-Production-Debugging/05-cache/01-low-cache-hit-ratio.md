# Problem

A cache keeps frequently needed data close to the application so repeated reads avoid slower database work.
A cache hit means the requested key was found and usable.
A cache miss means the application must use the authoritative store, usually MySQL, and may then populate Redis.
The hit ratio is `hits / (hits + misses)` for the same scope and time window.
This playbook covers a production drop from 95% to 60%, not merely the definition of that ratio.
A ratio can fall because keys changed, entries expired, Redis evicted them, traffic changed, or reads bypassed the cache.
A low ratio is a symptom; blindly flushing Redis creates more misses and can cause an outage.

Cache access patterns matter:

| Pattern | Read behavior | Write behavior | Main risk |
|---|---|---|---|
| Cache-aside | Application reads cache, then DB on miss | Application updates DB and invalidates or updates cache | Stale data and stampedes |
| Read-through | Cache loader obtains data on miss | Usually separate from reads | Loader latency and hidden DB pressure |
| Write-through | Cache writes cache and authoritative store synchronously | Both complete before success | Higher write latency and partial-failure handling |
| Write-behind | Cache accepts write and persists later | Asynchronous persistence | Data loss, ordering, and replay complexity |

# Production Situation

At 10:05 after `catalog-service` version 4.18 was deployed:

* Traffic remains 10,000 product reads/min.
* Redis hit ratio falls from 95% to 60%.
* Cache hits fall from 9,500/min to 6,000/min.
* DB-backed misses rise from 500/min to 4,000/min, an 8x increase.
* API p50 rises from 32 ms to 70 ms.
* API p95 rises from 110 ms to 1.4 s.
* API p99 rises from 240 ms to 3.8 s.
* Hikari active connections rise from 8/40 to 40/40.
* Hikari pending requests rise from 0 to 96.
* DB CPU rises from 28% to 82%.
* Redis command p99 remains 3 ms.
* Error rate reaches 7% because DB connection acquisition times out.

The aggregate ratio hides traffic weighting.
Endpoint A has 9,000 requests/min at 60% hits; endpoint B has 1,000 requests/min at 99% hits.
The weighted ratio is `(5,400 + 990) / 10,000 = 63.9%`, not the simple average of 79.5%.
I therefore segment by endpoint, tenant, key namespace, instance, response type, and deployment version.

# Architecture

```text
Client
  |
  v
API Gateway
  |
  v
Catalog Service (Spring Boot)
  |
  +---- GET product:v3:{tenantId}:{productId} ----> Redis
  |                         |
  |                         +---- hit: return value
  |                         |
  |                         +---- miss
  |                                |
  +--------------------------------v
                             HikariCP
                                |
                                v
                              MySQL
```

With cache-aside, MySQL is authoritative.
The application must use a key that represents every input affecting the response.
`product:v3:tenant-42:817` is safer than `product:817` because it includes schema version and tenant.

# What I Check FIRST

1. **Scope and timing.** I compare hit ratio by endpoint, tenant, instance, namespace, and version. A deployment-aligned drop suggests key-generation or bypass code; a global gradual drop suggests TTL, memory, or traffic change.
2. **Raw hits, misses, and request rate.** A ratio alone is ambiguous. Stable hits with rising misses can mean new traffic; falling hits with stable traffic can mean invalidation, expiration, or key changes.
3. **DB amplification and saturation.** I check DB calls, Hikari active/pending, query latency, and DB CPU. This establishes impact and whether a cache issue is becoming a database incident.
4. **Redis health.** I check latency, errors, used memory, evictions, expirations, and `maxmemory-policy`. Fast Redis with no evictions points away from server pressure.
5. **Recent change.** I compare key samples, TTL assignment, serialization, routing, tenant context, and feature flags between old and new application versions.

# Step-by-Step Investigation

### Step 1 - Confirm the ratio and its denominator

* **What I check:** Application cache hit and miss counters over identical labels and windows.
* **Why:** Ratios from different dashboards may count negative-cache hits, local L1 hits, errors, or refreshes differently.
* **Tool:** Prometheus query:

```promql
sum(rate(cache_gets_total{service="catalog-service",result="hit"}[5m]))
/
sum(rate(cache_gets_total{service="catalog-service",result=~"hit|miss"}[5m]))
```

* **Expected:** About 0.95 before the incident with stable request rate.
* **Bad:** 0.60 after 10:05 while traffic remains 10,000/min.
* **Meaning:** Extra authoritative-store work is real, not a percentage artifact.
* **Next:** Break down the numerator and denominator.

### Step 2 - Segment rather than average

* **What I check:** Hit and miss rates by route, cache name, tenant, region, pod, key version, and result type.
* **Why:** One high-volume route can dominate an aggregate, and one pod can run different code.
* **Expected:** Similar ratios for pods serving equivalent traffic.
* **Bad:** Only version 4.18 pods show 58-62%; version 4.17 pods show 94-96%.
* **Meaning:** Redis itself is probably healthy; the new client behavior is suspect.
* **Next:** Compare generated keys and cache lookup decisions.

### Step 3 - Inspect safe key samples

* **What I check:** Sanitized key names and hash fingerprints, never secrets or full personal data.
* **Why:** A version, tenant, locale, or whitespace change can turn every previously warm key into a miss.
* **Expected:** `product:v2:tenant-42:817` before and after if no schema migration was intended.
* **Bad:** New pods request `product:v3:tenant-42:817` while only v2 keys are warm.
* **Meaning:** A key-version rollout created a cold namespace.
* **Next:** Decide whether v3 was intentional and whether dual-read or warming was required.

### Step 4 - Validate cache correctness before improving hits

* **What I check:** Whether the key includes all response dimensions: tenant, authorization scope, locale, currency, feature version, and entity ID.
* **Why:** A high hit ratio is harmful if tenant A can receive tenant B's value.
* **Expected:** Semantically equal requests share keys; different tenants or representations do not.
* **Bad:** Key omits `tenantId`, or includes a random request ID.
* **Meaning:** Omission risks data leakage; over-specificity destroys reuse.
* **Next:** Correct and version the key deliberately, accepting a controlled warm-up.

### Step 5 - Check TTL assignment and distribution

* **What I check:** TTL at write time, remaining TTL samples, and expiration counts.
* **Why:** A missing unit conversion can turn 30 minutes into 30 seconds; identical TTLs can align expirations.
* **Safe samples:**

```text
redis-cli --tls -h redis.example -p 6379 TTL "product:v3:tenant-42:817"
redis-cli --tls -h redis.example -p 6379 PTTL "product:v3:tenant-42:817"
```

* **Expected:** Remaining TTLs spread across the intended range, such as 1,620-2,160 seconds with jitter.
* **Bad:** Many keys return 20-30 seconds or expire at the same minute.
* **Meaning:** Unit/config error or synchronized expiry is causing recurring miss waves.
* **Next:** Correlate `expired_keys` rate with miss spikes.

### Step 6 - Distinguish expiration from eviction

* **What I check:** `expired_keys`, `evicted_keys`, memory usage, `maxmemory`, and policy deltas.
* **Why:** Expiration is TTL-driven; eviction removes live keys under memory pressure.
* **Expected:** `evicted_keys` delta is zero and memory has headroom.
* **Bad:** Evictions increase while `used_memory` stays near `maxmemory`.
* **Meaning:** Redis is discarding useful entries because memory or policy no longer fits the working set.
* **Next:** Identify growth by namespace and validate the policy rather than immediately adding memory.

### Step 7 - Inspect Redis configuration safely

* **What I check:** Selected bounded server facts, not a production-wide key scan.
* **Commands:**

```text
redis-cli --tls -h redis.example -p 6379 INFO stats
redis-cli --tls -h redis.example -p 6379 INFO memory
redis-cli --tls -h redis.example -p 6379 INFO commandstats
redis-cli --tls -h redis.example -p 6379 CONFIG GET maxmemory
redis-cli --tls -h redis.example -p 6379 CONFIG GET maxmemory-policy
```

* **Expected:** No evictions, low errors, and an approved policy such as `allkeys-lfu` for a pure disposable cache.
* **Bad:** `noeviction` causes writes to fail, or `volatile-lru` has few TTL-bearing candidates.
* **Meaning:** Policy semantics do not match the data model.
* **Next:** Confirm whether Redis also stores non-cache correctness data before proposing a policy change.

### Step 8 - Check value population failures

* **What I check:** Cache put attempts, successes, serialization failures, oversized values, and Redis write errors.
* **Why:** Reads can miss repeatedly if DB results are never cached.
* **Expected:** Nearly every cacheable DB success produces a successful put.
* **Bad:** `cache_put_total{result="serialization_error"}` rises on version 4.18.
* **Meaning:** The miss path works but the refill path is broken.
* **Next:** Inspect one correlated trace and structured error log.

### Step 9 - Check bypass and eligibility logic

* **What I check:** Feature flags, `@Cacheable` conditions, null handling, transaction boundaries, and self-invocation in Spring proxies.
* **Why:** A method may silently bypass caching even though Redis is healthy.
* **Expected:** Eligible requests call Redis once and expose `cache.result`.
* **Bad:** A changed condition marks 40% of product requests `cacheable=false`.
* **Meaning:** Application logic, not Redis capacity, caused the ratio drop.
* **Next:** Compare request attributes that trigger bypass.

### Step 10 - Evaluate traffic and working-set change

* **What I check:** Unique keys/min, new-product traffic, tenant mix, bots, scans, and Zipf-like popularity.
* **Why:** The same cache capacity performs differently when traffic shifts from repeated popular keys to one-time keys.
* **Expected:** Stable unique-key rate and reuse distribution.
* **Bad:** A crawler requests 30,000 previously unseen IDs/min.
* **Meaning:** This is cache penetration or a changed working set, not faulty eviction alone.
* **Next:** Validate authorization/rate limits and negative caching.

### Step 11 - Investigate cache penetration

* **What I check:** Misses for nonexistent IDs, repeated invalid lookups, and DB `not found` queries.
* **Why:** If absence is not cached, repeated invalid keys always reach MySQL.
* **Expected:** Valid IDs dominate misses; bounded negative entries absorb repeated not-found reads.
* **Bad:** 55% of misses return DB 404 for the same small set of IDs.
* **Meaning:** Attackers, crawlers, or clients are penetrating the cache.
* **Next:** Add short negative caching after authorization and input validation.

Negative values need a distinct marker, a short TTL such as 15-60 seconds, and invalidation when the entity is created.
Never let negative caching reveal whether another tenant owns an identifier.
A Bloom filter can reduce obviously invalid lookups but introduces false positives and operational complexity.

### Step 12 - Quantify database amplification

* **What I check:** Requests multiplied by miss rate.
* **Calculation:** At 10,000 reads/min, 5% misses produce 500 DB reads/min; 40% misses produce 4,000 DB reads/min.
* **Why:** This 8x increase explains pool and latency effects without blaming a slow SQL plan.
* **Expected:** DB call rate closely follows cacheable request rate times miss rate.
* **Bad:** DB calls exceed misses due to retries or N+1 queries.
* **Meaning:** Cache misses plus retry/query amplification are cascading.
* **Next:** Temporarily cap fallback concurrency and retries.

### Step 13 - Separate mitigation from root repair

* **Safe mitigation:** Roll back 4.18, disable the bad key flag, rate-limit crawlers, and bound DB fallback concurrency.
* **Unsafe reaction:** `FLUSHALL`, a full `KEYS *`, unlimited warming, or raising DB pool size without capacity evidence.
* **Why:** Those reactions can create a larger cold-cache wave or move saturation into MySQL.
* **Root repair:** Correct versioned key generation and deploy with controlled dual-read plus asynchronous backfill.
* **Next:** Verify application, cache, and DB together.

# Metrics to Check

| Metric | High or increasing means | Low or decreasing means |
|---|---|---|
| Request rate | More offered load; ratios must be traffic-weighted | A ratio recovery may only reflect lost traffic |
| Cache hit rate | Warm reusable working set if values are correct | Bypass, cold keys, expiry, eviction, or new traffic |
| Cache miss rate | More DB fallback and possible penetration | Good only if hits and traffic remain stable |
| Unique keys/min | Churn, scans, or cardinality growth | Strong reuse or reduced traffic |
| Cache put failures | Refill is broken | Normal, provided puts are attempted |
| `expired_keys` delta | TTL-driven turnover | Entries may be long-lived or absent |
| `evicted_keys` delta | Memory pressure discards live keys | No active eviction pressure |
| Used memory/maxmemory | Imminent eviction or write rejection | Capacity headroom |
| Redis p50/p95/p99 | Server, network, pool, or command issue | Redis is unlikely to explain misses |
| DB calls/min | Miss amplification | Could mean requests are blocked before DB |
| Hikari active/pending | Pool saturation | Healthy headroom if request rate is normal |
| DB query p95 | Store is overloaded or plans changed | Pool wait may still dominate API time |
| API p95/p99 | User impact and queueing | Recovery, if success rate also recovers |

Relationships matter more than isolated values:

```text
DB reads ~= cacheable reads * miss rate * DB operations per miss * retry attempts
```

If hit ratio changes after deployment while Redis latency and evictions are flat, suspect client behavior.
If misses align with `expired_keys`, suspect TTL.
If misses align with `evicted_keys` and memory pressure, suspect capacity, cardinality, or policy.
If DB query rate falls while Hikari pending rises, requests may be waiting for connections rather than reaching MySQL.

# Distributed Trace Investigation

I select successful hits, ordinary misses, and slow/errors from the same endpoint and time window.

```text
traceId=8fd21c
Gateway                         1,430 ms
  Catalog Service              1,401 ms spanId=cat91
    Redis GET                      3 ms spanId=red11 cache.result=miss
    Hikari acquire               812 ms spanId=pool7
    MySQL SELECT                 554 ms spanId=db32 rows=1
    Redis SET                      4 ms spanId=red12 ttl=1800
```

The 3 ms Redis span proves this request did not wait inside Redis.
The miss caused a DB path; pool acquisition plus query time explains server latency.
Gateway client latency includes network and catalog processing.
Catalog server latency includes its queueing and children.
The DB span measures query work, not time waiting for a Hikari connection unless separately instrumented.
Retry spans with repeated DB calls reveal amplification.
A missing Redis child span may mean cache bypass, missing instrumentation, sampling, or failure before the client call.
It is not proof that Redis was never contacted.
I use `traceId` to find logs and `spanId` to distinguish the miss lookup from the refill write.

# Distributed Logs

```text
2026-09-13T10:06:14.221Z INFO service=catalog-service instance=catalog-4.18-7d9
traceId=8fd21c spanId=cat91 requestId=req-771 endpoint=/products/817 tenantIdHash=t-91
cache=product keyVersion=v3 cacheResult=miss redisLatencyMs=3 dbAcquireMs=812
dbLatencyMs=554 cachePut=success totalLatencyMs=1401
```

```text
2026-09-13T10:06:14.225Z WARN service=catalog-service instance=catalog-4.18-7d9
traceId=8fd21c spanId=red11 endpoint=/products/817 downstream=redis
event=cache_key_miss keyPattern=product:v3:{tenant}:{id} deployment=4.18
```

I search the trace ID, confirm timestamps and instance, then compare logs from 4.17 and 4.18.
I log a key pattern or one-way fingerprint, not raw tenant data or cached values.
The warning demonstrates a miss but does not prove root cause.
Only the deployment comparison, v2/v3 key evidence, normal Redis latency, and absence of evictions establish the cause.

# Commands / Tools

```text
Safe bounded observations:
redis-cli --tls -h redis.example -p 6379 PING
redis-cli --tls -h redis.example -p 6379 INFO stats
redis-cli --tls -h redis.example -p 6379 INFO memory
redis-cli --tls -h redis.example -p 6379 SLOWLOG GET 10
redis-cli --tls -h redis.example -p 6379 TTL "product:v3:tenant-42:817"
redis-cli --tls -h redis.example -p 6379 MEMORY USAGE "product:v3:tenant-42:817"
```

`PING` proves that this client can receive a Redis protocol response; it does not prove every command or application path is healthy.
`INFO stats` gives cumulative counters, so compare deltas over equal windows.
`INFO memory` shows allocator and memory state; it does not identify which application owns growth.
`SLOWLOG` measures server command execution time, not DNS, network, TLS, or client-pool wait.
`TTL` on a known safe key proves only that key's remaining lifetime.
`MEMORY USAGE` estimates one key's memory and must not be extrapolated from a biased sample.
Avoid `KEYS *`, broad `MONITOR`, `FLUSHDB`, and `FLUSHALL` in production.
If scanning is approved, use a small-count `SCAN` from an isolated operator session and stop if load changes.

# Root Cause

The 4.18 deployment changed the key prefix from v2 to v3 before the new namespace was warmed.

```text
Key version changed
  |
  v
95% of requests cannot reuse warm v2 entries
  |
  v
Hit ratio falls from 95% to 60%
  |
  v
DB reads rise from 500/min to 4,000/min
  |
  v
Hikari reaches 40/40 and requests queue
  |
  v
DB CPU and query latency rise
  |
  v
API p99 reaches 3.8 s and acquisition errors appear
```

Each arrow is supported by time-aligned metrics and miss traces.
Redis stayed fast and had no eviction growth, excluding Redis server saturation as the primary cause.

# Fix

Immediate mitigation:

* Roll back 4.18 so reads reuse the warm v2 namespace.
* Bound concurrent DB fallbacks so Redis misses cannot consume every DB connection.
* Disable retries on pool-acquisition timeout; they add work without changing capacity.
* Rate-limit abnormal invalid-key traffic by authenticated tenant.
* Do not flush either namespace.

Permanent fix:

* Keep versioned, tenant-safe keys.
* Deploy a controlled migration: read v3, then v2 on miss, populate v3, and remove fallback after coverage is high.
* Warm only known high-value keys at a rate below measured DB headroom.
* Add TTL jitter and cap warm-up concurrency.
* Test key compatibility, tenant isolation, serialization, and TTL units.
* Size Redis from working-set measurements and choose an eviction policy consistent with disposable cache data.

# Verification

| Signal | Before | After | Why it matters |
|---|---:|---:|---|
| Weighted hit ratio | 60% | 95.2% | Expected key reuse returned |
| Miss-backed DB reads | 4,000/min | 480/min | Amplification was removed |
| Redis p99 | 3 ms | 3 ms | Redis remained healthy |
| Hikari active | 40/40 | 9/40 | Pool headroom returned |
| Hikari pending | 96 | 0 | Requests no longer queue for DB |
| DB CPU | 82% | 31% | Cache recovery reduced store load |
| API p99 | 3.8 s | 230 ms | User-visible latency recovered |
| Error rate | 7% | 0.3% | Success recovered, not merely latency |

I compare at equal request rate and tenant mix for at least two TTL cycles.
I verify both cache hits and fresh DB-backed reads return correct tenant-specific values.
I check that v3 population grows gradually and that no expiration wave appears later.

# Prevention

* Alert on hit ratio and absolute miss rate by high-volume namespace.
* Alert on DB amplification: DB reads divided by cacheable requests.
* Dashboard hit/miss, puts, evictions, expirations, memory, Redis percentiles, Hikari, DB, and API RED metrics together.
* Add key-contract tests for version, tenant, locale, and authorization dimensions.
* Canary key-format changes and compare old/new pod ratios.
* Require an explicit cold-start and warming plan for namespace migrations.
* Add TTL jitter and monitor TTL distributions.
* Put a concurrency bulkhead around DB fallback.
* Use short negative caching for safe not-found results.
* Load-test cold-cache behavior at expected DB capacity.
* Record deployment annotations on dashboards.
* Keep rollback possible while old keys remain valid.

# Interview Answer

### What I would say in an interview

If a Redis hit ratio dropped from 95% to 60%, I would first validate raw hits, misses, traffic, and the affected endpoint, tenant, pods, and key namespace. At 10,000 reads per minute, misses rise from 500 to 4,000 per minute, so I would immediately watch Hikari pending, DB calls, DB latency, and API p99. Then I would distinguish client behavior, expiration, eviction, penetration, and Redis slowness using deployment timing, key samples, TTL distribution, `expired_keys`, `evicted_keys`, memory, traces, and logs. I would mitigate with rollback and bounded fallback, not a cache flush. Finally I would fix the key rollout, verify recovery at equal traffic, and test correctness across tenants.

### Common interviewer traps

* Quoting only the ratio without its request volume or labels.
* Treating a higher hit ratio as more important than correct tenant-scoped data.
* Assuming every miss is caused by eviction.
* Increasing DB pool size before checking DB capacity.
* Flushing Redis and creating a guaranteed cold-cache event.
* Using one cache-miss log as proof of root cause.

### Quick memory flow

```text
Confirm ratio and traffic
  -> segment scope
  -> compare key and TTL
  -> expiration versus eviction
  -> trace miss path
  -> quantify DB amplification
  -> mitigate safely
  -> fix rollout
  -> verify correctness and recovery
```

# Interview Follow-up Questions

### 1. Why can a 95% to 60% drop be severe?

At constant traffic, miss load changes from 5% to 40%, so DB work becomes eight times larger.

### 2. Why is the average of endpoint hit ratios wrong?

Endpoints carry different request volumes. Sum hits and requests first, then divide, or use a traffic-weighted average.

### 3. Expiration versus eviction?

Expiration removes a key because its TTL elapsed. Eviction removes a live key under memory pressure according to `maxmemory-policy`.

### 4. What does Redis `SLOWLOG` omit?

It records server command execution time. It omits DNS, connection acquisition, network, TLS, and most client-side queueing.

### 5. When is negative caching safe?

After authorization and validation, for non-sensitive not-found results, with a distinct marker, short TTL, and creation-time invalidation.

### 6. Why include tenant and version in keys?

Tenant prevents cross-customer data reuse. Version prevents incompatible serialized schemas or semantics from sharing entries.

### 7. Would you increase TTL to improve hits?

Only if staleness requirements allow it. First determine whether TTL is causal and ensure invalidation is reliable.

### 8. Why not flush the cache?

It removes valid warm data, forces nearly all reads onto the database, and can turn degradation into an outage.
