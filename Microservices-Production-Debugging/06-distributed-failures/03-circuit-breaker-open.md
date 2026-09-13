# Problem

## Circuit breaker opens on shipping quote failures

Checkout rejects shipping quote calls locally after a bad dependency rollout crosses the breaker's failure threshold.

I investigate the first failing constraint, contain amplification, preserve business truth, and prove recovery.

# Production Situation

- Checkout receives 900 quote requests/min; normal shipping P95 is 140 ms.
- Shipping version 5.7.1 has a stale OAuth audience on two of six instances.
- Those instances return 401 in 35 ms, producing a 33% aggregate failure rate.
- The breaker has a 20-call count window and minimum volume of 10.
- Its failure threshold is 30%, OPEN wait is 20 seconds, and HALF_OPEN permits 3 probes.
- The breaker opens in two seconds and local CallNotPermitted reaches 900/min.
- DNS, TCP, TLS, CPU, and memory are healthy.
- Cart can say shipping unavailable; checkout cannot invent a price or free shipping.

This is one continuous incident. Original demand, extra attempts, useful completions, and business failures are measured separately.

# Architecture

Client
  |
  v
Checkout Service
  | CircuitBreaker shippingQuotes
  +-- CLOSED: pass and measure
  +-- OPEN: reject locally
  +-- HALF_OPEN: 3 probes
  |
  v
Load Balancer -> Shipping v5.7.0 and bad v5.7.1 -> Carrier

Arrows show request direction. Queue time, retry delay, client wait, server work, and local circuit rejection are separate measurements.

# What I Check FIRST

### First check 1
- WHAT: Breaker state and transition reason.
- WHY: OPEN is protection, not root cause.
- RESULT: I look for failure rate crossed threshold after minimum volume.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 2
- WHAT: Exact outcomes in the rolling window.
- WHY: The configured window, not a lifetime average, causes transition.
- RESULT: I look for 7 failures among the latest 20 calls.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 3
- WHAT: Per-target version and config.
- WHY: Mixed instances disappear in aggregate data.
- RESULT: I look for all 401 responses come from v5.7.1.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 4
- WHAT: Remote calls versus local rejections.
- WHY: OPEN requests never reach shipping.
- RESULT: I look for CallNotPermitted has no shipping client span.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 5
- WHAT: Fallback business semantics.
- WHY: Technical availability cannot override pricing truth.
- RESULT: I look for SHIPPING_UNAVAILABLE rather than shippingCost=0.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

# Step-by-Step Investigation

### Step 1 - separate local and remote failure
- WHAT: Split CallNotPermitted from HTTP calls that reached shipping.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Open-state calls have no remote child span and near-zero duration.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Inspect preceding remote failures.

### Step 2 - read breaker configuration
- WHAT: Capture state, window type and size, minimum calls, thresholds, open wait, and probe count.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: With 20 calls, 7 failures exceed 30% after minimum 10.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Compare settings with traffic.

### Step 3 - inspect the triggering window
- WHAT: List each buffered outcome with target, status, duration, and exception mapping.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: 401 failures cluster only on v5.7.1.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Compare deployment and config history.

### Step 4 - explain CLOSED to OPEN
- WHAT: Verify calls passed in CLOSED until the qualified rolling failure rate crossed threshold.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: The transition prevents further futile calls and protects resources.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Do not count local rejection as remote 503.

### Step 5 - explain HALF_OPEN
- WHAT: After 20 seconds, observe exactly three controlled probes.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: All success closes; a configured failed probe reopens.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Prevent normal traffic from becoming probes.

### Step 6 - compare target dimensions
- WHAT: Group target by version, config hash, node, zone, and draining state.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Two bad targets share audience hash aud-v1.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Quarantine those targets.

### Step 7 - classify 401
- WHAT: Confirm the token audience is deterministic for this deployment.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: 401 is not retry eligible; repetition cannot fix audience.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Correct config instead of retrying.

### Step 8 - validate fallback
- WHAT: Trace cart and place-order behavior while OPEN.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Cart is explicitly degraded; place-order blocks without a real quote.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Search for zero-price completed orders.

### Step 9 - recover safely
- WHAT: Remove bad instances, correct audience, and wait for controlled probes.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Three successful probes close the breaker without a surge.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Watch OPEN-HALF_OPEN flapping.

### Step 10 - tune from evidence
- WHAT: Choose window, minimum volume, slow/failure thresholds, wait, and probes from measured traffic.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Noise cannot trip it, but a sustained outage opens promptly.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Test failure, slowness, and recovery.

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
traceId=cb-5510 breakerState=CLOSED
Checkout quote span 52 ms
  Shipping client span 38 ms status=ERROR http.status=401 peer=ship-5.7.1-2
traceId=cb-5511 breakerState=OPEN
Checkout quote span 4 ms status=DEGRADED
  CircuitBreaker span 1 ms event=call_not_permitted
  Shipping client span missing reason=local-open
  Fallback span 1 ms result=SHIPPING_UNAVAILABLE

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
- 2026-09-13T17:10:04.221Z WARN service=checkout-service instance=co-2ff1 traceId=cb-5510 spanId=ship-client requestId=req-610 endpoint=/quotes downstream=shipping-service error=HTTP_401 latencyMs=38 target=ship-5.7.1-2
- 2026-09-13T17:10:04.230Z WARN service=checkout-service instance=co-2ff1 traceId=none spanId=none requestId=none endpoint=/quotes downstream=shipping-service event=CIRCUIT_STATE_CHANGE from=CLOSED to=OPEN failureRate=35 bufferedCalls=20 minimumCalls=10
- 2026-09-13T17:10:05.004Z INFO service=checkout-service instance=co-2ff1 traceId=cb-5511 spanId=breaker requestId=req-611 endpoint=/quotes downstream=shipping-service error=CallNotPermitted latencyMs=1 fallback=SHIPPING_UNAVAILABLE
- 2026-09-13T17:10:25.231Z INFO service=checkout-service instance=co-2ff1 traceId=cb-probe spanId=probe requestId=req-700 endpoint=/quotes downstream=shipping-service event=HALF_OPEN_PROBE result=SUCCESS probe=1/3

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

Shipping 5.7.1 deploys with an obsolete OAuth audience.
  ->
Two of six targets return deterministic 401 responses.
  ->
The 20-call window records a 35% failure rate after minimum volume 10.
  ->
The 30% threshold is crossed, so CLOSED becomes OPEN.
  ->
OPEN correctly rejects calls locally and protects checkout.
  ->
Aggregate quote availability becomes zero although four targets are healthy.
  ->
Business-safe fallback preserves carts but cannot complete checkout.
  ->
Bad per-instance auth config is root cause; OPEN is expected protection.

The first line is the initiating fault. Later lines are amplifiers and propagation; fixing only the final timeout would leave the chain intact.

# Fix

### Immediate mitigation
- Remove v5.7.1 instances from readiness and retain the breaker.
- Correct OAuth audience and verify token acceptance on a canary.
- Allow only controlled HALF_OPEN probes before restoring normal traffic.

### Permanent design
- Validate audience, issuer, and secret references before readiness.
- Tune minimum volume and rolling windows to real traffic.
- Use separate failure and slow-call thresholds.
- Never retry deterministic 401 responses.
- Alert on transitions, per-target failures, fallback rate, and quote business success.
- Document retry eligibility, owner, attempts, timeout, backoff, jitter, budget, and idempotency.
- Keep mitigation and permanent repair distinct; reconcile business effects after technical recovery.
- Do not blindly add timeout, CPU, memory, workers, or retries without evidence of the constrained resource.

# Verification

### Comparable before and after
- Shipping 401: 33% -> 0% for every target.
- State: OPEN -> HALF_OPEN -> CLOSED after 3/3 probes.
- CallNotPermitted: 900/min -> 0/min.
- Remote traffic ramps to 900/min without a recovery herd.
- Business quote success: 0% -> 99.8%.
- No completed order has a synthetic zero shipping price.
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
I would begin by separating original demand, downstream attempts, and the required business outcome. Here, checkout rejects shipping quote calls locally after a bad dependency rollout crosses the breaker's failure threshold. I would compare RED and saturation metrics across every hop, find the first signal that changed, and inspect a failed trace including each retry span. I would correlate it with logs by traceId, target, version, config, node, zone, and remaining deadline. I would contain the incident with bounded concurrency, bulkheads, truthful load shedding, and a calibrated circuit, not blanket retries or longer timeouts. Any retry needs a transient failure, idempotency, one owner, exponential backoff with jitter, retry budget, and enough deadline. After fixing the initiating fault, I would verify useful throughput, tail latency, queues, attempts, and business correctness before declaring recovery.

### Common interviewer traps
- Calling OPEN the root cause stops one layer early.
- Force-closing sends traffic to known failures.
- Retrying 401 wastes capacity.
- HTTP 200 with free shipping is a false fallback.
- Ignoring minimum volume makes rate thresholds noisy.
- Treating a health endpoint as proof that the affected business operation works.
- Treating one log message or the last visible timeout as root cause.

### Quick memory flow
Symptom -> business invariant -> scope -> originals versus attempts -> first changed signal -> trace retries -> correlate logs -> target and saturation comparison -> contain -> root cause -> permanent fix -> verify business recovery

# Interview Follow-up Questions

### Q1. What are the states?
Answer: CLOSED passes and measures calls, OPEN rejects locally, and HALF_OPEN permits bounded recovery probes.

### Q2. Why did it open?
Answer: After minimum volume, 35% failures in the 20-call rolling window exceeded the 30% threshold.

### Q3. Why minimum volume?
Answer: One failure out of one call is 100%; minimum volume prevents noise-driven trips.

### Q4. What is a rolling window?
Answer: The recent count or time period whose outcomes determine failure and slow-call rates.

### Q5. Should 401 be retried?
Answer: Normally no; invalid audience or credentials are deterministic for that request.

### Q6. How do probes avoid a herd?
Answer: Only a small configured number pass, then traffic ramps after success.
