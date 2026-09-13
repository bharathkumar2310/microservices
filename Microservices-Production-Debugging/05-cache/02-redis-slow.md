# Problem

Redis is normally fast because most operations run in memory, but an application can still observe slow cache calls.
"Redis latency" is not one number.
The measured duration can include:

```text
application queue
  + client connection-pool wait
  + DNS/TCP/TLS setup
  + network transit
  + Redis command queue and execution
  + response transfer and deserialization
```

A high application timer does not prove the Redis server executed slowly.
Likewise, a clean Redis `SLOWLOG` does not prove the end-to-end cache path is fast.
This playbook separates command, server, network, and client-pool latency before changing timeouts or capacity.

# Production Situation

At 14:20, checkout reads become slow:

* Traffic is stable at 6,000 requests/min.
* Cache hit ratio remains 93%.
* API p50 rises from 45 ms to 75 ms.
* API p95 rises from 140 ms to 920 ms.
* API p99 rises from 260 ms to 2.4 s.
* Application Redis timer p99 rises from 4 ms to 1.8 s.
* Redis server CPU is 38%.
* Redis `SLOWLOG` shows no command above 8 ms.
* Lettuce pool is 32/32 active with 147 pending borrowers.
* Pool acquisition p99 is 1.72 s.
* Redis command execution p99 from traced spans is 5 ms after connection acquisition.
* One new release creates a blocking Redis client per request and holds pooled connections during JSON conversion.

The cache hit ratio is healthy, yet the cache path is slow.
Because most requests are hits, slow hit latency directly affects most users.

# Architecture

```text
Client
  |
  v
API Gateway
  |
  v
Checkout Service
  |
  +---- application executor queue
  |
  +---- Lettuce connection pool (max 32)
  |          |
  |          +---- wait for connection
  |          |
  |          v
  +------ network / TLS ------> Redis primary
                                  |
                                  +---- command queue
                                  +---- command execution
                                  +---- response encoding
```

Redis processes commands primarily on an event loop.
A single expensive command, large response, fork, persistence event, or host pause can delay otherwise cheap commands.
Client misuse can create the same symptom while the server remains fast.

# What I Check FIRST

1. **Impact and scope.** Compare endpoint, pod, availability zone, command, payload size, and deployment version. One-pod slowness suggests client or node-local networking.
2. **Latency decomposition.** Compare application Redis timer, connection acquisition, traced command duration, server latency, and network round-trip. The largest component directs the next check.
3. **Redis saturation and events.** Check CPU, memory, operations/sec, connected clients, blocked clients, evictions, fork/persistence, replication, and slow commands.
4. **Client pool and timeouts.** Check active, idle, pending, acquisition p95/p99, connection churn, command timeout, and whether connections are returned.
5. **Command shape.** Look for large values, unbounded collection reads, Lua scripts, `KEYS`, big deletes, or high-complexity commands.

# Step-by-Step Investigation

### Step 1 - Confirm user impact and time alignment

* **What I check:** API RED metrics, application Redis duration, error type, and incident start.
* **Why:** A slow cache timer matters only in relation to request latency and traffic.
* **Expected:** API and cache p99 are low and stable.
* **Bad:** Both rise at 14:20 while traffic is unchanged.
* **Meaning:** The cache path is on the critical path.
* **Next:** Split by instance and command.

### Step 2 - Compare percentiles, not averages

* **What I check:** p50, p95, p99, maximum, and histogram count.
* **Why:** A 10 ms average can hide a small group waiting two seconds.
* **Expected:** p50 and tail percentiles move together under broad server pressure.
* **Bad:** p50 is 4 ms while p99 is 1.8 s.
* **Meaning:** Tail queueing, a subset of connections, pods, zones, or large commands is likely.
* **Next:** Segment the tail.

### Step 3 - Segment by pod, zone, command, and key pattern

* **What I check:** `GET`, `MGET`, `HGETALL`, scripts, payload buckets, pod, and zone.
* **Why:** Aggregate data can combine cheap gets with one damaging command.
* **Expected:** Equivalent pods and commands have similar latency.
* **Bad:** Only version 5.7 pods have high pool wait; command time stays low.
* **Meaning:** Recent client code is more likely than Redis host pressure.
* **Next:** Inspect connection-pool metrics.

### Step 4 - Separate pool wait from command duration

* **What I check:** Time from borrow request to connection acquisition and from command write to response.
* **Why:** Many libraries report both under one `redis.operation` timer.
* **Expected:** Pool pending is zero and acquisition p99 is below 2 ms.
* **Bad:** 147 pending, acquisition p99 1.72 s, command p99 5 ms.
* **Meaning:** The bottleneck is before the Redis server executes the command.
* **Next:** Find why connections are occupied or leaked.

### Step 5 - Check connection lifecycle

* **What I check:** Open/close rate, active/idle, borrow/return counts, connection age, and stack traces around pool use.
* **Why:** Creating TLS connections per request is expensive; leaked or long-held connections cause queueing.
* **Expected:** Stable connections, borrow count approximately equals return count, brief hold time.
* **Bad:** Borrow exceeds return or hold time includes CPU-heavy JSON conversion.
* **Meaning:** Client lifecycle or critical-section scope is wrong.
* **Next:** Compare code and profiles with the previous deployment.

### Step 6 - Check client event-loop health

* **What I check:** Lettuce/Netty event-loop utilization, blocked thread stacks, executor queue, GC pauses, and CPU throttling.
* **Why:** Blocking work on an event-loop delays every response handled by that loop.
* **Expected:** Event-loop threads are RUNNABLE briefly and never perform blocking JSON or file work.
* **Bad:** Thread dump shows `lettuce-nioEventLoop` inside a blocking converter.
* **Meaning:** Apparent Redis latency is client-side event-loop starvation.
* **Next:** Move blocking work off the event loop and bound its executor.

### Step 7 - Check timeout layering

* **What I check:** Pool acquisition, connect, command, and caller deadlines.
* **Why:** A 2 s caller deadline with a 3 s pool wait guarantees useless work after the caller gives up.
* **Expected:** Each downstream budget fits inside the upstream deadline with time for fallback.
* **Bad:** Three retries each use a 2 s command timeout inside a 3 s API deadline.
* **Meaning:** Timeouts and retries amplify queueing.
* **Next:** Stop retries that cannot complete inside the remaining budget.

### Step 8 - Inspect Redis server command latency

* **What I check:** Bounded `SLOWLOG`, command statistics, and latency events.
* **Commands:**

```text
redis-cli --tls -h redis.example -p 6379 SLOWLOG LEN
redis-cli --tls -h redis.example -p 6379 SLOWLOG GET 20
redis-cli --tls -h redis.example -p 6379 INFO commandstats
redis-cli --tls -h redis.example -p 6379 LATENCY LATEST
redis-cli --tls -h redis.example -p 6379 LATENCY DOCTOR
```

* **Expected:** Cheap commands have low usec/call and no recent latency event.
* **Bad:** Repeated long Lua, `HGETALL`, or deletion events align with p99 spikes.
* **Meaning:** Server-side work may block unrelated commands.
* **Next:** Identify the owner, key size, and safe replacement.

### Step 9 - Understand what SLOWLOG proves

* **What I check:** The server execution threshold and timestamps.
* **Why:** `SLOWLOG` excludes network transfer and client waiting.
* **Expected:** It complements, not replaces, client and trace metrics.
* **Bad:** Operators conclude "Redis is fast" solely because `SLOWLOG` is empty.
* **Meaning:** The investigation may miss pool or network latency.
* **Next:** Compare Redis-side and client-side timestamps.

### Step 10 - Inspect memory and eviction pressure

* **What I check:** `used_memory`, `used_memory_rss`, fragmentation, `maxmemory`, evictions, and allocator behavior.
* **Why:** Memory pressure can trigger eviction work, swapping at the host, or failed writes.
* **Expected:** No host swap, headroom, stable RSS, and intended eviction policy.
* **Bad:** RSS greatly exceeds logical memory or host swaps.
* **Meaning:** Allocator fragmentation or host memory contention can cause pauses.
* **Next:** Inspect node metrics and key/value growth by approved sampling.

### Step 11 - Check large keys and response sizes safely

* **What I check:** Known suspect keys with `STRLEN`, `HLEN`, `LLEN`, `SCARD`, `ZCARD`, or `MEMORY USAGE`.
* **Why:** A large `HGETALL` can be fast enough to miss `SLOWLOG` yet slow to transfer and deserialize.
* **Expected:** Values remain within documented size budgets.
* **Bad:** One hash contains 800,000 fields or a value is 12 MB.
* **Meaning:** Network and serialization dominate, and the command can block the event loop.
* **Next:** Paginate, shard, or redesign the cached representation.

```text
redis-cli --tls -h redis.example -p 6379 TYPE "cart:v4:tenant-42:817"
redis-cli --tls -h redis.example -p 6379 MEMORY USAGE "cart:v4:tenant-42:817"
redis-cli --tls -h redis.example -p 6379 HLEN "cart:v4:tenant-42:817"
```

### Step 12 - Check dangerous command patterns

* **What I check:** Commandstats, ACL/audit data, and trace attributes for `KEYS`, broad scans, scripts, transactions, and bulk deletes.
* **Why:** Some operations monopolize Redis long enough to delay unrelated reads.
* **Expected:** Production code uses bounded commands and cursor iteration where required.
* **Bad:** A maintenance job runs `KEYS session:*` every minute.
* **Meaning:** A blocking command creates periodic latency waves.
* **Next:** Disable the job safely and replace it with indexed ownership or bounded scanning.

### Step 13 - Check persistence and fork pauses

* **What I check:** RDB/AOF status, last save, rewrite/fork duration, copy-on-write memory, and disk latency.
* **Why:** Persistence events can create latency even with moderate command CPU.
* **Expected:** Fork and fsync durations fit the SLO.
* **Bad:** p99 spikes align with AOF rewrite or fork.
* **Meaning:** Dataset size, write rate, or storage performance affects the server.
* **Next:** Tune persistence based on durability requirements, not by disabling it casually.

### Step 14 - Check replication and topology

* **What I check:** Role, replication lag, link state, failover events, redirects, and which endpoint each pod uses.
* **Why:** Cross-region replicas, repeated `MOVED`/`ASK`, or failover reconnects add latency.
* **Expected:** Clients use topology-aware routing and local approved endpoints.
* **Bad:** One zone resolves to a remote region or receives redirect storms.
* **Meaning:** Network/topology, not command complexity, drives the tail.
* **Next:** Correct discovery and cluster client configuration.

### Step 15 - Measure network and TLS separately

* **What I check:** DNS time, connect time, TLS handshake, retransmits, packet loss, and round-trip by zone.
* **Why:** Network delay is invisible to server command duration.
* **Expected:** Persistent connections avoid handshakes on ordinary operations.
* **Bad:** New connections/min jumps and TLS handshake p99 is 300 ms.
* **Meaning:** Connection churn or path degradation causes end-to-end slowness.
* **Next:** Reuse connections and engage platform networking with time-correlated evidence.

### Step 16 - Validate serialization and local resource pressure

* **What I check:** Payload bytes, compression, deserialize duration, JVM CPU, GC, and thread pools.
* **Why:** The timer may wrap deserialization after Redis returned.
* **Expected:** Deserialization is a small bounded portion.
* **Bad:** A 10 MB value takes 600 ms to allocate and parse, causing GC.
* **Meaning:** Cache representation is too large or the timer is mislabeled.
* **Next:** Store a smaller projection and instrument phases independently.

### Step 17 - Mitigate without moving the bottleneck

* **Immediate actions:** Roll back the client regression, disable the offending job, cap large operations, and shed optional cache work.
* **Pool increase:** Only after proving Redis and network have headroom and connection count is the actual safe limit.
* **Timeout increase:** Only when the operation legitimately needs longer and the end-to-end deadline permits it.
* **Fallback:** Bound DB concurrency; an unbounded cache bypass can overload MySQL.
* **Next:** Verify every latency component.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| Application Redis p50/p95/p99 | End-to-end client experience; high tail signals queueing or subsets |
| Pool acquisition p95/p99 | High means waiting before a command can be sent |
| Active/idle/pending connections | Max active plus pending indicates client saturation |
| Borrow versus return | Growing divergence suggests leaks or long holds |
| Connect/TLS latency | High with churn indicates lifecycle or network problems |
| Command execution p95/p99 | High points toward server command work |
| Redis ops/sec | Traffic context; a spike can create queueing |
| Redis CPU | High suggests command load, but low CPU does not exclude blocking |
| Blocked clients | High can be expected for blocking commands or indicate misuse |
| Connected clients | Sudden growth suggests churn or pool proliferation |
| Network bytes and retransmits | Large values or loss explain transfer delay |
| Payload size histogram | Tail growth often explains only some commands being slow |
| `evicted_keys` rate | Memory-pressure work and reduced hit quality |
| Fork/AOF/RDB duration | Correlation with spikes suggests persistence pauses |
| JVM GC and executor queue | High values can falsely look like Redis slowness |

If application p99 is high but command p99 is low, inspect pool, network, and serialization.
If both application and server command p99 rise, inspect command mix, server CPU, memory, and persistence.
If only one zone is slow, inspect routing and network before changing Redis.
If p99 changes immediately after deployment, compare pool lifecycle and instrumentation boundaries.

# Distributed Trace Investigation

```text
traceId=6ac903
Gateway                              2,410 ms
  Checkout Service                   2,380 ms spanId=chk20
    redis.connection.acquire         1,721 ms spanId=pool8
    redis.GET                            5 ms spanId=red44 db.system=redis
    cache.deserialize                  612 ms spanId=json9 bytes=10485760
    Pricing Service                     28 ms spanId=pr12
```

This trace locates 2.333 seconds outside Redis command execution.
The parent service span is server latency.
The gateway sees client latency to checkout.
Child spans split downstream, pool, and local conversion time.
Retry spans would show repeated GET attempts and explain extra load.
A missing Redis child can mean a local-cache hit, bypass, sampling, unsupported instrumentation, or failure before sending.
I do not infer a server problem from the parent span alone.

# Distributed Logs

```text
2026-09-13T14:23:51.028Z WARN service=checkout-service instance=checkout-5.7-b41
traceId=6ac903 spanId=pool8 requestId=req-912 endpoint=/checkout/summary
downstream=redis event=connection_acquire_slow poolActive=32 poolMax=32
poolPending=147 acquireMs=1721 command=GET
```

```text
2026-09-13T14:23:51.646Z INFO service=checkout-service instance=checkout-5.7-b41
traceId=6ac903 spanId=red44 endpoint=/checkout/summary downstream=redis
command=GET commandMs=5 responseBytes=10485760 deserializeMs=612 totalLatencyMs=2338
```

I correlate `traceId`, then use `spanId` to match pool and command phases.
The first log proves pool saturation for this request, not why connections were held.
Thread profiles, borrow/return metrics, deployment diff, and repeated traces prove the lifecycle regression.
Logs must avoid raw values, credentials, and sensitive keys.

# Commands / Tools

```text
redis-cli --tls -h redis.example -p 6379 PING
redis-cli --tls -h redis.example -p 6379 INFO server
redis-cli --tls -h redis.example -p 6379 INFO clients
redis-cli --tls -h redis.example -p 6379 INFO stats
redis-cli --tls -h redis.example -p 6379 INFO memory
redis-cli --tls -h redis.example -p 6379 INFO persistence
redis-cli --tls -h redis.example -p 6379 INFO replication
redis-cli --tls -h redis.example -p 6379 SLOWLOG GET 20
redis-cli --tls -h redis.example -p 6379 LATENCY LATEST
```

`INFO` is point-in-time/cumulative evidence and should be compared with monitoring history.
`PING` does not prove normal p99 or that the application pool is healthy.
`SLOWLOG` does not include network or pool wait.
Use `redis-cli --latency` only from an approved representative host for a bounded observation.
Use Spring Boot `/actuator/metrics` or `/actuator/prometheus` for client pool and timer series.
Use `/actuator/threaddump` under approved access to detect blocked event-loop threads.
Avoid `MONITOR`, `KEYS *`, unbounded collection reads, and large diagnostic scans on production.

# Root Cause

Release 5.7 held a pooled connection while converting a large cached cart response.

```text
Connection held beyond network read
  -> 612 ms JSON conversion occurs inside pool lease
  -> 32 connections stay occupied
  -> 147 requests wait for a connection
  -> acquisition p99 reaches 1.72 s
  -> application labels entire wait as Redis latency
  -> API p99 reaches 2.4 s
```

Redis command p99 of 5 ms and empty relevant `SLOWLOG` entries exclude command execution as the primary bottleneck.
The large value is a contributing design issue; the new connection scope is the deployment-triggered root cause.

# Fix

Immediate mitigation:

* Roll back 5.7.
* Rate-limit the affected summary endpoint if rollback takes time.
* Disable optional large cart enrichment.
* Keep DB fallback bounded to protect MySQL.

Permanent fix:

* Return the connection immediately after the response bytes are read.
* Deserialize outside the pool lease and off the Netty event loop.
* Cache a smaller summary projection rather than a 10 MB object.
* Reuse long-lived client resources and instrument acquisition, command, transfer, and deserialize separately.
* Set connect, acquisition, command, and caller timeouts from one deadline budget.
* Size the pool using measured concurrency and Redis connection limits, not guesswork.

# Verification

| Signal | Before | After |
|---|---:|---:|
| Application Redis p99 | 1.8 s | 7 ms |
| Pool acquisition p99 | 1.72 s | 1.4 ms |
| Pool active | 32/32 | 6/32 |
| Pool pending | 147 | 0 |
| Command p99 | 5 ms | 4 ms |
| Deserialize p99 | 612 ms | 18 ms |
| API p99 | 2.4 s | 245 ms |
| Error rate | 4.2% | 0.2% |

I verify under equal traffic and with large-cart requests included.
I watch at least one peak period and a failover drill.
I confirm borrow and return counts remain balanced and connection churn stays low.
I also verify cache values and tenant isolation remain correct.

# Prevention

* Dashboard client, network, and server latency separately.
* Alert on pool pending, acquisition p99, connection churn, and borrow/return divergence.
* Add payload-size budgets and reject unbounded cache objects.
* Load-test p99 with realistic value distributions, not only tiny keys.
* Ban blocking work on Redis/Netty event-loop threads.
* Review timeout budgets and retry count together.
* Record Redis persistence and failover events beside application latency.
* Restrict dangerous commands with ACLs.
* Canary client-library and pooling changes.
* Test degraded Redis with bounded DB fallback.
* Keep runbooks for large keys, failover, and safe diagnostics.

# Interview Answer

### What I would say in an interview

When Redis appears slow, I first separate the application timer into pool acquisition, connect/TLS, network transfer, server command, and deserialization. I compare p50, p95, and p99 by pod, zone, command, and payload. Then I use Redis CPU, memory, commandstats, latency events, persistence, replication, and a bounded `SLOWLOG` sample. In this incident the app p99 was 1.8 seconds, but command p99 was 5 ms and pool acquisition was 1.72 seconds. The release held connections during JSON conversion. I rolled it back, bounded fallback, fixed connection scope and payload size, then verified pool pending, API p99, and errors at equal traffic.

### Common interviewer traps

* Calling all client-observed time "Redis server latency."
* Using average latency instead of tails.
* Assuming an empty `SLOWLOG` clears the network and client.
* Increasing pool size without checking Redis limits and leak behavior.
* Increasing timeouts while queues continue growing.
* Falling back to MySQL without a bulkhead.

### Quick memory flow

```text
Scope
  -> percentiles
  -> pool wait
  -> network/TLS
  -> command/server
  -> payload/deserialize
  -> persistence/topology
  -> mitigate
  -> fix
  -> verify
```

# Interview Follow-up Questions

### 1. Why can Redis be slow with low CPU?

A blocking command, fork pause, network delay, pool wait, event-loop starvation, or large response can create latency without sustained high CPU.

### 2. What exactly does `SLOWLOG` measure?

Redis command execution on the server after the command is available for execution; it excludes client, network, and response-transfer time.

### 3. Should every Redis client use a pool?

No. Some multiplexed clients safely share connections. Use the client's documented model and isolate blocking operations where necessary.

### 4. Why are p95 and p99 important?

Queueing and large keys affect a minority of calls. Averages hide those user-visible tails.

### 5. When would you increase the connection pool?

Only after showing genuine concurrency demand, balanced returns, short holds, Redis headroom, and acceptable connection limits.

### 6. How do retries affect slow Redis?

They add commands and pool demand while the dependency is already delayed, often increasing queueing and tail latency.

### 7. Why can a large GET miss `SLOWLOG`?

Key lookup may be quick while transferring and deserializing a large response is slow.

### 8. What is a safe immediate fallback?

Serve a bounded stale value when correctness allows, shed optional work, or use rate-limited DB fallback with a concurrency bulkhead.
