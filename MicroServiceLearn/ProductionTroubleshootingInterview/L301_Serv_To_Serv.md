# Service-to-Service Production Troubleshooting Through Fifteen Incidents

## How this continuous story works

This guide follows one production engineer through fifteen incidents rather than presenting a theory manual.
The running system is an online store.
Order API, Service A, receives `POST /v1/orders` in Kubernetes.
A calls an Envoy gateway or mesh, which selects Inventory API, Service B.
B uses PostgreSQL, Redis, and Pricing Service C.
Examples use UTC timestamps, request IDs, instances, versions, addresses, and safe redacted output.
Metric names are examples: exact names and 502/503/504 semantics vary by stack and product.

```text
customer -> Order A -> A HTTP client/sidecar -> DNS -> TCP -> TLS
         -> gateway/load balancer/mesh -> Inventory B -> DB/cache/Pricing C
         -> reservation business outcome
```

## Terms I define before using them

A **counter** only increases until restart, such as `http_requests_total`; its rate over a window gives requests or errors per second.
A **gauge** rises and falls, such as queue depth or healthy-target count.
A **histogram** groups observations into buckets so duration distributions and percentiles can be estimated.
The **p50** is the median; **p95** and **p99** expose the slowest 5% and 1%; **max** is the largest sample.
Average can remain 120 ms while one percent wait five seconds, so I inspect p50/p95/p99/max and counts.

**RED** means Rate, Errors, Duration at each request boundary.
**USE** means Utilization, Saturation, Errors for resources.
Utilization is busy capacity; saturation means work waits; an error can be rejection, throttle, reset, or allocation failure.
**Queueing** is waiting before work begins.
**In-flight concurrency** is active work.
Little's Law approximates `concurrency = throughput x time`: 200 requests/s at 0.1 s needs about 20 in flight, but at 5 s tends toward 1,000.
That is why a flat request rate plus a slow dependency can exhaust threads and pools.

A **correlation ID** joins logs for one request.
A **trace** represents the distributed request; a **span** is one timed operation.
A **parent span** invokes or contains a **child span**, such as a B server parent and DB-query child.
Span status and attributes identify error, route, target, instance, version, zone, and retry.
Tracing may be sampled, dropped, misconfigured, or absent across legacy hops.
No B span directs me before B only after I corroborate sampling and access logs.

**DNS lookup duration** times name resolution.
**TCP connect duration** normally times the SYN/SYN-ACK/ACK handshake.
**TLS handshake duration** times secure negotiation and identity/trust validation.
**Time to first byte** ends at the first response byte; **total duration** ends after the response body.
Normal connect with slow first byte points after connection; normal first byte with slow total points to transfer or a slow consumer.

**Client metrics** describe the caller's observation; **server metrics** describe accepted work at the receiver.
A attempts rising while B accepts stay flat means work stops between them, an intermediary rejects it, or telemetry is incomplete.
A **retry attempt** is an extra try; retries can turn 200 user requests/s into 600 backend attempts/s.
A **backend/target label** identifies the selected instance and exposes one-target faults.

For HTTP/DB pools, **active** is checked out, **idle** is reusable, and **pending** waits to acquire.
Pool **acquisition** duration measures that wait.
A full pool does not prove it is undersized: leaks and slow holders are common causes, and enlarging it can overload downstream.
A **timeout** is a local wait limit.
A **deadline** is the remaining end-to-end budget; it should decrease downstream and include all retries.
More timeout does not make work faster.

## Dashboard and universal layered metric map

I use a single time cursor with deployment/config/certificate/network/job annotations.
The top row is orders attempted, reservations committed, and synthetic checkout.
Then I follow A server RED, A client phase RED, gateway downstream/upstream RED, B server RED, B dependency RED, and USE for every resource.
Every panel must split by route, status or exception, source, target, instance, zone, version, connection reuse, and retry attempt where applicable.

| Layer and example metric family | What/where it measures | Why it changes and how I split it | What it proves, does not prove, and next evidence |
|---|---|---|---|
| Config: `config_revision_info`, `build_info` | loaded revision/version from process or deployment | rollout/reload/drift; instance/version/zone | reported state, not code-path use; compare effective runtime config with good peer |
| DNS: `dns_lookup_duration_seconds`, `dns_queries_total{rcode}` | resolver latency, code, TTL, answers at A/DNS | slow forwarder, NXDOMAIN, SERVFAIL, stale answer; name/zone/resolver/source | resolver observation only; query from A and reconcile discovery |
| TCP attempts/success/refusal/timeout | outbound socket outcomes at A/proxy | refusal is fast RST/no listener; timeout often drop; source/IP/port/zone | client socket outcome, not owning firewall; inspect listener/flow/route |
| TCP handshake/retransmit/reset | connect time and transport quality at kernel/proxy | loss causes retransmission/tails; resets actively close; node/target/direction | transport symptom, not faulty device; compare nodes and flow telemetry |
| TLS handshake/failure/cert expiry | negotiation, reason, identity at each TLS peer | SNI/SAN/CA/mTLS/protocol/cert rotation; SNI/target/issuer/version | secure-session result, not HTTP; inspect effective chain/trust/policy |
| Gateway downstream status/duration | what gateway returned to A | policy or upstream result; route/generator/subreason/gateway | response at gateway; generator requires headers/log/product semantics |
| Gateway upstream connect/response | gateway-to-B connect, first byte, result | connect issue before B; response delay in/after B; cluster/target | gateway hop only; compare direct path and trace |
| Healthy/unhealthy targets and reasons | gateway/LB eligibility view | port/path/TLS/network/readiness/overload; target/zone/reason | configured probe only; compare listener and business contract |
| Active/idle/new connections and age | connection reuse/churn in A/proxy | churn, pending pool, stale keep-alive; target/protocol/reuse | pool state, not root cause; inspect holders/draining/dependency |
| Retry count and attempts/request | original/repeated attempts at client/proxy | retryable failure amplifies load; reason/attempt/target | amplification, not benefit; inspect deadline/downstream capacity |
| Circuit breaker state/rejections | closed/open/half-open admission | thresholds reached after failures; destination/caller | policy action, not dependency cause; inspect trigger and settings |
| A HTTP client RED and p50/p95/p99/max | caller request outcome and phase timing | demand/errors/tails; route/exception/target/source/version | A observation; compare gateway/B rate and trace |
| Request/worker queue and in-flight | waiting/active work at proxy/service | arrivals exceed completions or service time rises; route/instance | local saturation, not why service slowed; inspect child work/Little's Law |
| Thread active/max/queued/rejected | executor capacity and admission | active=max plus queue means saturation; pool/instance | executor pressure; more threads unsafe until blocked work/capacity known |
| B HTTP server RED | accepted rate/status/duration/in-flight | handler outcome; route/status/source/instance/version | B accepted work, not internal owner; navigate child spans |
| CPU/throttling | compute use/quota cap at container/node | load/code or quota; instance/node/version | low CPU can mean waiting; inspect queues/pools/dependencies |
| Memory heap/RSS/OOM | managed and resident memory | leak/cache/payload/preload; instance/version | symptom; correlate allocation, config, GC and heap evidence |
| GC pause/count | runtime collection cost | allocation/full heap causes pauses; instance/generation | long stop-the-world pause explains gaps; find heap cause |
| Network bytes/errors/loss | traffic and interface quality | payload/load/loss/bad link; source/target/node/zone | correlation only; compare direction/devices/flow |
| HTTP/DB pool active/idle/pending/acquire | checkout and wait inside A/B | leak/slow holder/pool exhaustion; pool/instance/dependency | wait boundary; inspect holder span and downstream capacity |
| B dependency RED | DB/cache/C rate/error/latency | slow/error/retry; operation/fingerprint/target/B instance | child boundary; inspect dependency evidence |
| DB query/pool/lock | execution, rows, waits, locks, connections | scan/index/lock/saturation; fingerprint/host | mechanism when trace-linked; read-only activity/plan evidence |
| Business success | orders/reservations/synthetic outcome | technical or semantic failure; region/product/tenant | customer outcome, not layer; work backward by request ID |

### Concrete metric-name examples

The names below are examples only and vary by library, runtime, proxy, gateway, exporter, and monitoring backend.
I verify the local metric definition, unit, histogram buckets, and label cardinality before writing a query or alert.

| Question I ask | Example names I may find |
|---|---|
| How much user and backend traffic exists? | `http_server_requests_total`, `http_client_requests_total`, `envoy_cluster_upstream_rq_total` |
| Which component generated each status? | `http_responses_total{generator,status}`, `envoy_http_downstream_rq_xx`, `nginx_ingress_controller_requests{status}` |
| What succeeds, errors, or times out? | `http_client_requests_total{outcome}`, `http_client_timeouts_total{phase}`, `inventory_reservations_total{outcome}` |
| What are p50/p95/p99/max? | `http_client_request_duration_seconds`, `http_server_request_duration_seconds`, `gateway_upstream_response_time_seconds` |
| Is name resolution slow or failing? | `dns_lookup_duration_seconds`, `dns_queries_total{rcode}`, `coredns_dns_request_duration_seconds` |
| Did TCP connect, refuse, or time out? | `tcp_connect_attempts_total`, `tcp_connect_success_total`, `tcp_connect_errors_total{reason}` |
| Is transport lossy or resetting? | `node_netstat_Tcp_RetransSegs`, `tcp_resets_total`, `node_network_receive_errs_total` |
| Did TLS negotiate and is expiry near? | `tls_handshake_duration_seconds`, `tls_handshake_failures_total{reason}`, `x509_cert_not_after` |
| Where did gateway time go? | `gateway_upstream_connect_time_seconds`, `gateway_upstream_response_time_seconds`, `envoy_cluster_upstream_cx_connect_ms` |
| Are targets eligible, and why not? | `healthy_target_count`, `unhealthy_target_count{reason}`, `probe_success{target}` |
| Are connections reused or churning? | `http_pool_active_connections`, `http_pool_idle_connections`, `http_client_connections_created_total` |
| Are retries amplifying demand? | `http_client_attempts_total{attempt}`, `http_client_retries_total{reason}`, `requests_per_original_request` |
| Is a circuit breaker rejecting calls? | `circuit_breaker_state`, `circuit_breaker_rejected_total`, `envoy_cluster_upstream_rq_pending_overflow` |
| Is work queueing? | `http_server_in_flight_requests`, `request_queue_depth`, `request_queue_wait_seconds` |
| Is the worker executor saturated? | `executor_active_threads`, `executor_pool_size_threads`, `executor_queued_tasks`, `executor_rejected_tasks_total` |
| Is CPU capped? | `container_cpu_usage_seconds_total`, `container_cpu_cfs_throttled_seconds_total` |
| Is memory or GC pausing work? | `jvm_memory_used_bytes`, `process_resident_memory_bytes`, `jvm_gc_pause_seconds`, `dotnet_gc_pause_time_seconds` |
| Is an HTTP pool saturated? | `http_pool_active`, `http_pool_idle`, `http_pool_pending`, `http_pool_acquire_duration_seconds` |
| Is the DB pool saturated? | `db_pool_active`, `db_pool_idle`, `db_pool_pending`, `db_pool_acquire_duration_seconds` |
| Is a B dependency slow? | `dependency_requests_total`, `dependency_request_duration_seconds`, `dependency_errors_total` |
| Is the DB executing, scanning, or waiting? | `db_query_duration_seconds{fingerprint}`, `db_rows_examined_total`, `db_lock_wait_seconds`, `db_deadlocks_total` |
| Did the business operation really complete? | `orders_completed_total`, `inventory_reservations_total`, `synthetic_checkout_success` |

## Shape relationships I use

If request rate rises first and B latency/errors rise after it, overload is possible because arrivals outpace completion.
I next inspect in-flight, queue wait, thread active/max/queued/rejected, CPU/throttling, pool pending, and downstream capacity.
If rate is flat but latency jumps at a rollout, service time, capacity, config, or dependency changed.
If A latency rises while B server-span latency stays normal, time is before B, after B, or in A pool/retries.
If B is slow and DB child is 4.7 s of 5.1 s, the DB path owns most observed latency.
If no B span exists, investigate before B while checking tracing completeness.
A flat B rate while A attempts rise is evidence of a pre-B loss/rejection, not a reason to inspect B's DB.
Low CPU during high latency suggests waiting, not health.
A spike pinned at exactly 3 or 5 seconds usually exposes a timeout boundary.
A fleet average can hide one bad replica; one of eight failing targets predicts about 12.5% errors under even balancing.

## Example trace waterfall and navigation

```text
A server total                                     5,120ms
  A HTTP pool acquire                                 11ms
  DNS                                                  2ms
  TCP                                                   8ms
  TLS                                                  14ms
  gateway                                           5,040ms
    upstream connect                                    7ms
    B server                                         4,930ms
      worker queue                                     52ms
      DB pool acquire                                 111ms
      DB query                                      4,702ms
      serialize                                         6ms
```

I begin at A's root, follow parent to client child, gateway server/upstream child, B server child, and dependency children.
I inspect status, exception event, route, target, instance, version, zone, attempt, reuse, and deadline.
Small arithmetic mismatches can come from overlap, clocks, and instrumentation boundaries.
I use the waterfall to choose logs/config/code/query evidence, not claim false precision.

## Production-safe command stories

I run bounded probes from A's real runtime or an approved equivalent; a laptop differs in DNS, route, identity, proxy, trust, and policy.
I never use traffic floods, scans, destructive DB/Kubernetes commands, blind restarts, secrets, or unredacted payloads.

```powershell
Resolve-DnsName inventory-b.internal -DnsOnly
Test-NetConnection inventory-b.internal -Port 8443 -InformationLevel Detailed
curl.exe --verbose --connect-timeout 3 --max-time 8 https://inventory-b.internal/health
```

`Resolve-DnsName` shows A-host resolver answers/TTL: NXDOMAIN means absent in that view; SERVFAIL means resolution failed.
`Test-NetConnection` proves one TCP handshake when true; false does not name the dropping device.
`curl` separates connect/TLS/HTTP for one sample; a 401 proves an HTTP responder, not business success.
Next I compare with metrics and a matched source/target.

```bash
getent ahosts inventory-b.internal
dig +time=2 +tries=1 inventory-b.internal
nc -vz -w 3 inventory-b.internal 8443
curl -sS -v --connect-timeout 3 --max-time 8 -o /dev/null -w 'code=%{http_code} remote=%{remote_ip} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://inventory-b.internal/health
openssl s_client -connect 10.42.7.18:8443 -servername inventory-b.internal -verify_return_error -brief </dev/null
```

`getent` follows application name-service configuration; `dig` exposes DNS details but may differ from runtime caching.
`nc` tests one TCP port: fast refusal selects listener/target; bounded timeout selects route/drop/reachability.
`curl` success example is `code=200 remote=10.42.7.18 connect=0.008 tls=0.021 ttfb=0.034 total=0.035`.
`openssl` tests explicit SNI; output `Verification: OK` succeeds, while hostname/issuer/mTLS errors select TLS.
Its trust store may differ from A's, so I inspect effective A trust next; disabling validation is never the fix.

```bash
kubectl get service -n shop inventory-b -o yaml
kubectl get endpointslice -n shop -l kubernetes.io/service-name=inventory-b -o wide
kubectl get pods -n shop -l app=inventory -o wide --show-labels
kubectl describe pod -n shop <inventory-b-pod>
kubectl get networkpolicy -n shop -o yaml
kubectl exec -n shop <service-a-pod> -- getent hosts inventory-b
```

These read selectors, ports, endpoints, readiness/events, labels, and policies; `exec` tests A's container namespace when tools exist.
A debug pod may have different labels, account, sidecar, DNS, or policy, so it is not automatically equivalent.
`Running` does not prove ready, listening, selected, reachable, or business-capable.

```bash
ss -lntp | grep ':8080'
```

On B, `0.0.0.0:8080` accepts non-loopback IPv4; `127.0.0.1:8080` is loopback only; no line means no listener in that namespace.
I compare with sidecar namespaces and port mappings before concluding.

```sql
SELECT pid, wait_event_type, wait_event, state, query_start
FROM pg_stat_activity
WHERE datname = 'inventory'
ORDER BY query_start;
```

This authorized read-only query can show a lock wait; it neither changes data nor links itself to a request.
I use the trace fingerprint and timestamp to establish that link and avoid expensive unbounded diagnostics.

## Incident discipline

In minute one I record UTC, route, exact exception/status, latency, request/trace IDs, source, target, version, zone, timeout, and generator.
By minute two I scope status/route/source/target/version/zone/tenant and new versus reused connection.
By minute three I open business plus layered RED/USE and change annotations.
By minute four I locate the last good boundary by adjacent rates and phase durations.
By minute five I preserve one failure and one matched success before any drain/rollback destroys evidence.
Mitigation reduces harm; root cause explains mechanism.
I never propose larger timeout, retries, pool, threads, replicas, or restart without evidence and downstream-capacity analysis.

---

# Incident 1: An Unknown Failure Becomes a TLS Diagnosis

## Interview question

> Service A is unable to connect to Service B. How would you investigate?

## The page arrives

At 09:14 UTC on 2026-09-13, an unknown failure becomes a TLS diagnosis.
The first failed request is `ord-7f93c2`, trace `01f92f3577b34da6a3ce929d0e0e0001`, from `order-a-4.18.2-k2m5q` toward `10.42.7.18:8443` and target label `inventory-b-r8x2p`.
The measured scope is 42% of POST /v1/orders in eu-west-1; catalog and us-east-1 remain normal.
The first metric sentence is: 220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `09:14 UTC | request=ord-7f93c2 | source=order-a-4.18.2-k2m5q | target=10.42.7.18:8443 | scope=42% of POST /v1/orders in eu-west-1; catalog and us-east-1 remain normal`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: 42% of POST /v1/orders in eu-west-1; catalog and us-east-1 remain normal.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `TLS between A's sidecar and B`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `01f92f3577b34da6a3ce929d0e0e0001`, sanitized logs for `ord-7f93c2`, target `inventory-b-r8x2p`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Business impact and A demand

I begin with the user result. Order intake remains 220 requests/s, but successful reservations fall from 99.95% to 58% at 09:12:41. Because demand did not rise before the failure, an overload explanation has no support. The loss appears only on `POST /v1/orders` in `eu-west-1`, so I compare the changed A configuration there with the unchanged region.

A's inbound rate and CPU remain steady, while its Inventory client changes from nearly all success to 92 TLS failures/s. There are no HTTP statuses on those failed attempts. An HTTP 4xx or 5xx would prove an HTTP responder was reached; a TLS exception says the exchange ended earlier.

### DNS, TCP, and TLS phases

DNS for `inventory-v2.internal` remains p50 1 ms, p99 3 ms, with `NOERROR` and address `10.42.7.18`. If lookup latency or SERVFAIL had risen, I would stay with the resolver path. Here the answer is fast and stable, so I carry the selected IP into the TCP panel.

TCP succeeds in 7 ms p95 with no refusal, timeout, retransmission, or reset increase. That rules out a missing listener and a dropped SYN for this failed dataset. If connect time had pinned at five seconds, I would inspect route and policy; if it had failed in 2 ms with refusal, I would inspect the listener.

The next phase changes sharply: `tls_handshake_failures_total{reason="hostname_mismatch",sni="inventory-v2.internal"}` rises from zero to 92/s. Successful handshakes using `inventory.internal` remain 14 ms p99. Certificate expiry is 41 days away, so expiry is not the issue; the reason label and SNI split point to identity selection.

### Gateway, target, and connection evidence

The sidecar records successful TCP connects but no upstream HTTP request for the failures. B's server request rate is flat rather than elevated, and no failed request ID appears in B access logs. That count boundary agrees with the missing B span: validation stopped before B's HTTP server.

Target-health count stays at three because the probe uses the old SNI. A green probe therefore does not clear the client-specific TLS name. Active and idle connection counts remain normal; new connections fail only after revision `cfg-a-883`, while old pooled TLS sessions briefly continue to work. That transition explains the 42% rather than immediate 100% impact.

Retries remain disabled for certificate validation errors, and the circuit breaker stays closed. Had retries risen, they would not repair deterministic identity mismatch and could amplify handshakes.

### B runtime and dependencies

B CPU is 34%, heap 57%, worker queue zero, DB pool pending zero, and DB/cache/Pricing C latency unchanged. Those normal values are expected because the failed calls never cross the TLS boundary. They are controls, not proof by themselves.

I now compare explicit SNI values from A's namespace. `openssl s_client` reports a hostname mismatch for `inventory-v2.internal` and `Verification: OK` for `inventory.internal`. The effective A config shows `cfg-a-883`, which changed only that destination name. That sequence links flat demand, successful DNS/TCP, failed TLS, absent B traffic, and the exact configuration change.

## Trace: follow parent to the failing child

I search trace `01f92f3577b34da6a3ce929d0e0e0001` and start at the A server parent.
The failed waterfall reads `A server 63ms -> inventory client 18ms ERROR -> DNS 1ms -> TCP 7ms -> TLS 10ms ERROR; no B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-7f93c2`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-r8x2p`, and `server.address=10.42.7.18:8443` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `SSLPeerUnverifiedException: no SAN matching inventory-v2.internal`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T09:14:22.417Z level=ERROR request_id=ord-7f93c2 trace_id=01f92f3577b34da6a3ce929d0e0e0001
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-r8x2p target=10.42.7.18:8443
message="SSLPeerUnverifiedException: no SAN matching inventory-v2.internal"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare openssl handshakes with old/new SNI.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-7f93c2 source=order-a-4.18.2-k2m5q target=10.42.7.18:8443
RESULT new SNI hostname mismatch; old SNI Verification: OK
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 09:14 UTC the triggering production state changed.
2. A config revision cfg-a-883 changed SNI to inventory-v2.internal while B's certificate SAN contained only inventory.internal.
3. Mechanically, wrong SNI caused identity validation to stop before HTTP.
4. Therefore `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s`.
5. Trace `01f92f3577b34da6a3ce929d0e0e0001` showed `A server 63ms -> inventory client 18ms ERROR -> DNS 1ms -> TCP 7ms -> TLS 10ms ERROR; no B span`.
6. It selected log evidence `SSLPeerUnverifiedException: no SAN matching inventory-v2.internal`.
7. The safe failed/control comparison showed `new SNI hostname mismatch; old SNI Verification: OK`.
8. That explains the customer scope: 42% of POST /v1/orders in eu-west-1; catalog and us-east-1 remain normal.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I roll back cfg-a-883 only.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** A config revision cfg-a-883 changed SNI to inventory-v2.internal while B's certificate SAN contained only inventory.internal.
It lives at `TLS between A's sidecar and B` and explains why wrong SNI caused identity validation to stop before HTTP.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to issue an overlapping certificate with the intended SAN and test SNI/trust pre-deploy.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: success >99.9%, TLS failures 0, p99 <180ms for 30 minutes.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `220/s demand is flat; success falls 99.95% to 58%; TLS failures jump to 92/s` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would first turn "unable to connect" into a precise layer failure. At 09:14 UTC, order demand was steady at 220 requests per second, but reservation success fell to 58% and A reported 92 TLS hostname failures per second. DNS completed in 1 ms and TCP in 7 ms, while B received no matching HTTP requests. The failed trace ended at the TLS child span, and an explicit caller-context handshake failed only with the new SNI. I found that configuration revision `cfg-a-883` changed the name to `inventory-v2.internal`, which was absent from B's certificate. I rolled back that revision to mitigate impact, then issued the correctly scoped certificate and added SNI and trust validation to deployment tests. I verified zero TLS failures, success above 99.9%, normal p99 latency, and correct reservations for thirty minutes.

---

# Incident 2: An Immediate Refusal Reveals a Bind Error

## Interview question

> Service A is getting "Connection Refused" while calling Service B. What could be the reasons?

## The page arrives

At 10:02 UTC on 2026-09-13, an immediate refusal reveals a bind error.
The first failed request is `ord-a61d09`, trace `02f92f3577b34da6a3ce929d0e0e0002`, from `order-a-4.18.2-v7n4j` toward `10.42.7.23:8080` and target label `inventory-b-3`.
The measured scope is one third of reservations, only target B-3.
The first metric sentence is: attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `10:02 UTC | request=ord-a61d09 | source=order-a-4.18.2-v7n4j | target=10.42.7.23:8080 | scope=one third of reservations, only target B-3`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: one third of reservations, only target B-3.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `TCP listener on B-3`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `02f92f3577b34da6a3ce929d0e0e0002`, sanitized logs for `ord-a61d09`, target `inventory-b-3`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Business impact and target probability

Reservation failures begin at 10:02 UTC without a demand increase: A still receives 225 requests/s. The fleet error rate is about 33%, matching one of three evenly weighted B targets. Splitting by `backend` turns the aggregate into a clean result: B-1 and B-2 succeed, while B-3 fails every connection.

A's client error is `Connection refused`, not timeout or HTTP failure. Connect duration for B-3 is 1-3 ms, and refusal count is 74/s. That fast, consistent response means a TCP RST returned; silent firewall drops normally consume the connect budget instead.

### Resolve the selected destination, then inspect TCP

DNS returns the expected three pod addresses in 2 ms p99, and the failed requests consistently select `10.42.7.23`. If answers had included a retired address, I would reconcile discovery. Here B-3 is a current endpoint, so I continue to its port.

There are no SYN timeout or retransmission spikes. `tcp_connect_errors_total{reason="refused",target="10.42.7.23:8080"}` alone rises. TLS metrics have no samples because TCP never establishes, and gateway upstream connect failures map one-for-one to B-3. The gateway's target label is essential; without it the failure looks random.

Target health is misleadingly green because the local probe reaches `127.0.0.1:8080`. New external connections to the pod IP fail; idle pooled connections to the other targets remain healthy. If all targets refused, I would suspect a shared port or deployment. The one-target split sends me to B-3's network namespace.

### Listener, server, and resource comparison

B-3 has zero HTTP server requests for failed IDs, while peers each receive about 75/s. Its CPU, heap, GC, worker queue, and DB pool are normal because rejected connections never execute application work. Dependency traffic from B-3 is also zero. Increasing workers, pools, or replicas would not make a socket listen on the pod interface.

A safe `ss -lntp` comparison shows B-3 listening at `127.0.0.1:8080`, while B-2 listens at `0.0.0.0:8080`. The runtime environment then shows `SERVER_ADDRESS=127.0.0.1` only on B-3 and config hash `legacy-71c`. A local curl succeeds and a pod-IP curl refuses, exactly as that bind predicts.

The alternate fast-refusal branches are a process crash, wrong target port, exhausted accept path, or a rejecting sidecar. Process uptime is continuous, the port mapping matches peers, and the listener output directly selects the bind-address branch.

## Trace: follow parent to the failing child

I search trace `02f92f3577b34da6a3ce929d0e0e0002` and start at the A server parent.
The failed waterfall reads `A 21ms -> client 3ms ERROR refused; no B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-a61d09`, `source=order-a-4.18.2-v7n4j`, `backend=inventory-b-3`, and `server.address=10.42.7.23:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `ConnectException: Connection refused /10.42.7.23:8080`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T10:02:22.417Z level=ERROR request_id=ord-a61d09 trace_id=02f92f3577b34da6a3ce929d0e0e0002
source_instance=order-a-4.18.2-v7n4j backend=inventory-b-3 target=10.42.7.23:8080
message="ConnectException: Connection refused /10.42.7.23:8080"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare ss listener output on B-3 and B-2.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-a61d09 source=order-a-4.18.2-v7n4j target=10.42.7.23:8080
RESULT B-3 127.0.0.1:8080; B-2 0.0.0.0:8080
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 10:02 UTC the triggering production state changed.
2. B-3 loaded SERVER_ADDRESS=127.0.0.1 from a stale node ConfigMap instead of 0.0.0.0.
3. Mechanically, kernel reset pod-IP traffic because only loopback had a listener.
4. Therefore `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero`.
5. Trace `02f92f3577b34da6a3ce929d0e0e0002` showed `A 21ms -> client 3ms ERROR refused; no B span`.
6. It selected log evidence `ConnectException: Connection refused /10.42.7.23:8080`.
7. The safe failed/control comparison showed `B-3 127.0.0.1:8080; B-2 0.0.0.0:8080`.
8. That explains the customer scope: one third of reservations, only target B-3.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I drain B-3 after preserving config and sockets.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** B-3 loaded SERVER_ADDRESS=127.0.0.1 from a stale node ConfigMap instead of 0.0.0.0.
It lives at `TCP listener on B-3` and explains why kernel reset pod-IP traffic because only loopback had a listener.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to remove override, validate non-loopback bind, and gate readiness on remote reachability.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: refusals 0 and B-3 balanced with peers for 30 minutes.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `attempts 225/s; refusals 74/s in 1-3ms; B-3 HTTP rate is zero` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> At 10:02 UTC I saw immediate connection refusals on roughly one third of reservations. The fraction matched one of three targets, so I split A and gateway metrics by backend. Every failure selected B-3, completed in 1-3 ms, and had no TLS phase or B server span. DNS was correct, peers were healthy, and B-3's application resources were idle. Comparing listening sockets showed B-3 bound to `127.0.0.1:8080`, whereas B-2 used `0.0.0.0:8080`; a stale node-specific ConfigMap supplied the loopback value. I preserved the instance evidence and drained B-3 as mitigation. I then removed the override, added startup validation for external binds, and required a remote readiness test. I verified balanced traffic to B-3, zero refusals, and stable business success for thirty minutes.

---

# Incident 3: A Connect Timeout Traces to Policy

## Interview question

> Service A is getting a connection timeout while calling Service B. How would you troubleshoot it?

## The page arrives

At 11:27 UTC on 2026-09-13, a connect timeout traces to policy.
The first failed request is `ord-18b44e`, trace `03f92f3577b34da6a3ce929d0e0e0003`, from `order-a-4.19.0-r4w9p` toward `10.42.7.18:8080` and target label `inventory-b-r8x2p`.
The measured scope is new A 4.19.0 only; old A succeeds; every B is affected.
The first metric sentence is: new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `11:27 UTC | request=ord-18b44e | source=order-a-4.19.0-r4w9p | target=10.42.7.18:8080 | scope=new A 4.19.0 only; old A succeeds; every B is affected`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: new A 4.19.0 only; old A succeeds; every B is affected.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `NetworkPolicy before B TCP`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `03f92f3577b34da6a3ce929d0e0e0003`, sanitized logs for `ord-18b44e`, target `inventory-b-r8x2p`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Version-scoped business and client signals

At 11:27 UTC the customer failure is confined to A version 4.19.0. Old 4.18.2 pods continue reserving against the same B fleet, which makes a shared B outage unlikely. Request rate is ordinary, but the new version records 61 connect timeouts/s, each ending at 5.000 seconds.

A's pool acquisition is 2 ms and DNS lookup is 3 ms with the same B addresses as old pods. Those normal phases place the wait after resolution and before TLS. If DNS were slow, total time would accumulate in the lookup bucket instead of the connect bucket.

### TCP path split by source version

New pods show SYN retransmissions at 4.1/s per pod and zero successful connects. Old pods connect to the identical `10.42.7.18:8080` target in 8 ms. There is no RST, so this is not a closed port; packets are disappearing before the handshake completes. The exact five-second plateau is A's connect timeout, not B execution time.

TLS, gateway upstream, and B server metrics remain flat for new-pod failures because those layers are never reached. B continues serving old A pods at 140 ms p99. A missing B span is therefore expected, and I do not inspect B's database.

### Policy and label evidence

I compare source dimensions before blaming the network generally. Failures follow version 4.19.0 on every node and zone, not one node interface. New pod labels contain `app.kubernetes.io/name=order-api`; the egress NetworkPolicy selects `app=order-api`. Old pods carry both labels and are allowed.

A bounded TCP probe from one old pod returns `Connected ... 8 ms`; the same probe from a new pod times out at 5,000 ms. Read-only policy and flow telemetry show drops for the new source identity. If both pods had timed out, I would move to the destination route/firewall; if one node alone failed, I would compare CNI and node routes.

B CPU, queues, thread pools, HTTP/DB pools, and dependency rates remain normal. Their flat shape is mechanically consistent with never receiving the new calls. Retry count remains one because connect retries are disabled; enabling them would multiply dropped SYNs and consume the order deadline.

The rollout annotation at 11:24:03 precedes the first new-source timeout. The selector mismatch explains the version boundary, retransmission pattern, absent downstream traffic, and recovery when the expected label is restored.

## Trace: follow parent to the failing child

I search trace `03f92f3577b34da6a3ce929d0e0e0003` and start at the A server parent.
The failed waterfall reads `A 5.006s -> client connect 5.001s ERROR; no B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-18b44e`, `source=order-a-4.19.0-r4w9p`, `backend=inventory-b-r8x2p`, and `server.address=10.42.7.18:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `ConnectTimeoutException after 5000ms`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T11:27:22.417Z level=ERROR request_id=ord-18b44e trace_id=03f92f3577b34da6a3ce929d0e0e0003
source_instance=order-a-4.19.0-r4w9p backend=inventory-b-r8x2p target=10.42.7.18:8080
message="ConnectTimeoutException after 5000ms"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare labels/selectors then bounded TCP probes from old/new pods.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-18b44e source=order-a-4.19.0-r4w9p target=10.42.7.18:8080
RESULT old Connected 8ms; new timed out 5000ms
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 11:27 UTC the triggering production state changed.
2. NetworkPolicy selected app=order-api but new pods used app.kubernetes.io/name=order-api.
3. Mechanically, policy dropped SYNs until the five-second connect deadline.
4. Therefore `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise`.
5. Trace `03f92f3577b34da6a3ce929d0e0e0003` showed `A 5.006s -> client connect 5.001s ERROR; no B span`.
6. It selected log evidence `ConnectTimeoutException after 5000ms`.
7. The safe failed/control comparison showed `old Connected 8ms; new timed out 5000ms`.
8. That explains the customer scope: new A 4.19.0 only; old A succeeds; every B is affected.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I pause rollout and restore the expected label.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** NetworkPolicy selected app=order-api but new pods used app.kubernetes.io/name=order-api.
It lives at `NetworkPolicy before B TCP` and explains why policy dropped SYNs until the five-second connect deadline.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to use stable identity labels and add caller-context policy tests.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: new-pod connect p99 <15ms, retransmits baseline, errors 0.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `new A connect timeouts 61/s pinned at 5.000s; B rate flat; SYN retransmits rise` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would scope a connection timeout by caller and target before changing any timeout. Here only A 4.19.0 failed: DNS completed in 3 ms, but TCP connect attempts retransmitted and ended exactly at the five-second budget, while old A pods reached the same B address in 8 ms. No TLS, gateway upstream, or B server span existed for the failed calls. Comparing the new pod labels with the NetworkPolicy showed that the policy selected `app=order-api`, but the rollout supplied only `app.kubernetes.io/name=order-api`. I paused the rollout and restored the stable identity label. The permanent correction used immutable labels and added policy connectivity tests. I verified sub-15-ms connects from new pods, baseline retransmissions, matching B accepts, zero customer errors, and a clean rollout.

---

# Incident 4: A Read Timeout Finds a DB Lock

## Interview question

> Service A connects to Service B, but the response takes too long and eventually gets a Read Timeout. What could be happening?

## The page arrives

At 12:41 UTC on 2026-09-13, a read timeout finds a DB lock.
The first failed request is `ord-c82ee1`, trace `04f92f3577b34da6a3ce929d0e0e0004`, from `order-a-4.18.2-k2m5q` toward `10.42.7.18:8080` and target label `inventory-b-r8x2p`.
The measured scope is warehouse 17 reservation writes; reads and other warehouses normal.
The first metric sentence is: A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `12:41 UTC | request=ord-c82ee1 | source=order-a-4.18.2-k2m5q | target=10.42.7.18:8080 | scope=warehouse 17 reservation writes; reads and other warehouses normal`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: warehouse 17 reservation writes; reads and other warehouses normal.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `B database child`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `04f92f3577b34da6a3ce929d0e0e0004`, sanitized logs for `ord-c82ee1`, target `inventory-b-r8x2p`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Business and A timing

At 12:41 UTC order volume is flat, but warehouse-17 reservations time out at 31/s. Reads and other warehouses remain below 180 ms. A's DNS, connect, and TLS p99 values stay at 3, 8, and 14 ms, so the five-second total is not connection setup.

A's time to first byte rises to the five-second read deadline. The client has 24 active connections, 76 idle, zero pending, and 2 ms pool acquisition. If pool pending were high, the wait would occur before the request left A; instead established connections wait for a response.

### Gateway and B receive the requests

Gateway upstream connect remains 8 ms, while upstream response p99 rises to 5.18 s for warehouse 17. The gateway and B rates match A attempts, and B access logs contain `ord-c82ee1`. That moves the investigation through transport into B.

B p50 remains 128 ms because most routes are healthy, while p99 reaches 5.2 s and max 6.4 s. The warehouse split exposes the tail that an average hides. B in-flight climbs from 28 to 173. At 220 requests/s, adding roughly 0.7 seconds of average residence predicts about 154 extra concurrent requests by Little's Law; the measured growth is plausible.

### Runtime, pools, and the dependency child

B CPU is only 38%, throttling zero, heap 59%, and GC max 27 ms. Workers are waiting rather than computing or pausing. The worker queue reaches 41 but no tasks are rejected. Adding threads would create more blocked transactions against the same rows.

The DB pool is active 38/40, idle 2, pending p99 11 ms. Pool acquisition is not the 4.7-second owner. The failed trace shows the subsequent `UPDATE inventory` child lasting 4.70 s, of which `db.lock.wait` is 4.63 s. Cache and Pricing C spans remain under 20 ms.

Database CPU and IO are normal, but lock-wait p99 for fingerprint `reserve-stock` jumps from 8 ms to 4.7 s. A read-only activity view identifies blocker `pid-2218`, application `inventory-reconcile`, job `job-882`. If rows examined had surged with no lock event, I would investigate an index/plan; if acquisition had consumed the time, I would look for slow holders or leaks.

The batch started at 12:39:52 and changed to one transaction for all warehouse-17 rows. It held locks long enough for A's five-second read deadline to expire, even though TCP and B remained alive. That causal sequence selects pausing the owned batch, not increasing the timeout.

## Trace: follow parent to the failing child

I search trace `04f92f3577b34da6a3ce929d0e0e0004` and start at the A server parent.
The failed waterfall reads `A 5.001s ERROR -> B 5.19s -> DB UPDATE 4.70s lock wait`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-c82ee1`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-r8x2p`, and `server.address=10.42.7.18:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `query=reserve-stock lock_wait_ms=4698`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T12:41:22.417Z level=ERROR request_id=ord-c82ee1 trace_id=04f92f3577b34da6a3ce929d0e0e0004
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-r8x2p target=10.42.7.18:8080
message="query=reserve-stock lock_wait_ms=4698"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to open longest trace child and inspect read-only lock activity.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-c82ee1 source=order-a-4.18.2-k2m5q target=10.42.7.18:8080
RESULT DB UPDATE 4.70s; lock.wait 4.63s blocked_by pid-2218
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 12:41 UTC the triggering production state changed.
2. batch job job-882 held warehouse-17 row locks for 4.7-6.1s in one transaction.
3. Mechanically, B workers connected then waited on locks beyond A's deadline.
4. Therefore `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%`.
5. Trace `04f92f3577b34da6a3ce929d0e0e0004` showed `A 5.001s ERROR -> B 5.19s -> DB UPDATE 4.70s lock wait`.
6. It selected log evidence `query=reserve-stock lock_wait_ms=4698`.
7. The safe failed/control comparison showed `DB UPDATE 4.70s; lock.wait 4.63s blocked_by pid-2218`.
8. That explains the customer scope: warehouse 17 reservation writes; reads and other warehouses normal.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I pause job-882 through its scheduler and let transaction finish.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** batch job job-882 held warehouse-17 row locks for 4.7-6.1s in one transaction.
It lives at `B database child` and explains why B workers connected then waited on locks beyond A's deadline.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to commit small batches, index predicate, and bound lock wait below deadline.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: lock p99 <20ms, B p99 <180ms, timeouts 0, reservations reconcile.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `A read timeout 31/s at 5s; B p99 5.2s; DB lock wait p99 4.7s; CPU 38%` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> At 12:41 UTC I found that read timeouts affected only warehouse-17 reservation writes. A's DNS, TCP, TLS, and pool acquisition were normal, but time to first byte reached the five-second deadline. Gateway and B rates matched A's attempts, and B p99 rose to 5.2 seconds. In the failed trace, a 4.70-second database update contained 4.63 seconds of lock wait; CPU, GC, and pool acquisition were normal. A read-only lock view linked the query fingerprint to reconciliation job `job-882`, which was holding the warehouse rows in one large transaction. I paused that job to mitigate impact. The durable fix used small commits, an indexed predicate, and a lock budget below the request deadline. I verified lock p99 below 20 ms, B p99 below 180 ms, zero timeouts, and correct reservation reconciliation.

---

# Incident 5: Intermittency Exposes Stale DNS

## Interview question

> Service A → Service B works sometimes but times out sometimes. How would you investigate?

## The page arrives

At 13:08 UTC on 2026-09-13, intermittency exposes stale DNS.
The first failed request is `ord-d50bc4`, trace `05f92f3577b34da6a3ce929d0e0e0005`, from `order-a-4.18.2-p8t6d` toward `10.42.18.37:8080` and target label `inventory-b-retired`.
The measured scope is 32-35% of all A calls, clustered on one of three answers and new connections.
The first metric sentence is: two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `13:08 UTC | request=ord-d50bc4 | source=order-a-4.18.2-p8t6d | target=10.42.18.37:8080 | scope=32-35% of all A calls, clustered on one of three answers and new connections`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: 32-35% of all A calls, clustered on one of three answers and new connections.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `service discovery target lifecycle`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `05f92f3577b34da6a3ce929d0e0e0005`, sanitized logs for `ord-d50bc4`, target `inventory-b-retired`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Compare successful and failed populations first

At 13:08 UTC the same route and payload sometimes succeed in about 100 ms and sometimes wait three seconds. I create two datasets rather than averaging them. Failures are 32-35% across A instances and versions, close to one of three DNS answers; successes use `10.42.7.18` or `.19`, while failures begin with `10.42.18.37`.

Business success is partly masked by retry: original orders remain 210/s, but Inventory attempts rise to 286/s, or 1.36 attempts per order. First-attempt success falls much farther than final success. That retry amplification loads the two healthy targets and consumes most of the deadline.

### DNS answer, connection age, and TCP result

DNS returns three addresses with `NOERROR` and normal 4 ms p99, so this is not a resolution failure. Correctness is the issue: the retired `.37` answer remains advertised. I retain answer order, TTL, selected IP, and cache age on every request.

New connections to `.18` and `.19` complete in 7-10 ms. New connections to `.37` retransmit and end at the three-second connect timeout. No refusal means no reachable closed listener. If every address timed out from one zone, I would inspect its route; the per-answer split selects discovery lifecycle.

Reused connections initially show fewer failures because pools already hold healthy `.18` and `.19` sockets. As those age out, failures approach one third. The opposite pattern - new connections succeeding while old ones reset - would point to stale keep-alive or draining instead. I keep that branch explicit because it becomes the mechanism in Incident 14.

### Gateway, targets, retries, and downstream controls

The gateway target inventory contains only the two current B instances, but A's direct discovery cache still includes `.37`. B server rate on `.18` and `.19` rises because of second attempts; there is no B span or access log for `.37`. Healthy-target count at the gateway therefore cannot validate A's separate discovery source.

Attempt 1 to `.37` consumes 3.000 s; attempt 2 to `.18` succeeds in 96 ms when enough deadline remains. Circuit breakers differ by A instance because each sees a local sample, so some open later than others. I cap retry amplification within the existing budget while removing the stale answer; I do not add retries.

B CPU rises modestly from 36% to 51% due to retries, but queues, GC, DB pool, and dependency latency remain healthy. That downstream headroom makes the temporary traffic distribution safe. Had B queue or pool pending climbed, I would reduce retries immediately and shed noncritical work.

Discovery audit logs show that retired VM B-legacy-4 was terminated at 12:56:11 after deregistration failed. Repeated bounded resolution produces `.18 OK 94 ms`, `.19 OK 101 ms`, and `.37 CONNECT_TIMEOUT 3000 ms`. Removing `.37` changes first-attempt success and attempts/order in the predicted direction.

## Trace: follow parent to the failing child

I search trace `05f92f3577b34da6a3ce929d0e0e0005` and start at the A server parent.
The failed waterfall reads `attempt1 target .37 3s ERROR -> attempt2 target .18 96ms OK`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-d50bc4`, `source=order-a-4.18.2-p8t6d`, `backend=inventory-b-retired`, and `server.address=10.42.18.37:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `attempt=1 resolved_ip=10.42.18.37 connect_timeout`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T13:08:22.417Z level=ERROR request_id=ord-d50bc4 trace_id=05f92f3577b34da6a3ce929d0e0e0005
source_instance=order-a-4.18.2-p8t6d backend=inventory-b-retired target=10.42.18.37:8080
message="attempt=1 resolved_ip=10.42.18.37 connect_timeout"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to correlate repeated bounded answers with outcome and attempt.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-d50bc4 source=order-a-4.18.2-p8t6d target=10.42.18.37:8080
RESULT .18 OK 94ms; .19 OK 101ms; .37 TIMEOUT 3000ms
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 13:08 UTC the triggering production state changed.
2. discovery retained a retired VM IP after failed deregistration.
3. Mechanically, one-third random selection hit a dead subnet and retries loaded healthy targets.
4. Therefore `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s`.
5. Trace `05f92f3577b34da6a3ce929d0e0e0005` showed `attempt1 target .37 3s ERROR -> attempt2 target .18 96ms OK`.
6. It selected log evidence `attempt=1 resolved_ip=10.42.18.37 connect_timeout`.
7. The safe failed/control comparison showed `.18 OK 94ms; .19 OK 101ms; .37 TIMEOUT 3000ms`.
8. That explains the customer scope: 32-35% of all A calls, clustered on one of three answers and new connections.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I remove stale discovery address and cap retry amplification.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** discovery retained a retired VM IP after failed deregistration.
It lives at `service discovery target lifecycle` and explains why one-third random selection hit a dead subnet and retries loaded healthy targets.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to make deregistration/draining transactional and reconcile answers to targets.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: all advertised targets healthy and attempts/request 1.00.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `two targets <120ms; 10.42.18.37 times out at 3s; retries amplify 210 to 286 attempts/s` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> For an intermittent timeout, I compare matched successes and failures rather than averaging them. Here failures were 32-35%, which suggested one of three addresses. DNS latency was normal, but every failed first attempt selected the stale address `10.42.18.37` and timed out after three seconds; the two current addresses connected in under 10 ms. Retries hid some customer failures while increasing backend attempts from 210 to 286 per second. Discovery logs showed that a retired VM had not deregistered. I removed that address and bounded retries as mitigation. The permanent fix made draining and deregistration transactional and added reconciliation between advertised answers and live targets. I verified every answer, first-attempt success above 99.9%, attempts per order back to 1.00, and stable capacity on both healthy targets.

---

# Incident 6: Running Pods Hide a Selector Error

## Interview question

> Service B is running, but Service A cannot reach it. What would you check?

## The page arrives

At 14:16 UTC on 2026-09-13, running pods hide a selector error.
The first failed request is `ord-e003f7`, trace `06f92f3577b34da6a3ce929d0e0e0006`, from `order-a-4.18.2-k2m5q` toward `inventory-b.shop.svc:8080` and target label `inventory-b-r8x2p`.
The measured scope is all Service calls; direct pod-IP probes succeed.
The first metric sentence is: EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `14:16 UTC | request=ord-e003f7 | source=order-a-4.18.2-k2m5q | target=inventory-b.shop.svc:8080 | scope=all Service calls; direct pod-IP probes succeed`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: all Service calls; direct pod-IP probes succeed.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `Kubernetes Service selection`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `06f92f3577b34da6a3ce929d0e0e0006`, sanitized logs for `ord-e003f7`, target `inventory-b-r8x2p`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Running is only the first observation

At 14:16 UTC A's business success through `inventory-b.shop.svc` falls to zero, but `kubectl get pods` reports three B pods as Running. Running means the containers exist; it does not prove readiness, Service selection, listening, or reachability. Direct pod-IP business probes succeed in 112-126 ms, so the B process and handler can work.

A request rate is unchanged. Its client receives an immediate mesh 503 in 3 ms rather than a DNS, connect, or read timeout. The short duration says no business execution occurred. I use the mesh response flag, `UH`, to identify `no_healthy_upstream` rather than treating all 503 responses alike.

### DNS, Service, and endpoint counts

Cluster DNS resolves `inventory-b.shop.svc` to the stable ClusterIP in 2 ms with no errors. That proves only the Service name exists. TCP to the virtual address reaches the sidecar path; there is no individual B connection attempt because the mesh cluster has zero eligible endpoints.

The decisive gauge is EndpointSlice ready-address count: it drops from three to zero at 14:14:29. B pod readiness itself remains true. If pod readiness had fallen, I would inspect its configured probe and runtime. Ready pods plus an empty EndpointSlice instead points to selection or publication.

I compare the Service selector `app=inventory` with actual labels `app=inventory-api`. No pod matches. Target port 8080 agrees with the listener, so a port mismatch is rejected. If EndpointSlice contained addresses but the mesh healthy count were zero, I would inspect mesh discovery or health; here both lose endpoints together.

### Downstream metrics stay flat for a reason

There are no B server spans or access logs for Service-routed failures, while direct control calls produce both. B CPU is 31%, queue zero, threads available, heap 55%, and HTTP/DB pools have idle capacity. DB, cache, and Pricing C rates fall only because routed traffic never arrives. Increasing B replicas would create more pods with labels that still do not match.

Connection and retry panels confirm immediate rejection: no upstream socket is created, retries stay at one, and the circuit breaker is not the generator. The mesh refuses because its cluster membership is empty.

Read-only manifest comparison shows the label changed in B revision 772 while the Service selector did not. Restoring a matching, stable label repopulates EndpointSlice with three ready addresses. I also check that the corrected selector does not accidentally include unrelated pods before restoring traffic.

## Trace: follow parent to the failing child

I search trace `06f92f3577b34da6a3ce929d0e0e0006` and start at the A server parent.
The failed waterfall reads `A 14ms -> sidecar 3ms ERROR no_healthy_upstream; no B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-e003f7`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-r8x2p`, and `server.address=inventory-b.shop.svc:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `response_flags=UH endpoints=0`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T14:16:22.417Z level=ERROR request_id=ord-e003f7 trace_id=06f92f3577b34da6a3ce929d0e0e0006
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-r8x2p target=inventory-b.shop.svc:8080
message="response_flags=UH endpoints=0"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare Service selector, pod labels, and EndpointSlice.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-e003f7 source=order-a-4.18.2-k2m5q target=inventory-b.shop.svc:8080
RESULT selector app=inventory; pods app=inventory-api; endpoints none
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 14:16 UTC the triggering production state changed.
2. B pods changed app=inventory to app=inventory-api while Service selector stayed app=inventory.
3. Mechanically, running listeners were not published as Service endpoints.
4. Therefore `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream`.
5. Trace `06f92f3577b34da6a3ce929d0e0e0006` showed `A 14ms -> sidecar 3ms ERROR no_healthy_upstream; no B span`.
6. It selected log evidence `response_flags=UH endpoints=0`.
7. The safe failed/control comparison showed `selector app=inventory; pods app=inventory-api; endpoints none`.
8. That explains the customer scope: all Service calls; direct pod-IP probes succeed.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I restore label or correct selector after checking unintended matches.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** B pods changed app=inventory to app=inventory-api while Service selector stayed app=inventory.
It lives at `Kubernetes Service selection` and explains why running listeners were not published as Service endpoints.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to render-test selectors and block zero-endpoint deployments.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: three ready addresses and balanced business calls.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `EndpointSlice ready addresses fall 3 to 0; sidecar returns no healthy upstream` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would not equate "Running" with reachable. At 14:16 UTC, all Service-routed Inventory calls failed immediately, while direct pod-IP calls succeeded. DNS resolved the Service normally, but the mesh returned `UH`, meaning no healthy upstream, and EndpointSlice ready addresses had fallen from three to zero. Comparing the Service selector with pod labels showed that deployment revision 772 changed `app=inventory` to `app=inventory-api`. B's CPU, queues, pools, and dependencies were normal because routed requests never arrived. I restored the matching label as mitigation after checking for unintended selector matches. The permanent fix used immutable labels, rendered-manifest selector tests, and a deployment gate for zero endpoints. I verified three ready addresses, balanced calls to every pod, zero mesh errors, and successful reservations.

---

# Incident 7: Ping Success Stops Before HTTPS

## Interview question

> Service A can ping Service B's server, but the API call fails. Why?

## The page arrives

At 15:05 UTC on 2026-09-13, ping success stops before HTTPS.
The first failed request is `ord-f82c11`, trace `07f92f3577b34da6a3ce929d0e0e0007`, from `order-a-vm-12` toward `10.42.7.18:8443` and target label `inventory-b-vm-04`.
The measured scope is ICMP from A node works; TCP 8443 across subnet fails.
The first metric sentence is: ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `15:05 UTC | request=ord-f82c11 | source=order-a-vm-12 | target=10.42.7.18:8443 | scope=ICMP from A node works; TCP 8443 across subnet fails`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: ICMP from A node works; TCP 8443 across subnet fails.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `L3/L4 firewall`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `07f92f3577b34da6a3ce929d0e0e0007`, sanitized logs for `ord-f82c11`, target `inventory-b-vm-04`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Separate ICMP from the application protocol

At 15:05 UTC `ping 10.42.7.18` returns in 1 ms with zero loss, but every API call from A's subnet times out. ICMP echo proves that one host answered one protocol. The API requires TCP 8443, then TLS, HTTP routing, authentication, and B execution; ping exercises none of those later stages.

Order demand remains normal, and A records 100% connect timeouts at exactly five seconds. DNS is not involved in the direct-IP reproduction, which removes name resolution from this test. A successful ping alongside a failed TCP connect is therefore entirely consistent.

### TCP evidence identifies the policy boundary

`Test-NetConnection` from `order-a-vm-12` reports `TcpTestSucceeded=False`. SYN retransmissions rise on A, B records no accepted sockets, and there are no resets. A closed port would normally refuse quickly; retransmission until deadline indicates silent drop or a path that cannot return SYN-ACK.

The failure follows source subnet `10.41.12.0/24` to destination port 8443 across both B instances. It does not follow one A host, B target, node, or DNS answer. That scope favors a shared route, ACL, security group, or firewall rule.

TLS has zero samples for failures, gateway upstream attempts are absent on the direct path, and B HTTP rate is flat. B health, CPU, worker queue, memory, GC, HTTP/DB pools, and DB/cache/C metrics remain normal. Those controls prevent an expensive detour into application code.

### Read-only flow evidence and branches

Firewall flow logs for the exact tuple show `src=10.41.12.24 dst=10.42.7.18 proto=TCP dstport=8443 action=DENY`. ICMP records show `ALLOW`. Policy revision `fw-2041` began at 15:01:06 and omitted the TCP rule.

If flow logs showed ALLOW but B saw no SYN, I would inspect routing and intermediate devices. If B saw SYN and sent SYN-ACK, I would inspect the reverse path. If TCP succeeded and TLS failed, I would move to SNI, trust, and mTLS. The DENY record selects the firewall branch directly.

Retries remain disabled, which avoids turning each user request into repeated five-second waits. Adding replicas or increasing the timeout would not alter the rule. I apply the approved least-privilege tuple and then verify TCP, TLS, authenticated HTTP, and business outcome in sequence.

## Trace: follow parent to the failing child

I search trace `07f92f3577b34da6a3ce929d0e0e0007` and start at the A server parent.
The failed waterfall reads `A 5.004s -> TCP connect 5s ERROR; no TLS/B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-f82c11`, `source=order-a-vm-12`, `backend=inventory-b-vm-04`, and `server.address=10.42.7.18:8443` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `connect timeout source=10.41.12.24`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T15:05:22.417Z level=ERROR request_id=ord-f82c11 trace_id=07f92f3577b34da6a3ce929d0e0e0007
source_instance=order-a-vm-12 backend=inventory-b-vm-04 target=10.42.7.18:8443
message="connect timeout source=10.41.12.24"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to run TCP probe from A and inspect read-only flow logs.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-f82c11 source=order-a-vm-12 target=10.42.7.18:8443
RESULT Ping 1ms; TcpTestSucceeded False; DENY dstport=8443
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 15:05 UTC the triggering production state changed.
2. firewall fw-2041 allowed ICMP but omitted TCP 8443 from 10.41.12.0/24.
3. Mechanically, different protocol matched default TCP drop despite ICMP success.
4. Therefore `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero`.
5. Trace `07f92f3577b34da6a3ce929d0e0e0007` showed `A 5.004s -> TCP connect 5s ERROR; no TLS/B span`.
6. It selected log evidence `connect timeout source=10.41.12.24`.
7. The safe failed/control comparison showed `Ping 1ms; TcpTestSucceeded False; DENY dstport=8443`.
8. That explains the customer scope: ICMP from A node works; TCP 8443 across subnet fails.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I apply approved least-privilege TCP rule.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** firewall fw-2041 allowed ICMP but omitted TCP 8443 from 10.41.12.0/24.
It lives at `L3/L4 firewall` and explains why different protocol matched default TCP drop despite ICMP success.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to generate rules from service contracts and test TCP before closure.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: TCP <10ms, TLS/API success, deny count 0 without broad access.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `ping loss 0% at 1ms; TCP timeout 100% at 5s; B accepts zero` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would explain that ping tests ICMP, not the API's TCP, TLS, or HTTP path. In this case ICMP completed in 1 ms, but A's TCP 8443 attempts retransmitted until the five-second connect deadline, and B accepted no sockets. The failure followed the Order subnet rather than an instance or target. Read-only firewall logs showed ICMP allowed but TCP 8443 denied after revision `fw-2041`. I applied the approved least-privilege rule for the exact source CIDR, destination, protocol, and port. The durable fix generated network rules from service contracts and added caller-context TCP tests to change validation. I verified a sub-10-ms handshake, successful TLS and authenticated API calls, zero matching denies, and restored reservation success without broadening access.

---

# Incident 8: DNS SERVFAIL Finds a Forwarder

## Interview question

> DNS resolution for Service B is failing. How would you troubleshoot it?

## The page arrives

At 16:22 UTC on 2026-09-13, DNS SERVFAIL identifies a failed forwarder.
The first failed request is `ord-01a849`, trace `08f92f3577b34da6a3ce929d0e0e0008`, from `order-a-4.18.2-k2m5q` toward `inventory-b.corp:443` and target label `inventory-b.corp`.
The measured scope is inventory.corp names fail cluster-wide; public and .svc names work.
The first metric sentence is: SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `16:22 UTC | request=ord-01a849 | source=order-a-4.18.2-k2m5q | target=inventory-b.corp:443 | scope=inventory.corp names fail cluster-wide; public and .svc names work`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: inventory.corp names fail cluster-wide; public and .svc names work.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `cluster resolver forwarding`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `08f92f3577b34da6a3ce929d0e0e0008`, sanitized logs for `ord-01a849`, target `inventory-b.corp`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Stop before TCP when resolution fails

At 16:22 UTC A's Inventory client rate becomes 100% name-resolution failure. No resolved address is attached to the failed requests, and TCP attempt count falls to zero. That ordering matters: a firewall at B cannot explain a request that never obtained a destination.

The DNS histogram rises from 4 ms p99 to 2.0 seconds, and the response code is `SERVFAIL`, not `NXDOMAIN`. NXDOMAIN would say the queried name does not exist in that view; SERVFAIL says the resolver could not complete resolution. Negative-cache and retry counters show two attempts inside each A request.

### Compare zones through the same resolver

From the affected pod, `inventory-b.corp` returns SERVFAIL after 2,001 ms. `kubernetes.default.svc` returns NOERROR in 3 ms, and a public control name returns in 11 ms. CoreDNS itself is accepting queries, so a total resolver outage is unlikely. Only the conditional zone `inventory.corp` fails.

The answer is identical across A instances, versions, nodes, and zones, which rejects one runtime cache or node-local resolver. If only one A process failed while `dig` succeeded, I would inspect JVM/.NET caching and search domains. If the result were NXDOMAIN everywhere, I would inspect the record and authoritative zone instead.

CoreDNS metrics show forwarding latency at the two-second timeout and errors only for upstream `10.40.0.53`. Its general CPU, memory, request queue, and cache hit ratio are normal. A route lookup and read-only network telemetry show that the conditional forwarder's address became unreachable after route-table revision `rt-corp-17`.

### Confirm that later layers are absent

A HTTP pool has idle connections but creates no new B connection because resolution fails. TCP, TLS, gateway upstream, target-health, B server, B runtime, pools, and dependencies have no failed-request samples. Their unchanged state is expected and does not weaken the DNS diagnosis.

Retrying DNS at the application level would multiply resolver traffic and consume the order deadline. The circuit breaker opens for name-resolution failure on some A instances, but that is a protective consequence, not the root cause.

Restoring the approved route makes `inventory-b.corp` return its expected private addresses in 7 ms. I verify TTL and answer ownership, then let normal TCP/TLS/API checks prove the later stages. Redundant forwarders and per-zone probes prevent a single conditional path from silently removing the service.

## Trace: follow parent to the failing child

I search trace `08f92f3577b34da6a3ce929d0e0e0008` and start at the A server parent.
The failed waterfall reads `A 2.01s -> dns.lookup 2s ERROR SERVFAIL; no TCP/TLS/B`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-01a849`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b.corp`, and `server.address=inventory-b.corp:443` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `UnknownHostException; server failure after 2 attempts`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T16:22:22.417Z level=ERROR request_id=ord-01a849 trace_id=08f92f3577b34da6a3ce929d0e0e0008
source_instance=order-a-4.18.2-k2m5q backend=inventory-b.corp target=inventory-b.corp:443
message="UnknownHostException; server failure after 2 attempts"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to query exact name plus public and .svc controls through A resolver.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-01a849 source=order-a-4.18.2-k2m5q target=inventory-b.corp:443
RESULT corp SERVFAIL 2001ms; .svc NOERROR 3ms
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 16:22 UTC the triggering production state changed.
2. conditional forwarder 10.40.0.53 became unreachable after route removal.
3. Mechanically, CoreDNS could resolve other zones but failed delegated zone before TCP.
4. Therefore `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero`.
5. Trace `08f92f3577b34da6a3ce929d0e0e0008` showed `A 2.01s -> dns.lookup 2s ERROR SERVFAIL; no TCP/TLS/B`.
6. It selected log evidence `UnknownHostException; server failure after 2 attempts`.
7. The safe failed/control comparison showed `corp SERVFAIL 2001ms; .svc NOERROR 3ms`.
8. That explains the customer scope: inventory.corp names fail cluster-wide; public and .svc names work.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I restore route or use designed redundant resolver.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** conditional forwarder 10.40.0.53 became unreachable after route removal.
It lives at `cluster resolver forwarding` and explains why CoreDNS could resolve other zones but failed delegated zone before TCP.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to configure two reachable forwarders and probe each conditional zone.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: expected answer <10ms from every zone and SERVFAIL 0.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `SERVFAIL 100%; lookup p99 2s; B TCP attempts fall to zero` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> At 16:22 UTC A failed before connection setup: DNS took two seconds and returned SERVFAIL for every `inventory.corp` lookup, while TCP attempts dropped to zero. From the affected pod, Kubernetes and public control names resolved quickly, so the resolver was alive; only the conditional corporate zone failed. CoreDNS forwarding metrics identified upstream resolver `10.40.0.53`, and read-only route evidence showed that network maintenance had removed its route. I restored the approved route, then configured the designed redundant forwarder and added conditional-zone probes. I verified the expected addresses and TTL in under 10 ms from every A zone, followed by successful TCP, TLS, API, and reservation checks. I did not waste time on B's database because the request never reached TCP.

---

# Incident 9: Laptop Success Reveals Split DNS

## Interview question

> The hostname works from your laptop but doesn't work from Service A. What could be wrong?

## The page arrives

At 17:11 UTC on 2026-09-13, laptop success reveals split-horizon DNS.
The first failed request is `ord-1a9e31`, trace `09f92f3577b34da6a3ce929d0e0e0009`, from `order-a-4.18.2-k2m5q` toward `inventory-api.corp:443` and target label `inventory-api.corp`.
The measured scope is corporate laptops resolve; prod-2 pods return NXDOMAIN.
The first metric sentence is: laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `17:11 UTC | request=ord-1a9e31 | source=order-a-4.18.2-k2m5q | target=inventory-api.corp:443 | scope=corporate laptops resolve; prod-2 pods return NXDOMAIN`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: corporate laptops resolve; prod-2 pods return NXDOMAIN.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `DNS view and resolver context`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `09f92f3577b34da6a3ce929d0e0e0009`, sanitized logs for `ord-1a9e31`, target `inventory-api.corp`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Treat the laptop as a different source, not a control

At 17:11 UTC a developer's laptop resolves `inventory-api.corp` to `10.70.8.12` in 8 ms. Production A pods return NXDOMAIN in 5 ms. The laptop success proves only its resolver view, route, trust, proxy, and identity; it does not contradict the pod failure.

A's order rate remains steady, but its Inventory client records `UnknownHostException` for every request. Pool, TCP, and TLS phase counts are zero after cached connections expire. The quick NXDOMAIN is different from the two-second SERVFAIL in Incident 8: this resolver is answering authoritatively that the name is absent from its view.

### Compare the actual resolver contexts

The laptop uses corporate resolver `10.1.0.53`; A uses cluster resolver `10.96.0.10`. I run the same fully qualified query without changing either context. The laptop gets `10.70.8.12`; the pod gets NXDOMAIN. Search suffixes do not explain it because the query is already fully qualified.

All prod-2 pods fail, while a corporate VM succeeds. The outcome follows resolver view, not A instance, version, node, or zone. If one pod alone failed, I would inspect local cache and DNS policy. If both resolvers returned the same address but only A failed, I would proceed to route, firewall, proxy, TLS trust, and identity.

Authoritative-zone inventory shows the record in the desktop private view but not in the prod-2 private zone. The service launch at 16:50 lacked that environment's DNS change. Hard-coding `10.70.8.12` or editing hosts would bypass TTL, failover, ownership, and audit controls, so I reject those workarounds.

### Later-layer and business evidence

Because A has no address, there are no new TCP attempts, TLS handshakes, gateway upstream calls, B server requests, or B dependency calls. B target health remains green from its own network. Those green panels do not prove A can resolve or reach it.

A retries one lookup inside the resolver and then fails within its deadline; application retries remain off. Business synthetic checks from the laptop are green, while the production-source synthetic fails. That source-labeled difference is exactly why synthetics must run from consumer environments.

After publishing the approved record into the production view, A resolves the address and completes an authenticated business request. I compare TTL and intended split-horizon behavior rather than requiring every network to receive an identical public/private answer by accident.

## Trace: follow parent to the failing child

I search trace `09f92f3577b34da6a3ce929d0e0e0009` and start at the A server parent.
The failed waterfall reads `A 7ms -> DNS 5ms NXDOMAIN; no connection/B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-1a9e31`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-api.corp`, and `server.address=inventory-api.corp:443` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `UnknownHostException from nameserver 10.96.0.10`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T17:11:22.417Z level=ERROR request_id=ord-1a9e31 trace_id=09f92f3577b34da6a3ce929d0e0e0009
source_instance=order-a-4.18.2-k2m5q backend=inventory-api.corp target=inventory-api.corp:443
message="UnknownHostException from nameserver 10.96.0.10"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare resolver config and same query from laptop and A.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-1a9e31 source=order-a-4.18.2-k2m5q target=inventory-api.corp:443
RESULT laptop 10.70.8.12; pod NXDOMAIN
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 17:11 UTC the triggering production state changed.
2. record existed in desktop DNS view but not prod-2 private zone.
3. Mechanically, different resolvers backed by different zone views gave different truth.
4. Therefore `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero`.
5. Trace `09f92f3577b34da6a3ce929d0e0e0009` showed `A 7ms -> DNS 5ms NXDOMAIN; no connection/B span`.
6. It selected log evidence `UnknownHostException from nameserver 10.96.0.10`.
7. The safe failed/control comparison showed `laptop 10.70.8.12; pod NXDOMAIN`.
8. That explains the customer scope: corporate laptops resolve; prod-2 pods return NXDOMAIN.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I publish approved record in production private zone.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** record existed in desktop DNS view but not prod-2 private zone.
It lives at `DNS view and resolver context` and explains why different resolvers backed by different zone views gave different truth.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to manage views as code and test from every consumer network.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: approved answer/TTL and authenticated call from A.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `laptop answer 10.70.8.12 in 8ms; pod NXDOMAIN 5ms; A TCP attempts zero` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would reproduce the lookup from Service A rather than rely on my laptop. Here the laptop used resolver `10.1.0.53` and returned `10.70.8.12`, while A used `10.96.0.10` and received NXDOMAIN in 5 ms. Because A never obtained an address, TCP, TLS, gateway, and B metrics had no failed-request samples. The record existed only in the corporate desktop DNS view; the production private zone had been omitted during launch. I published the approved production record instead of hard-coding an IP. The permanent fix managed both views as code and added source-specific DNS and business synthetics. I verified the expected address and TTL from A, then confirmed authenticated API and reservation success from each consumer environment.

---

# Incident 10: Gateway 502 Traces to Upstream TLS

## Interview question

> Service A receives HTTP 502 from the API Gateway. How would you investigate?

## The page arrives

At 18:03 UTC on 2026-09-13, Gateway 502 traces to upstream TLS.
The first failed request is `ord-249bc0`, trace `0af92f3577b34da6a3ce929d0e0e000a`, from `order-a-4.18.2-k2m5q` toward `10.42.7.19:8443` and target label `inventory-b-2`.
The measured scope is gateway gw-2 and B-2; direct correct-SNI HTTPS succeeds.
The first metric sentence is: gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `18:03 UTC | request=ord-249bc0 | source=order-a-4.18.2-k2m5q | target=10.42.7.19:8443 | scope=gateway gw-2 and B-2; direct correct-SNI HTTPS succeeds`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: gateway gw-2 and B-2; direct correct-SNI HTTPS succeeds.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `gateway-to-B TLS; Envoy generated 502`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `0af92f3577b34da6a3ce929d0e0e000a`, sanitized logs for `ord-249bc0`, target `inventory-b-2`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Establish who generated the 502

At 18:03 UTC A receives 48 HTTP 502 responses/s. The response carries `server: envoy`, request ID `ord-249bc0`, and `x-envoy-response-flags: UF`; A itself did not synthesize it, and B has no matching HTTP access record. In this Envoy configuration, `UF` means upstream connection failure. Another gateway may use 502 differently, so I use this product's documented flag and log.

Business demand is steady. A DNS, TCP, and TLS to `gateway.internal` remain normal, and gateway downstream duration is 24 ms. That proves A reached the gateway and received its error quickly. I now switch observation point from downstream to Envoy's B-facing upstream.

### Split gateway upstream phases by backend

Gateway upstream connect to B-2 completes in 9 ms. Refusals and connect timeouts are zero, so the B socket is reachable. Upstream TLS handshake then fails in 11 ms with `SAN_mismatch`, exactly 48/s. No upstream HTTP request or response-time sample exists after that failure.

The error follows B-2 after certificate rotation `cert-447`; other targets remain healthy. Gateway SNI is `inventory.default`, while B-2 presents a certificate for `inventory.shop.svc`. Expiry is normal, and the chain is trusted, narrowing the failure to hostname identity rather than age or CA.

Target health stays green because its probe uses plaintext port 8080, so it cannot validate production TLS on 8443. Active gateway connections to B-2 fall, new connection attempts rise, and no reusable TLS session survives the rotation. If reused sessions worked while only new handshakes failed, that would explain a gradual onset; here the rotation closed old sessions, producing an immediate spike.

### Direct-versus-gateway comparison

From an approved gateway-equivalent context, direct HTTPS with SNI `inventory.shop.svc` succeeds and reaches B-2 in 37 ms. Repeating with Envoy's effective SNI `inventory.default` returns `hostname mismatch`. Direct success does not clear the gateway path; it proves B can serve when the client presents the intended name.

B server rate for failed IDs is zero, and B CPU, queue, GC, HTTP/DB pools, DB/cache/C metrics remain normal. A retry could select another target, but attempts/order has already risen to 1.21, so more retries would hide the defect and increase load. The breaker is still closed because the aggregate cluster has healthy peers.

The chain is therefore A-to-gateway success, gateway-to-B TCP success, upstream TLS identity failure, then Envoy-generated 502. Restoring the known-good SNI binding removes the TLS errors and makes B HTTP counts rise by the same amount.

## Trace: follow parent to the failing child

I search trace `0af92f3577b34da6a3ce929d0e0e000a` and start at the A server parent.
The failed waterfall reads `A 31ms -> gateway 24ms 502 UF -> TCP 9ms -> TLS 11ms ERROR; no B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-249bc0`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-2`, and `server.address=10.42.7.19:8443` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `UF upstream_transport_failure_reason=TLS_error:SAN_mismatch`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T18:03:22.417Z level=ERROR request_id=ord-249bc0 trace_id=0af92f3577b34da6a3ce929d0e0e000a
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-2 target=10.42.7.19:8443
message="UF upstream_transport_failure_reason=TLS_error:SAN_mismatch"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare gateway logs/metrics with direct call using exact gateway SNI.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-249bc0 source=order-a-4.18.2-k2m5q target=10.42.7.19:8443
RESULT server envoy; flags UF; SAN_mismatch
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 18:03 UTC the triggering production state changed.
2. gateway SNI inventory.default mismatched B-2 renewed SAN inventory.shop.svc.
3. Mechanically, gateway completed TCP then rejected B identity before HTTP.
4. Therefore `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero`.
5. Trace `0af92f3577b34da6a3ce929d0e0e000a` showed `A 31ms -> gateway 24ms 502 UF -> TCP 9ms -> TLS 11ms ERROR; no B span`.
6. It selected log evidence `UF upstream_transport_failure_reason=TLS_error:SAN_mismatch`.
7. The safe failed/control comparison showed `server envoy; flags UF; SAN_mismatch`.
8. That explains the customer scope: gateway gw-2 and B-2; direct correct-SNI HTTPS succeeds.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I restore known-good gateway SNI/certificate binding.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** gateway SNI inventory.default mismatched B-2 renewed SAN inventory.shop.svc.
It lives at `gateway-to-B TLS; Envoy generated 502` and explains why gateway completed TCP then rejected B identity before HTTP.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to stage overlapping cert rotation and gateway-origin TLS tests.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: 502/TLS failures 0, B receives calls, direct/gateway both succeed.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `gateway 502 48/s; upstream TLS failures 48/s; connect 9ms; B HTTP zero` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would first identify the 502 generator and its product-specific subreason. At 18:03 UTC A received Envoy responses with flag `UF`, while A-to-gateway DNS, TCP, and TLS were normal. Envoy connected to B-2 in 9 ms, but its upstream TLS handshake failed with `SAN_mismatch`; B received no HTTP request. A direct call succeeded with SNI `inventory.shop.svc`, whereas the gateway's effective SNI, `inventory.default`, failed. Certificate rotation `cert-447` had exposed that mismatch. I restored the known-good gateway SNI binding to mitigate impact, then aligned the certificate and gateway configuration and added gateway-origin TLS tests with overlapping rotation. I verified zero 502 and TLS failures, matching gateway and B request counts, successful direct and gateway paths, normal retries, and restored business success.

---

# Incident 11: Gateway 503 Means Zero Healthy Targets

## Interview question

> Service A receives HTTP 503 from the Gateway. What could be the cause?

## The page arrives

At 19:20 UTC on 2026-09-13, Gateway 503 means zero healthy targets.
The first failed request is `ord-2f4d72`, trace `0bf92f3577b34da6a3ce929d0e0e000b`, from `order-a-4.18.2-k2m5q` toward `gateway.internal:443` and target label `inventory-b-7.6-v2m8s`.
The measured scope is all Inventory routes after B 7.6; direct pod port 8080 works.
The first metric sentence is: 503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `19:20 UTC | request=ord-2f4d72 | source=order-a-4.18.2-k2m5q | target=gateway.internal:443 | scope=all Inventory routes after B 7.6; direct pod port 8080 works`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: all Inventory routes after B 7.6; direct pod port 8080 works.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `gateway target health; gateway generated 503`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `0bf92f3577b34da6a3ce929d0e0e000b`, sanitized logs for `ord-2f4d72`, target `inventory-b-7.6-v2m8s`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Identify the 503 generator before interpreting it

At 19:20 UTC A receives 503 in 4-7 ms for every Inventory route. The response carries Envoy's `UH` flag and gateway instance ID; in this deployment `UH` means no healthy upstream. A 503 generated by B, a maintenance policy, a circuit breaker, or another gateway product could mean something else, so status alone is insufficient.

A-to-gateway DNS, TCP, TLS, and pool acquisition remain normal. A's request rate is unchanged and gateway downstream 503 count equals it. The short, flat duration proves the gateway rejects immediately rather than waiting for B.

### Target eligibility explains the absent upstream request

`healthy_target_count{cluster="inventory"}` drops from three to zero as the final old B target drains at 19:18:32. `unhealthy_target_count{reason="connection_refused"}` rises to three, and every reason names health-check port 8081. Gateway upstream request, connect, and response counters are zero because Envoy does not select an ineligible target.

B 7.6 pods are Running and ready according to Kubernetes. Direct calls to each pod on 8080 return a real reservation response in 118-134 ms. Direct calls to 8081 refuse in 2 ms. That comparison proves the application listener works but the configured gateway probe points at a closed port.

If health checks timed out instead of refusing, I would inspect network path or an overloaded probe handler. If checks returned 404, I would inspect the path. If targets were healthy but 503 carried an overflow or circuit-breaker flag, I would inspect gateway saturation and policy. Here `connection_refused:8081` selects the port contract.

### Connections, B resources, and dependencies

Gateway active upstream connections fall to zero and no new connection is attempted for user traffic. Retrying at A only produces another immediate 503, so attempts/request begins to rise without reaching B. I suppress unnecessary retries within the existing policy rather than increasing them.

B server rate contains only direct probes; CPU is 27%, worker queue zero, heap 52%, GC normal, and HTTP/DB pools have idle capacity. DB, cache, and Pricing C are healthy. Adding B replicas would create more healthy applications that the same wrong probe marks unhealthy.

The rollout diff shows B's management listener moved from 8081 to 8080, while gateway health configuration stayed unchanged. Correcting the health port returns targets one at a time; I wait for each real business probe before restoring full traffic.

## Trace: follow parent to the failing child

I search trace `0bf92f3577b34da6a3ce929d0e0e000b` and start at the A server parent.
The failed waterfall reads `A 9ms -> gateway 5ms 503 UH; no upstream attempt/B span`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-2f4d72`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-7.6-v2m8s`, and `server.address=gateway.internal:443` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `503 response_flags=UH healthy_hosts=0`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T19:20:22.417Z level=ERROR request_id=ord-2f4d72 trace_id=0bf92f3577b34da6a3ce929d0e0e000b
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-7.6-v2m8s target=gateway.internal:443
message="503 response_flags=UH healthy_hosts=0"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare target health reason and configured port to listeners.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-2f4d72 source=order-a-4.18.2-k2m5q target=gateway.internal:443
RESULT unhealthy ConnectionRefused 8081; LISTEN 8080
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 19:20 UTC the triggering production state changed.
2. health-check port remained 8081 after B management listener moved to 8080.
3. Mechanically, gateway removed every target and rejected without upstream attempt.
4. Therefore `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081`.
5. Trace `0bf92f3577b34da6a3ce929d0e0e000b` showed `A 9ms -> gateway 5ms 503 UH; no upstream attempt/B span`.
6. It selected log evidence `503 response_flags=UH healthy_hosts=0`.
7. The safe failed/control comparison showed `unhealthy ConnectionRefused 8081; LISTEN 8080`.
8. That explains the customer scope: all Inventory routes after B 7.6; direct pod port 8080 works.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I restore correct health port or roll back rollout.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** health-check port remained 8081 after B management listener moved to 8080.
It lives at `gateway target health; gateway generated 503` and explains why gateway removed every target and rejected without upstream attempt.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to derive check port from service contract and canary target health.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: healthy targets 3, 503 0, balanced business calls.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `503 100% in 4-7ms; healthy targets 3 to 0; checks refused on 8081` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would determine who generated the 503 and read that product's subreason. At 19:20 UTC Envoy returned `UH` in 4-7 ms, meaning no healthy upstream in this configuration. Its healthy-target count had fallen from three to zero, and every health check was refused on port 8081. Direct business calls to all B 7.6 pods on port 8080 succeeded, so B was serving but the gateway's probe contract was stale. I corrected the health-check port to mitigate impact rather than forcing targets healthy. The permanent fix derived the probe from the service contract and blocked rollouts that would reduce healthy targets to zero. I verified three eligible targets, balanced gateway traffic, zero 503 responses, normal retries, and successful direct and gateway reservation checks.

---

# Incident 12: Gateway 504 Finds a Missing Index

## Interview question

> Service A receives HTTP 504 from the Gateway. How would you investigate?

## The page arrives

At 20:07 UTC on 2026-09-13, Gateway 504 finds a missing index.
The first failed request is `ord-35d218`, trace `0cf92f3577b34da6a3ce929d0e0e000c`, from `order-a-4.18.2-k2m5q` toward `10.42.7.18:8080` and target label `inventory-b-r8x2p`.
The measured scope is SKU family 88 writes; direct calls return 200 only after 4.9s.
The first metric sentence is: 504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `20:07 UTC | request=ord-35d218 | source=order-a-4.18.2-k2m5q | target=10.42.7.18:8080 | scope=SKU family 88 writes; direct calls return 200 only after 4.9s`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: SKU family 88 writes; direct calls return 200 only after 4.9s.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `B DB surfaced as gateway-generated 504`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `0cf92f3577b34da6a3ce929d0e0e000c`, sanitized logs for `ord-35d218`, target `inventory-b-r8x2p`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Use the three-second shape to identify the waiting component

At 20:07 UTC A receives 27 HTTP 504 responses/s after 3.000 seconds. Envoy headers and response flag `UT` identify the gateway as generator; for this product, `UT` means upstream request timeout. A 504 does not by itself prove B was down or identify why B was slow.

Order demand remains normal, and only SKU family 88 is affected. A-to-gateway DNS, TCP, TLS, and pool acquisition remain below 20 ms. Gateway upstream connect to B is 8 ms, but upstream response time crosses the configured three-second budget. That contrast moves the search beyond connection setup.

### Compare gateway and direct B behavior

B receives every failed request and continues processing after the gateway returns 504. Direct calls that bypass the gateway eventually return 200 in about 4.9 seconds. Direct success is not acceptable performance; it confirms that the gateway timeout exposes a slower B operation rather than a gateway connection failure.

Healthy-target count remains eight, no upstream reset occurs, active connections are normal, and retry attempts rise to 1.18 per order. Retrying a five-second query inside a three-second budget cannot help and increases DB work, so I contain retries while diagnosing.

B p50 stays 132 ms, but p99 reaches 4.92 s only for SKU family 88. In-flight requests rise as slow work accumulates. Worker queue is 18, CPU 46%, throttling zero, heap 58%, and GC max 31 ms; runtime pressure is secondary, not the original five-second owner.

### Pool and database waterfall

DB pool acquisition is 14 ms with idle connections available. The failed B span contains a `SELECT stock` child of 4.71 s; cache and Pricing C children are normal. Database lock wait is low, so this differs from Incident 4. Query metric fingerprint `9ac2` shows 8.2 million rows examined and a sequential scan.

A read-only `EXPLAIN` comparison shows `Seq Scan stock rows=8,204,331 duration=4.71s`; the known-good plan uses `idx_stock_sku_warehouse` in 23 ms. Migration `2026.09.13` omitted that index. If lock wait dominated, I would find a blocker; if pool acquisition dominated, I would find slow holders. Neither branch fits these values.

The missing index makes B exceed Envoy's deadline, and missing cancellation lets DB work continue after A has already received 504. Raising the gateway timeout would hide the symptom while increasing concurrency. I use the prior compatible query path as mitigation, then deploy the reviewed index and cancellation fix.

## Trace: follow parent to the failing child

I search trace `0cf92f3577b34da6a3ce929d0e0e000c` and start at the A server parent.
The failed waterfall reads `A 3.02s -> gateway 3s 504 UT -> B 4.92s -> DB SELECT 4.71s`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-35d218`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-r8x2p`, and `server.address=10.42.7.18:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `UT; slow_query fingerprint=9ac2 duration_ms=4712`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T20:07:22.417Z level=ERROR request_id=ord-35d218 trace_id=0cf92f3577b34da6a3ce929d0e0e000c
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-r8x2p target=10.42.7.18:8080
message="UT; slow_query fingerprint=9ac2 duration_ms=4712"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to open 504 trace then compare read-only EXPLAIN with known-good plan.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-35d218 source=order-a-4.18.2-k2m5q target=10.42.7.18:8080
RESULT Seq Scan 8,204,331 rows 4.71s; expected Index Scan 23ms
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 20:07 UTC the triggering production state changed.
2. migration omitted idx_stock_sku_warehouse, scanning 8.2M rows.
3. Mechanically, B exceeded gateway budget in a full scan while cancelled work continued.
4. Therefore `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues`.
5. Trace `0cf92f3577b34da6a3ce929d0e0e000c` showed `A 3.02s -> gateway 3s 504 UT -> B 4.92s -> DB SELECT 4.71s`.
6. It selected log evidence `UT; slow_query fingerprint=9ac2 duration_ms=4712`.
7. The safe failed/control comparison showed `Seq Scan 8,204,331 rows 4.71s; expected Index Scan 23ms`.
8. That explains the customer scope: SKU family 88 writes; direct calls return 200 only after 4.9s.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I route to prior compatible query path and reduce batch load.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** migration omitted idx_stock_sku_warehouse, scanning 8.2M rows.
It lives at `B DB surfaced as gateway-generated 504` and explains why B exceeded gateway budget in a full scan while cancelled work continued.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to deploy reviewed index, plan regression test, and deadline cancellation.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: query p99 <80ms, 504 0, cancellation and reconciliation pass.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `504 27/s pinned 3s; upstream response >3s; connect 8ms; B continues` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would identify the 504 generator and compare connect time with upstream response time. Here Envoy returned `UT` exactly at its three-second upstream deadline. Connection to B took 8 ms, B received the request, and a direct call eventually returned 200 after 4.9 seconds. The B trace placed 4.71 seconds in database query fingerprint `9ac2`; pool acquisition and lock wait were normal, while the query scanned 8.2 million rows. Migration `2026.09.13` had omitted the expected index. I routed the affected operation through the prior compatible query path to mitigate impact. I then deployed the reviewed index, added plan regression tests, and propagated cancellation. I verified query p99 below 80 ms, zero 504 responses, normal attempts per order, stopped cancelled work, and reconciled business results.

---

# Incident 13: One Pod Fails Through GC

## Interview question

> Only one instance of Service B is failing while other instances work. How would you identify the problem?

## The page arrives

At 21:02 UTC on 2026-09-13, one pod fails through GC.
The first failed request is `ord-469f81`, trace `0df92f3577b34da6a3ce929d0e0e000d`, from `order-a-4.18.2-p8t6d` toward `10.42.7.27:8080` and target label `inventory-b-7`.
The measured scope is B-7 only; identical traffic succeeds on B-1 through B-6.
The first metric sentence is: B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `21:02 UTC | request=ord-469f81 | source=order-a-4.18.2-p8t6d | target=10.42.7.27:8080 | scope=B-7 only; identical traffic succeeds on B-1 through B-6`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: B-7 only; identical traffic succeeds on B-1 through B-6.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `B-7 runtime/configuration`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `0df92f3577b34da6a3ce929d0e0e000d`, sanitized logs for `ord-469f81`, target `inventory-b-7`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Let the failure fraction suggest, not prove, one instance

At 21:02 UTC fleet reservation errors are about 12%, close to one of eight evenly weighted B instances. I split by `service.instance.id` and backend rather than treating probability as proof. B-7 has 100% of the slow failures; B-1 through B-6 remain below 160 ms for matched requests.

The same A versions, routes, payload classes, and zones succeed when routed elsewhere. DNS answers, TCP connect at 8 ms, TLS at 13 ms, and gateway upstream connect are normal to B-7. Target health remains green because the lightweight probe happens between pauses.

A's client total pins at five seconds only for B-7. B-7's server span continues to 6.17 seconds, while its DB children total 91 ms. That rejects a shared DB query and leaves a 5.81-second gap inside B's runtime.

### Compare runtime and configuration with a healthy peer

B-7 CPU average is not exceptional, but heap is 96% versus B-6 at 54%. GC maximum jumps to 5.812 seconds with `Pause Full (Allocation Failure)`. A stop-the-world pause explains both the uninstrumented trace gap and why active requests make no progress despite normal dependency spans.

B-7 worker queue spikes during the pause and drains afterward; threads are not permanently maxed, and rejected count remains zero. HTTP and DB pools look active during the freeze because holders cannot run, but acquisition returns to normal immediately afterward. Increasing either pool would retain more paused work and memory.

Network bytes, retransmissions, and resets are normal. Retry attempts disproportionately hit healthy peers and raise their load from 31% to 44%; I account for that headroom before draining B-7. The circuit breaker is slow to open because pauses are intermittent.

Per-instance metadata reveals the same image version but a different config hash and node. B-7 alone has `FEATURE_PRELOAD_CATALOG=true` from a stale node-local injection and loaded a 3.4 GB snapshot. B-6 has `false`. If config matched but one node showed throttling or packet loss, I would pursue the node branch; if all new-version pods showed the pause, I would pursue the rollout.

I preserve GC logs, effective environment, heap summary, node identity, and traces before removing B-7 from service. Replacement without that preservation would recover traffic but destroy the explanation. The known-good configuration keeps heap below 65% and removes full pauses.

## Trace: follow parent to the failing child

I search trace `0df92f3577b34da6a3ce929d0e0e000d` and start at the A server parent.
The failed waterfall reads `A 5s ERROR -> B-7 6.17s with 5.81s uninstrumented gap; DB 91ms`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-469f81`, `source=order-a-4.18.2-p8t6d`, `backend=inventory-b-7`, and `server.address=10.42.7.27:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `Pause Full Allocation Failure 5812ms heap_after=3050MiB`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T21:02:22.417Z level=ERROR request_id=ord-469f81 trace_id=0df92f3577b34da6a3ce929d0e0e000d
source_instance=order-a-4.18.2-p8t6d backend=inventory-b-7 target=10.42.7.27:8080
message="Pause Full Allocation Failure 5812ms heap_after=3050MiB"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare B-7 version/config/node/heap/GC with B-6.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-469f81 source=order-a-4.18.2-p8t6d target=10.42.7.27:8080
RESULT B-7 preload=true heap96%; B-6 false heap54%
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 21:02 UTC the triggering production state changed.
2. B-7 alone loaded 3.4GB snapshot via stale FEATURE_PRELOAD_CATALOG=true.
3. Mechanically, heap pressure caused stop-the-world full GC while peers remained normal.
4. Therefore `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8`.
5. Trace `0df92f3577b34da6a3ce929d0e0e000d` showed `A 5s ERROR -> B-7 6.17s with 5.81s uninstrumented gap; DB 91ms`.
6. It selected log evidence `Pause Full Allocation Failure 5812ms heap_after=3050MiB`.
7. The safe failed/control comparison showed `B-7 preload=true heap96%; B-6 false heap54%`.
8. That explains the customer scope: B-7 only; identical traffic succeeds on B-1 through B-6.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I drain B-7 after preserving GC/config evidence.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** B-7 alone loaded 3.4GB snapshot via stale FEATURE_PRELOAD_CATALOG=true.
It lives at `B-7 runtime/configuration` and explains why heap pressure caused stop-the-world full GC while peers remained normal.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to remove node injection, enforce config hashes, and bound preload memory.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: every target p99 <200ms, heap <65%, GC p99 <50ms.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `B-7 p99 6.2s vs 160ms; GC max 5.8s; heap 96%; fleet errors 1/8` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would start with the failure fraction and then prove the target correlation. About 12% of calls failed, and every one selected B-7; matched calls to the other instances succeeded. DNS, TCP, TLS, gateway connect, and B's dependency spans were normal, but B-7's trace contained a 5.81-second gap. Its heap was 96% and the GC log showed a 5.812-second full pause. Comparing effective configuration found that only B-7 had loaded a 3.4 GB catalog snapshot from a stale node-local feature flag. I preserved the GC, heap, config, and node evidence before draining it. I removed the injection and added replica config-hash checks and memory bounds. I verified every target below 200 ms p99, heap below 65%, GC p99 below 50 ms, zero outlier errors, and safe peer capacity.

---

# Incident 14: Routing Finds a Stale Draining Target

## Interview question

> Requests routed to one particular Service B instance are failing. What could cause this?

## The page arrives

At 22:13 UTC on 2026-09-13, routing finds a stale draining target.
The first failed request is `ord-57d108`, trace `0ef92f3577b34da6a3ce929d0e0e000e`, from `order-a-4.18.2-k2m5q` toward `10.42.7.26:8080` and target label `inventory-b-6`.
The measured scope is LB-routed B-6 only; current endpoints and B-9 direct calls succeed.
The first metric sentence is: B-6 100% resets on reused connections; draining 27m; error share equals weight.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `22:13 UTC | request=ord-57d108 | source=order-a-4.18.2-k2m5q | target=10.42.7.26:8080 | scope=LB-routed B-6 only; current endpoints and B-9 direct calls succeed`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: LB-routed B-6 only; current endpoints and B-9 direct calls succeed.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `load-balancer lifecycle and stale keep-alive`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `0ef92f3577b34da6a3ce929d0e0e000e`, sanitized logs for `ord-57d108`, target `inventory-b-6`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Join every result to routing and connection state

At 22:13 UTC 14% of requests fail, matching B-6's configured load-balancer weight. I join success and failure datasets on backend, connection ID, connection age, reuse flag, source A, gateway, zone, and rollout state. Failures follow B-6 and reused connections; new connections to replacement B-9 succeed in 88 ms.

DNS no longer advertises B-6, and current EndpointSlice contains B-9 instead. That difference rules out a current multi-answer DNS problem, but it reveals disagreement between service discovery and the load balancer's target inventory.

TCP connects to active targets remain 8-11 ms. Requests on old B-6 keep-alives receive resets, not connect timeouts. A reset means an existing connection was forcibly closed; a timeout would suggest dropped new handshakes. The reuse split points directly to stale pooled state during draining.

### Follow target state through the rollout

The load balancer reports B-6 in `draining` for 27 minutes, even though normal drain time is 60 seconds. Its routing weight remains 0.14. Controller logs show its endpoint watch disconnected at 21:45:58, four seconds before B-6 termination began. Deregistration never completed.

Gateway upstream reset count for B-6 is 100% with reason `connection_termination`; B-9 has zero resets. Some first attempts retry to B-9 and succeed, raising attempts/order to 1.14 and making client-level failures depend on remaining deadline. The circuit breaker is per gateway worker, so stale connection distribution makes its state uneven.

B-6 has no current server metrics because the pod has terminated. B-9 and peers show normal CPU, GC, queues, HTTP/DB pools, and dependency latency. Adding replicas cannot delete stale routing state. Before removing B-6 from the load balancer, I preserve controller watch logs, target state, connection ages, pod termination timing, and the failed/success trace pair.

If failures followed one live B instance on both new and reused connections, I would inspect its config/runtime as in Incident 13. If all old connections reset across every target, I would inspect a broad rollout or keep-alive incompatibility. If DNS still returned B-6 directly, I would repair DNS/discovery as in Incident 5. Here only the load balancer inventory is stale.

Removing B-6 through the approved control plane stops resets immediately. The permanent solution makes watch recovery and deregistration idempotent, and aligns readiness false, preStop, endpoint withdrawal, load-balancer drain, and application termination.

## Trace: follow parent to the failing child

I search trace `0ef92f3577b34da6a3ce929d0e0e000e` and start at the A server parent.
The failed waterfall reads `LB upstream target=B-6 reused=true RESET -> retry B-9 88ms OK`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-57d108`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-6`, and `server.address=10.42.7.26:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `target_state=draining upstream_reset=connection_termination`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T22:13:22.417Z level=ERROR request_id=ord-57d108 trace_id=0ef92f3577b34da6a3ce929d0e0e000e
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-6 target=10.42.7.26:8080
message="target_state=draining upstream_reset=connection_termination"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to join outcome to target/reuse then compare LB inventory with EndpointSlice.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-57d108 source=order-a-4.18.2-k2m5q target=10.42.7.26:8080
RESULT failed B-6 reused reset; success B-9 new 88ms; B-6 absent endpoint
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `B-6 100% resets on reused connections; draining 27m; error share equals weight` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 22:13 UTC the triggering production state changed.
2. controller lost watch and failed to deregister terminating B-6.
3. Mechanically, routing state outlived compute; pooled connections selected a closing target.
4. Therefore `B-6 100% resets on reused connections; draining 27m; error share equals weight`.
5. Trace `0ef92f3577b34da6a3ce929d0e0e000e` showed `LB upstream target=B-6 reused=true RESET -> retry B-9 88ms OK`.
6. It selected log evidence `target_state=draining upstream_reset=connection_termination`.
7. The safe failed/control comparison showed `failed B-6 reused reset; success B-9 new 88ms; B-6 absent endpoint`.
8. That explains the customer scope: LB-routed B-6 only; current endpoints and B-9 direct calls succeed.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I remove stale target via approved control after preserving evidence.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** controller lost watch and failed to deregister terminating B-6.
It lives at `load-balancer lifecycle and stale keep-alive` and explains why routing state outlived compute; pooled connections selected a closing target.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to repair idempotent watches and align preStop/readiness/drain durations.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: B-6 absent LB, resets 0, controlled drain test passes.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `B-6 100% resets on reused connections; draining 27m; error share equals weight` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would correlate each failure with the selected backend and connection state. At 22:13 UTC the 14% error rate matched B-6's load-balancer weight, and every failure used a reused connection to that target. B-6 was absent from DNS and EndpointSlice but remained in the load balancer as `draining` for 27 minutes. Its connections reset, while new connections to B-9 succeeded in 88 ms. Controller logs showed that an endpoint watch had disconnected before B-6 terminated, so deregistration never completed. I preserved routing and termination evidence, then removed the stale target through the approved control plane. I fixed idempotent watch recovery and aligned readiness, preStop, deregistration, and drain timing. I verified no B-6 target or connection remained, resets were zero, retries normalized, and a controlled rollout drain passed.

---

# Incident 15: Green Health Hides Pool Exhaustion

## Interview question

> Service B is healthy according to its health endpoint, but actual API requests are failing. Why?

## The page arrives

At 23:04 UTC on 2026-09-13, green health hides pool exhaustion.
The first failed request is `ord-68ef55`, trace `0ff92f3577b34da6a3ce929d0e0e000f`, from `order-a-4.18.2-k2m5q` toward `10.42.7.18:8080` and target label `inventory-b-r8x2p`.
The measured scope is /live and /ready 200 in 3ms; business reservations fail fleet-wide.
The first metric sentence is: health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%.
I do not paraphrase this as 'B is down'; that would skip the failing layer.
I preserve error text, timestamps, address, generator, attempt, reuse, timeout, and remaining deadline.

## Why green health is not green business

I put five separate series on one panel rather than using one green status tile.

| Signal | What it asks | Observed value | Interpretation and next branch |
|---|---|---|---|
| Liveness `/live` | Is the B process event loop alive enough to answer? | 100% success, p99 3 ms | B should not be restarted by the platform; this says nothing about DB access |
| Readiness `/ready` | Should this instance receive traffic under the configured probe contract? | 100% success, p99 4 ms | The implemented probe is too shallow because it checks only local startup state |
| Startup probe | Has initialization completed within its startup budget? | Last ran at 22:42, succeeded after 11 s | It prevents premature liveness action during boot; it is not a continuous dependency test |
| Synthetic reservation | Can a controlled user execute the real reserve workflow? | Success falls from 99.9% to 18%, p99 5.1 s | The business path is broken and selects its handler, pool, and dependencies |
| Real business outcome | Did customer inventory reservations commit exactly once? | 22% success; attempts remain normal | Confirms customer impact and requires reconciliation after recovery |

The lightweight liveness and readiness handlers allocate no DB connection, perform no stock query, and call neither Redis nor Pricing C.
They remain fast because the event loop and local process state are healthy.
The business handler must acquire a DB connection, so its trace waits 4.9 seconds before any query span exists.
If readiness also used the DB pool and failed, I would still keep liveness independent to avoid a restart storm.
If the synthetic succeeded while real traffic failed, I would compare identity, payload, tenant, warehouse, rate limit, and data shape rather than clearing the incident.
If startup failed only on new instances, I would inspect initialization, migration, secret, and startup-budget evidence by version instead of the steady-state pool leak.

## What I do in the first five minutes

### Minute 0-1: make the symptom exact

I write `23:04 UTC | request=ord-68ef55 | source=order-a-4.18.2-k2m5q | target=10.42.7.18:8080 | scope=/live and /ready 200 in 3ms; business reservations fail fleet-wide`.
I acknowledge customer impact, freeze unrelated changes, assign owners, and use UTC everywhere.
I do not restart because it can erase sockets, heap, queues, connection age, config drift, and logs.

### Minute 1-2: define the blast radius

I split the dataset until I can state: /live and /ready 200 in 3ms; business reservations fail fleet-wide.
I split by route, status/exception, A source, B backend, instance, version, zone, attempt, new/reused connection, and safe business dimension.
I pair the failure with a success from the same minute, route, payload class, source version, and identity.
For intermittent behavior, the denominator and failure probability matter as much as a single stack trace.

### Minute 2-3: open the exact dashboard

I open the business outcome and service-path dashboard, not a host CPU dashboard alone.
The shared cursor starts ten minutes before the first failure and includes deploy/config/cert/network/job annotations.
Panels are A server RED, A client phases, DNS/TCP/TLS, gateway downstream/upstream, target health/connections, B server RED, B USE, pools, dependencies, and DB.
The working fault domain is `B business DB pool`, but it remains a hypothesis.

### Minute 3-4: count across boundaries

I compare A inbound user requests, A outbound attempts, gateway receives/upstream attempts, B accepts, DB/cache/C calls, and completed reservations.
A count mismatch identifies where work stopped; equal rates let me move deeper.
I align status/exception and latency phases rather than comparing unrelated totals.

### Minute 4-5: preserve evidence

I save trace `0ff92f3577b34da6a3ce929d0e0e000f`, sanitized logs for `ord-68ef55`, target `inventory-b-r8x2p`, effective config hashes, and a matched success.
I preserve per-instance/node/zone/version state before draining anything.
Only then do I select a reversible mitigation supported by the evidence.

## Metric-by-metric causal walk

### Put health and business outcomes on the same timeline

At 23:04 UTC `/live` and `/ready` remain 100% successful at 3-4 ms, but real reservation success falls to 22% and the controlled business synthetic falls to 18%. Request rate is normal. The disagreement is the starting evidence, not a paradox: each signal executes different work.

The startup probe last succeeded in 11 seconds at deployment time and is no longer running. Liveness checks only the event loop. Readiness checks local initialization. Neither acquires a DB connection, reads stock, calls Redis, or reaches Pricing C. Their low latency is therefore mechanically compatible with a broken reservation handler.

If liveness failed, I would investigate process scheduling, deadlock, or runtime failure. If readiness alone failed on new pods, I would inspect initialization and probe contract. If the synthetic succeeded while customer requests failed, I would compare identity, tenant, payload, data, and rate limits. Here both synthetic and real business paths fail.

### Follow the real request through healthy transport

A DNS, TCP, TLS, and gateway connect p99 values remain 3, 8, 14, and 7 ms. Healthy-target count stays eight, gateway status is 504 after waiting, and B receives every business request. This removes the health endpoint and network path from the critical branch.

B business p99 reaches 5.1 seconds, while health p99 remains 4 ms. CPU is only 29%, throttling zero, heap 56%, and GC max 24 ms. Worker in-flight rises and queue depth grows because handlers wait; low CPU is expected for blocked acquisition rather than compute saturation.

### The pool metrics reveal where work stops

Across B instances the DB pool progresses from active 31/40, idle 9, pending 0 at 22:55 to active 40/40, idle 0, pending 186 at 23:04. Acquisition p99 reaches 4.9 seconds and then times out. The failed trace has `db.pool.acquire=4902 ms` and no DB query child, proving the request never obtained a connection.

Database query rate falls even though B request rate is steady. DB CPU and lock waits are normal because leaked connections are checked out but not executing useful queries. Cache and Pricing C remain normal. Enlarging the pool could consume more DB sessions and only delay exhaustion.

Checkout/return counters diverge after B 7.7 deployment at 22:42:17, especially on validation-error requests. A matched success in 7.6 returns its connection in a `finally` block; 7.7 returns only on the success branch. The leak accumulates across every instance, explaining why impact worsens over time rather than mapping to one target.

Retries amplify pending work and are reduced within policy. The breaker opens only after acquisition failures, so it protects B late. I roll back 7.7, then drain gradually; a restart is used only after evidence capture to reclaim leaked resources, not presented as the root cause.

The code correction uses scoped resource management on every exit path. A failure-path and soak test assert checkout equals return, pending remains zero, and real business synthetics pass. Liveness stays shallow to avoid restart storms, while readiness and separate deep synthetic signals have explicit, different purposes.

## Trace: follow parent to the failing child

I search trace `0ff92f3577b34da6a3ce929d0e0e000f` and start at the A server parent.
The failed waterfall reads `health B 3ms no child; business B 5.1s -> pool.acquire 4.9s ERROR; no query`.
I expand A client, gateway server/upstream, B server, queue, and dependency children in order.
I inspect span status, exception events, route, source, resolved address, backend, instance, version, zone, attempt, reuse, and deadline.
I expect `request.id=ord-68ef55`, `source=order-a-4.18.2-k2m5q`, `backend=inventory-b-r8x2p`, and `server.address=10.42.7.18:8080` where instrumentation supports them.
If A client is slow but B is normal, I inspect pre-B phases and response transfer.
If B is slow and DB child is 4.7s of 5.1s, that DB path owns observed latency and selects its fingerprint.
If no B span exists, I investigate before B, but first confirm sampling and B access logs.
If gateway has no upstream child and `no_healthy_upstream`, I inspect target health, not B code.
If gateway ends at its deadline while B continues, I inspect cancellation and B's longest child.
A blank gap can be GC, queueing, uninstrumented code, dropped spans, or clock skew; metrics/logs must account for it.
Tracing is evidence, not guaranteed completeness.

## Trace-selected logs

The trace selects the narrow log evidence `pool active=40 idle=0 pending=186 acquire_timeout=5000`.
I query a small UTC window by request/trace ID and compare the matched successful request.
I retain source, target, version, zone, attempt, elapsed phase, and config hash, while redacting secrets and payloads.

```text
2026-09-13T23:04:22.417Z level=ERROR request_id=ord-68ef55 trace_id=0ff92f3577b34da6a3ce929d0e0e000f
source_instance=order-a-4.18.2-k2m5q backend=inventory-b-r8x2p target=10.42.7.18:8080
message="pool active=40 idle=0 pending=186 acquire_timeout=5000"
```

The log proves that this component emitted the record; its wording alone is not causal proof.
I verify nested exception, elapsed phase, target, and whether adjacent access logs contain the same request.

## Safe config, runtime, network, and direct-path test

The exact next test is to compare health and failed business traces, pool checkout/return logs.
I run one or a few bounded probes from A's real execution context and one matched control.

```text
FAIL request=ord-68ef55 source=order-a-4.18.2-k2m5q target=10.42.7.18:8080
RESULT ready 3ms children0; reserve 5104ms pool.acquire4902 query0
CONTROL same route, identity, payload class, and minute through known-good path succeeds
```

A single probe locates a layer but does not measure availability; the time series establishes scope.
If the command bypasses A's runtime DNS cache, sidecar, gateway, identity, or pool, I state that limitation.
I inspect effective runtime config, not only Git or deployment intent.

### Result branches

| Test result | Interpretation | Next evidence |
|---|---|---|
| Expected DNS answer quickly | DNS worked for this sample, not TCP | retain address/TTL and test exact IP/port |
| NXDOMAIN | name absent in this resolver view | compare FQDN, search suffix, zone/view, authority |
| SERVFAIL/timeout | resolver chain failed | compare control zones, forwarder route/health |
| Fast refusal | selected destination actively rejected | listener, bind, port map, target lifecycle |
| Connect timeout | handshake never completed | policy, firewall, route, flow, retransmission |
| TCP works; TLS fails | secure identity/policy boundary | SNI, SAN, CA, client cert, protocol |
| Gateway identity on response | intermediary generated/forwarded it | product subreason, target, upstream metrics |
| Direct B succeeds; gateway fails | intermediary path differs | SNI, protocol, route, target, identity |
| B span slow | B or child owns delay | queue, runtime, pool, dependencies |
| Health works; business fails | probe bypasses failing work | auth, handler, pool, dependency, data |

## Alternate branches kept alive

The symptom can have several causes; I reject each only with boundary evidence.

| Branch | Mechanism/shape | Deciding next test |
|---|---|---|
| Config | wrong scheme/host/port/path/proxy/SNI/deadline or drift; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | compare effective failed/good runtime config; reject if the failure/success pair conflicts |
| DNS | NXDOMAIN/SERVFAIL/stale multi-answer/cache/split horizon; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | query through A resolver and correlate answer to outcome; reject if the failure/success pair conflicts |
| Network/TCP | route/policy/firewall/refusal/loss/reset/NAT pressure; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | compare connect phases, listener, flow and counters; reject if the failure/success pair conflicts |
| TLS | expiry/SAN/CA/SNI/mTLS/protocol mismatch; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | inspect exact handshake and effective trust; reject if the failure/success pair conflicts |
| Gateway/LB/mesh | wrong cluster/port/protocol, health, stale target, queue/breaker; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | identify generator/subreason and backend; reject if the failure/success pair conflicts |
| A client | pool pending/stale keep-alive/proxy/retry/deadline; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | inspect phase, reuse, attempt, remaining budget; reject if the failure/success pair conflicts |
| B runtime | worker/CPU/GC/memory/deadlock/instance drift; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | split USE by instance and compare peer; reject if the failure/success pair conflicts |
| Dependency | DB lock/scan/pool, cache miss, Service C/retry; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | select exact child/fingerprint; reject if the failure/success pair conflicts |
| Contract/security | 401/403/404/429/500 from identity/route/quota/payload/code; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | confirm responder and sanitized decision log; reject if the failure/success pair conflicts |
| Telemetry | sampling/missing label/clock/drop/counter reset; compare with `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` | corroborate adjacent metrics/access logs; reject if the failure/success pair conflicts |

The winning explanation must account for start time, exact scope, probability, instance shape, waterfall, log, test, and recovery.
One abnormal chart without that chain is correlation, not root cause.

## Fully worked causal chain

1. At 23:04 UTC the triggering production state changed.
2. B 7.7 returned DB connections only on success; validation failures leaked them.
3. Mechanically, light health bypassed DB while business queued for leaked connections.
4. Therefore `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%`.
5. Trace `0ff92f3577b34da6a3ce929d0e0e000f` showed `health B 3ms no child; business B 5.1s -> pool.acquire 4.9s ERROR; no query`.
6. It selected log evidence `pool active=40 idle=0 pending=186 acquire_timeout=5000`.
7. The safe failed/control comparison showed `ready 3ms children0; reserve 5104ms pool.acquire4902 query0`.
8. That explains the customer scope: /live and /ready 200 in 3ms; business reservations fail fleet-wide.
9. I changed only the proven mechanism and watched the predicted metrics reverse.
10. Business outcomes recovered with the technical layer, completing causal verification.

## Mitigation is not root cause

**Mitigation:** I roll back 7.7 and drain gradually; restart only after evidence to recover state.
I first preserve evidence and confirm remaining targets/dependencies have capacity for shifted traffic.
I do not blindly restart, increase timeout, enable more retries, enlarge pools/threads, or add replicas.
Those actions can hide evidence, retain work longer, amplify load, or overwhelm the downstream.

**Root cause:** B 7.7 returned DB connections only on success; validation failures leaked them.
It lives at `B business DB pool` and explains why light health bypassed DB while business queued for leaked connections.
A workaround that clears current state does not correct this mechanism.

## Permanent correction

The primary fix is to use scoped/finally release and test error paths plus business synthetic.
I apply only controls supported by the incident evidence:

- Config: schema validation, one source, effective hash, drift detection, and safe reload.
- Code: deadline propagation, cancellation, resource release, bounded retry, and useful spans.
- Platform: readiness/startup/drain contracts, stable selectors, and per-instance telemetry.
- Network: least-privilege contract, caller-context probes, and DNS/certificate lifecycle.
- Gateway: tested cluster, port, protocol, SNI, health policy, subreason, and target reconciliation.
- Service/dependency: query/capacity guardrails, pool ownership, and real business synthetic coverage.

## Verification of recovery

The incident-specific target is: pending 0, idle available, business >99.9%, no leak under soak.
I verify the same caller, route, identity class, and formerly failing target/path.
I confirm business reservation success and reconciliation, not only HTTP 200 or /health.
I confirm A error reason and every phase histogram return to baseline.
I confirm gateway target/subreason and B/dependency rate/latency/queue/pool are stable.
I confirm p50/p95/p99/max by instance, zone, and version for the observation window.
I confirm attempts/request returns to normal and no hidden retry amplification remains.
I check for duplicate, orphaned, or late reservations from retries/cancelled work.
I execute a controlled canary/deployment test at the repaired boundary.
Only matching technical and business recovery closes the incident.

## Recurrence prevention

I alert on business SLO plus `health 100%; business 22%; DB pool 40/40, idle0, pending186, acquire p99 4.9s; CPU29%` with useful route/source/target/version labels.
I add adjacent-rate mismatch alerts such as A attempts without B accepts.
I separate DNS, refusal, connect timeout, TLS, 502 subreason, 503 health reason, 504 timeout, read timeout, and application errors.
I alert on per-instance p99/error outliers, queues/rejections, pool pending/acquisition, retries, and breaker rejection.
The runbook links dashboards, safe caller-context tests, evidence preservation, owner, rollback/drain control, and numeric verification.
Deployment tests cover DNS, TCP, TLS identity, gateway route, readiness, business dependency, deadlines, retries, and draining.

## Interview-ready answer

> I would compare liveness, readiness, startup, synthetic, and real business metrics instead of trusting one green tile. At 23:04 UTC liveness and readiness were 100% successful in 3-4 ms, but reservations succeeded only 22%. Transport and B runtime were healthy; the failed business trace spent 4.9 seconds acquiring a DB connection and never produced a query span. Every pool was active 40 of 40, idle zero, with 186 pending requests. Checkout and return counters diverged after B 7.7 because validation failures skipped connection release. I rolled back 7.7 and drained safely after preserving evidence; restart only reclaimed leaked state. I fixed resource handling with scoped cleanup and failure-path tests. I verified idle capacity, zero pending work, balanced checkout/return, business success above 99.9%, and a clean soak test.

---

# Reusable spoken interview story template

> At `[UTC]`, `[alert]` showed `[exact error/status/duration]` on `[route]`.
> In five minutes I captured `[request/trace/source/target/version/zone/attempt/deadline]`, froze unrelated changes, and scoped by `[dimensions]`.
> Business plus layered RED/USE showed `[metric A]` changed while adjacent `[metric B]` stayed normal/flat.
> That located the last good boundary and triggered `[exact safe test]`.
> If `[alternate shape]` had appeared, I would have moved to `[alternate layer]` instead.
> The failed trace showed `[waterfall]`; a matched success showed `[contrast]`.
> The longest/failed child selected `[log/config/code/query/network evidence]`.
> Root cause was `[mechanism at named layer]`, triggered by `[change]`.
> Mitigation was `[reversible action]`; it was not the permanent fix.
> I fixed `[control]`, verified the same path, business result, p95/p99, every instance, retries, and downstream capacity for `[window]`, then added `[alert/runbook/test]`.

# Final combined case study: metrics to trace to code to durable recovery

At 08:31 UTC on 2026-09-14, reservation success falls to 91.4% while order rate is a normal 240/s.
A p50 is 92ms, p95 180ms, p99 5.02s, while average is only 141ms and hides the tail.
Read timeouts are 20.6/s pinned at the five-second deadline.
Both A versions and all eight B instances fail, but 96% of failures are warehouse 17.
DNS p99 is 3ms with zero errors; TCP p99 9ms with no refusal/retransmit; TLS p99 14ms with zero failure.
Gateway connect p99 is 8ms but upstream response p99 is 5.11s; eight targets stay healthy.
A attempts and B accepts match, proving calls reach B quickly.
B CPU is 44%, throttle zero, heap 61%, GC p99 21ms, workers 96/200, queue 4, rejected zero.
DB pool acquisition is 11ms, but query fingerprint `9ac2` has p99 4.82s and lock wait 4.68s for warehouse 17.

```text
Order A server                                  5,008ms ERROR
  Inventory client                              5,001ms read_timeout
    Gateway                                     4,997ms downstream_cancel
      Inventory B                               5,143ms cancelled
        auth                                         7ms
        DB pool acquire                             11ms
        UPDATE stock                              4,742ms
          db.lock.wait                            4,681ms
```

A successful warehouse-12 trace has the same connection phases but a 19ms update and no lock child.
The failed trace identifies request `ord-70c912`, fingerprint `9ac2`, DB host `pg-inventory-2`, and blocker application.
A narrow B log shows `lock_wait_ms=4681 blocked_by_app=inventory-reconcile job_id=job-901`.
An approved read-only DB activity view shows job-901 opened a transaction at 08:29:54.
The batch diff shows version 2.3 changed from commits every 500 rows to one transaction per warehouse.
This mechanism explains the warehouse scope, flat CPU, healthy transport, and exact timeout shape.

The owner pauses job-901 through its scheduler and lets the transaction finish: mitigation.
We do not increase timeouts, pools, retries, or replicas because they add blocked work against the same rows.
The permanent fix restores small commits, consistent row order, a bounded lock timeout, and cancellation propagation.
A concurrency test runs reservations and reconciliation on the same warehouse.
At 08:40 lock p99 is 14ms, B p99 151ms, gateway p99 162ms, and timeouts zero.
At 08:45 business success is 99.97%, attempts/order is 1.00, and every B p99 is below 190ms.
Reconciliation finds no duplicate or missing reservations; batch 2.3.1 canary has max lock wait 46ms.
After thirty stable minutes we close with an alert linking business failure, lock fingerprint, and blocker.

> My causal chain was metrics to boundary, trace to child, child to log/query/config, mechanism to targeted fix, then the same technical and business measurements to verification. I did not call one correlated chart a root cause.
