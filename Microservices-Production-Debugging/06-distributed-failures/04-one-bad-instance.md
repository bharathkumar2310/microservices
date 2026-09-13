# Problem

## Intermittent failures from one bad catalog instance

About one eighth of product requests time out because one ready instance has stale database configuration.

I investigate the first failing constraint, contain amplification, preserve business truth, and prove recovery.

# Production Situation

- Catalog has eight instances and receives 4,800 requests/min.
- Global error rate is 11.8%, near the 12.5% probability of hitting one of eight targets.
- catalog-7c9f in zone-c has 94% timeout rate and P95 of 1.6 s.
- Seven peers have error below 0.4% and P95 below 120 ms.
- The bad pod uses config hash 91ab; peers use c440.
- It points to retired host catalog-db-old and times out after 1.5 s.
- Retries often land on healthy targets and hide final errors while doubling work.
- Readiness checks JVM liveness only, so the dependency-broken pod remains eligible.

This is one continuous incident. Original demand, extra attempts, useful completions, and business failures are measured separately.

# Architecture

Product API
  |
  v
Service load balancer
  +--> seven healthy catalog pods -> catalog-db-primary
  +--> catalog-7c9f BAD zone-c
          | config hash 91ab
          v
      catalog-db-old -> connect timeout

Arrows show request direction. Queue time, retry delay, client wait, server work, and local circuit rejection are separate measurements.

# What I Check FIRST

### First check 1
- WHAT: Metrics by target instance.
- WHY: Aggregates dilute one outlier.
- RESULT: I look for catalog-7c9f has 94% timeout.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 2
- WHAT: Version, config, node, and zone.
- WHY: A runtime difference often explains one target.
- RESULT: I look for only config hash differs.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 3
- WHAT: Traffic share and readiness.
- WHY: Blast radius depends on eligible routing weight.
- RESULT: I look for bad ready endpoint receives 12.1%.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 4
- WHAT: First-attempt versus final success.
- WHY: Retries can mask the defect.
- RESULT: I look for second attempts succeed on healthy targets.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

### First check 5
- WHAT: Draining and endpoint propagation.
- WHY: Removal must stop new traffic safely.
- RESULT: I look for active calls reach zero before termination.
- MEANING: This decides whether I isolate a target, contain a caller, or continue into the dependency.
- NEXT: Preserve the timestamp and dimensions, then validate the next causal link.

# Step-by-Step Investigation

### Step 1 - quantify the fraction
- WHAT: Graph first-attempt errors by target and minute.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: A stable rate near 1/N suggests one of N similarly weighted targets.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Verify actual traffic weights.

### Step 2 - build a target comparison
- WHAT: List traffic, errors, P95, version, config, node, zone, and readiness.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: catalog-7c9f is the only outlier.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Inspect its dependency spans.

### Step 3 - calculate probability
- WHAT: Use p=1/8 for an independent first choice among equal targets.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Observed 11.8% is close to expected 12.5%.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Check locality and sticky routing.

### Step 4 - expose retry masking
- WHAT: Measure first-attempt and final success separately.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: A second independent attempt has only 1/64 chance to hit the bad target twice.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Do not confuse retry success with repair.

### Step 5 - compare configuration
- WHAT: Compare environment, mounted config, image digest, JVM args, and resolved DB host.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: The bad pod resolves catalog-db-old.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Trace the stale bundle source.

### Step 6 - test node and zone hypotheses
- WHAT: Compare healthy pods on the same node and zone.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Healthy zone-c peers make a zone-wide fault unlikely.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Keep scope on pod config.

### Step 7 - inspect lifecycle
- WHAT: Check readiness, endpoint registration, termination grace, and active requests.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Liveness-only readiness admits a pod unable to query.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Mark it unready and drain.

### Step 8 - judge retry safety
- WHAT: Catalog GET is idempotent, but a retry still needs deadline, budget, jitter, and target diversity.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: One bounded retry may mitigate; repeated POST would require a key.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Avoid retry pinning.

### Step 9 - remove the outlier
- WHAT: Mark catalog-7c9f unready, wait for endpoint removal and active=0, then terminate.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: Global errors drop by about its former share.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Watch remaining target load.

### Step 10 - prevent drift
- WHAT: Deploy immutable image/config pairs and gate on per-target canary metrics.
- WHY: This tests one causal link rather than guessing from the final symptom.
- TOOL: Use a five-minute metrics overlay, sampled trace comparison, and structured-log query scoped by target.
- EXPECTED RESULT: A stale canary fails readiness before receiving production traffic.
- BAD RESULT: A value outside baseline or concentrated on one dimension identifies a capacity, correctness, or dependency divergence.
- MEANING: The signal changing first is a cause candidate; later queue, timeout, and rejection signals may be consequences.
- NEXT: Automate config fingerprint checks.

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
traceId=bi-8812 routeTarget=catalog-7c9f
Product API span 1518 ms
  Catalog attempt=1 span 1502 ms status=DEADLINE_EXCEEDED peer=catalog-7c9f
    Catalog server span 1496 ms
      MySQL connect span 1490 ms peer=catalog-db-old status=TIMEOUT
traceId=bi-8813 routeTarget=catalog-3
Product API span 96 ms
  Catalog attempt=1 span 81 ms status=OK peer=catalog-3
    MySQL query span 14 ms peer=catalog-db-primary

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
- 2026-09-13T17:20:14.102Z ERROR service=catalog-service instance=catalog-7c9f traceId=bi-8812 spanId=db-connect requestId=req-812 endpoint=/products/44 downstream=mysql error=ConnectTimeout latencyMs=1490 target=catalog-db-old version=2.14.1 configHash=91ab node=node-17 zone=zone-c
- 2026-09-13T17:20:14.111Z WARN service=product-api instance=prod-a12f traceId=bi-8812 spanId=cat-a1 requestId=req-812 endpoint=/products/44 downstream=catalog-service error=ReadTimeout latencyMs=1502 target=catalog-7c9f attempt=1
- 2026-09-13T17:20:15.009Z INFO service=catalog-service instance=catalog-3 traceId=bi-8813 spanId=cat-server requestId=req-813 endpoint=/products/44 downstream=mysql error=none latencyMs=81 target=catalog-db-primary version=2.14.1 configHash=c440
- 2026-09-13T17:21:02.700Z INFO service=catalog-service instance=catalog-7c9f traceId=none spanId=none requestId=none endpoint=none downstream=none event=READINESS_FALSE activeRequests=17 draining=true

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

One catalog pod receives a stale configuration bundle.
  ->
Its datasource points to a retired MySQL host that drops connections.
  ->
Readiness checks only whether Spring Boot is alive.
  ->
The load balancer sends about one eighth of traffic to the pod.
  ->
Selected requests wait 1.5 seconds and time out.
  ->
Retries often succeed elsewhere, masking first-attempt failure.
  ->
Aggregate dashboards dilute the 94% target error to 11.8%.
  ->
Configuration drift plus inadequate readiness causes the incident.

The first line is the initiating fault. Later lines are amplifiers and propagation; fixing only the final timeout would leave the chain intact.

# Fix

### Immediate mitigation
- Mark catalog-7c9f unready and verify endpoint removal.
- Drain its 17 active requests, then terminate only that pod.
- Keep at most one bounded GET retry only while capacity and deadline allow.

### Permanent design
- Make image and configuration immutable and atomic.
- Expose nonsecret config fingerprints in metrics and logs.
- Gate canaries on per-target error, latency, and business results.
- Make readiness test a bounded representative dependency operation.
- Test graceful draining and cautious outlier ejection.
- Document retry eligibility, owner, attempts, timeout, backoff, jitter, budget, and idempotency.
- Keep mitigation and permanent repair distinct; reconcile business effects after technical recovery.
- Do not blindly add timeout, CPU, memory, workers, or retries without evidence of the constrained resource.

# Verification

### Comparable before and after
- Global errors: 11.8% -> 0.3% after endpoint removal.
- First-attempt success: 87.9% -> 99.7%.
- Retry ratio: 1.11 -> 1.00 at 4,800/min useful demand.
- All replacement pods have hash c440 and catalog-db-primary.
- Active requests drain 17 -> 0 before exit.
- Healthy target P95 stays below 120 ms.
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
I would begin by separating original demand, downstream attempts, and the required business outcome. Here, about one eighth of product requests time out because one ready instance has stale database configuration. I would compare RED and saturation metrics across every hop, find the first signal that changed, and inspect a failed trace including each retry span. I would correlate it with logs by traceId, target, version, config, node, zone, and remaining deadline. I would contain the incident with bounded concurrency, bulkheads, truthful load shedding, and a calibrated circuit, not blanket retries or longer timeouts. Any retry needs a transient failure, idempotency, one owner, exponential backoff with jitter, retry budget, and enough deadline. After fixing the initiating fault, I would verify useful throughput, tail latency, queues, attempts, and business correctness before declaring recovery.

### Common interviewer traps
- Aggregate metrics hide target heterogeneity.
- Retries mask the bad capacity instead of fixing it.
- Restarting every pod discards healthy capacity.
- One bad pod in zone-c does not prove a zonal outage.
- Liveness is not business readiness.
- Treating a health endpoint as proof that the affected business operation works.
- Treating one log message or the last visible timeout as root cause.

### Quick memory flow
Symptom -> business invariant -> scope -> originals versus attempts -> first changed signal -> trace retries -> correlate logs -> target and saturation comparison -> contain -> root cause -> permanent fix -> verify business recovery

# Interview Follow-up Questions

### Q1. Why does one of eight matter?
Answer: Equal random routing gives each first attempt a 12.5% chance of hitting it.

### Q2. How do you prove the target?
Answer: Compare peer instance, version, config, node, zone, errors, latency, and business results.

### Q3. Why can retry hide it?
Answer: A second selection usually lands on one of seven healthy pods, lowering final errors but adding work.

### Q4. Should it be restarted immediately?
Answer: Remove readiness and drain first so in-flight work is not abruptly duplicated.

### Q5. What if all zone-c pods fail?
Answer: Then test zonal routing and dependencies; healthy zone-c peers here contradict that scope.

### Q6. Can outlier detection solve it?
Answer: It mitigates with minimum volume and ejection limits, but config validation and readiness fix the cause.
