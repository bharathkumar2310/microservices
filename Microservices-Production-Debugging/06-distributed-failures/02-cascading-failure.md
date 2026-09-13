# Problem

## Cascading saturation from inventory to checkout

A lock-heavy inventory deployment holds order threads until queues and failures spread through otherwise healthy services.

I investigate the first failing constraint, contain amplification, preserve business truth, and prove recovery.

# Production Situation

- Gateway traffic is 2,000 requests/min; normal is 1,900 requests/min.
- Inventory P95 rises from 180 ms to 4.6 s after version 3.18.0.
- Order in-flight requests rise from 35 to 510 and workers reach 250/250.
- Gateway pending requests reach 3,800; checkout P99 reaches 12.4 s.
- Error rate is 44%, but CPU is 38% and heap is 61%.
- Thread dumps show workers WAITING on inventory HTTP reads.
- Status and cancellation share the same worker pool as checkout.
- Retries create bursts against inventory after lock waits release.

This is one continuous incident. Original demand, extra attempts, useful completions, and business failures are measured separately.

# Architecture

Clients
  |
  v
Gateway queue
  |
  v
Order Service shared worker pool
  +--> Inventory Service -> MySQL locks
  +--> Payment Service
  +--> Order status and cancellation

Arrows show request direction. Queue time, retry delay, client wait, server work, and local circuit rejection are separate measurements.

# What I Check FIRST

### First check 1
- WHAT: The first latency increase across services.
- WHY: Chronology separates trigger from propagation.
- RESULT: I look for inventory DB lock spans rise before upstream queues.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 2
- WHAT: In-flight work, workers, and queues.
- WHY: Low CPU can coexist with waiting saturation.
- RESULT: I look for 250/250 order workers and 510 in flight.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 3
- WHAT: Little's Law estimate.
- WHY: L = lambda x W predicts concurrency pressure.
- RESULT: I look for 33 requests/s x 4.6 s is at least 152 in flight before tail and retries.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 4
- WHAT: Bulkhead boundaries.
- WHY: A shared pool lets one dependency starve unrelated work.
- RESULT: I look for cancellation and checkout compete for the same threads.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 5
- WHAT: Deadline and cancellation propagation.
- WHY: Abandoned work must release scarce capacity.
- RESULT: I look for inventory continues after gateway timeout.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

# Step-by-Step Investigation

### Step 1 - establish chronology
- WHAT: Overlay RED and saturation metrics for gateway, order, inventory, payment, and MySQL.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Inventory latency changes four minutes before gateway errors.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Trace requests at the transition.

### Step 2 - apply Little's Law
- WHAT: Calculate L = lambda x W using original arrival rate and average residence time.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: At 33 requests/s and 4.6 s, at least 152 calls are concurrently resident.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Explain excess with queueing, tails, and retries.

### Step 3 - inspect order workers
- WHAT: Check active/max, queue, rejection, and RUNNABLE, WAITING, or BLOCKED thread states.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Most workers wait on inventory socket reads.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Identify shared endpoint pools.

### Step 4 - inspect inventory and DB
- WHAT: Compare server duration, DB lock wait, query duration, connection pending, and version.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: MySQL lock wait dominates only version 3.18.0 requests.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Inspect transaction order.

### Step 5 - budget timeouts
- WHAT: Reserve edge time for gateway and order work, then give inventory less than the remaining budget.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: A 10-second inventory timeout outlives the 5-second user deadline.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Propagate deadline and cancellation.

### Step 6 - measure retry load
- WHAT: Count inventory attempts and graph retry delays.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Immediate retries double load while lock capacity is lowest.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Remove duplicate retry ownership.

### Step 7 - isolate healthy operations
- WHAT: Put inventory checkout work in a bounded bulkhead separate from status and cancellation.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: A full inventory pool rejects only inventory-dependent work.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Validate priority capacity.

### Step 8 - shed excess admission
- WHAT: Reject above concurrency and bounded queue limits with honest 503 or 429.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Short queues recover while useful completion rises.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Track shed rate and admitted success.

### Step 9 - remove the trigger
- WHAT: Roll back inventory 3.18.0 and drain affected pods.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: DB lock wait falls before upstream queues drain.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Do not add workers against a fixed locked resource.

### Step 10 - test containment
- WHAT: Inject bounded inventory latency at production concurrency in staging.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Checkout sheds predictably while status and cancellation remain within SLO.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Automate the regression gate.

### Resilience mechanics used in this investigation

- Retry eligibility has four gates: transient classification, safe or idempotent operation, enough remaining deadline, and available retry budget.
- An exception alone does not justify retry; validation, authorization, and deterministic business failures need correction.
- A total deadline bounds user wait across all hops; each attempt plus backoff and response margin must fit the remaining budget.
- Per-attempt connect and read timeouts are smaller controls inside that total deadline, not substitutes for it.
- Capped exponential backoff increases spacing after repeated failures.
- Full jitter chooses a random delay inside the cap so thousands of callers do not wake together.
- A retry budget caps extra attempts as a fraction of successful or original traffic and preserves useful capacity.
- Retries multiply across layers: three layers with three total attempts each can create 27 leaf calls.
- A thundering herd is a synchronized burst after timeout, retry, cache expiry, or recovery.
- Idempotency means repeating one logical operation creates one durable effect, usually through a stable key and stored result.
- CLOSED passes calls and records outcomes; OPEN rejects locally; HALF_OPEN admits a small number of recovery probes.
- A breaker should require minimum volume and a rolling count or time window before evaluating failure or slow-call rate.
- It opens because the qualified rate crossed threshold, protecting threads and the dependency; OPEN is not itself root cause.
- A bulkhead reserves bounded capacity so one dependency cannot consume every worker or connection.
- A concurrency limit caps in-flight work, load shedding rejects beyond capacity, and rate limiting controls admission by caller or tenant.
- Little's Law is L = lambda x W: at fixed arrival rate, longer residence time requires proportionally more concurrent work.
- Low CPU can coexist with total saturation when threads wait on sockets, locks, queues, or pools.
- Graceful degradation must state unavailable, stale, or pending when allowed; it must never imitate authoritative success.
- A success-shaped false fallback can violate charging, inventory, tax, shipping, or fulfillment correctness.
- Dependency analysis must verify both technical transport and the required business outcome.

# Metrics to Check

| Metric | If HIGH | If LOW | If it changes after deployment |
|---|---|---|---|
| Original request rate | High may be real demand overload. | Low while attempts stay high suggests orphaned or repeated work. | A jump only in attempts implicates resilience policy. |
| Attempt/original ratio | High proves retry amplification. | Near one reduces retry-storm likelihood. | A caller-specific jump identifies retry ownership. |
| Error rate by outcome | High technical or business failure needs classification. | Low can be false if fallback returns 200. | A version-aligned jump implicates rollout. |
| P50/P95/P99/max latency | High tail shows queueing, slow targets, or dependency delay. | Low with errors suggests fast rejection. | A threshold change can alter behavior. |
| Useful business throughput | High means capacity produces valid outcomes. | Low while HTTP success is high exposes false fallback. | A fall proves user impact. |
| In-flight concurrency | High means long residence time or excess arrival. | Low with errors suggests early rejection. | A rise after latency matches Little's Law. |
| Worker active/max | High means no execution headroom. | Low shifts focus away from executor saturation. | A new blocking path can cause a jump. |
| Queue depth and age | High means delayed admission and stale work. | Low is healthy only if useful throughput is normal. | Acceleration predicts collapse. |
| Rejected or shed requests | High protects admitted work but signals overload. | Zero during saturation may mean missing guardrail. | A controlled rise after a policy change is expected. |
| Dependency client/server latency | High server time points downstream; client-only gap points local pool or network. | Low with failures means fast response or rejection. | A target-specific jump narrows scope. |
| Connection pool pending | High means work waits for scarce connections. | Zero weakens pool exhaustion. | A rise after attempts suggests secondary saturation. |
| Retry delay and count | High count means instability or bad policy. | Low may mean healthy service or exhausted budget. | Zero delay reveals herd risk. |
| Circuit state and calls denied | OPEN and high denial mean local protection. | CLOSED does not prove health. | Transition must match window evidence. |
| Per-target error and latency | High on one target reveals an outlier. | Uniform low values shift to caller or shared path. | New version or config correlation is actionable. |
| CPU, heap, and GC | High may constrain service and needs JVM evidence. | Low does not rule out blocked-thread saturation. | A rollout change can reveal code cost. |

### How I combine the signals
- RED means Rate, Errors, and Duration at every service boundary.
- Saturation adds active/max workers, in-flight calls, queues, connection pools, rejections, and shedding.
- I compare baseline, onset, mitigation, and recovery windows at similar original traffic.
- A falling request rate can be backpressure or rejection, not recovery.
- A falling dependency query rate can mean callers are stuck before reaching it, not that it is healthy.
- I require a business-success counter beside HTTP status.
- Per-target comparison includes version, config hash, node, zone, start time, and draining state.

# Distributed Trace Investigation

### Representative traces
traceId=cf-9031 edgeDeadlineMs=5000
Gateway span 5002 ms status=504
  Order POST /checkout span 4978 ms status=CANCELLED
    executor.queue span 890 ms
    Inventory reserve span 4070 ms status=DEADLINE_EXCEEDED
      Inventory server span 6200 ms cancelled=false
        MySQL UPDATE span 5980 ms db.lockWaitMs=5660
    Payment span missing reason=inventory-not-complete

### How I read the trace
- traceId identifies the end-to-end request; spanId identifies one operation; parent and child links preserve causality.
- Client latency includes local queue, connection acquisition, network wait, server time, and response handling.
- Server latency starts after the downstream accepts work; a client-only gap suggests pool, queue, network, or instrumentation delay.
- A dominant DB or external child narrows delay to that dependency; large parent self-time points to local work or queueing.
- Every retry is a sibling span with attempt, trigger, target, timeout, delay, and deadline remaining.
- Overlapping attempts suggest hedging or failed cancellation; sequential attempts reveal cumulative deadline consumption.
- A missing child can mean OPEN circuit, executor rejection, failure before instrumentation, sampling, context loss, or no call.
- I verify missing spans with client counters and logs rather than declaring network loss.
- I compare a failure with a success from the same minute and group trace attributes by target and version.

# Distributed Logs

### Correlated event sequence
- 2026-09-13T17:02:41.091Z WARN service=inventory-service instance=inv-9c4a traceId=cf-9031 spanId=db-44 requestId=req-448 endpoint=/reserve downstream=mysql error=LockWait latencyMs=5980 version=3.18.0
- 2026-09-13T17:02:42.006Z WARN service=order-service instance=ord-2ea1 traceId=cf-9031 spanId=inv-client requestId=req-448 endpoint=/checkout downstream=inventory-service error=ReadTimeout latencyMs=4070 workerActive=250 queueDepth=912
- 2026-09-13T17:02:42.030Z INFO service=gateway instance=gw-7ac2 traceId=cf-9031 spanId=edge requestId=req-448 endpoint=/checkout downstream=order-service error=GatewayTimeout latencyMs=5002
- 2026-09-13T17:02:43.200Z WARN service=order-service instance=ord-5bb7 traceId=cf-cancel spanId=cancel requestId=req-771 endpoint=/orders/cancel downstream=none error=ExecutorRejected latencyMs=8

### Correlation rules
- Start from a failed traceId and sort all services by timestamp.
- Join asynchronous work with requestId and a stable business key such as orderId.
- instance, target, version, config hash, node, and zone expose heterogeneous behavior.
- attempt, timeout, deadlineRemainingMs, breaker state, and retry delay explain resilience decisions.
- Compare a successful request from the same interval to remove unrelated background noise.
- One log proves only what that component observed; it is not root-cause proof.
- A timeout log proves the caller stopped waiting, not that the server failed or rolled back.
- Clock skew, sampling, duplicate ingestion, and late asynchronous completion can alter apparent ordering.
- Root cause needs chronology plus metrics, trace, logs, and responsible-component evidence.

# Commands / Tools

### Windows
- `Resolve-DnsName dependency.internal` proves name resolution from this host now; it does not prove application health.
- `Test-NetConnection dependency.internal -Port 8080` proves TCP establishment only, not TLS, HTTP, or business correctness.
- `curl.exe -v --max-time 5 http://dependency.internal:8080/actuator/health/readiness` proves a bounded HTTP exchange on that route only.
- `curl.exe -v --max-time 5 -H "X-Debug-Request: incident-20260913" http://dependency.internal:8080/actuator/health/readiness` adds an approved correlation marker.

### Linux
- `dig dependency.internal` and `getent hosts dependency.internal` inspect DNS and resolver results.
- `nc -vz -w 3 dependency.internal 8080` tests TCP connection establishment only.
- `curl -v --max-time 5 http://dependency.internal:8080/actuator/health/readiness` tests bounded HTTP readiness, not a business transaction.
- `ss -lntp` on an authorized host shows listeners and owning processes, not downstream health.

### Platform and JVM evidence
- Prometheus and Grafana compare RED, saturation, retries, breaker transitions, targets, and useful throughput.
- OpenTelemetry with Jaeger or Zipkin finds failed traces and retry child spans.
- Spring Boot `/actuator/metrics` and `/actuator/prometheus` expose HTTP, executor, HikariCP, and resilience metrics when secured.
- `/actuator/threaddump` distinguishes WAITING, BLOCKED, and RUNNABLE concentration; capture it sparingly.
- Read-only endpoint views verify readiness, version, zone, and endpoint removal before any mutation.
- Safe progression is DNS -> IP route -> TCP -> TLS -> HTTP -> business validation; each tool proves only one layer.
- Ping success never proves TCP port, TLS, HTTP, authorization, dependency, or business correctness.

# Root Cause

Inventory 3.18.0 changes transaction order and creates long MySQL lock waits.
  ->
Inventory residence time rises from 180 ms to several seconds.
  ->
At constant arrival rate, Little's Law predicts much more in-flight work.
  ->
Order's shared worker pool fills with inventory waits.
  ->
Unbounded queueing delays checkout, status, and cancellation.
  ->
Retries add load and synchronized bursts.
  ->
Gateway queues behind order while healthy payment cannot be reached.
  ->
A local lock regression becomes a cross-service cascade.

The first line is the initiating fault. Later lines are amplifiers and propagation; fixing only the final timeout would leave the chain intact.

# Fix

### Immediate mitigation
- Roll back inventory 3.18.0 and drain the affected pods.
- Disable inventory retries, cap checkout concurrency, and shed excess demand.
- Reserve a separate cancellation and status bulkhead.

### Permanent design
- Use consistent lock order and shorter inventory transactions.
- Use one end-to-end deadline and cancel abandoned downstream work.
- Size bounded queues from tolerated waiting time, not memory.
- Configure a slow-call breaker with minimum volume and limited half-open probes.
- Load-test dependency latency and unrelated endpoint isolation.
- Document retry eligibility, owner, attempts, timeout, backoff, jitter, budget, and idempotency.
- Keep mitigation and permanent repair distinct; reconcile business effects after technical recovery.
- Do not blindly add timeout, CPU, memory, workers, or retries without evidence of the constrained resource.

# Verification

### Comparable before and after
- Inventory P99: 6.3 s -> 240 ms; lock wait P99: 5.7 s -> 12 ms.
- Order in-flight: 510 -> 42.
- Workers: 250/250 -> 74/250; queue: 1,900 -> 4.
- Gateway pending: 3,800 -> 15; checkout P99: 12.4 s -> 710 ms.
- Useful checkout throughput: 760/min -> 1,910/min.
- Cancellation P99 remains below 180 ms during a repeat fault test.
- Verify p50, p95, p99, max, errors, queues, rejections, and circuit state over multiple rolling windows.
- Verify original demand stayed comparable so lower load is not mistaken for repair.
- Verify useful business throughput, not just fast HTTP responses.
- Verify no late, duplicate, stale, or missing business effects remain.
- Repeat a bounded fault test and confirm containment works before closing the incident.

Lower latency caused only by fast rejection is not recovery. Capacity and correct business outcomes must return.

# Prevention

- Alert on original rate, attempt ratio, errors, tail latency, saturation, and business success together.
- Dashboard retries by caller, attempt, reason, target, and policy.
- Dashboard breaker state, transition reason, buffered calls, failure rate, slow-call rate, and probe outcomes.
- Define one total deadline and derive per-hop timeouts from its remaining budget.
- Test that cancellation reaches downstream work and closes connections.
- Enforce stable idempotency keys and durable deduplication for state changes.
- Use full-jitter exponential backoff and a strict retry budget.
- Use dependency bulkheads, tested concurrency limits, bounded queues, rate limits, and explicit load shedding.
- Validate truthful fallback contracts with automated business-invariant tests.
- Compare target version, config, node, zone, and draining status in dashboards.
- Make readiness represent ability to serve and use graceful draining.
- Test partial latency, errors, one bad instance, and recovery-herd behavior.
- Gate canaries on P99, first-attempt success, queueing, retry ratio, and business outcome.
- Keep runbooks for rollback, containment, reconciliation, and evidence preservation.

# Interview Answer

### What I would say in an interview
I would begin by separating original demand, downstream attempts, and the required business outcome. Here, a lock-heavy inventory deployment holds order threads until queues and failures spread through otherwise healthy services. I would compare RED and saturation metrics across every hop, find the first signal that changed, and inspect a failed trace including each retry span. I would correlate it with logs by traceId, target, version, config, node, zone, and remaining deadline. I would contain the incident with bounded concurrency, bulkheads, truthful load shedding, and a calibrated circuit, not blanket retries or longer timeouts. Any retry needs a transient failure, idempotency, one owner, exponential backoff with jitter, retry budget, and enough deadline. After fixing the initiating fault, I would verify useful throughput, tail latency, queues, attempts, and business correctness before declaring recovery.

### Common interviewer traps
- Scaling order workers increases pressure on the locked DB.
- Low CPU does not mean spare concurrency.
- A longer timeout raises in-flight work through Little's Law.
- A fake successful reservation would oversell inventory.
- Treating a health endpoint as proof that the affected business operation works.
- Treating one log message or the last visible timeout as root cause.

### Quick memory flow
Symptom -> business invariant -> scope -> originals versus attempts -> first changed signal -> trace retries -> correlate logs -> target and saturation comparison -> contain -> root cause -> permanent fix -> verify business recovery

# Interview Follow-up Questions

### Q1. What is a cascading failure?
Answer: A local dependency fault consumes shared upstream resources until healthy services and operations also fail.

### Q2. How does Little's Law help?
Answer: L = lambda x W shows that longer residence time at the same arrival rate requires more concurrent work.

### Q3. Why use a bulkhead?
Answer: It bounds one dependency's resource use and preserves unrelated or higher-priority operations.

### Q4. Why shed load?
Answer: Early honest rejection protects admitted work and avoids universal timeout through queue growth.

### Q5. Can checkout fall back?
Answer: It may report availability unknown, but it cannot claim inventory was reserved.

### Q6. When should the breaker open?
Answer: After minimum volume and rolling failure or slow-call rate cross configured thresholds.
