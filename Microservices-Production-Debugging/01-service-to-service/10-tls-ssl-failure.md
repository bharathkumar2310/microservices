# Problem

A TLS/SSL failure occurs after TCP establishes but before a trusted encrypted application session is ready. Causes include expired/not-yet-valid certificate, hostname/SAN mismatch, unknown CA, incomplete chain, wrong SNI, missing/invalid client certificate in mTLS, protocol/cipher mismatch, and clock skew.

```text
DNS OK -> TCP OK -> TLS ClientHello/ServerHello/certificate X -> no HTTP
```

TLS failure is not connection refusal, connect timeout, HTTP 401/403, or read timeout.

# Production Situation

At `2026-09-13T21:36:09Z`, Order A cannot call Inventory B after certificate rotation.

* route `POST /v1/inventory/reservations`
* requestId `ord-t91c40`, traceId `10f92f3577b34da6a3ce929d0e0e0010`
* A `order-a-4.20.0-b7j2p`, Java 17, zone `eu-west-1c`
* B `inventory-b-7.6-q4n1d`, target `10.42.7.29:8443`
* normal TLS p99 18 ms, success 99.96%
* abnormal TLS failures 116/s, handshake 13 ms, B HTTP 0/s
* exception `PKIX path building failed: unable to find valid certification path`
* new B cert issuer `Shop-Intermediate-2026`; A trust bundle lacks that intermediate/root path
* old A 4.19.0 succeeds because its trust bundle was updated

# Architecture

```text
Order A 4.20.0 trust-v41 --TCP 9ms--> Inventory B
        |
        `-- TLS chain: leaf -> Shop-Intermediate-2026 -> Corp-Root-2
              X trust-v41 lacks Corp-Root-2

No HTTP authorization or application handler runs.
```

A gateway may terminate TLS and succeed while a direct A mTLS path fails, or the reverse. Treat each TLS peer boundary separately.

# What I Check FIRST

1. **Handshake reason.** WHAT: deepest exception/alert, SNI, peer, direction. WHY: "SSL error" is broad. LOOK FOR: PKIX trust-path failure.
2. **TCP success.** WHAT: connect duration/refusal/timeout. WHY: TLS cannot start without TCP. LOOK FOR: 9 ms success.
3. **Population.** WHAT: A/B version, target, zone, new/reused connection. WHY: trust/cert rotations often affect new handshakes only. LOOK FOR: A 4.20.0.
4. **Certificate and effective trust.** WHAT: SAN, issuer chain, validity, EKU, trust hash. WHY: certificate can be valid but untrusted. LOOK FOR: missing root.
5. **HTTP absence.** WHAT: gateway/B access and spans. WHY: TLS precedes HTTP. LOOK FOR: zero B requests for failed IDs.

# Step-by-Step Investigation

### Step 1 - Classify the TLS error

* **What I check:** Java nested exception, TLS alert, handshake stage, peer/SNI.
* **Why:** hostname mismatch, trust, mTLS, and protocol need different fixes.
* **Expected result:** handshake completes under 20 ms.
* **Bad result:** `SSLHandshakeException` caused by PKIX path building.
* **Meaning:** Java cannot build a chain to a trusted anchor.
* **Next branch:** inspect presented chain and A's effective trust.

### Step 2 - Confirm DNS and TCP first

* **What I check:** resolved IP, TTL, connect success/duration, exact port.
* **Why:** avoid calling a pre-TLS failure SSL.
* **Expected result:** expected `.29`, TCP 9 ms.
* **Bad result:** refused or timeout.
* **Meaning:** move to listener/network instead; TLS was never negotiated.
* **Next branch:** with TCP good, capture safe handshake metadata.

### Step 3 - Inspect presented certificate chain

* **What I check:** subject/SAN, issuer, chain order/completeness, notBefore/notAfter, EKU.
* **Why:** peer presentation may omit an intermediate or use wrong identity.
* **Expected result:** complete chain, correct SAN, current validity, serverAuth.
* **Bad result:** chain ends at `Corp-Root-2` unknown to A.
* **Meaning:** trust-anchor mismatch for this client.
* **Next branch:** compare effective JVM trust stores across A versions.

### Step 4 - Compare effective client trust and clock

* **What I check:** trust bundle hash, loaded path, JVM arguments, container mount, system clock.
* **Why:** repository intent may not be the loaded store; clock errors mimic validity failure.
* **Expected result:** all A instances use `trust-v42`, time synchronized.
* **Bad result:** A 4.20.0 uses `trust-v41`; A 4.19.0 uses v42.
* **Meaning:** packaging regression removed the new trust anchor.
* **Next branch:** compare image layers/build manifest and canary.

### Step 5 - Check mTLS and protocol alternatives

* **What I check:** client certificate presentation, issuer/EKU, SNI, TLS versions/ciphers, server policy.
* **Why:** server may reject A even when A trusts B.
* **Expected result:** mutual identities accepted with TLS 1.2/1.3.
* **Bad result:** `certificate_required`, `unknown_ca`, `protocol_version`, or `handshake_failure`.
* **Meaning:** follow identity or compatibility branch, not trust bundle assumed here.
* **Next branch:** inspect exact alert from the terminating peer.

### Step 6 - Compare direct and gateway paths

* **What I check:** which component terminates TLS, SNI/trust/client cert at each hop.
* **Why:** direct curl and gateway call can have different identities.
* **Expected result:** equivalent trust/SNI succeeds on both.
* **Bad result:** gateway succeeds with v42 while direct A fails with v41.
* **Meaning:** A-specific trust packaging, not B outage.
* **Next branch:** fix A image/trust, not B certificate validation.

### Step 7 - Verify no HTTP work occurred

* **What I check:** B access log/server spans, HTTP status counts, resources/dependencies.
* **Why:** TLS errors happen before route/auth/business.
* **Expected result:** no failed request ID at B; metrics remain normal.
* **Bad result:** B returns 401/403.
* **Meaning:** TLS succeeded; investigate HTTP authentication/authorization.
* **Next branch:** reclassify rather than mixing security layers.

### TLS failure branch matrix

| Handshake result | What it proves | What it does not prove | Exact next check |
|---|---|---|---|
| Hostname/SAN mismatch | Presented identity does not match requested name | Chain is untrusted or expired | Compare SNI/URL host with SAN and intended service contract |
| PKIX path failure | Client cannot build chain to trusted anchor | Server hostname is wrong | Inspect complete presented chain and loaded client trust |
| Certificate expired | Peer certificate is outside validity window | Rotation alone will restore every client | Check issuer/SAN/trust, clock, and overlapping deployment |
| Certificate not yet valid | Client clock/cert start time conflict | Certificate was issued incorrectly | Compare NTP/clock skew and `notBefore` across instances |
| `certificate_required` | TLS server requires a client certificate | A sent a valid client identity | Inspect A key/cert mount, selection, EKU, issuer, and expiry |
| Server `unknown_ca` | Server rejects A's client chain | A distrusts B | Inspect server trust bundle and A-presented chain |
| `protocol_version` | Peers have no allowed TLS version in common | Cipher suites are the only mismatch | Compare enabled protocol policies on both peers |
| Generic `handshake_failure` | Negotiation failed | Exact cause is known | Inspect peer alert, cipher/protocol, client cert, and SNI |
| TCP reset during handshake | Socket closed during TLS exchange | Certificate validation failed | Correlate terminator logs, lifecycle, and packet timing |
| TLS succeeds; HTTP 401 | Secure transport and HTTP responder work | Caller has business authorization | Inspect HTTP identity/token and authorization decision |

### Certificate-chain reasoning

The server normally presents the leaf and required intermediates.

The client trust store normally contains an approved root or trust anchor.

Trusting a leaf directly can work but makes routine rotation fragile unless
intentional certificate pinning is designed and operated.

An incomplete server chain may work on a developer machine that cached an
intermediate and fail in a minimal container.

`openssl` output must be interpreted with the CA file it actually used.

The JVM's loaded PKCS12/JKS trust store is the authority for this Java call,
not the host curl trust store.

I compare fingerprints and metadata, never private keys.

### Rotation timeline

At 21:30, B begins presenting the chain rooted at `Corp-Root-2`.

At 21:31, old A 4.19.0 with `trust-v42` continues successfully.

At 21:34, A 4.20.0 with regressed `trust-v41` receives traffic.

At 21:36, TLS failures reach 116/s as new connections are created.

Pooled old sessions can delay the onset; I split handshake outcomes by
connection reuse and creation time.

If failures begin before certificate rotation, I compare A image rollout and
trust loading first.

### mTLS identity checks

For mTLS I verify both directions: A trusts B's server identity and B trusts
A's client identity.

The client certificate needs appropriate validity, issuer, key usage/EKU, and
private-key access.

A missing client certificate is different from an untrusted client issuer.

Service mesh sidecars may terminate mTLS before Java, so Java logs can show
plain HTTP while the sidecar records the TLS failure.

I identify the actual TLS terminator before changing Java configuration.

### Recovery branches

If v42 trust loads but PKIX failures remain, I inspect server chain
completeness and the actual trust-store path selected by JVM arguments.

If trust succeeds but SAN mismatch appears, the root cause has moved to SNI or
certificate identity and is not fully fixed.

If handshakes recover but B returns 403, transport is fixed and authorization
requires a separate investigation.

If only one A instance fails, I compare its mounted secret/config hash and
clock before fleet-wide certificate changes.

If only new connections fail, I keep monitoring through maximum connection
age so pooled success does not hide residual risk.

# Metrics to Check

| Metric | Interpretation |
|---|---|
| TLS failures by reason | High PKIX/unknown CA selects trust; SAN selects identity; alert selects peer |
| Handshake duration | Low failure at 13 ms means deterministic validation; high may mean network/crypto load |
| TCP connect | Low 9 ms clears TCP sample; refusal/timeout occurs earlier |
| Cert expiry/notBefore | Low days-to-expiry risks outage; not sufficient for trust/SAN correctness |
| A outbound vs B HTTP | A attempts high, B flat means stop before HTTP |
| New/reused connections | New-only failures show existing TLS sessions masking rollout |
| Per-version/trust hash | One A version failing strongly narrows loaded trust |
| HTTP status | Flat zero for failures confirms no HTTP; 401 means TLS succeeded |
| CPU/GC | High CPU can slow handshake; low/flat here fits validation error |
| Retry attempts | High repeats deterministic failure and handshake cost |
| p50/p95/p99/max | Fast failures can make duration look good; pair with success |
| After rotation/deploy | Temporal alignment guides comparison; chain/trust evidence proves cause |

# Distributed Trace Investigation

```text
traceId=10f92f3577b34da6a3ce929d0e0e0010
Order A server                         34ms span=j001 ERROR
  Inventory client                    25ms span=j002
    dns.lookup                         2ms span=j003
    tcp.connect                        9ms span=j004
    tls.handshake                     13ms span=j005 ERROR PKIX
  [no HTTP request span]
  [no B server span]
```

Inspect peer address, SNI, certificate fingerprint/issuer, TLS protocol, source version, trust revision, zone, and reuse. Do not put private keys or full sensitive certificates in spans.

Missing B HTTP span matches handshake failure, but sampling and access logs must corroborate. A server-side TLS terminator may log the failed handshake without an application span.

# Distributed Logs

```text
2026-09-13T21:36:09.118Z level=ERROR service=order-service
instance=order-a-4.20.0-b7j2p version=4.20.0 zone=eu-west-1c
traceId=10f92f3577b34da6a3ce929d0e0e0010 spanId=j002 requestId=ord-t91c40
endpoint=POST_/v1/inventory/reservations downstream=inventory-service
target=10.42.7.29:8443 sni=inventory.shop.svc trust_revision=trust-v41
latency_ms=25 error="SSLHandshakeException: PKIX path building failed"
```

The log proves this JVM rejected a chain. It does not alone show which certificate was presented or why trust differed. Correlate with handshake metadata, trust hash, deployment manifest, and a good A instance.

# Commands / Tools

```powershell
Resolve-DnsName inventory.shop.svc -DnsOnly
Test-NetConnection 10.42.7.29 -Port 8443
curl.exe -v --connect-timeout 3 --max-time 8 https://inventory.shop.svc/actuator/health
```

Windows curl may use a different trust provider than Java, so success does not clear the JVM.

```bash
openssl s_client -connect 10.42.7.29:8443 -servername inventory.shop.svc \
  -showcerts -verify_return_error -brief </dev/null
keytool -list -keystore /app/truststore.p12 -storetype PKCS12
```

Run `keytool` only through approved access without printing passwords. `openssl` tests its configured CA set, not automatically Java's.

```bash
kubectl get pods -n shop -l app=order -o wide
kubectl describe pod -n shop order-a-4.20.0-b7j2p
kubectl get configmap -n shop order-trust-metadata -o yaml
```

Inspect metadata/hashes, never secret key material. Do not fix with curl `-k`, Java trust-all managers, or disabled hostname verification.

# Root Cause

Order A 4.20.0 image packaging regressed from `trust-v42` to `trust-v41`, which lacked `Corp-Root-2` needed for B's rotated chain.

```text
A image regression + B cert rotation -> TCP succeeds
-> JVM cannot build trusted chain -> TLS stops before HTTP
-> B handler receives nothing -> reservations fail
```

# Fix

**Immediate mitigation:** pause A 4.20.0 and route to A 4.19.0 with verified capacity, or deploy the approved v42 trust bundle. Preserve trust/cert hashes.

**Root cause correction:** repair image packaging and ensure the exact intended trust bundle loads.

**Permanent fix:** overlap cert rotations, test old/new client versions, validate trust/SAN/mTLS/protocol at build and canary, and monitor trust revision. Never disable verification.

# Verification

Before: TLS failures 116/s, B HTTP 0/s from A 4.20.0, business success 47%, handshake failures in 13 ms.

After: TLS failures 0/s, handshake p99 17 ms, B accepts 224/s, attempts/request 1.00, business success 99.97% for 30 minutes. Every A version/zone passes direct and gateway-equivalent TLS and one reservation commits.

# Prevention

* Alert on TLS reason, SNI, client version, target, and trust revision.
* Certificate inventory covers expiry, SAN, issuer, EKU, and rotation overlap.
* Build test opens the packaged JVM trust store and validates expected anchors.
* Canary runs mTLS from real Service A identity through each path.
* Runbook separates hostname, trust, validity, client-cert, and protocol failures.
* No retries for deterministic certificate validation errors.

# Interview Answer

### What I would say in an interview

I prove TCP completed, then classify the exact TLS failure. Here A connected in 9 ms but Java failed PKIX validation in 13 ms, and B saw no HTTP request. The failure followed A 4.20.0, while old A reached the same rotated B certificate. Comparing loaded trust hashes showed the new image had regressed to `trust-v41`, which lacked the new corporate root. I paused that rollout, restored the approved trust bundle, fixed packaging tests, and verified zero TLS failures, normal handshake p99, matching B traffic, and business success above 99.9%.

### Common interviewer traps

Do not call TLS failure a 401, disable validation, assume non-expired means valid, or trust a curl using a different CA store. Do not expose private keys while debugging.

### Quick memory flow

TCP proved -> exact alert -> SAN/chain/validity -> client trust/clock -> mTLS/protocol -> path terminator -> safe rollback -> full verify.

# Interview Follow-up Questions

1. **Hostname mismatch versus PKIX?** Hostname mismatch rejects SAN identity; PKIX cannot build to a trusted anchor.
2. **What does SNI do?** It tells the server which certificate/virtual host the client requests.
3. **Can a valid certificate fail?** Yes, due to trust, SAN, EKU, incomplete chain, client cert, or protocol.
4. **Why might old connections work?** Existing TLS sessions avoid a new handshake temporarily.
5. **TLS versus 401?** TLS happens before HTTP; 401 proves HTTP and asks for authentication.
6. **Why might gateway work but direct fail?** They use different trust stores, SNI, and client identities.
7. **Should I import the leaf cert?** Prefer the approved CA trust design; leaf pinning complicates rotation unless intentional.
8. **What is safe certificate logging?** Fingerprint, issuer, serial, SAN summary, validity; never private key.
