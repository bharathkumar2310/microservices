# Problem

Redis unavailable means an application cannot complete required Redis operations within its deadline.
The cause can be DNS, TCP, TLS, authentication, client-pool exhaustion, server failure, cluster failover, or a network partition.
Availability policy must follow correctness:

* **Fail open** means continue without Redis, usually through a bounded database fallback or a safe stale value.
* **Fail closed** means reject the operation because proceeding could violate security, uniqueness, ordering, quota, or data integrity.

A product-description cache can often fail open.
A Redis-backed idempotency key, rate-limit decision, distributed lock, or session may need fail closed or a separately designed authoritative fallback.
Calling every Redis use "just a cache" is dangerous.
Blindly flushing or restarting Redis is not a diagnosis and can worsen recovery.

# Production Situation

At 09:42 the managed Redis primary becomes unreachable during a network change:

* Checkout traffic is 4,000 requests/min.
* Normal cache hit ratio is 96%.
* Redis connection errors jump from 0 to 3,800/min.
* Application Redis p99 reaches its 800 ms command timeout.
* Each request retries twice, so attempted Redis calls reach about 12,000/min.
* Product DB reads rise from 160/min to 3,200/min before bulkheads engage.
* Hikari active reaches 40/40 and pending reaches 121.
* API p99 rises from 280 ms to 5.2 s.
* Product display can tolerate five minutes of staleness.
* Checkout idempotency and inventory reservation cannot safely guess.
* Redis recovers through failover in 75 seconds, but reconnect storms continue for three minutes.

The response must differ by data class.
Failing open for product display protects availability.
Failing closed for reservation ownership protects correctness.

# Architecture

```text
                          +--> Product cache keys
                          |      fail open: bounded DB/stale
Client -> Gateway -> Checkout Service -> Redis cluster
                          |      fail closed: no unsafe guess
                          +--> Idempotency / lock keys
                          |
                          +--> HikariCP -> MySQL
                                   ^
                                   |
                          bounded fallback only
```

Cache-aside product reads use MySQL as authority.
Redis-backed coordination is not equivalent to disposable cached data.
The runbook starts by classifying the operation before choosing fallback.

# What I Check FIRST

1. **Customer impact and operation class.** Identify which Redis-backed features fail and whether stale/bypass behavior is correct. This prevents a well-intended fail-open from creating duplicate charges or cross-tenant data.
2. **Failure layer.** Read the exact error: DNS, connection refused, connect timeout, TLS, `NOAUTH`, command timeout, pool timeout, `MOVED`, or `READONLY`. Each belongs to a different stage.
3. **Scope and topology.** Compare pods, zones, nodes, primary/replica, endpoints, and deployment versions. Partial impact often points to routing, stale DNS, or one client configuration.
4. **Amplification.** Check retries, Redis attempts, fallback DB rate, Hikari pending, worker queues, and caller deadlines. The secondary overload may outlast Redis failure.
5. **Provider/failover events.** Correlate role change, replication, maintenance, certificates, ACL rotation, and network policy with the first failure.

# Step-by-Step Investigation

### Step 1 - Declare the affected capability, not only "Redis down"

* **What I check:** Endpoints, use cases, tenants, regions, and error/success rates.
* **Why:** Product reads, sessions, locks, and idempotency have different correctness needs.
* **Expected:** A dependency map identifies every Redis use and owner.
* **Bad:** Teams enable a global bypass without knowing what Redis protects.
* **Meaning:** Mitigation could be more harmful than the outage.
* **Next:** Classify each use as disposable cache, durable data, or coordination.

### Step 2 - Decide fail-open versus fail-closed explicitly

| Use | Default during outage | Reason |
|---|---|---|
| Public product description | Bounded stale or DB fallback | Slight staleness is acceptable |
| Tenant-specific price | Bounded authoritative lookup | Wrong price is not acceptable |
| Authentication session | Usually fail closed | Cannot establish identity safely |
| Idempotency key | Fail closed or authoritative DB design | Guessing can duplicate side effects |
| Inventory lock | Fail closed | Concurrent writes can oversell |
| Optional recommendation | Omit feature | Availability matters more than enrichment |

* **Expected:** Policy is documented and tested.
* **Bad:** Generic exception handler treats all keys the same.
* **Meaning:** Outage behavior is undefined.
* **Next:** Apply the safest pre-approved mode per capability.

### Step 3 - Capture exact errors and timestamps

* **What I check:** Exception class, message, remote endpoint, phase, timeout, first occurrence, and pod.
* **Why:** `UnknownHostException`, `ConnectException`, TLS alert, `NOAUTH`, command timeout, and pool timeout are not interchangeable.
* **Expected:** One dominant failure signature aligns with the incident.
* **Bad:** Many error strings are collapsed into `CACHE_ERROR`.
* **Meaning:** Observability cannot locate the failed layer.
* **Next:** Use traces and client metrics to reconstruct the phase.

### Step 4 - Walk the connection progression

```text
DNS
  -> IP route
  -> TCP connection
  -> TLS handshake
  -> Redis authentication
  -> cluster routing
  -> command
  -> response
```

* DNS failure means no usable address was obtained.
* Connection refused means the destination actively rejected TCP, often no listener or wrong port.
* Connection timeout means the TCP handshake did not complete, often routing, firewall, or silent drop.
* TLS failure means TCP worked but identity/protocol/certificate negotiation failed.
* `NOAUTH` or `WRONGPASS` means protocol connectivity worked but credentials/ACL failed.
* Command timeout means the earlier stages may have succeeded but no response arrived in budget.
* **Next:** Test only from the affected runtime network, not merely a laptop.

### Step 5 - Check DNS safely

Windows:

```text
Resolve-DnsName redis.example.internal
nslookup redis.example.internal
```

Linux:

```text
dig redis.example.internal
getent hosts redis.example.internal
```

* **Expected:** Approved addresses and TTL.
* **Bad:** NXDOMAIN, timeout, or different answers across pods/zones.
* **Meaning:** Discovery or resolver path failed before TCP.
* **Next:** Compare application resolver behavior and endpoint TTL; do not hard-code a transient primary IP.

### Step 6 - Check TCP reachability

Windows:

```text
Test-NetConnection redis.example.internal -Port 6379
```

Linux:

```text
nc -vz -w 3 redis.example.internal 6379
```

* **Expected:** TCP succeeds from an approved host in the same network path.
* **Bad:** Refused indicates reachable destination without an accepting listener; timeout indicates no completed handshake.
* **Meaning:** TCP evidence narrows network/listener failure but does not prove TLS, authentication, Redis health, or application-pool health.
* **Next:** Perform an authenticated TLS Redis check.

### Step 7 - Check protocol and authentication

```text
redis-cli --tls -h redis.example.internal -p 6379 PING
```

* **Expected:** `PONG` through the approved secret mechanism.
* **Bad:** Certificate error, `NOAUTH`, `WRONGPASS`, timeout, or redirect loop.
* **Meaning:** The response identifies TLS, credential, server, or cluster configuration failure.
* **Next:** Compare certificate expiry/SAN, ACL rotation time, and client trust configuration.

Never paste credentials on the command line where process listings or shell history can expose them.
Use the platform's approved secret injection and a read-only diagnostic identity.

### Step 8 - Determine scope by pod and zone

* **What I check:** Error rate by instance, node, zone, client version, and resolved IP.
* **Why:** A global Redis failure affects all paths; one-zone impact suggests networking or stale endpoint state.
* **Expected:** Scope matches provider topology.
* **Bad:** Zone A fails while zone B succeeds against the same logical endpoint.
* **Meaning:** Zone routing, DNS cache, network policy, or NAT exhaustion is more likely than total server failure.
* **Next:** Compare routes and connection telemetry between zones.

### Step 9 - Inspect Redis topology and failover

* **What I check:** Primary role, replica link, failover start/end, replication offset/lag, cluster state, and application redirects.
* **Why:** During promotion, old connections may see `READONLY`, resets, or topology redirects.
* **Expected:** New primary is healthy and clients refresh topology promptly.
* **Bad:** Clients keep writing to the old primary or retry `MOVED` indefinitely.
* **Meaning:** Client discovery/failover configuration extends the outage.
* **Next:** Refresh topology through supported mechanisms and correct endpoint use.

### Step 10 - Inspect client connection behavior

* **What I check:** Active, idle, pending, connect rate, reconnect attempts, backoff, and jitter.
* **Why:** Thousands of clients reconnecting simultaneously can overload a recovered cluster.
* **Expected:** Exponential backoff with jitter and bounded pending work.
* **Bad:** Every pod reconnects every 10 ms with no jitter.
* **Meaning:** A reconnect storm delays recovery.
* **Next:** Cap attempts, add jitter, and shed noncritical requests.

### Step 11 - Inspect retries and deadline budgets

* **What I check:** Attempts per request, timeout per attempt, remaining caller deadline, and retryable error list.
* **Why:** Three 800 ms attempts consume 2.4 seconds before fallback and triple load.
* **Expected:** At most a small bounded retry for transient, idempotent operations within the deadline.
* **Bad:** Retries occur on auth failures, pool exhaustion, or after the caller timed out.
* **Meaning:** Retries cannot repair the fault and amplify it.
* **Next:** Disable inappropriate retries and use a circuit breaker.

### Step 12 - Protect the database fallback

* **What I check:** DB calls, Hikari active/idle/pending, acquisition latency, DB CPU, and queue depth.
* **Why:** At 4,000 requests/min, bypassing a 96% cache changes DB reads from 160/min toward 4,000/min, 25x.
* **Expected:** A bulkhead limits fallback below tested DB headroom.
* **Bad:** Hikari reaches 40/40 and pending reaches 121.
* **Meaning:** Cache unavailability has become a database and thread-pool incident.
* **Next:** Rate-limit or shed fallback rather than increasing the pool blindly.

### Step 13 - Use bounded stale data when correct

* **What I check:** Last refresh time, data classification, maximum acceptable staleness, and tenant scope.
* **Why:** A known five-minute-old product description may be better than an outage.
* **Expected:** Stale serving is explicit, observable, and time-bounded.
* **Bad:** Stale price, permission, or revocation data is served indefinitely.
* **Meaning:** Availability policy violates business correctness.
* **Next:** Fail closed or query an authority within a bulkhead for sensitive fields.

### Step 14 - Treat distributed locks cautiously

* **What I check:** Lease duration, renewal, pause behavior, owner token, and guarded resource.
* **Why:** A client can pause past lease expiry and continue believing it owns the lock.
* **Bad sequence:**

```text
Worker A obtains lease token 41
Worker A pauses
Lease expires
Worker B obtains token 42 and writes
Worker A resumes and writes stale data
```

* **Meaning:** Mutual exclusion alone is insufficient under pauses/partitions.
* **Next:** Use a monotonically increasing fencing token and make the authoritative resource reject token 41 after token 42.

Redis locks do not create cross-system transactions.
Lock safety depends on assumptions about time, failure, and the protected store.
For correctness-critical ownership, prefer a database constraint, conditional update, queue partition, or consensus-backed coordinator where appropriate.

### Step 15 - Investigate data after recovery

* **What I check:** Whether keys survived, replica promotion lost acknowledged writes, cache is cold, and versions are compatible.
* **Why:** Transport recovery does not guarantee warm or correct data.
* **Expected:** Disposable entries refill at a controlled rate; correctness data is reconciled from authority.
* **Bad:** All clients warm the same keys at once or stale replica data wins.
* **Meaning:** Recovery can trigger stampede or correctness defects.
* **Next:** Warm high-value keys gradually and reconcile critical state explicitly.

### Step 16 - Separate mitigation from root cause

* **Mitigation:** Circuit-break noncritical Redis calls, serve bounded stale data, omit optional features, cap DB fallback, and use reconnect backoff.
* **Root repair:** Correct the network policy and fix retry/reconnect behavior.
* **Not a fix:** Raising command timeout, adding unlimited connections, or flushing Redis.
* **Next:** Verify transport, application behavior, fallback, and data correctness.

# Metrics to Check

| Metric | High/increasing suggests | Low/decreasing suggests |
|---|---|---|
| Redis connection errors | DNS/TCP/TLS/auth/topology failure | Connectivity recovered |
| Redis timeout rate | No response in budget or client queueing | Healthy only if traffic remains |
| Retry attempts/request | Amplification and longer user latency | Controlled failure behavior |
| Circuit state/open count | Dependency isolated | Closed may be healthy or misconfigured |
| Client active/pending | Pool saturation or reconnect backlog | Headroom |
| Reconnect rate | Topology churn or outage | Stable connections |
| Redis connected clients | Reconnect storm or pool multiplication | May fall during network partition |
| Fallback DB rate | Cache bypass impact | Stale/omit strategy or lost traffic |
| Hikari active/pending | Database fallback saturation | DB headroom |
| API p50/p95/p99 | User impact and timeout tails | Recovery if success also improves |
| Error rate by operation class | Fail-closed impact | Could hide unsafe fail-open behavior |
| Stale-response age | Risk approaching business limit | Fresh or recently cached data |
| Failover and replication lag | Promotion/recovery risk | Stable topology |

If Redis errors rise before DB pending, Redis is the initiating failure.
If DB pending stays high after Redis recovers, fallback work or retries are prolonging the incident.
If error rate is low but stale age grows beyond policy, the apparent success is unsafe.
If only one zone fails, aggregate availability can hide a routing problem.

# Distributed Trace Investigation

```text
traceId=91be20
Gateway                                  5,210 ms
  Checkout Service                       5,180 ms spanId=co71
    Redis GET attempt=1                    801 ms spanId=r11 error=timeout
    Redis GET attempt=2                    802 ms spanId=r12 error=timeout
    Redis GET attempt=3                    801 ms spanId=r13 error=timeout
    Hikari acquire                       1,903 ms spanId=h44
    MySQL product lookup                   821 ms spanId=d88
```

The repeated sibling spans prove retry amplification.
The DB query span is 821 ms, while Hikari acquisition is 1.903 seconds, so "DB slow" is incomplete.
Gateway client latency includes all checkout work.
Checkout server latency shows its entire request.
A missing Redis child span can indicate fail-fast circuit breaker, local stale response, missing instrumentation, or failure before send.
I inspect span attributes such as exception type, peer address, attempt, cache policy, fallback result, and remaining deadline.

# Distributed Logs

```text
2026-09-13T09:42:07.611Z ERROR service=checkout-service instance=checkout-a-17
traceId=91be20 spanId=r11 requestId=req-140 endpoint=/checkout
downstream=redis peer=redis.example.internal:6379 operation=GET attempt=1
error=RedisCommandTimeoutException timeoutMs=800 latencyMs=801 zone=zone-a
```

```text
2026-09-13T09:42:10.141Z WARN service=checkout-service instance=checkout-a-17
traceId=91be20 spanId=co71 endpoint=/checkout cachePolicy=fail-open-bounded
fallback=mysql fallbackResult=pool-timeout poolActive=40 poolPending=121
totalLatencyMs=5180
```

I correlate logs using trace ID and distinguish each attempt by span ID.
The timeout log proves this client received no command response in 800 ms.
It does not prove whether network, failover, server pause, or client queueing caused it.
Provider events, topology state, network evidence, and cross-zone comparison establish the root cause.

# Commands / Tools

```text
Windows network stages:
Resolve-DnsName redis.example.internal
Test-NetConnection redis.example.internal -Port 6379

Linux network stages:
dig redis.example.internal
getent hosts redis.example.internal
nc -vz -w 3 redis.example.internal 6379

Safe Redis observations:
redis-cli --tls -h redis.example.internal -p 6379 PING
redis-cli --tls -h redis.example.internal -p 6379 INFO replication
redis-cli --tls -h redis.example.internal -p 6379 INFO clients
redis-cli --tls -h redis.example.internal -p 6379 INFO stats
```

DNS commands prove resolver answers, not TCP connectivity.
TCP commands prove only that a connection can be established from that source at that moment.
`PING` proves protocol response for the diagnostic identity, not application credentials or business correctness.
`INFO replication` exposes role/link facts but does not prove clients refreshed topology.
Spring Boot `/actuator/health` may show Redis down, but a green overall endpoint does not prove checkout correctness.
`/actuator/metrics`, `/actuator/prometheus`, and controlled traces provide rates and latency.
Do not use `FLUSHALL`, `FLUSHDB`, `KEYS *`, or `MONITOR` as outage diagnostics.

# Root Cause

A network policy rollout silently dropped traffic from zone A to the Redis endpoint.
The client then amplified the failure with two retries and synchronized reconnects.

```text
Network policy drops TCP packets
  -> connect/command attempts time out
  -> each request performs three attempts
  -> worker threads and client pending queue grow
  -> fail-open product reads reach MySQL
  -> DB reads increase up to 25x
  -> Hikari reaches 40/40
  -> fallback also times out
  -> API p99 and errors rise
```

Zone B remained healthy, the policy timestamp matched the first failure, and zone A TCP tests timed out.
Redis itself did not fail globally.
Retry and fallback design were contributing causes that increased impact and recovery time.

# Fix

Immediate mitigation:

* Roll back the network policy.
* Open the circuit for optional Redis operations to stop futile retries.
* Serve bounded stale product descriptions and omit recommendations.
* Fail closed for idempotency and reservation operations without a safe authority.
* Limit concurrent MySQL fallback below tested capacity.
* Add exponential reconnect backoff with jitter.

Permanent fix:

* Add pre-deployment connectivity tests from every zone.
* Use one deadline budget and retry only transient, idempotent operations.
* Document fail-open/fail-closed behavior by key namespace.
* Add a stale-value path with tenant-safe versioned keys and maximum age.
* Protect DB fallback with rate and concurrency limits.
* Use fencing tokens or authoritative conditional writes for critical coordination.
* Test failover, network partition, cold recovery, and credential rotation.

# Verification

| Signal | During incident | After fix |
|---|---:|---:|
| Redis connection success | 5% in zone A | >99.99% |
| Redis attempts/request | 3.0 | 1.01 |
| Reconnect rate/pod | 100/s | <0.1/s steady state |
| Fallback DB reads | 3,200/min | 150/min |
| Hikari active | 40/40 | 7/40 |
| Hikari pending | 121 | 0 |
| API p99 | 5.2 s | 270 ms |
| Error rate | 16% | 0.2% |
| Unsafe stale responses | Unknown | 0 |

I verify from each zone and from application pods, not only an operator host.
I confirm product fallback, idempotency, sessions, and inventory each follow their intended policy.
I watch through DNS TTL, a connection recycle, and a controlled failover.
I reconcile correctness-critical operations created during the incident.

# Prevention

* Maintain an inventory of Redis namespaces and correctness classification.
* Alert separately on connect, TLS, auth, command, pool, and redirect errors.
* Dashboard retry attempts, circuit state, fallback DB load, and stale age.
* Add zone-specific synthetic Redis protocol checks with read-only credentials.
* Canary network, certificate, ACL, and topology changes.
* Use reconnect and retry jitter.
* Cap pending requests and DB fallback concurrency.
* Load-test complete cache loss, not only ordinary misses.
* Run failover and network-partition game days.
* Require fencing or authoritative conditional writes for critical locks.
* Keep secrets out of logs and commands.
* Never make a health endpoint the only proof of business readiness.

# Interview Answer

### What I would say in an interview

For Redis unavailability, I first classify what Redis is doing. A product cache may fail open through bounded stale or database fallback, while idempotency, sessions, or inventory locks may need fail closed. Then I identify the failed layer from DNS through TCP, TLS, auth, topology, command, and response, comparing pods and zones. I watch retries and fallback because losing a 96% hit cache can increase DB reads 25 times. In this case a zone network policy caused timeouts, while retries and unbounded fallback amplified impact. I rolled back the policy, opened the circuit, bounded fallback, and then fixed retry, reconnect, and correctness policies before verifying every operation class.

### Common interviewer traps

* Saying "always fail open because Redis is only a cache."
* Treating TCP success as proof that authentication and commands work.
* Retrying authentication errors or every timeout.
* Sending all misses to MySQL without a bulkhead.
* Assuming failover completion means clients and data are recovered.
* Trusting a green health endpoint over business checks.

### Quick memory flow

```text
Classify correctness
  -> scope
  -> exact error
  -> DNS/TCP/TLS/auth
  -> topology
  -> retry and reconnect
  -> bounded fallback
  -> recovery correctness
  -> verify
```

# Interview Follow-up Questions

### 1. What is fail open?

Continue with a safe degraded path, such as bounded authoritative lookup or time-limited stale content, when correctness allows.

### 2. When should Redis failure fail closed?

When proceeding cannot safely establish identity, authorization, uniqueness, idempotency, quota, or ownership.

### 3. Why can complete cache loss overload MySQL?

At a 96% hit rate, MySQL normally receives 4% of reads. Full bypass can approach 100%, a 25x increase before retries.

### 4. Why is connection timeout different from refused?

Timeout means the TCP handshake did not complete; refused means the destination was reachable and actively rejected the connection.

### 5. What does a circuit breaker add?

It stops repeated calls to a known-failing dependency, preserves threads and deadlines, and lets a bounded probe determine recovery.

### 6. Are Redis distributed locks completely safe?

No. Lease expiry, pauses, partitions, and stale owners require fencing or enforcement by the authoritative resource.

### 7. Why use reconnect jitter?

It prevents all application instances from reconnecting simultaneously and overwhelming a newly recovered Redis node.

### 8. Why not increase timeouts?

Longer waits consume threads and caller budgets without repairing DNS, policy, auth, or topology failures.
