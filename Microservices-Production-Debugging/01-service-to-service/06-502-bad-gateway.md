# Problem

HTTP 502 Bad Gateway means a reverse proxy received an unusable result while
acting as an HTTP gateway to an upstream. The status does not identify the
failed layer: refusal, reset, premature close, malformed response, or protocol
mismatch are possible. This playbook uses NGINX Ingress, whose access and error
logs provide the subreason; Envoy response flags do not apply.

# Production Situation

At `2026-09-13T18:03:22Z`, Order Service A calls Inventory Service B through
NGINX Ingress route `POST /v1/inventory/reservations`.

* requestId `ord-249bc0`
* traceId `0af92f3577b34da6a3ce929d0e0e000a`
* A instance `order-a-4.18.2-k2m5q`
* NGINX instance `ingress-nginx-controller-6d7b9`
* B target `10.42.7.19:8443`
* B instance/version `inventory-b-2`/`7.5`
* zone `eu-west-1b`
* normal rate 220 requests/s
* normal success 99.95%
* normal gateway p99 182 ms
* abnormal NGINX 502 rate 48/s
* abnormal response duration 22-28 ms
* NGINX upstream connect time 8 ms
* NGINX upstream response time is empty because no HTTP response header arrived
* B application HTTP request count is zero for failed request IDs
* direct `https://inventory.shop.svc:8443` succeeds in 37 ms

At `18:00:41Z`, an ingress change removed the HTTPS upstream annotation.
NGINX used plaintext HTTP on B's TLS-only port 8443. B accepted TCP, received
bytes that were not a TLS ClientHello, and closed before HTTP existed.

# Architecture

```text
Order A
   |
   | HTTPS, valid request
   v
NGINX Ingress
   |
   | proxy_pass http://10.42.7.19:8443
   |              wrong upstream protocol
   v
Inventory B TLS listener :8443
   |
   X closes connection before HTTP response header

NGINX -> A: HTTP 502 Bad Gateway
```

The direct control uses HTTPS to B; the gateway uses HTTP to the same TLS port,
so the paths are not equivalent.

# What I Check FIRST

1. **Generator.** WHAT: response headers, request ID, ingress instance, and
   access log. WHY: Service B or another proxy could also emit 502. LOOK FOR:
   `server: nginx` and the same request ID in NGINX access logs.
2. **NGINX error reason.** WHAT: bounded error-log entry for the upstream and
   timestamp. WHY: 502 is only the HTTP category. LOOK FOR:
   `upstream prematurely closed connection while reading response header`.
3. **Upstream phase.** WHAT: connect time, header time, response time, target,
   and protocol. WHY: fast TCP plus no header differs from refusal or timeout.
   LOOK FOR: connect 8 ms, no upstream status/header.
4. **Direct-versus-gateway contract.** WHAT: scheme, port, SNI, Host, trust,
   route, and identity. WHY: direct HTTPS success can coexist with proxy HTTP
   failure. LOOK FOR: `https` direct versus `http` upstream.
5. **B boundary.** WHAT: TLS terminator logs, application access logs, server
   spans, and per-target rate. WHY: protocol rejection happens before B HTTP.
   LOOK FOR: socket/TLS parse failure and no application request.

# Step-by-Step Investigation

### Step 1 - Preserve the exact 502 response

* **What I check:** status, `Server` header, request ID, ingress instance,
  route, UTC time, and total duration.
* **Why:** those fields identify the observation point before logs rotate.
* **Expected result:** NGINX forwards B's 201 in under 200 ms.
* **Bad result:** A receives 502 in 24 ms with `Server: nginx`.
* **Meaning:** NGINX generated or served the 502 for this request.
* **Next branch:** query NGINX access and error logs by request ID and time.

### Step 2 - Read the product-specific NGINX error

* **What I check:** error-log message, upstream address, request route, and
  connection/request identifiers.
* **Why:** NGINX explains whether connect, header read, or response parsing
  failed.
* **Expected result:** no error entry and `upstream_status=201`.
* **Bad result:** `upstream prematurely closed connection while reading
  response header from upstream`.
* **Meaning:** TCP connected, but NGINX did not receive a complete HTTP
  response header before the upstream closed.
* **Next branch:** inspect upstream protocol, TLS, target lifecycle, and B logs.

### Step 3 - Separate connection from response-header failure

* **What I check:** `$upstream_connect_time`, `$upstream_header_time`,
  `$upstream_response_time`, `$upstream_status`, and target.
* **Why:** these fields distinguish refusal from a post-connect close.
* **Expected result:** connect 0.008 s, header 0.120 s, status 201.
* **Bad result:** connect 0.008 s, header `-`, response `-`, status `502`.
* **Meaning:** the socket opened, but no valid upstream HTTP header arrived.
* **Next branch:** compare configured upstream scheme and port.

### Step 4 - Inspect the rendered NGINX upstream configuration

* **What I check:** generated location/upstream configuration, Service port,
  backend protocol annotation, and reload revision.
* **Why:** source YAML may differ from the configuration NGINX loaded.
* **Expected result:** `proxy_pass https://inventory-backend`.
* **Bad result:** rendered upstream uses plaintext HTTP for port 8443.
* **Meaning:** NGINX sends an HTTP request to a TLS-only listener.
* **Next branch:** reproduce one bounded protocol comparison from an approved
  ingress-equivalent context.

### Step 5 - Compare HTTP and HTTPS to the exact target

* **What I check:** same IP/port, correct Host/SNI, one HTTP probe, and one
  HTTPS probe.
* **Why:** keeping target constant isolates the scheme.
* **Expected result:** HTTPS completes TLS and receives a valid status.
* **Bad result:** HTTP connection closes with an empty reply while HTTPS
  returns the health response in 37 ms.
* **Meaning:** B is TLS-only and the gateway uses the wrong protocol.
* **Next branch:** verify B's TLS and application logs match that mechanism.

### Step 6 - Confirm where B stops

* **What I check:** B TLS listener counters/logs, application access logs,
  HTTP server metrics, and trace search.
* **Why:** plaintext bytes reach the socket but not the HTTP handler.
* **Expected result:** an HTTPS request creates a B server span and access log.
* **Bad result:** failed gateway request has no B application span/access log;
  TLS listener records an invalid first record or early close.
* **Meaning:** failure occurs between TCP acceptance and B HTTP handling.
* **Next branch:** preserve a matched direct success and gateway failure.

### Step 7 - Exclude nearby 502 mechanisms

* **What I check:** refusal, timeout, reset timing, malformed response, response
  header size, target lifecycle, and NGINX buffer errors.
* **Why:** several NGINX failures can generate 502.
* **Expected result:** only protocol mismatch evidence fits all observations.
* **Bad result:** `connect() failed (111: Connection refused)` would select a
  listener/port branch; `upstream sent invalid header` would select response
  protocol/framing.
* **Meaning:** error-log wording changes the next owner.
* **Next branch:** follow the exact NGINX reason rather than generic 502 advice.

### Step 8 - Check retries and peer capacity

* **What I check:** original requests, upstream attempts, NGINX retry policy,
  selected targets, B peer load, and remaining deadline.
* **Why:** automatic `proxy_next_upstream` can hide a bad target or multiply
  writes.
* **Expected result:** one attempt per idempotent reservation.
* **Bad result:** attempts/request rises to 1.21 and healthy peers gain load.
* **Meaning:** retry amplification is a secondary risk.
* **Next branch:** limit unsafe retries while correcting the protocol.

### NGINX 502 result branches

| NGINX evidence | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| `connect() failed (111: Connection refused)` | Upstream TCP actively rejected the port | B JVM is stopped | Check listener, bind, target port, sidecar, and lifecycle |
| `upstream timed out while connecting` | NGINX could not complete TCP in budget | Which device dropped traffic | Check retransmits, flow telemetry, target SYN, and reverse route |
| `upstream prematurely closed connection while reading response header` | Upstream closed after connect before complete header | Protocol mismatch is certain | Compare scheme/port, upstream resets, TLS listener, and lifecycle |
| `upstream sent no valid HTTP/1.0 header` | Bytes were not a valid expected HTTP response | B business logic failed | Check HTTP version/protocol configuration and raw safe response metadata |
| `upstream sent too big header` | Header exceeds configured proxy buffer | More buffer is always the right fix | Inspect unexpected cookie/header growth and application contract |
| `recv() failed (104: Connection reset by peer)` | Established upstream socket was reset | NGINX caused the reset | Correlate B/sidecar lifecycle, protocol, and connection reuse |
| 502 only for one target | Backend state/version predicts failure | NGINX fleet is healthy | Compare target listener, protocol, config, and instance lifecycle |
| 502 only for one ingress instance | Ingress-local config/state predicts failure | B is healthy from all clients | Compare rendered config, reload, node, and controller version |
| Direct HTTPS succeeds | B can serve that direct TLS request | Gateway uses HTTPS or same identity | Compare gateway scheme, SNI, trust, Host, and target |
| B access log contains request | B HTTP layer received it | B generated the 502 | Correlate B response/close with NGINX upstream log |

### Protocol mismatch mechanics

TCP is byte-oriented and does not know whether payload is HTTP or TLS. NGINX
connects to `10.42.7.19:8443`, then its `http` upstream writes a plaintext HTTP
method. B expects a TLS record and ClientHello, cannot parse those bytes, and
closes. NGINX receives EOF instead of an HTTP response header and generates
502. No B Spring MVC/WebFlux handler, authentication filter, or DB call runs.

### Direct-versus-gateway comparison

The direct command uses `https://inventory.shop.svc:8443`, performing TCP, TLS
with correct SNI, then HTTP. The rendered gateway uses
`http://inventory-backend`, performing TCP then plaintext HTTP. Direct success
therefore does not contradict gateway failure.

If both paths used HTTPS but only NGINX failed, I would compare SNI, trust
bundle, client certificate, Host header, and protocol version.

If both paths used HTTP and direct worked, I would compare selected target,
route rewrite, connection reuse, and NGINX-specific policy.

### Gateway status mapping caveat

Status mapping is product and configuration specific.

This file's 502 is generated by NGINX and explained by its error log.

For Envoy, upstream connection failure flag `UF` is normally returned as HTTP
503, not 502.

Envoy `UH` is also HTTP 503 and means no healthy upstream.

Envoy `UT` is HTTP 504 and means upstream request timeout.

Custom filters or external gateways can alter mappings, so I always retain
generator, status, product version, and subreason together.

# Metrics to Check

| Metric | High / low / flat interpretation |
|---|---|
| NGINX 502 rate | High confirms gateway-visible failures; split ingress/route/upstream/target |
| Downstream request rate | Flat 220/s rejects a demand spike; high could reveal overload |
| Upstream connect time | Low 8 ms proves TCP for samples; high/refused selects network/listener |
| Upstream header time | Missing means no complete header; high means slow header response |
| Upstream response time | Missing with premature close differs from a long upstream response |
| Upstream status | 502 generated locally is not B's application status |
| B HTTP request rate | Zero for failed IDs supports pre-HTTP stop; check sampling/access logs |
| B TLS invalid-record count | High after ingress change supports plaintext-to-TLS mismatch |
| Per-target 502 | One target suggests instance compatibility; all targets suggest shared config |
| Per-ingress 502 | One controller suggests config/reload drift; all suggest shared route config |
| Connection reuse | New and reused patterns distinguish protocol config from stale sockets |
| Retry attempts | High adds load and can duplicate non-idempotent work |
| CPU/heap/GC | Low/flat on B is expected when handlers do not run |
| p50/p95/p99/max | Fast 502 can lower duration while success collapses; pair with counts |
| After ingress change | Immediate 502/TLS-invalid spike directs diff; rendered config proves cause |

If connect time becomes high, I leave the protocol branch and inspect TCP.

If B HTTP rate rises but 502 continues, I inspect response parsing and closes.

If only one B target reports TLS invalid records, I compare that target's port
and listener configuration.

# Distributed Trace Investigation

```text
traceId=0af92f3577b34da6a3ce929d0e0e000a
Order A server                              31ms span=f001
  NGINX client                             25ms span=f002 status=502
    NGINX upstream                         20ms span=f003 ERROR
      tcp.connect target=10.42.7.19         8ms span=f004
      wait_for_response_header             10ms span=f005 EOF
      [no upstream TLS span: NGINX configured HTTP]
      [no Inventory B application span]
```

The absence of an upstream TLS span is itself consistent with rendered
plaintext upstream configuration.

It does not prove NGINX skipped TLS if instrumentation is incomplete.

I confirm the scheme in rendered configuration and compare B TLS listener
evidence.

A missing B application span can also result from sampling, broken context
propagation, exporter loss, or missing server instrumentation.

I check B access logs and HTTP request counters before claiming non-arrival.

The trace's parent/child waterfall narrows the boundary; it does not replace
the NGINX error log.

# Distributed Logs

```text
2026-09-13T18:03:22.417Z level=ERROR service=nginx-ingress
instance=ingress-nginx-controller-6d7b9 version=1.11.2 zone=eu-west-1b
traceId=0af92f3577b34da6a3ce929d0e0e000a spanId=f003 requestId=ord-249bc0
endpoint=POST_/v1/inventory/reservations upstream=10.42.7.19:8443
status=502 request_time=0.024 upstream_connect_time=0.008
upstream_header_time=- upstream_response_time=-
error="upstream prematurely closed connection while reading response header from upstream"
```

```text
2026-09-13T18:03:22.429Z level=WARN service=inventory-tls
instance=inventory-b-2 version=7.5 zone=eu-west-1b
remote=10.42.5.14:43122 local=10.42.7.19:8443
event=tls_invalid_record requestId=unavailable
```

The NGINX log proves its observed upstream close.

The TLS log proves B's listener rejected non-TLS bytes from that source.

Neither line alone proves the rendered proxy scheme caused the fleet incident.

The config diff, matched HTTPS success, metric timing, and reversal after
correction establish causality.

# Commands / Tools

```powershell
curl.exe -sS -D - --connect-timeout 3 --max-time 8 `
  https://gateway.internal/v1/inventory/health -o NUL
Test-NetConnection 10.42.7.19 -Port 8443
curl.exe -v --connect-timeout 3 --max-time 8 `
  https://inventory.shop.svc:8443/actuator/health
```

`Server: nginx` identifies the responder for that sample.

TCP success proves only one socket handshake.

Direct HTTPS success does not prove the NGINX upstream scheme.

```bash
curl -sS -v --connect-timeout 3 --max-time 8 \
  http://10.42.7.19:8443/actuator/health
curl -sS -v --connect-timeout 3 --max-time 8 \
  --resolve inventory.shop.svc:8443:10.42.7.19 \
  https://inventory.shop.svc:8443/actuator/health
openssl s_client -connect 10.42.7.19:8443 \
  -servername inventory.shop.svc -verify_return_error -brief </dev/null
```

The HTTP probe may report `Empty reply from server`; it is one bounded
protocol test, not availability measurement.

`openssl` uses its configured trust, which may differ from NGINX.

Never use disabled TLS verification as a permanent fix.

```bash
kubectl get ingress -n shop inventory-api -o yaml
kubectl describe ingress -n shop inventory-api
kubectl get service -n shop inventory-b -o yaml
kubectl get endpointslice -n shop \
  -l kubernetes.io/service-name=inventory-b -o wide
kubectl logs -n ingress-nginx ingress-nginx-controller-6d7b9 --since=10m
```

These commands inspect declared resources and bounded logs.

The controller's generated configuration or supported diagnostic endpoint is
the runtime truth; source annotations alone do not prove it reloaded.

Do not print secrets, issue broad traffic loops, or invoke a mutating
reservation route without an approved idempotent synthetic.

# Root Cause

Ingress revision `ing-884` removed
`nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`.

NGINX rendered the Inventory upstream as HTTP while the Service still targeted
B's TLS-only port 8443.

```text
annotation removed
-> NGINX renders plaintext HTTP upstream
-> TCP to B:8443 succeeds
-> B TLS listener rejects non-TLS bytes and closes
-> NGINX receives no HTTP response header
-> NGINX generates 502 Bad Gateway
-> reservation success falls
```

The chain explains the start time, all-target scope, fast failure, direct HTTPS
success, B application absence, NGINX error, and B TLS invalid-record count.

# Fix

**Immediate mitigation:** preserve ingress configuration and failed/success
evidence, then roll back `ing-884` or restore the HTTPS backend protocol.

I confirm healthy target capacity and avoid adding retries.

**Root cause correction:** render `proxy_pass https://inventory-backend` for
port 8443 with the intended SNI and trust policy.

**Permanent fix:** version the upstream scheme/port/TLS contract, render-test
Ingress changes, run a gateway-origin canary, and require a valid upstream
response before rollout proceeds.

Increasing timeout cannot make plaintext become TLS.

Increasing buffers cannot repair a connection that closes before a response
header.

# Verification

Before:

* NGINX 502 rate 48/s
* business success 78%
* upstream connect p99 9 ms
* upstream header time absent
* B application accepts 0/s for failed IDs
* B TLS invalid-record rate 48/s
* attempts/request 1.21

After:

* NGINX 502 rate 0/s
* business success 99.96%
* upstream connect p99 9 ms
* upstream header p99 132 ms
* gateway route p99 181 ms
* B receives its balanced request share
* B TLS invalid-record rate 0/s
* attempts/request 1.00

I hold these values for 30 minutes across every ingress instance, B target,
version, and zone.

A controlled reservation through NGINX commits exactly once.

A direct HTTPS control also succeeds.

Reconciliation finds no duplicate, late, or orphan reservation from retries.

The recovered header time and B acceptance prove the expected boundary changed,
while business success proves user recovery.

# Prevention

* Alert on NGINX 502 by route, ingress, upstream, target, and error category.
* Alert when upstream connect succeeds but header/response fields are absent.
* Dashboard B TLS invalid-record count beside NGINX 502 and deployment events.
* Render-test backend scheme, port, SNI, trust, Host, and path rewrite.
* Canary from NGINX's real network and identity context.
* Contract test direct HTTPS and gateway HTTPS without changing target.
* Runbook includes exact NGINX error strings and next branches.
* Preserve generator/version because other proxies map failures differently.
* Keep write retries bounded, deadline-aware, and idempotent.
* Validate generated NGINX configuration before and after reload.
* Block rollout if gateway synthetic fails while direct control succeeds.

# Interview Answer

### What I would say in an interview

I first identify the 502 generator and read its product-specific error. Here
`Server: nginx` and the ingress log showed that the upstream prematurely closed
while NGINX waited for a response header. Upstream TCP connected in 8 ms, but B
had no application request; its TLS listener reported invalid records. A direct
HTTPS call succeeded. The ingress revision had removed the HTTPS backend
annotation, so NGINX sent plaintext HTTP to TLS port 8443. I rolled back that
change, restored HTTPS, and verified zero 502s, normal upstream header time,
matching B traffic, one attempt per request, and business success above 99.9%.

### Common interviewer traps

Do not say every 502 means B is down.

Do not use Envoy `UF` as an HTTP 502 example; Envoy normally returns 503 for
that flag.

Do not treat direct HTTPS success as proof that NGINX uses HTTPS.

Do not increase timeout or buffers before reading the NGINX error reason.

### Quick memory flow

502 -> identify NGINX -> exact error log -> connect/header phases -> rendered
scheme/port -> equivalent direct comparison -> root cause -> rollback -> verify
gateway and business.

# Interview Follow-up Questions

1. **What does this 502 prove?** NGINX could not obtain a usable upstream HTTP
   response; its error log identifies the narrower mechanism.
2. **Why is this not connection refused?** Upstream connect completed in 8 ms;
   the peer closed while NGINX waited for a response header.
3. **Why is direct HTTPS successful?** It uses TLS, while the broken NGINX
   upstream used plaintext HTTP.
4. **Would NGINX always return 502 for every upstream problem?** No; mapping
   depends on failure, configuration, and product/version, so inspect logs.
5. **How does Envoy differ here?** Envoy `UF` normally maps to HTTP 503,
   `UH` to 503, and `UT` to 504.
6. **Why is there no B application span?** The TLS listener closed before HTTP;
   sampling and access logs must still corroborate absence.
7. **Why not increase proxy timeout?** More time cannot correct an HTTP-to-TLS
   protocol mismatch.
8. **What permanently prevents recurrence?** Rendered-config contract tests
   plus a gateway-origin HTTPS canary and numeric business verification.
