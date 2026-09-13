# Production Troubleshooting Study Chapter: Caching

## Purpose

This chapter teaches caching from first principles through production operation. A cache trades freshness, memory, and invalidation complexity for lower latency and less origin load. The goal is not merely a high hit ratio; it is correct behavior, predictable origin protection, and graceful degradation.

Use sanitized keys and metadata when debugging. Cache values, command traces, and memory dumps may contain credentials or customer data. Never flush a shared production cache or expose Redis diagnostics without authorization.

## Learning goals

After studying this chapter, you should be able to:

1. Explain cache-aside, read-through, write-through, write-behind, and their failure windows.
2. Select keys, TTLs, eviction policies, and consistency rules from business requirements.
3. Diagnose stale data, misses, evictions, hot keys, penetration, and memory pressure.
4. Prevent stampedes with single-flight, soft expiration, jitter, stale-while-revalidate, and controlled warming.
5. Decide fail-open versus fail-closed per data class.
6. Handle Redis latency or outage without causing a database collapse.
7. Explain why distributed locks reduce duplicate work but are not a general correctness proof.
8. Build useful cache observability and safe alerts.

---

# 1. Mental model

## 1.1 What a cache is

A cache is a faster, usually smaller copy or computed representation of authoritative data. It can live in a process, sidecar, CDN, gateway, Redis cluster, database buffer pool, or client. Every cache entry has:

```text
key -> value + metadata

metadata can include:
- creation/version time
- hard and soft expiration
- source version
- tenant or authorization scope
- negative-result marker
```

Before caching, answer:

1. What is the source of truth?
2. How stale may a value be?
3. What event makes it invalid?
4. What happens on a miss or outage?
5. Can two users safely share it?
6. Can it be reconstructed?
7. How much origin concurrency is safe?

A cache is not automatically a source of truth, durable store, distributed lock service, or authorization boundary.

## 1.2 Common caching patterns

### Cache-aside

```text
read:
  get cache
  hit  -> return
  miss -> read origin -> populate cache -> return

write:
  update origin -> invalidate or update cache
```

The application controls loading. It is simple and only caches demanded data, but a miss can stampede and a write/invalidation race can leave stale data.

### Read-through

The application asks the cache; the cache abstraction loads the origin on miss. It centralizes loading behavior but does not remove consistency or failure decisions. The loader is still origin traffic.

### Write-through

Writes synchronously update the authoritative store and cache through one abstraction. Reads are warm and the cache usually reflects completed writes, but write latency and failure coordination increase. Atomicity across two systems is not implied.

### Write-behind

Writes reach the cache or buffer first and are persisted asynchronously. It can batch writes and lower apparent latency, but introduces loss, reordering, replay, durability, and read-after-write risks. Use only with explicit business semantics, durable buffering, idempotency, and recovery.

### Refresh-ahead and stale-while-revalidate

Refresh-ahead reloads likely-needed entries before hard expiry. Stale-while-revalidate serves a bounded stale value while one refresh runs. Both reduce expiry cliffs but consume background capacity and need a hard freshness limit.

## 1.3 Freshness and consistency

TTL limits how long an entry is eligible to remain, but it is not a guarantee that:

- the value was fresh when cached,
- invalidation succeeded,
- replicas agree,
- clock assumptions are correct,
- an in-flight reader did not repopulate old data,
- the entry survives until TTL.

Typical invalidation approaches:

- Short TTL: simple bounded staleness, more origin load.
- Explicit delete/update after commit: faster freshness, delivery and race concerns.
- Transactional outbox/change-data-capture: durable invalidation events, asynchronous lag.
- Versioned keys: immutable versions avoid overwrite races but require pointer/version management.
- Tag/dependency invalidation: useful for aggregates, but fan-out and completeness are hard.

Example race:

```text
Reader R misses and reads database version 10.
Writer W commits version 11 and deletes the cache.
Reader R then writes version 10 into the empty cache.
Stale version 10 remains until another invalidation or TTL.
```

Version checks, ordering, short TTL, or a design that populates only known-current versions can close this window.

## 1.4 TTL, eviction, and memory

- **Expiration** removes or makes an entry ineligible because its time policy ended.
- **Eviction** removes an entry early to reclaim memory according to policy.
- **Invalidation** removes/updates an entry because source data changed.

TTL should reflect permitted staleness and origin capacity, not a single global habit. Add bounded random **jitter** so many entries do not expire simultaneously.

Redis memory includes key/value data, object and allocator overhead, replication/output buffers, client buffers, and fragmentation. `used_memory` below host memory does not prove safety. `maxmemory`, policy, RSS, fragmentation, fork/replication needs, and cgroup/host headroom matter.

## 1.5 Hit ratio and effective value

```text
request hit ratio = hits / (hits + misses)
byte hit ratio = bytes served from cache / total requested bytes
origin offload = origin work avoided, not just request count
```

A 99 percent hit ratio can still be bad if the 1 percent misses are expensive or synchronized. A lower ratio can be acceptable if cheap objects miss. Segment by cache, operation, route, tenant class, key family, and result type while controlling label cardinality.

Hit ratio does not measure correctness, staleness, cache latency, or origin survival.

## 1.6 Stampede, hot keys, and penetration

- **Stampede/dogpile:** many callers regenerate the same missing/expired value.
- **Hot key:** one key receives disproportionate traffic or CPU/network load.
- **Cache penetration:** repeated requests for values that do not exist or are deliberately uncacheable.
- **Avalanche:** many entries expire or disappear together, shifting broad load to the origin.

Controls include:

- Per-key single-flight inside a process.
- Cross-instance lease or lock with bounded wait and fallback.
- Soft TTL plus stale-while-revalidate.
- TTL jitter.
- Negative caching with a short TTL where absence is safe.
- Admission filters for known-invalid identifiers.
- Request coalescing and origin concurrency limits.
- Proactive warming for a bounded known working set.
- Replication or local near-cache for safe hot values.

## 1.7 Distributed-lock limitations

A lock can reduce duplicate cache fills. It does not alone guarantee business correctness:

- The holder can pause past lease expiry and continue.
- Network partitions can create uncertainty.
- Failover can lose a lock before replication.
- Unlock must verify ownership.
- Clock and lease assumptions can fail.
- A lock service outage can block all fills.

Use a unique owner token and atomic compare-and-delete for release. Keep leases bounded, make work idempotent, use fencing tokens when a downstream resource can reject stale owners, and retain a correctness mechanism independent of the lock. For cache regeneration, duplicate computation may be safer than indefinite blocking.

## 1.8 Fail-open and fail-closed

**Fail-open** bypasses cache failure and consults the authoritative source. This can preserve correctness for ordinary data, but only if origin load is bounded.

**Fail-closed** rejects when the cache cannot answer. It may be required when the cache is deliberately part of security or abuse control and no authoritative safe fallback is available.

Examples:

- Product description cache: usually fail open to database with concurrency limiting.
- Cached authorization decision: do not grant access merely because cache failed. Re-evaluate at the authority or deny according to policy.
- Rate-limit state: policy may require local conservative limits or denial for sensitive operations; document the tradeoff.

The choice is per cache and operation, not a global Redis setting.

## 1.9 Glossary

| Term | Meaning |
|---|---|
| Origin | Authoritative service or datastore used to build the cached value |
| Cache-aside | Application loads cache after a miss |
| Read-through | Cache abstraction invokes a configured loader |
| Write-through | Write synchronously updates origin and cache abstraction |
| Write-behind | Persistence happens asynchronously after accepting a write |
| TTL | Time-to-live before an entry expires |
| Soft TTL | Time after which refresh should begin while bounded stale use may continue |
| Hard TTL | Maximum time after which stale content must not be served |
| Eviction | Early removal due to memory policy |
| Invalidation | Removal/update because source data changed |
| Single-flight | Coalesces concurrent loads for one key into one execution |
| Negative caching | Temporarily caches an absent result |
| Hot key | Key with disproportionate access or resource cost |
| Stampede/dogpile | Concurrent regeneration of the same missing entry |
| Penetration | Repeated misses for nonexistent or uncacheable data |
| Jitter | Random bounded variation, commonly added to TTL or retry delay |
| Near-cache | Small local cache in front of a remote cache |
| Fencing token | Monotonic token allowing a resource to reject stale lock holders |

---

# 2. Metrics and evidence

| Signal | Why it matters | Explicit interpretation |
|---|---|---|
| Gets/hits/misses by cache and key family | Measures lookup outcomes | A miss increase can be expiry, eviction, key drift, flush, cold start, or errors |
| Request and byte hit ratio | Measures different offload dimensions | Compare with origin calls and cost; ratio alone is insufficient |
| Cache command p50/p95/p99 | Detects cache slowness | A hit that takes longer than origin may have negative value |
| Errors/timeouts by operation | Separates unavailable from ordinary misses | Do not convert errors to misses in telemetry |
| Origin calls, latency, errors, concurrency | Shows fallback impact | Miss rise plus origin pool wait indicates cascading pressure |
| Evictions and expirations | Separates memory removal from TTL | Eviction rise with memory ceiling supports pressure |
| Memory used, RSS, max, fragmentation | Evaluates memory headroom | RSS above logical used memory may indicate fragmentation/buffers |
| Key count and TTL distribution | Finds churn and synchronized expiry | A TTL spike at one timestamp predicts an avalanche |
| Load/regeneration duration and coalesced waiters | Diagnoses stampede | Many waiters with one origin call means single-flight works |
| Per-node CPU/network/commands | Finds hot nodes | Cluster averages hide slot/key concentration |
| Connection count, rejected clients, pool wait | Finds connection pressure | High client pool wait can occur before Redis CPU saturates |
| Replication lag, failover, rejected writes | Shows topology health | A failover event can explain transient errors or lost recent cache writes |
| Stale-served count and age | Makes freshness visible | Compare age with business hard limit |

Safe evidence sources include application metrics and traces, Redis `INFO` and slow-log summaries through approved access, managed-service dashboards, client-pool telemetry, deployment/config history, and origin metrics. Do not log full keys or values when they reveal personal or secret data; hash or classify keys.

---

# 3. Generic cache investigation workflow

## Step 1 - Define correctness and impact

Identify the cache, key family, source of truth, allowed staleness, affected operation, time window, and user impact. Decide whether the symptom is stale data, miss rate, latency, errors, memory, or origin overload.

## Step 2 - Stabilize both cache and origin

Cap fallback concurrency, protect the origin with bulkheads/rate limits, stop a harmful rollout, and serve bounded stale data only where policy permits. Do not flush the cache or create an unlimited fail-open path.

## Step 3 - Separate outcome types

Measure hit, miss, negative hit, stale hit, timeout, connection error, decode error, and loader failure separately. Treating every error as a miss hides incidents.

## Step 4 - Trace one key lifecycle

Using a sanitized representative key, follow key construction, lookup, value/version/TTL, origin read, population, write invalidation, and replication. Compare an affected key with a healthy key.

## Step 5 - Correlate the timeline

Overlay deploys, schema/serializer changes, key-prefix/version changes, invalidation consumer lag, evictions, expirations, failovers, memory, and traffic.

## Step 6 - Prove the mechanism

Examples:

- Key absent plus high evictions and maxmemory pressure supports eviction.
- Key present with old source version and missed invalidation supports invalidation failure.
- Timeout counter rising with normal keyspace does not mean a miss problem.
- One origin load plus many coalesced waiters shows single-flight is effective.

## Step 7 - Mitigate and verify

Verify cache latency/errors, origin concurrency, application SLO, freshness age, and recovery. Warm only a bounded high-value set at a controlled rate.

## Step 8 - Make it durable

Repair consistency protocol, key/version design, TTL/jitter, memory sizing, overload controls, client behavior, tests, and alerts.

---

# 4. Original cache questions

## 1. The application is returning stale data from cache. How would you investigate?

### Meaning and what it does not prove

The returned cached representation is older than the business freshness contract or source version. A different value from the database does not automatically prove a cache bug: replica lag, transaction isolation, eventual-consistency policy, client/CDN caches, and read timestamps may explain it.

### Issue locations

Browser/CDN/gateway/local/Redis caches; key construction; tenant and authorization dimensions; TTL; invalidation producer/outbox/consumer; write order; replicas; serializers; near-cache; clocks; and source read replicas.

### Causal mechanisms and example

Common causes are an overly long TTL, missed invalidation, wrong key, in-flight repopulation race, local cache not invalidated, event lag, or reading a lagging replica.

Example: a profile update commits version 42 and deletes Redis. An earlier miss has already read version 41 and writes it after the delete. Redis now serves version 41 until TTL. Including a source version and refusing to overwrite a newer version prevents regression.

### Ordered investigation and why

1. Capture a sanitized key, returned version, source version, timestamps, route, and cache layers; "stale" needs a precise comparison.
2. Query the authoritative source using the correct consistency path; a replica may itself lag.
3. Inspect entry value metadata, TTL, creation/version, and which layer served it.
4. Reconstruct the write commit and invalidation/update timeline.
5. Check invalidation publication, outbox, consumer lag/errors, and local-cache fan-out.
6. Compare key format, namespace, tenant, schema, and deployment version.
7. reproduce the race with controlled concurrent read/write.

### Tools, metrics, evidence, and interpretation

Use traces with cache outcome and value version, sanitized key inspection, TTL distribution, event/outbox lag, deployment diffs, DB primary/replica positions, and structured invalidation logs. Entry version older than primary plus no invalidation receipt proves the failed path. A fresh Redis entry but stale CDN response moves the issue outward.

### Immediate mitigation

Invalidate only affected keys or namespace through an approved tool, bypass the affected layer for critical reads with bounded origin concurrency, pause a broken invalidation rollout, or shorten TTL prospectively. Do not flush the entire shared cache reflexively.

### Permanent fix/design

Define freshness contracts, version entries, use durable outbox/CDC invalidation, close read-populate races, include all correctness dimensions in keys, and use soft/hard TTLs appropriate to data.

### Prevention and alerts

Measure invalidation end-to-end lag/failure, stale-served age, source/cache version mismatch sampling, and near-cache propagation. Test concurrent read/write races and event loss/replay.

### Common mistakes

- Assuming TTL guarantees freshness.
- checking only Redis while a CDN or local cache serves the value.
- logging full sensitive values or keys.
- deleting all keys before preserving evidence.
- updating cache before the database transaction commits.

### Interview-ready answer

I define the freshness contract and compare the returned version with the authoritative source at the same time. I identify the serving cache layer, inspect key and TTL, then reconstruct commit, invalidation, and repopulation order. I mitigate only affected data with bounded fallback, and permanently use durable invalidation, versioning, race-safe population, and freshness telemetry.

## 2. Cache suddenly stops working and database traffic increases dramatically. What could happen?

### Meaning and what it does not prove

This is loss of cache offload causing an origin surge, often called a cache avalanche when broad entries disappear together. Increased DB traffic does not prove Redis is down; a key-version rollout, expiry wave, eviction, client timeout, or serialization failure can turn hits into misses.

### Issue locations

Cache cluster/network/DNS/TLS, client pool and timeout, application key namespace, serializer, TTL policy, eviction/memory, bulk flush, failover, deployment, and DB fallback path.

### Causal mechanisms and example

If 95 percent of 20,000 reads/s were hits, the DB handled 1,000 reads/s. Losing the cache sends up to 20,000 reads/s, before retries. The DB slows, connection pools fill, timeouts cause retries, and both systems may fail.

### Ordered investigation and why

1. Stabilize the DB with fallback concurrency limits, rate limits, and bounded stale serving.
2. Separate cache misses from timeouts, connection, auth, and decode errors.
3. correlate onset with deploy, failover, expiry counts, evictions, key count, and memory.
4. compare old/new key prefixes and serializer versions.
5. inspect client connection pools, DNS/TLS, cache command latency, and cluster nodes.
6. measure origin attempts per request and retry amplification.
7. restore cache carefully, then warm a small high-value set at a DB-safe rate.

### Tools, metrics, evidence, and interpretation

Use hit/miss/error counters, Redis availability/latency/memory/eviction/expiration/failover metrics, application traces, pool metrics, and DB queries/connections/waits. Near-zero hits with normal cache commands and a new key prefix indicates a cold namespace. Cache timeout growth with stable key count indicates reachability/latency, not expiration.

### Immediate mitigation

Protect the DB first: shed noncritical reads, cap loader concurrency, reduce retries, serve policy-approved stale values, roll back a bad key/serializer change, and restore cache service. Warm gradually with jitter.

### Permanent fix/design

Use circuit breakers that do not create unlimited origin load, bulkheads, stale-while-revalidate, TTL jitter, multi-zone cache topology, backward-compatible key migrations, and DB capacity reserved for cache loss.

### Prevention and alerts

Alert on hit-ratio change, cache errors, evictions, synchronized expirations, origin/cache amplification, DB pool wait, and keyspace drops. Run cache-loss and cold-start exercises.

### Common mistakes

- Treating errors as misses.
- failing open to an unbounded DB.
- restarting all application instances and making every local cache cold.
- warming the full keyspace at maximum speed.
- increasing DB pools during DB saturation.

### Interview-ready answer

I protect the database, then distinguish true misses from cache timeouts, connection, auth, and decode failures. I correlate deployment, key prefix, expiry, eviction, memory, and failover evidence. I restore or roll back safely and warm gradually. Long term I use bounded fallback, stale-while-revalidate, jitter, compatible migrations, and cache-loss capacity tests.

## 3. Redis becomes unavailable. How should the application behave?

### Meaning and what it does not prove

Behavior depends on the data's correctness and security role. "Redis unavailable" may mean connection failure, timeout, partial cluster failure, or excessive latency. It does not imply every operation should fail open or every operation should fail closed.

### Issue locations

Client timeout/retry/circuit breaker, local fallback, origin bulkhead, security policy, Redis topology/failover, DNS/TLS/auth, connection pool, and operation-specific code.

### Causal mechanisms and example

If every request waits two seconds for Redis and then queries the DB, cache failure adds latency and still multiplies DB load. Short bounded cache timeouts and a circuit breaker reduce wasted waiting, while an origin concurrency limiter prevents collapse.

For a catalog read, the application may use bounded stale data or DB fallback. For a cached authorization decision, it must revalidate at the authority or deny according to policy; it must never grant access merely because Redis failed.

### Ordered investigation and why

1. Classify each cache by reconstructability, freshness, correctness, and security impact.
2. Detect errors separately from misses and use short operation-specific timeouts.
3. apply a circuit breaker to avoid repeated slow cache attempts.
4. enforce origin concurrency/rate budgets before fail-open.
5. choose bounded stale, authoritative fallback, degraded response, or fail-closed per operation.
6. verify recovery does not trigger synchronized warming or retry storms.
7. investigate Redis nodes, failover, network, DNS/TLS/auth, and client pools.

### Tools, metrics, evidence, and interpretation

Use command errors/latency, client-pool wait, circuit state, fallback and stale-served counters, origin concurrency, DB latency, and Redis topology events. An open circuit with stable origin load shows isolation. High fallback attempts and DB pool wait means degradation is unsafe.

### Immediate mitigation

Activate documented degraded modes, protect origins, disable nonessential cache-dependent features, use bounded stale data where allowed, and repair connectivity/topology. Avoid unlimited retries.

### Permanent fix/design

Define per-cache failure policy, multi-zone topology, tested client timeouts, jittered reconnect, circuit breakers, bulkheads, origin reserve, stale limits, and recovery warming controls.

### Prevention and alerts

Run Redis outage/failover tests. Alert on error rate, command p99, fallback load, circuit state, origin saturation, replication/failover health, and recovery surge.

### Common mistakes

- A global fail-open rule.
- granting authorization from missing cache state.
- long cache timeouts followed by DB calls.
- retries at client, library, and service layers.
- assuming a Redis replica makes application behavior tested.

### Interview-ready answer

I define failure behavior per data class. Ordinary reconstructable reads may use bounded stale data or a concurrency-limited authoritative fallback; security-sensitive decisions must revalidate or deny, never grant on failure. I use short timeouts, a circuit breaker, bulkheads, and jittered recovery, then verify both Redis and origin health.

## 4. A cache stampede occurs during high traffic. How would you prevent it?

### Meaning and what it does not prove

A stampede is many callers concurrently loading the same missing or expired value. High DB traffic alone does not prove a stampede; broad key misses, cache penetration, a flush, or a key-version change can look similar.

### Issue locations

TTL policy, key popularity, loader, process-local and distributed coordination, origin concurrency, deployments/warming, negative results, and failure/retry logic.

### Causal mechanisms and example

At 10,000 RPS, a hot key expires. If origin load takes 500 ms, about 5,000 requests can enter during regeneration. Without coalescing they execute 5,000 identical queries. A per-key single-flight makes one load while others wait briefly or receive bounded stale data.

### Ordered investigation and why

1. Identify whether misses concentrate on the same keys and timestamp.
2. compare cache miss count with origin load count; many loads per missing key prove failed coalescing.
3. inspect TTL distribution, deployment/flush, hot-key rate, and loader duration.
4. check single-flight/lock acquisition, wait, lease expiry, ownership-safe release, and errors.
5. inspect negative-key traffic and invalid input.
6. verify origin concurrency and stale fallback.
7. test expiry under load and a slow/failing loader.

### Tools, metrics, evidence, and interpretation

Use hashed-key frequency samples, miss-to-load amplification, coalesced waiter count, loader p99, lock contention/expiry, TTL histograms, and origin query concurrency. Ten thousand misses with one loader call and bounded wait shows successful coalescing. Many lock holders after lease expiry reveals a lease too short or unsafe ownership.

### Immediate mitigation

Serve bounded stale data, enable or tighten per-key single-flight, cap origin loaders, extend hot-key TTL with jitter through a reviewed change, and rate-limit abusive/invalid keys. Warm only known hot entries.

### Permanent fix/design

Use soft TTL plus stale-while-revalidate, jittered hard TTL, single-flight, safe negative caching, proactive hot-key refresh, origin bulkheads, and versioned values. Treat distributed locks as duplicate-work reduction, not sole correctness.

### Prevention and alerts

Alert on miss-to-origin-load amplification, loader concurrency, coalesced waiters, hot-key share, TTL cliffs, negative misses, and stale age. Test lock-holder pause and cache failover.

### Common mistakes

- One global lock for all keys.
- lock release without checking owner token.
- indefinite waiting for regeneration.
- identical TTL on a bulk import.
- warming every key and stampeding the origin deliberately.

### Interview-ready answer

I prove that concurrent misses for the same key create multiple origin loads, then inspect TTL concentration, hot keys, loader time, and coordination. I mitigate with bounded stale serving, per-key single-flight, jitter, and origin concurrency limits. Long term I add soft/hard TTL, safe refresh-ahead, negative caching, and tests for slow loaders and lock expiry.

## 5. Cache hit ratio suddenly drops. What would you investigate?

### Meaning and what it does not prove

The share of cacheable lookups returning values decreased for a defined scope. It does not prove cache failure or user impact. New traffic, cheap misses, low request volume, or a denominator/instrumentation change can alter the ratio.

### Issue locations

Metric definition, workload and key cardinality, key construction/version/prefix, deployment, TTL/expiration, eviction/memory, flush/restart/failover, invalidation volume, cache errors mislabeled as misses, and bypass logic.

### Causal mechanisms and example

A release changes `product:123` to `v2:product:123`; every lookup is a cold miss though old entries remain. Alternatively, memory pressure evicts keys before TTL. A marketing event can bring many one-time product IDs, reducing ratio without a defect.

### Ordered investigation and why

1. Validate numerator, denominator, time window, traffic volume, and instrumentation.
2. segment by cache, key family, route, version, tenant class, and result type.
3. separate miss, error, bypass, negative hit, and stale hit.
4. correlate deploys, key-prefix/serializer changes, restarts, flushes, and failovers.
5. compare expiration versus eviction rates, memory/RSS, key count, and TTL distribution.
6. inspect workload cardinality and reuse distance; demand may have changed.
7. measure origin cost and application SLO to prioritize response.

### Tools, metrics, evidence, and interpretation

Use cache counters, deployment/config diffs, Redis keyspace/memory/eviction/expiration telemetry, sampled hashed-key cardinality, traces, and origin metrics. Evictions rising at maxmemory indicates pressure. Misses only on new version indicate namespace migration. Lower ratio with stable origin cost and SLO may be expected traffic.

### Immediate mitigation

Roll back an accidental key change, restore capacity, correct expiry policy, protect origin, or warm a bounded valuable set. Do not chase the percentage without impact.

### Permanent fix/design

Use versioned migration plans with dual-read or controlled warm-up, size memory from working set and overhead, choose eviction policy deliberately, and instrument outcome reasons.

### Prevention and alerts

Alert on change rate plus origin amplification, not ratio alone. Track evictions, errors, keyspace drops, memory headroom, and per-key-family SLO.

### Common mistakes

- Dividing by all requests rather than cache lookups.
- combining errors and misses.
- using fleet-wide ratio that hides one cache.
- flushing to "reset" the problem.
- optimizing hit ratio instead of latency, correctness, and origin cost.

### Interview-ready answer

I validate the metric and segment it, then separate misses from errors and bypasses. I correlate deployments, key format, restart/failover, expiry, eviction, memory, and workload cardinality. I judge impact through origin load and SLO, mitigate the actual cause, and add reason-specific metrics and safe migration/warming controls.

---

# 5. High-value additional questions

## 6. How do you choose a caching pattern and consistency model?

### Meaning and what it does not prove

The choice follows read/write ratio, freshness, durability, ownership, and failure semantics. A named pattern does not make a multi-system write atomic.

### Issue locations

Application and cache abstraction, source transaction, outbox/event stream, cache loader, retry/reconciliation, and consumers of stale data.

### Causal mechanisms and example

Cache-aside suits read-heavy rebuildable data but permits miss and invalidation races. Write-through warms reads but a cache write can fail after source commit. Write-behind improves apparent write throughput but accepted writes can be lost without a durable log.

### Ordered investigation and why

1. Define source of truth and permitted stale/lost/reordered behavior.
2. quantify read/write ratio, object cost, and reuse.
3. map each partial failure and retry.
4. select a pattern and explicit reconciliation/version rule.
5. test concurrent writes, cache outage, replay, and recovery.
6. measure correctness and origin protection.

### Tools, metrics, evidence, and interpretation

Use versioned integration tests, fault injection, outbox lag, cache/source mismatch samples, and retry/reconciliation counters. A happy-path test proves no failure semantics.

### Immediate mitigation

For inconsistency, prefer authoritative reads for critical operations and targeted invalidation while preserving source durability.

### Permanent fix/design

Document the consistency contract, use durable eventing where needed, version values, make replay idempotent, and keep cache reconstructable unless intentionally designed as durable state.

### Prevention and alerts

Test every partial-failure boundary and alert on event lag, reconciliation failures, and mismatch samples.

### Common mistakes

- calling write-through atomic.
- using write-behind without durable replay.
- caching data with no acceptable stale behavior.
- omitting tenant/security context from keys.

### Interview-ready answer

I choose from business semantics: source of truth, freshness, durability, read/write mix, and partial failures. Cache-aside is simple for rebuildable reads; read/write-through centralize behavior; write-behind needs durable idempotent replay. In every case I define versioning, invalidation, fallback, and reconciliation and test failures, not only hits.

## 7. A Redis node has high memory pressure and evictions. How would you respond?

### Meaning and what it does not prove

Memory pressure means the cache approaches an operational or configured limit; eviction is policy-driven removal. Evictions do not prove a leak, and no evictions do not prove safety if policy rejects writes instead.

### Issue locations

Working-set growth, large values, key churn, TTL absence, allocator fragmentation, client/replication buffers, persistence fork overhead, maxmemory/policy, cluster slot balance, and local near-caches.

### Causal mechanisms and example

One million small entries carry key and allocator overhead. A large pipeline or slow client can grow output buffers. Under `allkeys-lru`, valuable entries are evicted and misses load the DB. Under `noeviction`, writes fail instead.

### Ordered investigation and why

1. Compare logical used memory, RSS, maxmemory, host/cgroup headroom, and trend.
2. identify policy and whether writes fail or keys evict.
3. inspect key count, size distribution, TTL coverage, and largest key families safely.
4. check fragmentation, client/output buffers, replication and persistence activity.
5. examine node/slot imbalance and hot keys.
6. correlate evictions with hit ratio, origin load, and SLO.
7. reduce growth or add capacity through a tested migration.

### Tools, metrics, evidence, and interpretation

Use managed Redis metrics or approved `INFO MEMORY`, keyspace summaries, sampled size scans, client/replication stats, and application outcomes. High RSS with stable logical memory suggests fragmentation or buffers. Logical data growth with missing TTL identifies retention policy.

### Immediate mitigation

Protect origin, stop an offending writer, add TTL to the affected class prospectively, remove only approved reconstructable data, or scale memory/shards safely. Avoid blocking full-keyspace commands.

### Permanent fix/design

Set object-size and key-count budgets, TTL defaults, correct eviction policy, value compression only when CPU tradeoff is measured, shard planning, and bounded clients.

### Prevention and alerts

Alert on projected time to maxmemory, eviction/write rejection, fragmentation, buffer growth, large-key regressions, and node imbalance.

### Common mistakes

- equating RSS and dataset bytes.
- running dangerous broad key scans in production.
- deleting unknown keys.
- adding memory without controlling unbounded key growth.

### Interview-ready answer

I separate dataset growth, fragmentation, and client or replication buffers, then confirm maxmemory policy and impact. I inspect safe size/TTL samples and node balance, protect the origin, and stop the offending growth or add tested capacity. I permanently enforce key, value, TTL, buffer, and shard budgets.

## 8. How do hot keys and cache penetration differ?

### Meaning and what it does not prove

A hot key is a valid key with disproportionate traffic; penetration is repeated demand for absent or uncacheable keys. High commands on one node may also reflect slot imbalance, not one key.

### Issue locations

Key distribution, cluster slots, popular content, bot/abusive input, negative-cache rules, local near-cache, origin loader, and rate limiting.

### Causal mechanisms and example

A celebrity profile can saturate one Redis shard despite a high hit ratio. Random nonexistent IDs always miss, traverse Redis, and query the DB; attackers can vary IDs to defeat per-key coalescing.

### Ordered investigation and why

1. Separate hit-heavy concentrated traffic from high-cardinality misses.
2. sample hashed key frequency and key family without exposing data.
3. inspect per-node CPU/network/commands and slot distribution.
4. validate missing-ID input and negative-cache safety.
5. check tenant/source rate and origin load.
6. test mitigation for correctness and abuse resistance.

### Tools, metrics, evidence, and interpretation

Use privacy-safe heavy-hitter sampling, per-node telemetry, miss cardinality estimates, negative-hit rate, and WAF/rate-limit data. One hashed key dominating hits is hot-key evidence; unique misses tracking request count suggests penetration.

### Immediate mitigation

For hot keys, use safe local replication/near-cache, request coalescing, or shard strategy. For penetration, validate input, rate-limit sources, and short-negative-cache safe absences.

### Permanent fix/design

Design even keys, replicate immutable hot values, use admission filters for known namespaces, enforce tenant budgets, and protect origins.

### Prevention and alerts

Alert on top-key share, per-node skew, miss cardinality, negative-miss surge, and origin queries per cache miss.

### Common mistakes

- adding more cache nodes without changing one key's placement.
- negative-caching transient errors as "not found."
- logging raw sensitive keys.
- trusting a probabilistic filter as an authorization check.

### Interview-ready answer

Hot keys are concentrated valid demand; penetration is repeated absent or uncacheable demand. I distinguish them with privacy-safe frequency, hit/miss cardinality, node skew, and origin calls. I replicate or near-cache safe hot values, while penetration needs validation, fair rate limits, safe short negative caching, and origin protection.

---

# 6. Decision trees

## 6.1 Cache outcome worsens

```text
Are cache operations errors/timeouts?
|-- Yes -> inspect client pool, DNS/TLS/auth, network, node/failover;
|          open circuit and protect origin according to policy.
`-- No
    |
    Are keys absent?
    |-- Yes
    |   |-- expirations high -> TTL cliff or expected expiry.
    |   |-- evictions high -> memory/policy pressure.
    |   |-- new namespace/version -> cold deployment.
    |   `-- high unique nonexistent keys -> penetration.
    `-- No
        |
        Is value version stale?
        |-- Yes -> invalidation, race, event lag, near-cache, source replica.
        `-- No -> metric definition, bypass, decode, or wrong cache layer.
```

## 6.2 Redis is unavailable

```text
Does the operation affect authorization, abuse control, or correctness?
|-- Yes -> revalidate at authority or deny per documented policy.
|         Never grant merely because cache is unavailable.
`-- No
    |
    Is bounded stale data within the freshness contract?
    |-- Yes -> serve stale while limiting refresh.
    `-- No
        |
        Does origin have protected capacity?
        |-- Yes -> fail open through concurrency/rate bulkhead.
        `-- No -> degrade or reject quickly; protect the system.
```

## 6.3 Miss surge

```text
Same small set of keys?
|-- Yes -> stampede/hot keys; single-flight, stale refresh, jitter.
`-- No
    |
    High-cardinality absent keys?
    |-- Yes -> penetration; validate, limit, safe negative cache.
    `-- No
        |
        Expiry or eviction spike?
        |-- expiry -> stagger/jitter TTL and controlled refresh.
        |-- eviction -> memory/working-set/policy investigation.
        `-- neither -> key version, flush, restart, bypass, instrumentation.
```

---

# 7. Final cheat sheet

## Pattern selection

| Need | Starting pattern | Main risk |
|---|---|---|
| Rebuildable read-heavy data | Cache-aside | Miss stampede and invalidation race |
| Centralized transparent loader | Read-through | Hidden origin load/failure behavior |
| Warm reads after writes | Write-through | Dual-write partial failure |
| Buffered high write throughput | Write-behind with durable log | Loss, order, replay, read-after-write |
| Hot data with bounded staleness | Soft TTL/stale-while-revalidate | Serving beyond hard freshness limit |

## Fast evidence sequence

1. Correctness and freshness contract.
2. Cache layer and sanitized key family.
3. Hit, miss, stale, bypass, error, and negative-hit separately.
4. Command latency plus client-pool wait.
5. Expiration, eviction, memory, key count, and TTL distribution.
6. Origin load and miss-to-load amplification.
7. Deploy, key/serializer version, invalidation lag, and failover timeline.

## Production rules

- Cache correctness follows explicit business semantics.
- TTL is not proof of freshness.
- Errors are not misses.
- Protect the origin before failing open.
- Never grant access because a security cache failed.
- Use per-key single-flight; keep waits and leases bounded.
- Add TTL and retry jitter.
- Warm a bounded valuable set at an origin-safe rate.
- Distributed locks reduce duplicates; they do not establish correctness alone.
- Optimize correctness, latency, and origin offload, not hit ratio alone.
