# Production Troubleshooting Study Chapter: Security

## Purpose

This chapter explains authentication, authorization, JWT validation, service identity, TLS, secrets, and safe production diagnosis. Security incidents require two goals at once: restore legitimate access and preserve the controls that prevent unauthorized access. Never disable certificate validation, JWT signature validation, issuer or audience validation, or authorization checks as a fix.

Use request IDs, token fingerprints, claim names, key IDs, and sanitized metadata. Do not paste bearer tokens, private keys, session cookies, client secrets, authorization headers, or personal claims into logs, terminals, tickets, or chat.

## Learning goals

After studying this chapter, you should be able to:

1. Distinguish authentication from authorization and interpret 401 versus 403.
2. Explain JWT encoding, signing, validation, registered claims, scopes, and roles.
3. Diagnose issuer, audience, signature, JWKS, clock, and claim-mapping failures.
4. Reason about gateway and downstream trust boundaries.
5. Choose token propagation, delegation, or token exchange safely.
6. Explain mTLS, workload identity, secret and certificate lifecycle, and rotation.
7. Investigate without exposing tokens or weakening controls.
8. Detect deployment regressions and build durable security observability.

---

# 1. Mental model

## 1.1 Authentication and authorization

**Authentication** answers "which principal presented verifiable credentials?"
**Authorization** answers "may that principal perform this action on this resource in this context?"

A valid identity can still be forbidden. A request may pass edge authentication yet fail downstream because each service is a separate trust boundary.

Typical HTTP semantics:

- **401 Unauthorized** means valid authentication credentials are absent, invalid, expired, or otherwise not accepted. The name is historical; the problem is authentication.
- **403 Forbidden** means the server understood the authenticated identity but policy denies the action. Some systems deliberately return 404 to avoid revealing a resource.

Implementations sometimes misuse these codes, so verify the response body, `WWW-Authenticate` metadata where safely available, gateway/service logs, and policy path. Status alone does not prove the cause.

## 1.2 JWT structure and signature

A JSON Web Token commonly has three Base64url-encoded parts:

```text
base64url(header).base64url(payload).base64url(signature)
```

The header can contain:

- `alg`: signing algorithm.
- `kid`: key identifier used to select a verification key.
- `typ`: optional token type metadata.

The payload contains claims such as:

- `iss`: issuer.
- `sub`: subject.
- `aud`: intended audience or audiences.
- `exp`: expiration.
- `nbf`: not valid before.
- `iat`: issued at.
- `jti`: token identifier.
- `scope` or `scp`: delegated permissions.
- `roles`: application roles.
- tenant, client, assurance, or custom claims.

The signature protects header and payload integrity/authenticity according to the algorithm and key. JWT contents are encoded, not encrypted by default. Anyone holding a normal signed JWT can often decode its claims; therefore do not place secrets in it.

Validation must:

1. Allow only configured algorithms.
2. Select a trusted key for the token's `kid`.
3. Verify the signature.
4. Match exact trusted issuer rules.
5. Match the expected audience for this API.
6. Enforce `exp` and `nbf` with a small documented clock-skew allowance.
7. Confirm token type/use where the identity platform distinguishes access and ID tokens.
8. Apply tenant, client, scope, role, subject, and resource policy.

Successful decoding proves none of these. A valid signature proves only that the holder of the signing key signed those bytes; it does not grant access to every API.

## 1.3 Issuer, audience, scopes, and roles

- **Issuer** names the authority that created the token. Trust must be explicitly configured.
- **Audience** names the intended recipient/API. API A must not accept a token intended for API B.
- **Scope** commonly represents delegated permission granted to a client acting for a user.
- **Role** commonly represents an application or principal assignment.

Authorization may also need tenant, ownership, resource state, client application, authentication strength, network context, and separation-of-duties rules.

Do not interchange scopes and roles merely because both are strings. Policy must match the identity provider's contract and token flow.

## 1.4 JWKS discovery, caching, and rotation

Asymmetric JWT validation obtains public keys from a trusted JSON Web Key Set (JWKS), often discovered from issuer metadata. Validators cache keys to avoid a network call per request.

Rotation sequence:

```text
identity provider publishes new public key
-> begins signing some tokens with new kid
-> validators refresh and accept new key
-> old tokens expire
-> old key can eventually be retired
```

Failure happens when a new `kid` is used before validators see it, caches ignore refresh, outbound access to JWKS fails, proxies cache stale metadata, or old keys are removed too soon. A safe validator can refresh on an unknown `kid` with rate limiting and continue using still-valid cached keys during a bounded metadata outage. It must not trust an arbitrary key URL from an untrusted token header.

## 1.5 Clock skew

`exp`, `nbf`, and `iat` use timestamps. Hosts with incorrect clocks can reject valid tokens as expired or not-yet-valid. A small configured skew handles normal distributed timing differences; a large allowance weakens expiration enforcement and hides time-sync faults.

Compare sanitized timestamps and clock offsets, not raw tokens. Fix time synchronization and configuration rather than enlarging skew indefinitely.

## 1.6 Gateway and downstream validation

A gateway can authenticate at the edge, but downstream services still need an authenticated, integrity-protected identity appropriate to their trust model. Common designs:

1. Propagate the original access token if it is intended for and safely usable by the downstream API.
2. Exchange the token for a downstream-audience token that preserves delegated identity.
3. Use workload identity for the calling service and transmit end-user context through a separately protected delegation mechanism.
4. Let a trusted mesh/gateway assert identity in signed or mutually authenticated metadata, but only when direct bypass is impossible and headers are stripped/recreated at the boundary.

Blindly forwarding a user token can violate audience and least privilege. Blindly trusting `X-User` or role headers allows spoofing if callers can reach the service.

## 1.7 Workload identity and mTLS

Mutual TLS authenticates both sides during TLS:

- The client validates the server certificate and hostname/service identity.
- The server validates the client certificate against trusted issuers and policy.

mTLS protects a connection and establishes certificate identity. It does not automatically authorize an operation, prove the human user, or protect data after TLS termination. Map certificate identity to an authorized workload.

Workload identity platforms issue short-lived credentials bound to a service account, pod, VM, or workload. They reduce static secret use but still require audience, subject, trust-domain, and policy checks.

## 1.8 Secret and certificate lifecycle

Failures can arise from:

- Expired/not-yet-valid certificates.
- Rotation deployed to one side only.
- Missing intermediate CA or trust-store update.
- Wrong hostname/SAN or workload subject.
- Revoked credential.
- Secret version absent, disabled, or access denied.
- Mounted secret not reloaded by a long-running process.
- Key and certificate mismatch.
- Clock drift.

Monitor usable lifetime, not only secret existence. Test rotation overlap and reload. Never log secret values to prove which version is loaded; use version identifiers, certificate serial/fingerprint, subject, issuer, and validity dates.

## 1.9 Safe evidence and logging

Useful fields:

```text
request/correlation ID
service and version
authentication stage and failure category
expected issuer and observed issuer identifier
expected audience and observed audience identifier
kid and algorithm
token type
exp/nbf/iat as timestamps or age, without the token
principal/tenant pseudonymous identifier
required and present scope/role names
policy/rule identifier and decision
certificate subject, issuer, serial/fingerprint, validity
JWKS cache age and refresh outcome
```

Redact authorization headers and cookies at ingestion. Avoid logging full claim sets. Limit access and retention of security logs.

## 1.10 Glossary

| Term | Meaning |
|---|---|
| Principal | Authenticated user, client, or workload identity |
| Credential | Evidence used to authenticate, such as token, certificate, or secret |
| Bearer token | Possession is generally sufficient to use it; protect against disclosure |
| JWT | Compact claims format that may be signed and/or encrypted |
| JWS | Signed content format commonly used for JWT access tokens |
| JWE | Encrypted content format |
| Claim | Named assertion in a token |
| Issuer | Authority that issued the token |
| Audience | Intended recipient of the token |
| Scope | Delegated permission commonly associated with user/client consent |
| Role | Assigned application permission commonly evaluated by policy |
| JWKS | Published set of public verification keys |
| `kid` | Key identifier selecting a key from a trusted set |
| OIDC | Identity layer and discovery conventions built on OAuth 2.0 |
| OAuth 2.0 | Authorization framework for delegated access |
| Token exchange | Obtaining a token suited to another audience or delegation hop |
| mTLS | TLS where client and server both present authenticated certificates |
| Trust store | Trusted certificate authorities/anchors |
| SAN | Certificate Subject Alternative Name used for identity/hostname matching |
| RBAC | Role-based access control |
| ABAC | Attribute-based access control |
| Least privilege | Grant only permissions needed for the operation |

---

# 2. Metrics and evidence

| Signal | Why it matters | Explicit interpretation |
|---|---|---|
| 401/403 rate by route, service, version, and reason | Locates authn/authz failure | A status increase needs reason classification; status alone is insufficient |
| Token validation failures by category | Separates expiry, signature, issuer, audience, key, and format | A new unknown-`kid` spike points toward rotation/cache |
| Authorization decisions by policy ID | Identifies denied rule | Log names and decisions, not sensitive policy input |
| JWKS cache age, refresh success/latency, unknown `kid` | Shows key lifecycle | Refresh failure with still-known keys may not affect existing tokens yet |
| Identity-provider token issuance errors/latency | Finds upstream identity issue | Normal issuance does not prove API validation is correct |
| Token age and time-to-expiry distributions | Finds stale clients and short lifetimes | Never label by token or subject |
| Host clock offset/time-sync state | Validates temporal claims | Offset near the allowed skew can cause intermittent failures |
| mTLS handshake failures by category | Separates trust, hostname, expiry, and protocol | Generic connection reset is not enough |
| Certificate days remaining and loaded fingerprint | Detects expiry/reload mismatch | Store public fingerprint only |
| Secret version and last successful reload | Detects stale process state | Presence in vault does not prove process loaded it |
| Gateway-to-service propagation/exchange failures | Finds hop mismatch | Edge success plus downstream invalid audience indicates token contract |
| Deployment/config change events | Finds regressions | Correlation requires a matching mechanism |

Evidence sources include gateway and application security events, identity-provider audit/health telemetry, trusted discovery/JWKS metadata, deployment manifests and diffs, service mesh/TLS metrics, certificate metadata, time synchronization, and a synthetic test identity with minimum permissions.

---

# 3. Generic security investigation workflow

## Step 1 - Protect and scope

Determine first bad time, route, status, environment, tenant class, client/workload, service version, and whether all or a subset of requests fail. Preserve controls. Stop a proven bad rollout or route traffic to a known-good version; do not bypass validation.

## Step 2 - Identify the rejecting hop

Trace gateway, mesh/ingress, application authentication middleware, authorization policy, and downstream. The response seen by the caller may be generated by any one of them.

## Step 3 - Classify the failure safely

Use reason codes and sanitized metadata to separate:

```text
missing/malformed credential
unsupported algorithm or token type
unknown kid / key refresh failure
bad signature
issuer mismatch
audience mismatch
expired / not yet valid
missing scope or role
tenant/client/resource policy denial
mTLS trust/hostname/expiry failure
secret unavailable/expired/not reloaded
```

## Step 4 - Compare one failed and one successful path

Compare client flow, route, gateway, service version, expected metadata, `kid`, issuer identifier, audience identifier, scope/role names, token age, and certificate fingerprint. Never share raw credentials.

## Step 5 - Check changes and dependencies

Inspect release/config, identity-provider app registration, key rotation, JWKS cache and network path, clock health, secret/certificate rotation, policy version, and service mesh changes.

## Step 6 - Test the causal hypothesis

Use a minimum-permission synthetic identity and approved environment. Predict which reason code and metric should change. A rollback or corrected config should restore valid requests while invalid test requests remain rejected.

## Step 7 - Mitigate without weakening security

Roll back incorrect configuration/code, restore trusted JWKS connectivity/cache behavior, deploy the correct certificate/secret through approved rotation, fix time sync, or correct token exchange and policy. Keep issuer, audience, signature, certificate, and authorization validation enabled.

## Step 8 - Make it durable

Add contract tests, staged rotations, expiration alerts, policy-as-code review, negative security tests, safe reason telemetry, and deployment canaries.

---

# 4. Original security questions

## 1. A previously working API suddenly returns 401. How would you investigate?

### Meaning and what it does not prove

A component rejected authentication or did not receive acceptable credentials. A 401 does not prove the password is wrong, token is expired, or application generated the response. A gateway, mesh, proxy, or downstream can return it.

### Issue locations

Client token acquisition/storage, authorization header propagation, gateway/ingress, service authentication middleware, issuer/audience config, JWT key cache, clock, identity provider, route policy, proxy size/header rules, cookies, mTLS, and deployment.

### Causal mechanisms and example

The token may be absent, malformed, expired, not yet valid, signed by an unknown/incorrect key, for the wrong issuer/audience, or stripped by a proxy. A certificate-based client flow may fail to obtain a token after secret expiry.

Example: the identity provider rotates to `kid=K2`. One service's JWKS cache remains on K1 because outbound DNS is broken. Tokens still signed with K1 work; newly issued K2 tokens return 401 only on that service.

### Ordered investigation and why

1. Record time, route, client class, correlation ID, rejecting hop, version, and failure reason; this defines scope without exposing credentials.
2. Check whether the Authorization header or expected cookie reaches the trusted boundary, using presence/format telemetry only.
3. Compare failed and successful sanitized token metadata: issuer identifier, audience, `kid`, algorithm, type, and time claims.
4. inspect validator configuration and recent deployments/config changes.
5. inspect JWKS cache age, unknown-`kid`, refresh outcome, trusted discovery URL, and identity-provider rotation.
6. check clock offset and token acquisition errors.
7. reproduce with a minimum-permission synthetic client and verify invalid controls remain rejected.

### Tools, metrics, evidence, and interpretation

Use request tracing, security reason counters, gateway/service logs, deployment diffs, identity-provider audit/health, trusted JWKS metadata, and time-sync telemetry. Failures only for one `kid` plus failed refresh strongly support rotation/cache trouble. Missing-header evidence at the service but present at gateway points to propagation/proxy config. `exp` earlier than trusted current time proves expiration, not why the client failed to refresh.

### Immediate mitigation

Roll back a bad validator/proxy/config release, restore trusted JWKS connectivity and refresh, repair token acquisition, correct time synchronization, or route to a known-good validated version. Preserve all validation.

### Permanent fix/design

Use standards-based libraries, exact issuer/audience configuration, bounded JWKS refresh on unknown `kid`, rotation overlap, redundant trusted metadata access, safe error categories, and contract/synthetic tests.

### Prevention and alerts

Alert on 401 rate by safe reason, unknown `kid`, JWKS refresh failure/cache age, token issuance errors, clock offset, and secret/certificate expiry. Canary key and configuration changes.

### Common mistakes

- Decoding a JWT and calling it valid.
- logging the token or Authorization header.
- treating all clients and instances as affected.
- increasing clock skew indefinitely.
- disabling signature, issuer, audience, or certificate checks.

### Interview-ready answer

I identify which hop returned 401 and classify the authentication reason using sanitized metadata. I compare a failed and successful request across header presence, issuer, audience, `kid`, algorithm, token type, and time claims, then check deployment, JWKS refresh, identity provider, clock, and credential acquisition. I roll back or repair the trusted configuration without weakening validation.

## 2. The API returns 403 even though the JWT is valid. Why?

### Meaning and what it does not prove

Cryptographic and claim validation established an accepted identity, but authorization policy denied the operation, or the implementation mapped an authentication failure to 403. A valid JWT does not prove the principal has the required scope, role, tenant, resource ownership, or action.

### Issue locations

Gateway authorization, application route/method policy, scope/role mapping, identity-provider assignment/consent, tenant/client allow-list, resource ownership, policy engine, claim transformation, and deployment.

### Causal mechanisms and example

The token can have correct issuer/audience/signature/expiry but lack `orders.write`. It may carry delegated scopes while the endpoint expects an application role, or the role claim mapping changed. Policy may deny cross-tenant access even with the right role.

Example: a service account receives `roles=["Orders.Read"]` and calls `POST /orders`, whose policy requires `Orders.Write`. Authentication succeeds; authorization correctly returns 403.

### Ordered investigation and why

1. Confirm the rejecting component and authorization policy/rule ID.
2. identify required action, resource, scope/role, tenant, and client constraints.
3. compare sanitized present claim names/values allowed for diagnosis with expected policy.
4. verify token flow: user-delegated scope versus application permission.
5. check role assignment, consent, group-to-role mapping, propagation delay, and overage behavior.
6. inspect policy/config/deployment changes and route/method matching.
7. test allowed and denied synthetic principals to protect negative behavior.

### Tools, metrics, evidence, and interpretation

Use policy decision logs, safe claim summaries, identity-provider app-role assignments, route policy configuration, and audit logs. A named policy denial for missing `Orders.Write` is direct evidence. Seeing the string in an ID token is irrelevant if the API expects an access token. Admin UI assignment without a newly issued token may not affect the old token.

### Immediate mitigation

Restore the correct least-privilege assignment or policy mapping, obtain a fresh token after approved assignment, or roll back an erroneous policy release. Do not grant a broad role to everyone.

### Permanent fix/design

Centralize policy semantics, separate scopes and roles, use resource-level checks, policy-as-code review, least privilege, explicit denial reasons internally, and authorization contract tests.

### Prevention and alerts

Alert on unexpected 403 changes by policy/route/client class. Test positive, negative, cross-tenant, wrong-role, wrong-scope, and ownership cases in CI and canaries.

### Common mistakes

- Equating valid signature with permission.
- checking only roles when the flow uses scopes.
- giving administrator access as a quick fix.
- returning sensitive policy detail to untrusted clients.
- overlooking HTTP method or resource ownership.

### Interview-ready answer

A valid JWT proves accepted authentication, not authorization. I find the denying policy, determine required action, scope or role, tenant, client, and resource ownership, and compare those with sanitized claims and the actual token flow. I correct the least-privilege assignment or policy and verify both allowed and intentionally denied cases.

## 3. Authentication works at the Gateway but fails at the downstream service. What would you check?

### Meaning and what it does not prove

The gateway accepted one credential, while the downstream did not receive or accept an identity valid for its trust boundary. Gateway success does not prove that forwarding the same token is safe or that the downstream should trust unsigned identity headers.

### Issue locations

Gateway route/filter, header stripping, token relay, token exchange, downstream issuer/audience/type config, service mesh, mTLS, proxy/header limits, clock, JWKS cache, and direct network access.

### Causal mechanisms and example

The original token may target the gateway audience, not the downstream API. A gateway may terminate authentication and fail to relay a token. A downstream may expect workload mTLS plus delegated user context. Proxy rules can remove an Authorization header.

Example: the gateway accepts audience `api-gateway`. It forwards that bearer token to inventory, which correctly requires audience `inventory-api`, so inventory returns 401. The safe fix is a designed token exchange or downstream authorization architecture, not relaxing audience validation.

### Ordered investigation and why

1. Trace the same correlation ID and identify the exact rejecting downstream.
2. document the intended trust model: token relay, exchange, workload identity, or protected gateway assertion.
3. verify credential presence and scheme at each hop without logging its value.
4. compare downstream expected issuer/audience/token type with sanitized observed metadata.
5. inspect gateway route/filter and token exchange errors.
6. inspect downstream JWKS, clock, and deployment/config.
7. verify direct callers cannot spoof gateway-generated identity headers.
8. test end-to-end with valid, wrong-audience, and missing-credential cases.

### Tools, metrics, evidence, and interpretation

Use distributed traces, gateway filter logs, header-presence telemetry, token-exchange audit data, downstream reason codes, mTLS identity logs, and config diffs. Gateway success plus downstream audience mismatch proves a token contract mismatch. Missing auth at downstream with gateway relay enabled points to forwarding/proxy behavior.

### Immediate mitigation

Roll back a bad route/filter release, restore approved token relay/exchange, repair mTLS identity, or route to a known-good version. Maintain downstream audience and signature validation.

### Permanent fix/design

Define authentication per hop, use audience-specific tokens/token exchange, protect service-to-service connections with workload identity, strip untrusted identity headers at ingress, and authorize again at the resource service.

### Prevention and alerts

Add end-to-end auth contract tests and alerts for token exchange, downstream 401 reasons, gateway/downstream decision mismatch, and mTLS handshake errors.

### Common mistakes

- Trusting plain identity headers from any network caller.
- accepting the gateway's audience downstream.
- forwarding tokens indiscriminately to every service.
- logging bearer tokens to compare them.
- removing downstream validation.

### Interview-ready answer

I first define the intended identity propagation model and trace the rejecting hop. I verify credential presence, token exchange, issuer, downstream audience, type, keys, time, and mTLS identity using sanitized metadata. I repair relay or exchange and keep downstream validation and authorization, because gateway success alone is not a downstream trust proof.

## 4. Service-to-service authentication suddenly stops working. What could be wrong?

### Meaning and what it does not prove

One workload can no longer establish or present an accepted identity to another. It may fail during credential acquisition, TLS handshake, token validation, or authorization. A generic connection error does not prove authentication.

### Issue locations

Workload identity agent, service account mapping, OAuth client credentials, secret vault, certificate issuance/mount/reload, mTLS trust stores/SANs, token endpoint, DNS/network, issuer/audience, JWKS, policy, and deployment.

### Causal mechanisms and example

A client secret or certificate can expire; a rotated secret may exist in the vault but the process still holds the old version. A CA rotation may update server certificates before client trust stores. A service account rename can change workload identity subject.

Example: sidecars rotate client certificates successfully, but a Java service loaded its key store only at startup. After the old certificate expires, new connections fail while existing pooled connections temporarily continue, producing a gradual incident.

### Ordered investigation and why

1. Locate the stage: DNS/TCP, TLS handshake, token acquisition, token validation, or authorization.
2. scope caller/callee instances, zones, versions, and new versus reused connections.
3. inspect safe certificate metadata or secret version, validity, subject/SAN, issuer, and loaded fingerprint.
4. verify trust chain, intermediate certificates, revocation status, and workload subject policy.
5. inspect token endpoint errors, client assignment, audience, and clock.
6. correlate secret/certificate/CA/policy/deployment rotation.
7. test with an approved synthetic workload identity and verify unauthorized identities still fail.

### Tools, metrics, evidence, and interpretation

Use TLS handshake reason telemetry, service mesh logs, certificate metadata tools, vault audit/version data, workload identity events, token endpoint errors, and config diffs. Failures only on new connections suggest rotation/reload. Unknown CA points to trust-chain rollout; hostname mismatch points to SAN/routing. `invalid_client` at token endpoint points to client credential/config, not downstream JWT validation.

### Immediate mitigation

Deploy the correct approved credential/trust chain with overlap, trigger a supported secure reload or rolling replacement, repair workload identity mapping, roll back bad policy, or restore token service connectivity. Keep certificate and token validation enabled.

### Permanent fix/design

Prefer short-lived workload identity over static secrets, automate rotation with overlap, support reload, monitor loaded identity, use least privilege, and exercise CA/secret expiry and rollback.

### Prevention and alerts

Alert well before certificate/secret expiry, on reload failure, token acquisition error, mTLS handshake reason, trust-bundle drift, and identity issuance failure. Inventory owners and dependencies.

### Common mistakes

- Checking vault contents but not what the process loaded.
- renewing a certificate without its intermediate chain.
- reusing a human or broad shared service credential.
- restarting everything before identifying rotation scope.
- bypassing hostname, trust-chain, or client-certificate validation.

### Interview-ready answer

I classify the failure stage first, then scope callers, callees, versions, and new connections. I compare safe loaded certificate or secret versions, validity, SAN, chain, workload subject, token acquisition, audience, keys, clock, and rotation events. I restore the correct overlapping credential or mapping through approved reload while preserving validation.

## 5. JWT validation suddenly fails after deployment. How would you troubleshoot it?

### Meaning and what it does not prove

A release changed code, library, configuration, routing, environment, or runtime such that tokens are rejected. Temporal correlation strongly prioritizes the deployment but does not prove application code; identity-provider rotation or clock drift can coincide.

### Issue locations

JWT library/version, algorithm allow-list, issuer normalization, audience parser, claim mapper, discovery/JWKS URL, proxy/CA trust, cache behavior, environment substitution, clock/timezone handling, token type, and gateway routing.

### Causal mechanisms and example

A library upgrade may enforce audience arrays correctly where old code accepted a string loosely. Configuration may point to a trailing-slash-different issuer. A new container may lack the internal CA needed to fetch trusted JWKS. Claim mapping may change `scp` to `scope`.

Example: the new image has `AUTH_AUDIENCE=orders` instead of `orders-api`. Every correctly signed token is rejected for audience mismatch, while decoding appears normal.

### Ordered investigation and why

1. Compare 401 rate and validation reason by old/new version; this tests release specificity.
2. stop or roll back the bad rollout if impact is material while retaining evidence.
3. compare effective sanitized configuration, library versions, trust store, and clock.
4. compare one accepted-old and rejected-new validation using issuer, audience, `kid`, algorithm, type, and temporal result, never raw token.
5. inspect discovery/JWKS fetch, cache, proxy, DNS/TLS, and unknown-`kid`.
6. inspect claim and authority mapping plus route security rules.
7. run positive and negative contract tests against the candidate fix.

### Tools, metrics, evidence, and interpretation

Use per-version reason metrics, deployment/config diff, dependency lockfile, startup config diagnostics with secrets redacted, trusted metadata/JWKS checks, clock telemetry, and synthetic tokens from the approved test issuer. Failures isolated to new pods prove a version/config environment difference. Audience reason plus changed effective audience identifies the mechanism.

### Immediate mitigation

Pause/roll back the release, correct configuration or trusted CA bundle, restore compatible library behavior that still performs full validation, or repair JWKS access. Verify invalid signature, issuer, audience, and expired test cases remain rejected.

### Permanent fix/design

Pin and review security library changes, validate startup configuration, use environment-specific contract tests, canary deployments, safe reason metrics, trusted CA/JWKS health checks, and negative tests.

### Prevention and alerts

Alert on validation failures by version and reason. Gate deployments on correct issuer/audience, signature, key rotation, expiry, not-before, and malformed-token tests.

### Common mistakes

- Printing the token to compare environments.
- accepting multiple broad audiences to make tests pass.
- falling back to an insecure algorithm or skipping signature checks.
- forgetting trust-store and proxy changes in the image.
- testing only a valid token.

### Interview-ready answer

I compare old and new versions by sanitized validation reason, roll back if needed, and diff effective issuer, audience, algorithms, library, JWKS/discovery, CA trust, claim mapping, and clock. I reproduce with approved synthetic credentials and require positive plus negative tests. The fix preserves full signature, issuer, audience, time, and authorization checks.

---

# 5. High-value additional questions

## 6. How should JWKS key rotation and outages be handled?

### Meaning and what it does not prove

Validators need fresh trusted public keys without depending on the identity provider for every request. An unknown `kid` may be legitimate rotation or an attack designed to force refreshes; it is not permission to trust token-supplied locations.

### Issue locations

Issuer discovery, trusted JWKS URI, HTTP/DNS/TLS, proxy/CDN cache, validator cache, refresh throttling, identity-provider publication/signing order, and old-key retirement.

### Causal mechanisms and example

If signing begins before the new key is globally published, new tokens fail. If every unknown `kid` triggers a fetch, attackers can cause a metadata denial of service. If the cache never refreshes, rotation causes prolonged 401s.

### Ordered investigation and why

1. confirm exact trusted issuer and configured JWKS endpoint.
2. compare failed `kid` with current trusted set and publication timeline.
3. inspect cache age, refresh attempts, HTTP status, TLS, DNS, and proxy age.
4. verify known cached keys still validate during bounded metadata outage.
5. check refresh rate limiting and concurrent-refresh coalescing.
6. test staged new-key publication and old-key overlap.

### Tools, metrics, evidence, and interpretation

Use public key identifiers, validator refresh telemetry, trusted endpoint response metadata, identity-provider rotation events, and synthetic signed test tokens. Unknown `kid` plus trusted JWKS containing it but stale local cache proves refresh behavior. A token's `jku` is not a trusted source.

### Immediate mitigation

Restore access to the configured trusted endpoint, trigger the library's supported bounded refresh, roll back a bad proxy/cache rule, or have the identity team restore safe key overlap. Never accept an unverified token.

### Permanent fix/design

Publish before signing, overlap keys through maximum token lifetime plus skew, cache responsibly, refresh periodically and on unknown `kid` with rate limits/single-flight, and provide resilient trusted metadata access.

### Prevention and alerts

Run rotation drills and alert on cache age, refresh failure, unknown-`kid` rate, endpoint certificate expiry, and validation failures by key ID.

### Common mistakes

- Fetching arbitrary `jku` or `x5u` from a token.
- refreshing for every request.
- removing old keys before old tokens expire.
- making each request synchronously depend on JWKS.

### Interview-ready answer

I trust only the issuer-configured JWKS endpoint, cache keys, refresh periodically and on unknown `kid` with rate limiting and single-flight, and retain usable cached keys through bounded outages. Rotation publishes the new key before use and overlaps old keys until tokens expire. I alert on refresh and unknown-key failures.

## 7. When should a service propagate a token versus exchange it?

### Meaning and what it does not prove

Propagation preserves the original bearer token; exchange obtains a token intended for another hop and may preserve delegated identity. A technically accepted propagated token does not prove least privilege or correct audience.

### Issue locations

Gateway, calling service, identity provider/token exchange, downstream audience, delegated scope, workload identity, logs, and retry/cache of tokens.

### Causal mechanisms and example

A user token for `orders-api` should not automatically be accepted by `payments-api`. Token exchange can issue a short-lived payment-audience token with only `payment.authorize` and delegation context. A background job without a user should use workload/application identity instead of fabricating user context.

### Ordered investigation and why

1. identify whether the call acts as user, service, or both.
2. define the downstream audience and minimum permission.
3. determine whether original token contract explicitly includes downstream.
4. choose relay only when audience/trust and exposure are appropriate; otherwise exchange.
5. protect token storage, logs, retries, and lifetime.
6. test confused-deputy, wrong-audience, and missing-delegation cases.

### Tools, metrics, evidence, and interpretation

Use architecture contracts, identity-provider exchange audit, safe audience/scope telemetry, and authorization tests. Downstream accepting a gateway-only audience is evidence of an overly broad trust rule, not successful design.

### Immediate mitigation

Restore the approved exchange/relay path or route to a known-good version; do not broaden accepted audiences.

### Permanent fix/design

Use audience-specific least-privilege tokens, short lifetimes, workload identity, explicit delegation, and centralized supported token acquisition libraries.

### Prevention and alerts

Alert on exchange failures, wrong-audience validation, unexpected caller identity, and broad permission regressions.

### Common mistakes

- forwarding a bearer token to every dependency.
- using an ID token as an API access token.
- losing user delegation and authorizing only the middle-tier service.
- caching tokens beyond their safe context or expiry.

### Interview-ready answer

I identify whether the downstream action is on behalf of a user or the workload, then require a token for that downstream audience with minimum permission. I relay only when the original token contract explicitly supports the hop; otherwise I use token exchange or workload identity with protected delegation and test wrong-audience and confused-deputy cases.

## 8. How do you debug security failures safely without exposing credentials?

### Meaning and what it does not prove

Safe debugging uses metadata and correlation rather than raw secrets. Redaction alone does not prove logs are safe; payloads, URLs, stack traces, traces, and support exports can leak data.

### Issue locations

Application/gateway logs, APM headers, trace baggage, error pages, shell history, tickets, CI artifacts, crash dumps, identity audit, and debugging proxies.

### Causal mechanisms and example

Copying a JWT into a decoder or ticket gives every reader bearer access until expiry and reveals claims. Instead, compare a one-way fingerprint, `kid`, issuer/audience identifiers, timestamps, reason code, and correlation ID inside trusted tools.

### Ordered investigation and why

1. establish approved tools, access, retention, and incident channel.
2. use a synthetic minimum-permission identity where possible.
3. record token presence and sanitized metadata, never raw value.
4. use request IDs to join gateway, service, policy, and identity-provider events.
5. inspect secret/certificate public metadata and version identifiers.
6. verify redaction across logs, traces, errors, and support bundles.
7. if exposure occurs, revoke/rotate and follow incident procedure.

### Tools, metrics, evidence, and interpretation

Use structured reason codes, centralized access-controlled logs, identity audit, configuration fingerprints, certificate fingerprints, and secret-scanner controls. A matching token fingerprint correlates events without making the token reusable, but fingerprints should still be access-controlled.

### Immediate mitigation

Stop unsafe capture, restrict access, remove exposed artifacts under policy, revoke/rotate affected credentials, and continue with sanitized evidence.

### Permanent fix/design

Default-deny sensitive headers in telemetry, field allow-lists, centralized redaction tests, short-lived credentials, synthetic diagnostics, and documented playbooks.

### Prevention and alerts

Scan logs/artifacts for credential patterns, audit sensitive-log access, and test telemetry redaction in CI.

### Common mistakes

- pasting tokens into public web decoders.
- enabling full request-header logging.
- putting secrets on command lines or in shell history.
- assuming Base64url encoding hides claims.

### Interview-ready answer

I use correlation IDs, validation reason, `kid`, issuer/audience identifiers, time claims, token fingerprints, and certificate or secret version metadata in access-controlled systems. I use synthetic least-privilege credentials, verify telemetry redaction, and revoke and rotate immediately if a credential is exposed.

## 9. How do you distinguish an authentication deployment regression from an identity-provider incident?

### Meaning and what it does not prove

Both can begin suddenly. Timing near a deploy suggests a regression but requires version and dependency comparison; a provider status page alone does not prove local health.

### Issue locations

Canary/stable versions, configuration, library, trust store, network, token issuance, discovery/JWKS, provider region/tenant, and clock.

### Causal mechanisms and example

If only new pods reject the same class of token, effective config/library/trust differs. If old and new versions across services fail token acquisition or unknown new keys simultaneously, the identity provider or shared network path is more likely.

### Ordered investigation and why

1. split validation results by service version, instance, client, issuer tenant, and region.
2. separate token issuance from API validation.
3. compare old/new effective safe config and dependencies.
4. check provider audit/health, JWKS/discovery, DNS/TLS, and clock.
5. use an approved synthetic flow against old and new versions.
6. roll back canary only if version evidence predicts recovery.

### Tools, metrics, evidence, and interpretation

Use per-version reason counters, deployment events, provider issuance metrics, trusted JWKS checks, and synthetic probes. Only-new-version audience failures indicate regression. Cross-version issuance failures indicate provider/client-credential path. Cross-service unknown-`kid` can indicate rotation or shared stale proxy.

### Immediate mitigation

Roll back a proven bad deployment, restore provider/network access, or apply the provider's approved recovery path. Keep validation intact.

### Permanent fix/design

Canary security changes, separate issuance and validation SLIs, add configuration fingerprints, redundant metadata paths, and provider/rotation exercises.

### Prevention and alerts

Alert by version and reason, on issuance failures, JWKS refresh, clock drift, and canary-versus-stable divergence.

### Common mistakes

- blaming the last deploy without a version split.
- trusting status pages instead of local evidence.
- changing several auth settings simultaneously.
- broadening trust during uncertainty.

### Interview-ready answer

I split failures by version and distinguish token issuance from local validation. I compare effective config, libraries, CA trust, keys, and clocks while checking provider and network evidence. A canary-only failure supports rollback; cross-version issuance or key failures point to shared identity infrastructure. In either case I preserve all trust checks.

---

# 6. Decision trees

## 6.1 401 or 403

```text
Which component generated the response?
|-- Gateway/mesh -> inspect credential presence, route policy, mTLS, exchange.
`-- Service
    |
    Authentication validation failed?
    |-- missing/malformed -> client or proxy propagation.
    |-- signature/kid -> trusted JWKS, rotation, cache, algorithm.
    |-- issuer/audience/type -> token contract or configuration.
    |-- exp/nbf -> client refresh, clock, small documented skew.
    `-- No; authenticated
        |
        Authorization denied?
        |-- scope/role -> flow, assignment, consent, mapping.
        |-- tenant/client -> allow-list or cross-tenant policy.
        |-- resource/action -> ownership, method, state, ABAC rule.
        `-- implementation mapped status incorrectly -> correct semantics.
```

## 6.2 mTLS failure

```text
Did TCP connect?
|-- No -> DNS, route, firewall, listener.
`-- Yes
    |
    TLS handshake reason?
    |-- unknown CA -> trust bundle/intermediate/rotation.
    |-- expired/not yet valid -> validity, loaded cert, clock.
    |-- hostname/SAN mismatch -> routing or certificate identity.
    |-- client certificate rejected -> client chain, subject, policy.
    |-- protocol/cipher -> compatible secure configuration.
    `-- handshake succeeds -> investigate application authz separately.
```

## 6.3 JWT fails after release

```text
Only new version fails?
|-- Yes -> diff effective issuer, audience, algorithm, library,
|          trust store, JWKS/proxy, claim mapping, and clock.
`-- No
    |
    Only new kid fails?
    |-- Yes -> trusted JWKS publication/cache/refresh/rotation.
    `-- No
        |
        Issuance also fails?
        |-- Yes -> identity provider, client credential, network.
        `-- No -> shared validator config, clock, policy, or routing.
```

---

# 7. Final cheat sheet

## 401 versus 403

```text
401: no acceptable authentication for this request.
403: accepted identity, denied action/resource by policy.
Reality: verify the component and reason because implementations vary.
```

## JWT validation checklist

1. Expected token type and approved algorithm.
2. Signature against a key from the trusted issuer's JWKS.
3. Exact trusted issuer.
4. Intended audience for this API.
5. Expiration and not-before with small documented skew.
6. Tenant/client constraints.
7. Required delegated scope or application role.
8. Resource/action/ownership policy.

## Fast evidence sequence

1. Time, route, client/workload, version, and rejecting hop.
2. Safe failure category and correlation ID.
3. Header/credential presence without value.
4. Issuer, audience, `kid`, algorithm, type, and time metadata.
5. JWKS cache/refresh, identity-provider, and clock.
6. Policy ID, required/present permissions, tenant/resource.
7. Secret/certificate version, validity, loaded fingerprint, and trust chain.
8. Positive and negative synthetic verification after the fix.

## Production rules

- A decodable JWT is not a validated JWT.
- A valid JWT is not authorization.
- Audience is a security boundary, not an inconvenience.
- Validate and authorize at each designed trust boundary.
- Trust only configured issuer metadata and JWKS locations.
- Rotate keys and certificates with overlap and tested reload.
- Prefer short-lived workload identity over static shared secrets.
- Never put raw credentials in logs, tickets, terminals, or chat.
- Never disable certificate or JWT validation as a fix.
