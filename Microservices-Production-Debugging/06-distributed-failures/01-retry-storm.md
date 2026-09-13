# Problem

## Retry storm during payment authorization

A partial payment slowdown becomes a checkout outage because three retry layers multiply traffic.

I investigate the first failing constraint, contain amplification, preserve business truth, and prove recovery.

# Production Situation

- Original checkout traffic is 1,260 requests/min; normal is 1,200 requests/min.
- Payment P95 rises from 280 ms to 1.9 s after one database replica develops storage waits.
- Gateway, order-service, and the payment client each allow two retries.
- One checkout can therefore create 3 x 3 x 3 = 27 payment attempts.
- Payment receives 5,900 attempts/min, an amplification ratio of 4.7 in the observed mix.
- Checkout P99 reaches 8.4 s, errors reach 31%, and useful authorizations fall to 640/min.
- Order workers reach 240/240 and queue depth reaches 1,900 while CPU is only 52%.
- Some payment calls commit after order-service times out, so missing idempotency risks duplicate charges.

This is one continuous incident. Original demand, extra attempts, useful completions, and business failures are measured separately.

# Architecture

Client
  | one checkout
  v
API Gateway -- up to 3 attempts
  |
  v
Order Service -- up to 3 attempts
  |
  v
Payment Client -- up to 3 attempts
  |
  v
Payment Service -> MySQL replica

Arrows show request direction. Queue time, retry delay, client wait, server work, and local circuit rejection are separate measurements.

# What I Check FIRST

### First check 1
- WHAT: Original request rate versus downstream attempt rate.
- WHY: This directly quantifies amplification.
- RESULT: I look for 1,260 originals/min versus 5,900 payment attempts/min.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 2
- WHAT: Retries grouped by caller, outcome, and attempt.
- WHY: This finds every multiplying layer.
- RESULT: I look for gateway, order, and payment client all retry read timeouts.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 3
- WHAT: The first metric that changed.
- WHY: Retries may magnify a fault without initiating it.
- RESULT: I look for payment replica wait rises before retry count.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 4
- WHAT: End-to-end deadline and per-attempt timeouts.
- WHY: Attempts that cannot fit are guaranteed waste.
- RESULT: I look for three 2-second attempts inside a 3-second deadline.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 5
- WHAT: Idempotency coverage and late success.
- WHY: A timeout does not prove payment failed.
- RESULT: I look for one order produces more than one authorization ID.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

# Step-by-Step Investigation

### Step 1 - scope the storm
- WHAT: Compare region, endpoint, caller, version, original rate, attempt rate, and useful success.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Only checkout calls through payment are affected; status reads remain healthy.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Calculate amplification by caller.

### Step 2 - measure retry multiplication
- WHAT: Count attempt=1, attempt=2, and attempt=3 spans at every layer.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: The observed 4.7 ratio is below the theoretical 27 but still consumes most new capacity.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Find which error classifications trigger each retry.

### Step 3 - rebuild the deadline budget
- WHAT: Start with the 3,000 ms edge deadline and subtract gateway, order work, response time, and safety margin.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Only about 1,800 ms remains for all payment work.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Reject attempts that cannot fit attempt timeout plus backoff.

### Step 4 - classify retry eligibility
- WHAT: Inspect exception types and HTTP outcomes instead of retrying any failure.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Transient 502, 503, reset, and selected timeout outcomes may qualify; 400, 401, 403, and payment declines do not.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Confirm operation safety and idempotency.

### Step 5 - inspect backoff and jitter
- WHAT: Plot attempt timestamps and retry-delay histograms.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Zero-delay retries align callers into sharp waves.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Use capped exponential backoff with full jitter.

### Step 6 - find the initiating slowdown
- WHAT: Compare payment server spans, DB spans, pool waits, and replica metrics before retries rise.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: The replica storage wait begins 90 seconds before amplification.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Remove the replica from service.

### Step 7 - check saturation propagation
- WHAT: Graph payment queue, DB pending, order active workers, and gateway pending requests.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Each queue rises after attempt traffic, showing cascading saturation.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Protect admitted work with limits and shedding.

### Step 8 - check late completion and idempotency
- WHAT: Join attempts by orderId and idempotency key, including responses after caller timeout.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: One key should map to one stored authorization outcome.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Pause unsafe retries and reconcile duplicates.

### Step 9 - mitigate without hiding failure
- WHAT: Disable gateway and order retries, retain at most one owner retry, and shed overload truthfully.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Attempts fall before latency and queues recover.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Watch useful throughput and duplicate effects.

### Step 10 - prove the permanent policy
- WHAT: Load-test partial latency with one owner, deadline propagation, budget, idempotency, backoff, jitter, breaker, and bulkhead.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Retry ratio remains below 1.10 and no synchronized waves appear.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Ship canary guards and alerts.

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
traceId=rt-7a21 deadlineMs=3000
Gateway span 18 ms
  Order POST /checkout span 2990 ms
    Payment attempt=1 span 1005 ms status=DEADLINE_EXCEEDED
      Payment server span 1180 ms status=OK late=true
    retry.delay span 0 ms policy=immediate
    Payment attempt=2 span 1002 ms status=DEADLINE_EXCEEDED
    Payment attempt=3 span 960 ms status=CANCELLED deadlineRemainingMs=0

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
- 2026-09-13T16:55:11.203Z WARN service=order-service instance=order-6d8f traceId=rt-7a21 spanId=pay-a1 requestId=req-991 endpoint=/checkout downstream=payment-service attempt=1 error=ReadTimeout latencyMs=1005 deadlineRemainingMs=1958
- 2026-09-13T16:55:11.205Z INFO service=order-service instance=order-6d8f traceId=rt-7a21 spanId=delay-a2 requestId=req-991 endpoint=/checkout downstream=payment-service retryEligible=true retryDelayMs=0
- 2026-09-13T16:55:11.378Z INFO service=payment-service instance=pay-4b2c traceId=rt-7a21 spanId=server-a1 requestId=req-991 endpoint=/authorizations orderId=O-771 idempotencyKey=missing result=AUTHORIZED latencyMs=1172
- 2026-09-13T16:55:13.170Z ERROR service=order-service instance=order-6d8f traceId=rt-7a21 spanId=checkout requestId=req-991 endpoint=/checkout downstream=payment-service error=DeadlineExceeded attempts=3 latencyMs=2990

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

A payment database replica develops 1.8-second storage waits.
  ->
Authorization crosses the 1-second client read timeout but can still commit.
  ->
Three independent layers classify the timeout as retryable.
  ->
Immediate retries synchronize callers and multiply offered load.
  ->
Payment queues and DB waits grow, making every later attempt slower.
  ->
Order workers fill while waiting, despite moderate CPU.
  ->
Missing idempotency allows repeated effects for one order.
  ->
A degraded replica becomes a broad checkout outage.

The first line is the initiating fault. Later lines are amplifiers and propagation; fixing only the final timeout would leave the chain intact.

# Fix

### Immediate mitigation
- Remove the slow replica and stop unsafe retries at gateway and order-service.
- Shed excess checkout load with an honest 503 and Retry-After where clients can honor it.
- Drain saturated instances so admitted requests can finish.

### Permanent design
- Give the payment client sole retry ownership and at most one eligible retry.
- Require an order-scoped idempotency key and persist the first outcome.
- Propagate one deadline and cancellation through Gateway, Feign or WebClient, and payment.
- Use full-jitter exponential backoff and a rolling retry budget below 10% of successful originals.
- Add a payment bulkhead and tested concurrency limit.
- Document retry eligibility, owner, attempts, timeout, backoff, jitter, budget, and idempotency.
- Keep mitigation and permanent repair distinct; reconcile business effects after technical recovery.
- Do not blindly add timeout, CPU, memory, workers, or retries without evidence of the constrained resource.

# Verification

### Comparable before and after
- Payment attempts: 5,900/min -> 1,310/min at comparable original demand.
- Amplification ratio: 4.7 -> about 1.04.
- Checkout P99: 8.4 s -> 620 ms; error rate: 31% -> 0.7%.
- Order workers: 240/240 -> 82/240; queue: 1,900 -> 12.
- Useful authorizations: 640/min -> 1,180/min.
- Every idempotency key maps to one durable authorization.
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
I would begin by separating original demand, downstream attempts, and the required business outcome. Here, a partial payment slowdown becomes a checkout outage because three retry layers multiply traffic. I would compare RED and saturation metrics across every hop, find the first signal that changed, and inspect a failed trace including each retry span. I would correlate it with logs by traceId, target, version, config, node, zone, and remaining deadline. I would contain the incident with bounded concurrency, bulkheads, truthful load shedding, and a calibrated circuit, not blanket retries or longer timeouts. Any retry needs a transient failure, idempotency, one owner, exponential backoff with jitter, retry budget, and enough deadline. After fixing the initiating fault, I would verify useful throughput, tail latency, queues, attempts, and business correctness before declaring recovery.

### Common interviewer traps
- Adding more retries because payment is flaky increases offered load.
- Increasing timeout without capacity evidence increases in-flight work.
- Treating the timeout as proof of failed payment ignores late success.
- Declaring recovery from lower errors while useful throughput stays low accepts a false result.
- Treating a health endpoint as proof that the affected business operation works.
- Treating one log message or the last visible timeout as root cause.

### Quick memory flow
Symptom -> business invariant -> scope -> originals versus attempts -> first changed signal -> trace retries -> correlate logs -> target and saturation comparison -> contain -> root cause -> permanent fix -> verify business recovery

# Interview Follow-up Questions

### Q1. When is a retry eligible?
Answer: Only for a likely transient failure, a safe or idempotent operation, enough remaining deadline, and available retry budget.

### Q2. Why exponential backoff with jitter?
Answer: Backoff gives recovery time; jitter prevents synchronized retry waves.

### Q3. What is a retry budget?
Answer: A cap on extra attempts relative to useful traffic or successes so retries cannot consume all capacity.

### Q4. How do retries multiply?
Answer: Three layers with three total attempts each can create 27 leaf calls from one original request.

### Q5. Why is timeout not proof of failure?
Answer: The caller stopped waiting, but the server may complete and commit later.

### Q6. What proves recovery?
Answer: Original rate, attempt ratio, useful throughput, queues, tail latency, errors, and duplicate effects all normalize.
