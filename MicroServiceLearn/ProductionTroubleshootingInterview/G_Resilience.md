# Production Troubleshooting Study Chapter: Resilience

## Purpose and safety

Resilience is the ability to preserve defined critical outcomes under partial failure and to recover without spreading damage. It is not "never fail," and it is not a pile of retry annotations. Every protection mechanism spends something: timeouts abandon work, retries add load, breakers reject calls, bulkheads reserve capacity, and fallbacks may reduce correctness.

Never enable blanket retries for writes, raise timeouts without a capacity model, or turn errors into success-shaped empty/default responses. During an incident, preserve idempotency, authoritative state, auditability, and a bounded recovery path.

## Learning goals

After studying this chapter, you should be able to:

1. Build an end-to-end deadline budget and distinguish connect, acquisition, attempt, and total timeouts.
2. Decide retry eligibility from failure semantics, idempotency, and remaining deadline.
3. Design exponential backoff with jitter, retry budgets, and `Retry-After`.
4. Explain and prevent multi-layer retry amplification and retry storms.
5. Tune circuit-breaker states, windows, thresholds, minimum volume, and half-open probes.
6. Combine bulkheads, concurrency limits, bounded queues, load shedding, and rate limits.
7. Design truthful fallbacks and explicit graceful degradation.
8. Diagnose cascading failure and thundering-herd behavior using production evidence.

---

# 1. Mental model

## 1.1 Failure is normal; unbounded waiting is not

```text
caller deadline
   |
   +-- admission/queue wait
   +-- attempt 1: acquire connection + connect/TLS + server work + response
   +-- backoff
   +-- attempt 2
   +-- response processing
```

The whole operation must fit one **deadline**. A timeout limits a phase or attempt; a deadline is the latest useful completion time. If each layer independently waits and retries, the actual latency and work can greatly exceed the user budget.

```text
remaining = deadline - monotonic_now
attempt_timeout <= remaining - response_and_safety_margin
```

Propagate the deadline or remaining budget. Stop queued/running work when cancellation is safe and useful. A caller timing out does not prove the server stopped: it may commit later, creating an ambiguous result.

## 1.2 Protection hierarchy

```text
admission control
  -> rate limit fair share and abusive/excess arrival rate
  -> concurrency limit cap in-flight work
  -> bounded queue absorb only short bursts
  -> load shed work that cannot meet its objective
  -> bulkhead isolate resources by dependency/priority/tenant
  -> timeout/deadline bound waiting
  -> bounded eligible retry handle rare transient failure
  -> circuit breaker fail fast while a dependency is unhealthy
  -> truthful fallback/degradation preserve an explicitly weaker outcome
```

These mechanisms complement rather than replace one another. A breaker does not cap concurrency. A timeout does not stop arrivals. A rate limit does not isolate a connection pool. A fallback is not correct merely because it returns `200`.

## 1.3 Retry decision

Retry only when all are true:

```text
failure is plausibly transient
AND operation is safe to repeat or durably idempotent
AND next attempt has a meaningful chance on a different condition
AND total deadline has enough budget
AND retry budget/capacity permits it
AND server did not provide a stronger instruction not to retry
```

Common candidates: a failed connect before a request was accepted, selected `408`, `429`, or `503` responses with policy and `Retry-After`, or transport resets for a provably idempotent operation. Do not retry validation errors, authentication failures, most deterministic `4xx`, permanent resource absence, or ambiguous non-idempotent writes without an idempotency protocol.

HTTP method alone is insufficient. A nominal `GET` can trigger side effects; a `POST` can be safely retried with a durable idempotency key and stored outcome.

## 1.4 Idempotency and ambiguous completion

An operation is idempotent when repeating the same logical request produces no additional business effect.

```text
client sends create-payment(key=K)
server commits payment and result for K
response is lost
client retries key=K
server returns stored result; it does not charge again
```

The deduplication key must identify the logical operation, be scoped to the caller/operation, persist at least through the retry/replay window, and be committed atomically with the business effect. Same key plus different payload must be rejected. An in-memory cache is not durable idempotency.

## 1.5 Backoff, jitter, and budgets

Base exponential delay:

```text
cap_n = min(max_delay, base_delay * 2^n)
```

Useful jitter choices:

```text
full jitter:        sleep = random(0, cap_n)
equal jitter:       sleep = cap_n/2 + random(0, cap_n/2)
decorrelated jitter:sleep = min(max_delay, random(base_delay, prior_sleep*3))
```

Full jitter spreads synchronized clients well. Always cap attempts, delay, elapsed time, and concurrency. Honor a valid `Retry-After` within the caller's deadline and policy.

A **retry budget** bounds retry traffic relative to useful traffic, for example:

```text
retry_ratio = retry_attempts / initial_attempts
```

The chosen limit is workload specific; it is not automatically 10%. A token bucket can earn retry tokens from initial successes/requests and spend one per retry. When exhausted, fail fast rather than amplifying an outage.

## 1.6 Multi-layer amplification

With `r` retries at each of `d` independently retrying layers, one top-level request can create up to:

```text
attempts at deepest dependency = (r + 1)^d
```

Three retries at three layers can mean `4^3 = 64` deepest attempts. Even lower actual counts can overload pools and queues. Prefer one designated retry layer, usually the layer with the best semantics and deadline knowledge. Make attempt counts observable across layers.

## 1.7 Circuit breaker state machine

```text
                 failure threshold met
        +------------------------------------+
        |                                    v
     CLOSED ------------------------------> OPEN
       ^                                      |
       |                                      | open interval expires
       |                                      v
       +------ successful probes -------- HALF_OPEN
                         failed probe --------> OPEN
```

- **Closed:** calls flow; outcomes populate a count- or time-based window.
- **Open:** calls fail fast or take an explicitly correct fallback.
- **Half-open:** a bounded number of probes test recovery.

Tune by dependency and operation using minimum call volume, failure/slow-call threshold, window, open duration, half-open permits, and exception classification. Do not count caller cancellation, validation errors, or local bulkhead rejection as dependency failures unless the policy explicitly intends that. Breakers need metrics and must not share one state across unrelated operations without reason.

## 1.8 Bulkheads, concurrency, queues, and shedding

A bulkhead isolates scarce resources:

```text
critical calls -> pool/permits A ----> dependency
batch calls ----> pool/permits B ----> dependency
```

Concurrency limits bound in-flight work; rate limits bound arrivals over time. Little's Law gives the intuition:

```text
in_flight ~= throughput * average_time
```

When latency rises tenfold at the same arrival rate, concurrency demand rises roughly tenfold. Unbounded queues turn overload into latency and memory failure. A bounded queue should reject work that cannot meet its deadline. Shed low-priority or excess work early with explicit `429`/`503` and retry guidance where appropriate.

## 1.9 Fallback and graceful degradation

A fallback is correct only if its weaker contract is acceptable and visible:

```text
safe examples:
  recommendation unavailable -> omit optional recommendation section
  stale catalog read -> serve age-labeled cache within approved staleness
  analytics write unavailable -> durable local/outbox buffer within bounds

unsafe examples:
  payment failed -> return "paid"
  inventory unknown -> return quantity 0 as authoritative
  authorization unavailable -> allow access
  dependency error -> empty list indistinguishable from truly no data
```

Mark degraded responses, expose freshness/source, preserve error semantics, and never bypass security or financial correctness.

## 1.10 Cascades and herds

A cascading failure often follows:

```text
dependency latency rises
 -> caller in-flight requests and queues grow
 -> threads/connections/memory exhaust
 -> unrelated endpoints wait
 -> timeouts trigger synchronized retries
 -> load rises further
 -> health checks/autoscaling/cache behavior add churn
```

A **thundering herd** is synchronized work after a shared event: retry timers, cache expiry, service recovery, deployment, token refresh, or scheduled jobs. Jitter, request coalescing/single-flight, staggered TTLs, bounded warm-up, retry budgets, and admission control break synchronization.

---

# 2. Glossary

| Term | Meaning |
|---|---|
| Admission control | Decision to accept, queue, degrade, or reject work before scarce execution |
| Backpressure | Signal/mechanism causing producers to slow when consumers lack capacity |
| Bulkhead | Isolated resource/concurrency partition limiting blast radius |
| Circuit breaker | State machine that fails calls fast after evidence of dependency failure |
| Concurrency limit | Maximum simultaneous in-flight work |
| Deadline | Absolute latest useful completion time for the entire operation |
| Fallback | Explicit alternative behavior with a defined weaker contract |
| Graceful degradation | Preserving critical functions while reducing noncritical quality/features |
| Idempotency key | Stable logical-operation key used to deduplicate retries |
| Jitter | Randomness added to delay to avoid synchronization |
| Load shedding | Rejecting work that cannot be served safely or on time |
| Rate limit | Maximum arrivals/actions over an interval, often token/leaky bucket based |
| Retry budget | Bound on extra attempts allowed relative to useful traffic/capacity |
| Timeout | Maximum wait for one phase or attempt |

---

# 3. Essential metrics and evidence

## 3.1 Per operation and dependency

Collect by normalized operation, dependency, outcome, region, and version:

```text
initial request rate
attempt rate and retry rate
success/error/timeout/rejection rate
latency histogram for total operation and each attempt
remaining deadline at attempt start
in-flight concurrency
queue depth and queue wait
connection/thread/permit pool active, idle, pending, rejected
breaker state, transitions, window volume, failure/slow rate
rate-limit allowed/rejected and token availability
bulkhead accepted/rejected
fallback/degraded response count and freshness
idempotency hit, conflict, storage error, and key age
business success, duplicate, pending, and reconciliation count
```

Never use request ID, user ID, or raw URL as metric labels. Attempt number is bounded only if policy is bounded.

Example PromQL:

```promql
sum(rate(client_requests_total{client="checkout",dependency="payment"}[5m])) by (outcome)

sum(rate(client_attempts_total{client="checkout",dependency="payment",attempt!="1"}[5m]))
/
sum(rate(client_attempts_total{client="checkout",dependency="payment",attempt="1"}[5m]))

max(bulkhead_in_flight{service="checkout"}) by (dependency)

sum(increase(circuit_breaker_transitions_total{
  service="checkout",to_state="open"
}[15m])) by (dependency,operation)
```

Interpret ratios only with matching scopes and sufficient volume. Breaker state gauges need transition counters; a scrape may miss a short state. A low downstream request rate during an open breaker can mean protection is working, not recovery.

## 3.2 Incident evidence worksheet

```text
User-visible and business impact:
First/last failure time in UTC:
Caller operation, version, region, instance:
Dependency operation and endpoint:
Initial requests/second and attempt requests/second:
Error classes and who generated them:
Total deadline and remaining budget:
Connect/acquire/read/attempt timeouts:
Maximum attempts and actual attempt distribution:
Backoff/jitter and Retry-After behavior:
Idempotency guarantee and ambiguous writes:
Breaker state/config/window/sample count:
Bulkhead/concurrency/queue usage and rejection:
Rate-limit/load-shed/fallback outcomes:
Dependency latency/error/saturation:
Recent code/config/traffic/deployment change:
Telemetry drops or missing context:
```

## 3.3 What evidence proves

| Evidence | It supports | It does not prove |
|---|---|---|
| Client timeout | Client did not complete within its wait | Server did no work |
| Breaker open | Local policy crossed its configured threshold | Dependency is globally down |
| `429` | A limiter rejected this request | Retrying immediately is safe |
| `503` | Responder declared temporary unavailability | Operation was not committed |
| Retry success | A later attempt got a success response | Earlier write had no effect |
| Queue depth high | Work is waiting | Root cause is CPU |
| Fallback count high | Degraded path is active | Users received correct data |

---

# 4. Generic resilience design and incident workflow

1. **Define critical outcome.** Identify what must remain correct and what can degrade.
2. **Map the call graph and ownership.** Include gateways, clients, queues, storage, and external dependencies.
3. **Set one end-to-end deadline.** Allocate queue/acquisition/connect/attempt/backoff/response budgets from measured distributions and SLOs.
4. **Classify operations.** Read/write, idempotent/non-idempotent, synchronous/async, critical/optional.
5. **Classify failures.** Definite pre-accept failure, explicit transient rejection, deterministic permanent error, or ambiguous completion.
6. **Control admission.** Rate/concurrency limits, bounded queues, priority/fairness, and load shedding.
7. **Isolate.** Separate dependency and priority resources with bulkheads.
8. **Retry narrowly.** One owner, eligible errors only, durable idempotency, capped attempts/time, jitter, budget, deadline.
9. **Break persistent failure.** Tune a circuit breaker from observed volume and recovery behavior.
10. **Degrade truthfully.** Use only business-approved, distinguishable fallbacks.
11. **Observe and test.** Measure initial versus attempt load, rejection, breaker state, deadline, saturation, fallback, and business correctness.
12. **Operate.** During failure, reduce amplification first, preserve state, mitigate the bad cohort, and verify recovery under gradually restored load.

---

# 5. Original interview questions

## 1. Service B is down. How should Service A behave?

### Exact meaning and limits of the symptom

"Down" could mean DNS failure, refusal, timeout, `503`, one bad instance, or an open breaker. A cannot assume whether B accepted an ambiguous write. Required behavior depends on whether B is critical and whether the operation is a read, write, or asynchronous submission.

### Possible failure locations and cause mechanisms

B may be unreachable, overloaded, deploying, dependency-blocked, or healthy while the A-to-B path fails. A can worsen it through long waits, unbounded queues, synchronized retries, shared pool exhaustion, or a false-success fallback.

### Ordered design/debugging reasoning

1. Identify exact error generator and whether B received/committed the operation.
2. Check the remaining end-to-end deadline and operation idempotency.
3. Fail fast on deterministic/permanent errors.
4. For a narrowly eligible transient failure, permit a bounded jittered retry only within budget.
5. Open a per-operation breaker after enough evidence, with limited half-open probes.
6. Isolate B calls in a bulkhead; bound queue/concurrency so unrelated A endpoints survive.
7. Shed excess work explicitly.
8. Use only a truthful fallback: stale labeled read, optional omission, or durable async acceptance.
9. Return a clear unavailable/degraded result and correlation ID.
10. Reconcile ambiguous writes before replay.

### Evidence, tools, queries, and interpretation

Compare A initial versus attempt rate, B receive rate, client error classes, pool pending, in-flight work, breaker state/window, retry budget, and business outcomes. An A `DEADLINE_EXCEEDED` with a later B commit means retrying under a new key can duplicate the effect.

### Immediate mitigation

Disable or reduce retries, lower admitted noncritical traffic, enable an already-approved breaker/degraded mode, isolate B's pool, and communicate unavailability. Do not improvise success-shaped data.

### Permanent correction/design

Create deadline propagation, durable idempotency/status lookup, bounded queues, per-dependency bulkheads, tuned breaker/retry budgets, and an approved degradation contract.

### Prevention and alerts

Alert on A's business/SLO impact, attempt amplification, pool saturation, breaker transitions, and degraded-mode duration. Chaos-test B unavailability and verify unrelated A routes remain healthy.

### Common mistakes

- Returning `200 []` for dependency failure.
- Retrying every exception.
- Waiting indefinitely because B is "critical."
- Sharing B's exhausted pool with unrelated work.
- Assuming timeout means no write.

### Interview-ready answer

A should bound the call by the remaining deadline, isolate B resources, retry only eligible idempotent work within a jittered retry budget, and use a tuned breaker to fail fast during persistent failure. It should shed overload and expose a truthful unavailable or approved degraded result, then reconcile ambiguous writes rather than replay blindly.

## 2. Service B is intermittently failing. Would you use retry? How?

### Exact meaning and limits of the symptom

Intermittent means some attempts fail, but it does not establish transience, independence, or retry safety. Alternating failures can come from one bad instance or deterministic input; retries might only hide it.

### Possible failure locations and cause mechanisms

One bad B instance, connection reuse, packet loss, overload, rolling deploy, rate limiting, lock contention, bad payload cohort, or B's dependency can appear intermittent. Immediate retries can land on the same condition and amplify it.

### Ordered design/debugging reasoning

1. Segment failures by B instance, request class, version, region, and attempt.
2. Identify exact error and whether the first attempt may have committed.
3. Verify operation idempotency or use a durable key/stored result.
4. Retry only classified transient outcomes; respect `Retry-After`.
5. Prefer a small attempt count, full jitter exponential backoff, and a strict total deadline.
6. Consume a retry-budget token and stop when budget/remaining time is insufficient.
7. Retry at one layer; expose attempt metadata and do not nest policies.
8. Use a breaker and admission controls if failures persist.
9. Measure whether retries improve end-to-end success without harming tail latency/load.

### Evidence, tools, queries, and interpretation

```text
initial_rps=1000, retry_rps=400, recovered_success_rps=20
```

Here 40% extra load recovers only 2% of initial traffic; the policy likely harms the system. Inspect attempt outcome by B instance and error class. A second-attempt success on a different instance points toward an instance-specific issue, not a reason to keep masking it.

### Immediate mitigation

Cap/disable harmful retry classes, drain a proven bad instance, honor backoff, and reduce incoming load. Preserve keys for ambiguous writes.

### Permanent correction/design

Fix the intermittent mechanism, centralize retry ownership, add idempotency, use jitter/deadlines/budgets, and test failure matrices.

### Prevention and alerts

Alert on retry ratio, recovered-success efficiency, attempt latency, duplicate/conflict count, and failures by instance/version.

### Common mistakes

- Retrying all `5xx`, including deterministic failures.
- Retrying a non-idempotent write after read timeout.
- Fixed sleeps and synchronized clients.
- Independent retries in gateway, SDK, and service.
- Measuring only final successes.

### Interview-ready answer

I retry only if the failure is plausibly transient, the operation is idempotent or keyed, the deadline and retry budget allow it, and one layer owns the policy. I use few attempts with exponential full jitter and `Retry-After`, record every attempt, and validate recovered success against added load and tail latency.

## 3. Retries are making the production problem worse. Why?

### Exact meaning and limits of the symptom

The evidence should show extra attempts increasing load, latency, saturation, or duplicates. Retries may be the amplifier while an underlying slowdown or outage remains the trigger.

### Possible failure locations and cause mechanisms

Immediate/fixed-delay retries synchronize clients; nested layers multiply attempts; long timeouts retain old work; unbounded queues preserve doomed work; non-idempotent writes duplicate effects; recovery causes queued retries to flood the dependency.

### Ordered debugging reasoning

1. Graph initial requests separately from all attempts.
2. Calculate retry ratio and deepest-dependency amplification by call path.
3. Inspect attempt timing to identify synchronization and nested retries.
4. Compare added load with recovered successful operations.
5. Check deadline remaining, in-flight concurrency, queues, pools, and B saturation.
6. Identify retry-owning layers and error eligibility.
7. Check duplicate/ambiguous business effects.
8. Reduce amplification, then diagnose the original B failure.

### Evidence, tools, queries, and interpretation

Trace attributes should include `attempt`, remaining deadline, backoff reason, and outcome. Logs should record a stable logical-operation key, not secrets. If arrival remains 2,000 initial RPS while B receives 7,500 RPS and pool waits spike, retries are multiplying demand. If breaker opens only after long timeouts, it reacts too late to protect local resources.

### Immediate mitigation

Disable or sharply cap retries at duplicate layers, enforce retry budget and load shedding, shorten waits within a valid deadline design, and use the breaker to fail fast. Ramp changes gradually and monitor business correctness.

### Permanent correction/design

Choose one retry owner, classify failures, implement jitter and durable idempotency, propagate deadlines/cancellation, bound queues/concurrency, and stagger recovery.

### Prevention and alerts

Alert on attempt/initial ratio, retry budget exhaustion, queue wait, in-flight growth, duplicate rate, and synchronized spikes. Failure-test complete call graphs, not one client in isolation.

### Common mistakes

- Treating retries as free.
- Disabling all resilience while leaving unbounded waits.
- Fixing retry amplification but not the trigger.
- Counting final responses without attempt load.

### Interview-ready answer

Retries add positive feedback: slower dependencies cause timeouts, timeouts create more attempts, and extra attempts increase queueing and latency. Nested policies multiply this. I separate initial from attempt load, find retry owners and synchronization, cap or disable amplification, enforce deadlines/jitter/budgets/idempotency, and then fix the triggering dependency.

## 4. How would you prevent one failing service from bringing down the entire system?

### Exact meaning and limits of the symptom

The goal is blast-radius containment while preserving critical correctness. No single pattern prevents every cascade, and isolation consumes reserved capacity.

### Possible failure locations and cause mechanisms

Shared thread/connection pools, unbounded queues, synchronous fan-out, retry storms, common caches/databases, priority inversion, autoscaling lag, and misleading health checks can propagate one dependency failure.

### Ordered design/debugging reasoning

1. Map dependencies and shared resources; identify critical and optional paths.
2. Set end-to-end deadlines and propagate cancellation.
3. Bound global and per-dependency concurrency using capacity/latency measurements.
4. Isolate critical versus batch traffic with bulkheads.
5. Bound queues and shed requests unlikely to finish.
6. Apply fair rate limits so one tenant/route cannot consume all capacity.
7. Use narrow retry policies and budgets; prevent multi-layer amplification.
8. Use per-operation breakers for persistent faults.
9. Degrade optional functions truthfully; preserve auth, payment, and data integrity.
10. Scale on leading saturation signals where useful, but retain admission control.
11. Test dependency loss and recovery at realistic load.

### Evidence, tools, queries, and interpretation

Monitor pool use per dependency, queue wait, rejection by priority, initial/attempt traffic, breaker transitions, CPU/memory, business completion, and unrelated-route SLOs. If B fails and A's unrelated health endpoint remains up but critical C traffic starves in a shared pool, the missing control is resource isolation, not another health probe.

### Immediate mitigation

Shed noncritical traffic, isolate or disable the B-dependent feature, stop retry amplification, reserve capacity for critical work, and roll back a causal change.

### Permanent correction/design

Build bulkheads, bounded adaptive/static concurrency, priority/fair admission, deadline propagation, breaker/retry budgets, and explicit degraded contracts.

### Prevention and alerts

Chaos-test blast radius and assert unrelated SLOs. Alert before exhaustion on queue wait, permit/pool utilization, retry ratio, and rejected critical work.

### Common mistakes

- One shared executor for every dependency.
- Autoscaling as the only protection.
- Huge queues to avoid rejection.
- A global breaker that disables healthy operations.
- Fallback that bypasses security or lies about success.

### Interview-ready answer

I contain failure with end-to-end deadlines, per-dependency and per-priority bulkheads, bounded concurrency/queues, fair rate limits, and early load shedding. Retries are narrow and budgeted, breakers fail persistent faults fast, and only approved optional behavior degrades. I test that unrelated critical paths remain healthy when one dependency fails.

## 5. When would you use a circuit breaker?

### Exact meaning and limits of the symptom

Use a breaker when repeated calls to a dependency are likely to fail or be too slow and fast rejection protects caller resources and dependency recovery. It is not a replacement for timeout, concurrency limit, retry classification, or health monitoring.

### Possible failure locations and cause mechanisms

Persistent connection failure, high timeout/slow-call rate, overload, or dependency maintenance can justify opening. Poorly classified local cancellations, a tiny sample, one low-volume error, or mixed operations can open incorrectly.

### Ordered design reasoning

1. Define the protected dependency and operation scope.
2. Classify outcomes that represent dependency failure; exclude caller cancellation and business rejection as appropriate.
3. Select a count/time window and meaningful minimum call volume.
4. Choose failure and slow-call thresholds from SLO/capacity evidence.
5. Set open duration to reduce pressure without delaying recovery excessively.
6. Limit half-open probes and isolate them from a user surge.
7. Define explicit open behavior: clear unavailable result or correct fallback.
8. Combine with timeout, bulkhead, admission control, and telemetry.
9. Test low traffic, burst traffic, partial instance failure, and recovery.

### Evidence, tools, queries, and interpretation

Expose state, transition reason/time, window calls, failure/slow rates, rejected calls, and probe outcomes. A breaker with 100% failure from one call should not open if minimum volume is 20. A breaker cycling open/half-open means either B has not recovered, probes overload B, classification is wrong, or thresholds/open duration are poorly tuned.

### Immediate mitigation

If B is failing, an approved breaker can stop waste; if a misconfigured breaker blocks healthy B, correct/rollback configuration rather than repeatedly forcing it closed under load.

### Permanent correction/design

Tune per operation from real distributions, coordinate retries, make half-open bounded, and define degraded semantics and operator controls with audit.

### Prevention and alerts

Alert on sustained open state and transition flapping, not every transition alone. Dashboard sample count and underlying dependency health.

### Common mistakes

- Opening on every exception.
- No minimum calls.
- One breaker for unrelated endpoints.
- Unlimited half-open probes.
- Manual reset without verifying dependency capacity.

### Interview-ready answer

I use a circuit breaker for persistent or high-probability dependency failures where fail-fast behavior protects local capacity. I scope and classify it per operation, require sufficient sample volume, tune failure/slow thresholds and open time, bound half-open probes, and pair it with deadlines, bulkheads, and truthful degraded behavior.

## 6. Circuit breaker is constantly opening. How would you investigate?

### Exact meaning and limits of the symptom

Frequent opening means the configured rolling window repeatedly crosses a threshold. It could reflect a real dependency problem, local resource failure misclassified as remote, traffic shape, or configuration.

### Possible failure locations and cause mechanisms

B latency/errors, one bad B instance, A pool acquisition timeout, caller cancellations counted as B failures, low minimum volume, retry amplification, shared breaker scope, too-short open duration, aggressive half-open probes, or configuration drift.

### Ordered debugging reasoning

1. Record transition times, reasons, policy version, and operation scope.
2. Inspect window size, minimum calls, thresholds, slow duration, and classified exceptions.
3. Correlate each opening with B server RED and A client attempt metrics.
4. Separate connect, pool-acquire, queue, read, deadline, `5xx`, `429`, and local rejection.
5. Segment by B instance/zone/version and A instance.
6. Inspect retry load and whether half-open probes create a burst.
7. Compare with known healthy policy/config instances.
8. Fix the real B fault or tune the demonstrated misclassification/window issue.
9. Verify a stable recovery under gradually restored load.

### Evidence, tools, queries, and interpretation

```text
breaker window: calls=6, failures=3, minimum_calls=5, threshold=50%
B server errors: 0
A pool acquisition timeouts: 3
```

This points to local pool saturation being counted as B failure, not proof B is unhealthy. Conversely, matching B `503` and saturation supports a real B fault. Review dynamic-config/audit history.

### Immediate mitigation

Reduce retry/traffic pressure, isolate the operation, correct an obviously bad policy through controlled configuration, or keep the breaker open while B recovers. Do not force-close it repeatedly.

### Permanent correction/design

Fix B or A pool/root issue, correct exception classification, minimum volume, scope, open duration, and half-open concurrency; eliminate nested retries.

### Prevention and alerts

Alert on flapping and long-open duration with underlying metrics. Version resilience config and canary changes.

### Common mistakes

- Blaming B from breaker state alone.
- Resetting state before collecting the rolling-window evidence.
- Raising threshold until alerts stop.
- Ignoring local queue/pool timeout classification.

### Interview-ready answer

I inspect the exact rolling-window samples and transition reason, correlate A client outcomes with B server metrics, and split local queue/pool/deadline failures from true B errors. I check volume, scope, retries, config drift, and half-open probe behavior. Then I fix B or tune the proven classification/window problem and validate recovery under controlled load.

## 7. A downstream service takes 30 seconds to respond. How would you protect your service?

### Exact meaning and limits of the symptom

Thirty seconds may be normal for a batch operation but disastrous for an interactive SLO. It includes some combination of client queue/acquisition/network/server work. A shorter client timeout alone may leave downstream work running.

### Possible failure locations and cause mechanisms

Downstream queueing, slow query, lock, remote dependency, streaming, or overload can hold A's threads, connections, permits, and memory. Raising timeouts increases in-flight demand; aggressive retries multiply it.

### Ordered design/debugging reasoning

1. Define user deadline and whether synchronous completion is required.
2. Decompose 30 seconds into acquisition, connect, server queue/work, and transfer.
3. Propagate remaining deadline and cancellation; reserve response margin.
4. Apply per-dependency concurrency limit/bulkhead and bounded queue.
5. Reject or degrade work that cannot meet deadline.
6. Do not retry slow/ambiguous writes merely because A timed out.
7. Use a slow-call-aware breaker if persistent.
8. For legitimately long work, redesign as durable asynchronous submission with status/callback rather than holding an HTTP request.
9. Fix the downstream bottleneck and capacity.

### Evidence, tools, queries, and interpretation

Use latency histograms, trace phase spans, queue/pool wait, in-flight count, cancellation receipt, B completion after A timeout, and business-state reconciliation. At 100 RPS and 30-second average, approximately 3,000 operations are in flight before retries; that explains resource risk.

### Immediate mitigation

Shed noncritical traffic, cap concurrency, disable harmful retry, open the breaker when warranted, and use an approved async/degraded route.

### Permanent correction/design

Optimize B, introduce async job semantics for long operations, propagate cancellation/deadlines, and capacity-test bulkhead/queue limits.

### Prevention and alerts

Alert on queue wait, in-flight demand, deadline remaining, late completions, and slow-call breaker rate before total exhaustion.

### Common mistakes

- Setting A timeout to 31 seconds with no capacity analysis.
- Retrying at 30 seconds.
- Assuming cancellation automatically stops B.
- Returning accepted before durable enqueue.

### Interview-ready answer

I start from the end-to-end SLO, decompose the 30 seconds, propagate deadline/cancellation, and protect A with a per-dependency bulkhead, concurrency cap, bounded queue, and shedding. I avoid retrying ambiguous slow writes. If 30 seconds is legitimate, I use durable async acceptance and status rather than tying up synchronous resources.

## 8. How would you handle temporary downstream failures?

### Exact meaning and limits of the symptom

Temporary must be inferred from explicit signals or prior behavior, not assumed for every error. Handling must cover overload protection and business correctness, not only retry.

### Possible failure locations and cause mechanisms

Deploy restarts, short network loss, leader election, throttling, brief saturation, DNS refresh, or dependency failover may recover. Authentication, validation, schema mismatch, and deterministic data faults generally will not.

### Ordered design/debugging reasoning

1. Classify exact error and acceptance/commit ambiguity.
2. Apply a short attempt timeout inside the total deadline.
3. Retry only safe/idempotent operations and selected transient outcomes.
4. Use capped exponential full jitter and honor valid `Retry-After`.
5. Enforce attempt/elapsed/retry budgets at one layer.
6. Isolate with bulkhead and bound concurrency/queues.
7. Open a breaker if the "temporary" fault persists.
8. Use durable queue/outbox for deferrable work; expose pending status.
9. Reconcile uncertain outcomes and monitor recovery.

### Evidence, tools, queries, and interpretation

Measure failures by class, recovery time distribution, retry success by attempt, added attempt load, DLT/pending age, and duplicates. A `401` that succeeds after token refresh is a credential-refresh workflow, not a generic HTTP retry. A `429 Retry-After: 10` instructs pacing; immediate retry violates it.

### Immediate mitigation

Reduce attempt pressure, honor throttling, defer eligible work durably, or expose temporary unavailability. Maintain a bounded backlog.

### Permanent correction/design

Codify the failure matrix, retry ownership, durable idempotency, budgets, breaker, and async recovery/reconciliation.

### Prevention and alerts

Alert on sustained transient-class errors, retry efficiency, budget exhaustion, pending-work age, and duplicates. Test dependency failover and extended outage.

### Common mistakes

- Infinite retry because the error is labeled transient.
- In-memory queue as durable recovery.
- Retrying expired work.
- Treating every `503` or transport reset as definitely uncommitted.

### Interview-ready answer

I classify the failure and ambiguity first. For an eligible idempotent operation, one layer may perform a few deadline-bounded, budgeted retries with exponential jitter and `Retry-After`. Bulkheads and admission controls prevent overload; persistent failure opens a breaker, while deferrable work uses a durable queue and reconciliation.

## 9. When would you use timeout vs retry vs circuit breaker?

### Exact meaning and limits of the symptom

These solve different problems and are often combined. Choosing one excludes neither the need for the others nor admission control.

### Possible failure locations and mechanisms

A timeout bounds a slow attempt; a retry handles a rare transient attempt failure; a breaker suppresses repeated attempts during a persistent fault. Incorrect combinations cause late results, amplification, or prolonged rejection.

### Ordered design reasoning

1. Always begin with an end-to-end deadline and phase/attempt timeouts.
2. Classify operation safety and failure semantics.
3. Add a retry only for eligible transient outcomes with idempotency, jitter, budget, and remaining time.
4. Add a breaker when rolling evidence predicts repeated calls will fail/timeout and fail-fast protects capacity.
5. Add bulkhead/concurrency/admission controls because none of the three caps arrivals/in-flight work alone.
6. Define explicit behavior for timeout, exhausted retry, breaker open, and ambiguous writes.
7. Test timing algebra and observe initial versus attempt traffic.

### Evidence, tools, queries, and interpretation

```text
total deadline = 800 ms
queue/acquire budget = 80 ms
attempt 1 cap = 300 ms
full-jitter backoff cap = 40 ms
attempt 2 cap <= remaining deadline minus 100 ms response margin
```

These numbers are illustrative, not universal. Derive them from SLO, latency distributions, and capacity. Trace remaining deadline and attempt number; graph breaker state and local rejection.

### Immediate mitigation

If overloaded, first reduce admission/retry amplification; merely shortening or raising timeouts can shift symptoms. Use breaker only with evidence and safe open behavior.

### Permanent correction/design

Centralize policy, propagate deadlines, implement idempotency and budgets, tune breaker windows, and combine resource isolation.

### Prevention and alerts

Validate configuration relationships automatically: total retry worst case must fit deadline; every client has finite timeouts; attempt count is bounded.

### Common mistakes

- Saying timeout, retry, and breaker are alternatives.
- Per-attempt timeouts whose sum exceeds deadline.
- Breaker without timeout.
- Retry inside and outside breaker with unclear accounting.

### Interview-ready answer

Timeouts bound every call within one end-to-end deadline. Retries are optional and only for safe transient failures within remaining time and a retry budget. A circuit breaker fails fast when recent evidence makes continued calls wasteful. I combine them with bulkheads, bounded concurrency, and admission control, and define ambiguous-write behavior.

## 10. What happens if 1000 requests retry simultaneously after a downstream failure?

### Exact meaning and limits of the symptom

The simultaneous retry burst is a thundering herd. "1000" describes clients or requests, but nested layers and multiple attempts can produce far more downstream calls.

### Possible failure locations and cause mechanisms

Fixed retry delays, common timeout boundaries, service recovery, cache expiry, scheduled tasks, or mass reconnect synchronize requests. The burst exhausts accept queues, connections, threads, CPU, DB, rate limits, or autoscaling capacity and can knock a recovering service down again.

### Ordered design/debugging reasoning

1. Measure initial versus attempt traffic at high-resolution time buckets.
2. Identify synchronization source: equal delay, `Retry-After`, deadline, cache TTL, or recovery event.
3. Count retry layers and maximum amplification.
4. Check dependency and caller concurrency, queue, pool, and rejection.
5. Stop excess attempts with a retry budget and breaker.
6. Spread eligible retries with full/decorrelated jitter and cap concurrency.
7. Apply rate/admission limits and server-guided retry pacing.
8. Use single-flight/request coalescing for identical cache/key work.
9. Stagger cache TTL/refresh and warm recovery gradually.
10. Verify recovery at ramped load, not a one-shot flood.

### Evidence, tools, queries, and interpretation

One-second request/attempt rates reveal a herd that five-minute averages hide. Correlate a sawtooth pattern at the fixed delay with attempt number. Check connection creation, TLS handshakes, rate-limit rejection, cache misses, and autoscaler lag. `Retry-After` should itself be jittered by clients within semantics when a large fleet shares it.

### Immediate mitigation

Open the breaker, reduce retry tokens, rate-limit admission, shed noncritical traffic, and ramp traffic as B recovers. Do not restart every caller simultaneously.

### Permanent correction/design

Implement jitter, budgets, centralized retry ownership, concurrency caps, single-flight, staggered TTLs/jobs, and controlled warm-up.

### Prevention and alerts

Alert on attempt spikes, synchronization periodicity, reconnect storms, cache-miss bursts, and recovery flapping. Load-test outage plus recovery, not only steady state.

### Common mistakes

- Exponential backoff without jitter.
- Giving every client the same random seed.
- Unlimited half-open probes.
- Declaring recovery and releasing all queued work.
- Looking only at minute averages.

### Interview-ready answer

The 1000 retries form a thundering herd that can re-overload a recovering dependency, and nested retries can multiply it. I use high-resolution attempt metrics to find synchronization, then apply full jitter, retry budgets, capped concurrency, rate/load shedding, breaker-controlled probes, single-flight where valid, and gradual recovery.

---

# 6. Additional important interview questions

## 11. How do you choose timeout and deadline values?

### Meaning, failure locations, and mechanisms

Too short creates false failures and retries; too long holds resources and hides overload. One copied "30 second timeout" ignores network, queue, operation, SLO, and downstream latency differences.

### Ordered reasoning, evidence, and tools

1. Start from the user/business SLO and maximum useful completion time.
2. Measure normal and degraded phase distributions by route/region, including queue/acquisition.
3. Reserve ingress/egress and response margin.
4. Allocate child budgets so nested worst-case work cannot exceed parent remaining time.
5. Set connect shorter than total attempt where fast failure is expected; distinguish pool-acquire and read.
6. Include at most the retry plan that fits; use monotonic time.
7. Propagate remaining deadline and cancellation.
8. Load-test tail behavior and late completion.

Trace `deadline.remaining_ms`, phase spans, timeouts, cancellation receipt, and server completion after caller timeout. Revisit values after topology or latency changes.

### Mitigation, permanent correction, prevention, and mistakes

During overload, reduce admitted work rather than blindly lowering/raising timeouts. Permanently maintain per-operation budgets in reviewed config, validate nesting, and alert on deadline-exhaustion and late completions. Do not set every child timeout equal to the top-level deadline or forget queue time.

### Interview-ready answer

I derive deadlines from the useful business/SLO limit, measured tail distributions, and a response margin. I budget queue, acquisition, connect, attempt, and optional retry under one propagated deadline, use monotonic elapsed time, and verify cancellation and late writes under load.

## 12. Rate limiting, concurrency limiting, and load shedding: what is the difference?

### Meaning and mechanisms

Rate limiting controls arrivals over time, concurrency limiting controls simultaneous in-flight work, and load shedding rejects work the system cannot serve safely or before deadline. A service can obey an average rate limit yet overload when latency rises and concurrency accumulates.

### Ordered design, evidence, and tools

1. Identify capacity bottleneck and fair-share dimensions.
2. Apply edge rate limits for quotas/bursts using token or leaky buckets.
3. Cap per-dependency and global in-flight work from tested capacity.
4. Keep queues bounded and include queue time in deadlines.
5. Shed lowest-priority, expired, or excess work first with explicit `429` or `503`.
6. Protect a critical reserve and avoid starvation.
7. Return meaningful retry guidance only when retry is expected to succeed.
8. Measure allowed/rejected, queue wait, permits, fairness, and business impact.

### Mitigation, permanent correction, prevention, and mistakes

During overload, lower noncritical admission and preserve critical capacity. Permanently capacity-test limits, use adaptive concurrency only with guardrails, and document status semantics. Do not use an unbounded queue as a limiter, return `200`, or rate-limit all tenants equally when one causes the surge.

### Interview-ready answer

Rate limits shape arrivals, concurrency limits cap in-flight resource demand, and shedding rejects work that cannot meet safety or latency objectives. I combine them with bounded queues, fairness and priority, explicit responses, and metrics on rejection and business outcomes.

## 13. How do you design a safe fallback?

### Meaning, failure locations, and mechanisms

A fallback changes the product contract under failure. Its main risk is silent wrongness: stale, empty, fabricated, or unauthorized data that looks successful.

### Ordered design, evidence, and tools

1. Identify the critical invariant: authorization, money, inventory, ordering, or freshness.
2. Ask product/domain owners which weaker outcome is acceptable.
3. Define trigger, maximum age, scope, duration, and exit criteria.
4. Make degradation explicit in status/body/metadata and UI.
5. Ensure fallback has independent capacity and cannot recursively call the failed dependency.
6. Never bypass auth or claim a write committed.
7. Measure fallback count, age, correctness, and user impact.
8. Test transition into and out of fallback, including stale cache and recovery herd.

### Mitigation, permanent correction, prevention, and mistakes

Activate only a pre-approved fallback; otherwise return explicit unavailability. Permanently document/test it and reconcile deferred work. Do not turn exceptions into empty collections, serve unbounded stale data, or cache sensitive decisions beyond validity.

### Interview-ready answer

A safe fallback preserves named invariants while offering an explicitly weaker, business-approved result. It is bounded by scope and freshness, visibly marked, independently resourced, monitored, and tested through recovery. If no such contract exists, failing clearly is safer than fake success.

## 14. How do you recover without causing another outage?

### Meaning, failure locations, and mechanisms

Recovery can unleash queued requests, retries, cache misses, reconnects, half-open probes, and autoscaled callers before dependencies are warm. This is a recovery herd.

### Ordered reasoning, evidence, and tools

1. Confirm root condition is improving using dependency saturation and successful canaries.
2. Keep admission and retry budgets restrictive.
3. Permit a small number of breaker probes.
4. Ramp traffic/concurrency in stages while watching tail latency, queue wait, errors, and business correctness.
5. Warm caches/connections with bounded work; stagger TTLs and scheduled jobs.
6. Drain durable backlog at a rate below spare capacity with priority.
7. Reconcile ambiguous/pending operations before replay.
8. Define rollback thresholds and stop the ramp if leading saturation returns.

### Mitigation, permanent correction, prevention, and mistakes

Use controlled ramp and backlog throttling now. Permanently automate progressive recovery, jitter, single-flight, and capacity-aware drain controls. Never force-close all breakers, release the whole queue, restart every caller, or confuse one successful probe with recovered capacity.

### Interview-ready answer

I recover progressively: bounded probes, low initial concurrency, staged traffic ramp, controlled cache/connection warm-up, and backlog drain only from measured spare capacity. I monitor leading saturation and business state, reconcile uncertain writes, and roll back the ramp if thresholds regress.

---

# 7. Decision trees

## 7.1 Should this failure be retried?

```text
Did the total deadline expire or lack useful remaining time?
  yes -> do not retry
  no
   |
   +-- Is error deterministic/permanent (validation, authz, bad schema, not found)?
   |     yes -> do not retry; correct request/config
   |     no
   |
   +-- Could the first attempt have committed?
   |     yes -> durable idempotency/status lookup available?
   |              no -> do not blindly replay; reconcile
   |              yes -> continue with same logical key
   |
   +-- Is outcome explicitly transient and policy-approved?
   |     no -> fail/route to explicit recovery
   |     yes
   |
   +-- Is this the designated retry layer?
   |     no -> propagate classified failure
   |     yes
   |
   +-- Retry budget, concurrency, and attempt cap available?
         no -> fail fast
         yes -> wait capped jitter/Retry-After, retry once within deadline
```

## 7.2 Which resilience mechanism?

```text
Work waits without bound?
  -> deadline + phase/attempt timeout + cancellation

Rare eligible transient failure?
  -> bounded jittered retry + idempotency + retry budget

Persistent high-probability dependency failure?
  -> circuit breaker with bounded half-open probes

One dependency/traffic class consumes shared resources?
  -> bulkhead + per-class concurrency limit

Arrival rate exceeds policy/capacity?
  -> fair rate limit

Accepted work cannot finish before objective?
  -> early load shedding and bounded queue

Optional feature unavailable?
  -> explicit approved fallback/graceful degradation

Long but legitimate operation?
  -> durable async submission + status/callback
```

## 7.3 Breaker keeps opening

```text
Read exact transition samples
 |
 +-- enough minimum volume?
 |     no -> tune sampling/window; do not infer dependency outage
 |
 +-- failures generated by dependency?
 |     no -> separate local queue/pool/cancellation/rejection classification
 |     yes
 |
 +-- one instance/operation/version?
 |     yes -> isolate/drain/fix that cohort
 |
 +-- retry traffic inflating failure/load?
 |     yes -> remove nested retries; enforce budget
 |
 +-- half-open probes fail under burst or too-short recovery?
       yes -> bound probes/open longer based on evidence
       no  -> repair dependency and validate progressive recovery
```

## 7.4 Overload and cascade response

```text
Latency/queue/in-flight rising
 |
 +-- stop amplification: retries, fan-out, duplicate work
 +-- preserve critical capacity: bulkhead/priority/fairness
 +-- reject doomed work early: bounded queue + shedding
 +-- reduce optional work and traffic
 +-- keep deadlines/cancellation and correctness
 +-- fix triggering dependency/resource
 +-- recover with probes and staged ramp
 +-- reconcile pending/ambiguous business operations
```

---

# 8. Cheat sheets

## 8.1 Safe retry checklist

```text
[ ] Exact transient error allowlist
[ ] Operation idempotent or durable key + stored outcome
[ ] Same logical key retained across attempts
[ ] One retry-owning layer
[ ] End-to-end deadline propagated
[ ] Few attempts; elapsed-time cap
[ ] Exponential full/decorrelated jitter
[ ] Valid Retry-After honored within deadline
[ ] Retry budget and concurrency cap
[ ] Initial and attempt traffic measured separately
[ ] Ambiguous completion reconciled
[ ] No auth/validation/deterministic failures retried
```

## 8.2 Circuit-breaker checklist

```text
[ ] Scoped by dependency and compatible operation
[ ] Dependency-failure classification documented
[ ] Window and minimum call count meaningful
[ ] Failure and slow-call thresholds evidence-based
[ ] Open duration and bounded half-open probes
[ ] Clear breaker-open response/fallback
[ ] Timeout, bulkhead, and admission control also present
[ ] State, transitions, sample count, reason observable
[ ] Low-volume, partial failure, and recovery tested
[ ] Config versioned and canaried
```

## 8.3 Cascade warning signs

```text
dependency p99 rising
caller in-flight, queue wait, or pool pending rising
attempt rate diverging from initial rate
timeouts followed by periodic retry spikes
unrelated endpoint latency rising
breaker flapping
health-check or autoscaling churn
cache hit rate falling and origin load rising
late writes, duplicates, or pending reconciliation increasing
telemetry/export drops during peak load
```

## 8.4 Failure response matrix

| Outcome | Retry? | Key caution |
|---|---|---|
| Validation/schema error | No | Correct request/contract |
| Authentication/authorization failure | No generic retry | Controlled credential refresh may be separate |
| Explicit throttling | Only by policy | Honor `Retry-After`, deadline, budget |
| Connect failure before send | Sometimes | Operation and route policy still matter |
| Read timeout after write | Not blindly | Completion is ambiguous; query/deduplicate |
| `503` before accepted work | Sometimes | Must know responder semantics |
| Local bulkhead rejection | Usually no immediate retry | Caller is already saturated |
| Breaker open | Not at inner layer | Respect open interval; avoid probe herd |
| Deterministic not found | No | Unless domain explicitly defines eventual creation |

## 8.5 Concise interview framework

```text
Define correctness and SLO
 -> one propagated deadline
 -> classify operation and failure ambiguity
 -> admission + bounded concurrency/queue
 -> isolate with bulkheads
 -> narrow idempotent jittered retry under budget
 -> tuned breaker for persistent fault
 -> truthful fallback only when approved
 -> observe initial/attempt load and business outcomes
 -> recover progressively and reconcile
```

Resilience is successful when the system fails **boundedly, truthfully, and without corrupting state**, while retaining enough capacity to recover.
