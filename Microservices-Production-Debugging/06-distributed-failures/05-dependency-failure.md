# Problem

## External tax dependency failure with business-safe degradation

An external tax outage threatens checkout capacity and correctness; the service must not present untaxed orders as successful.

I investigate the first failing constraint, contain amplification, preserve business truth, and prove recovery.

# Production Situation

- Checkout receives 1,500 requests/min; tax P95 is normally 220 ms.
- The provider begins returning 503 and 8-second read timeouts in two regions.
- Tax success falls to 14% and checkout P99 reaches 10.2 s.
- Tax and checkout both retry, raising provider attempts to 4,100/min.
- Tax bulkhead fills and order worker utilization reaches 96%.
- Cart viewing can state tax is unavailable without finalizing an order.
- Order placement legally requires authoritative tax or an explicit TAX_PENDING workflow.
- An old estimate or tax=0 is not a valid authoritative fallback.

This is one continuous incident. Original demand, extra attempts, useful completions, and business failures are measured separately.

# Architecture

Client
  |
  v
Checkout Service -> Order DB
  |
  v
Tax Service
  | circuit + bulkhead + deadline
  v
External Tax Provider

Cart: explicit unavailable estimate is allowed
Place order: authoritative tax is required

Arrows show request direction. Queue time, retry delay, client wait, server work, and local circuit rejection are separate measurements.

# What I Check FIRST

### First check 1
- WHAT: Business success versus HTTP success.
- WHY: A fallback can hide a correctness failure.
- RESULT: I look for no completed order lacks authoritativeTax=true.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 2
- WHAT: Remote 503, timeout, or local rejection.
- WHY: Each outcome proves a different stage.
- RESULT: I look for provider spans contain 503 and read timeout; OPEN has no child.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 3
- WHAT: Deadline and nested retry count.
- WHY: Late repeated calculations waste capacity.
- RESULT: I look for 4,100 attempts from 1,500 originals.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 4
- WHAT: Tax bulkhead and order workers.
- WHY: Dependency waits can cascade into checkout.
- RESULT: I look for tax pool full and order utilization 96%.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 5
- WHAT: Allowed degradation by operation.
- WHY: Cart and place-order have different invariants.
- RESULT: I look for cart says unavailable; place-order blocks or is explicitly pending.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

# Step-by-Step Investigation

### Step 1 - define the invariant
- WHAT: Confirm completed orders require authoritative tax and a durable taxDecisionId.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Any completed order without both is a correctness defect.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Audit incident orders.

### Step 2 - scope provider failure
- WHAT: Compare outcomes by region, endpoint, account, and time.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Both regions show 503 and timeouts while internal calls are healthy.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Check provider status and credentials.

### Step 3 - separate network stages
- WHAT: Check DNS, route, TCP, TLS, HTTP request, and response in order.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: TLS succeeds; some HTTP responses are 503 and others exceed read timeout.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Classify remote outcomes accurately.

### Step 4 - build the deadline budget
- WHAT: Reserve 500 ms local work and 300 ms margin from a 4-second deadline.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: At most 3.2 seconds remains for all tax attempts.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Stop attempts that cannot fit.

### Step 5 - decide eligibility
- WHAT: Retry selected transient 503 or timeout only for idempotent calculation.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: 400, 401, 403, and expired deadlines do not qualify.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Honor Retry-After and retry budget.

### Step 6 - measure multiplication
- WHAT: Compare checkout originals, tax requests, provider attempts, and retry timestamps.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Nested immediate retries create 4,100 attempts and waves.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Give tax-service sole retry ownership.

### Step 7 - contain saturation
- WHAT: Check bulkhead active/max, queue, rejects, order waits, and provider pool.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Bounded tax concurrency rejects before all order workers block.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Shed excess place-order traffic honestly.

### Step 8 - configure the breaker
- WHAT: Use rolling window, minimum volume, failure and slow thresholds, open wait, and few probes.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Qualified provider failures open it; OPEN rejections are local.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Keep cart and order behavior distinct.

### Step 9 - degrade by business path
- WHAT: Return taxStatus=UNAVAILABLE for cart; fail place-order or persist controlled TAX_PENDING.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: No tax=0 or expired estimate becomes final.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Block payment and fulfillment until tax succeeds.

### Step 10 - reconcile and recover
- WHAT: Half-open gradually, process pending work idempotently, and audit incident orders.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Backlog reaches zero without duplicate payment or fulfillment.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Verify provider and business recovery.

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
traceId=df-4401 edgeDeadlineMs=4000 operation=PLACE_ORDER
Checkout span 3998 ms status=ERROR taxStatus=UNAVAILABLE
  validation span 84 ms
  Tax attempt=1 span 1805 ms status=503 retryEligible=true
    Provider HTTP span 1710 ms status=503 retryAfterMs=500
  retry.delay span 500 ms jittered=true
  Tax attempt=2 span 1590 ms status=CANCELLED deadlineRemainingMs=0
    Provider span 3200 ms status=OK late=true
  persist-order span missing reason=authoritative-tax-required

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
- 2026-09-13T17:31:00.010Z WARN service=tax-service instance=tax-44c1 traceId=df-4401 spanId=provider-a1 requestId=req-401 endpoint=/tax/calculate downstream=external-tax error=HTTP_503 latencyMs=1710 attempt=1 retryAfterMs=500
- 2026-09-13T17:31:00.511Z INFO service=tax-service instance=tax-44c1 traceId=df-4401 spanId=retry-delay requestId=req-401 endpoint=/tax/calculate downstream=external-tax retryEligible=true retryDelayMs=500 deadlineRemainingMs=1601
- 2026-09-13T17:31:02.102Z ERROR service=checkout-service instance=co-91aa traceId=df-4401 spanId=checkout requestId=req-401 endpoint=/orders downstream=tax-service error=TaxUnavailable latencyMs=3998 businessResult=NOT_PLACED
- 2026-09-13T17:31:03.709Z WARN service=tax-service instance=tax-44c1 traceId=df-4401 spanId=provider-a2 requestId=req-401 endpoint=/tax/calculate downstream=external-tax error=LateResponse latencyMs=3200 taxDecisionId=TD-882 ignored=true

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

The external provider loses capacity and returns 503 or stalls.
  ->
Tax latency exceeds the checkout dependency budget.
  ->
Checkout and tax-service both retry.
  ->
Provider attempts rise from 1,500 to 4,100/min.
  ->
Long waits fill the tax bulkhead and occupy order workers.
  ->
The outage begins affecting unrelated checkout work.
  ->
A generic fallback risks presenting estimates as authoritative.
  ->
Provider loss initiates the incident; retries, weak isolation, and unsafe fallback magnify it.

The first line is the initiating fault. Later lines are amplifiers and propagation; fixing only the final timeout would leave the chain intact.

# Fix

### Immediate mitigation
- Remove checkout retry and retain at most one tax-owned eligible retry.
- Open the provider circuit, cap tax concurrency, and shed excess place-order calls.
- Return explicit unavailable or controlled TAX_PENDING; stop fulfillment until authoritative tax exists.

### Permanent design
- Propagate one 4-second deadline and cancellation.
- Use full-jitter backoff and a retry budget below 10% of healthy successes.
- Deduplicate tax requests by cart version and jurisdiction.
- Use bulkhead, concurrency limiter, bounded queue, and calibrated breaker.
- Test that no success response can omit authoritative tax.
- Document retry eligibility, owner, attempts, timeout, backoff, jitter, budget, and idempotency.
- Keep mitigation and permanent repair distinct; reconcile business effects after technical recovery.
- Do not blindly add timeout, CPU, memory, workers, or retries without evidence of the constrained resource.

# Verification

### Comparable before and after
- Provider attempts: 4,100/min -> 1,560/min at 1,500/min demand.
- Tax queue: 1,200 -> 0; active remains below tested limit 80.
- Order worker utilization: 96% -> 43%.
- Completed untaxed orders remain zero throughout degradation.
- Breaker moves OPEN -> HALF_OPEN with five probes -> CLOSED without a surge.
- TAX_PENDING backlog drains 214 -> 0 without duplicate payment.
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
I would begin by separating original demand, downstream attempts, and the required business outcome. Here, an external tax outage threatens checkout capacity and correctness; the service must not present untaxed orders as successful. I would compare RED and saturation metrics across every hop, find the first signal that changed, and inspect a failed trace including each retry span. I would correlate it with logs by traceId, target, version, config, node, zone, and remaining deadline. I would contain the incident with bounded concurrency, bulkheads, truthful load shedding, and a calibrated circuit, not blanket retries or longer timeouts. Any retry needs a transient failure, idempotency, one owner, exponential backoff with jitter, retry budget, and enough deadline. After fixing the initiating fault, I would verify useful throughput, tail latency, queues, attempts, and business correctness before declaring recovery.

### Common interviewer traps
- tax=0 with HTTP 200 violates business correctness.
- Retrying auth, validation, or expired work consumes capacity.
- A longer timeout increases in-flight work and cascade risk.
- OPEN rejection is not a fresh provider 503.
- An unlabeled old estimate is not graceful degradation.
- Treating a health endpoint as proof that the affected business operation works.
- Treating one log message or the last visible timeout as root cause.

### Quick memory flow
Symptom -> business invariant -> scope -> originals versus attempts -> first changed signal -> trace retries -> correlate logs -> target and saturation comparison -> contain -> root cause -> permanent fix -> verify business recovery

# Interview Follow-up Questions

### Q1. How do you handle a required dependency outage?
Answer: Protect capacity and fail closed or use an explicit pending workflow; never claim completion without required data.

### Q2. What can degrade?
Answer: Cart can say tax unavailable, while final order placement requires authoritative tax.

### Q3. Which failures are retryable?
Answer: Selected transient 503 or timeout for idempotent calculation, within deadline and retry budget.

### Q4. Why breaker and bulkhead?
Answer: The breaker stops likely failures; the bulkhead bounds resource use while evidence accumulates.

### Q5. What is business verification?
Answer: Every completed order has authoritative tax, pending work is reconciled, and payment is not duplicated.

### Q6. Is provider 503 the whole cause?
Answer: It initiates the event, but nested retries, long budgets, weak isolation, and fallback explain the larger impact.
