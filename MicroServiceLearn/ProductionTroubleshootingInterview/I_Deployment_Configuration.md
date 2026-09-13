# Production Troubleshooting Study Chapter: Deployment and Configuration

## Purpose

This chapter teaches a diff-first, evidence-led way to diagnose failures introduced during deployment. It begins with how an artifact becomes a running instance, then covers rollout strategies, configuration and secret resolution, database compatibility, Kubernetes lifecycle behavior, service discovery, health probes, graceful draining, mixed versions, feature flags, rollback, roll-forward, and 5xx attribution.

Examples use Kubernetes-style terminology, but the reasoning applies to virtual machines, managed container platforms, serverless revisions, and traditional release systems.

> Safety: Start with read-only evidence. Do not print Secret values, tokens, private keys, or full environment dumps. Do not disable probes, TLS verification, authentication, or authorization to "make it work." Before rollback or restart, preserve the failing artifact identity, configuration fingerprints, events, logs, metrics, and a representative request trace. Use approved deployment controls and verify each change.

## Learning goals

After studying this chapter, you should be able to:

1. Explain artifact, runtime, configuration, dependency, traffic, and data compatibility as separate deployment dimensions.
2. Compare old and new instances using immutable identities and effective configuration.
3. Diagnose canary, blue-green, and rolling failures while mixed versions are live.
4. Trace configuration precedence without exposing secrets.
5. Evaluate secret and certificate identity, mounting, permissions, validity, and rotation.
6. Design expand-migrate-contract database changes that remain safe during rollback.
7. Interpret startup, readiness, liveness, discovery, endpoint, and graceful-drain evidence.
8. Attribute 5xx responses to the layer that generated them.
9. Choose rollback, roll-forward, pause, traffic shift, or feature disable based on compatibility and evidence.
10. Convert an incident into immutable release controls, tests, alerts, and a runbook.

---

# 1. Foundational mental model

## 1.1 A deployment changes more than code

A running instance is the result of several inputs:

```text
Running behavior
  = immutable application artifact
  + runtime and base image
  + startup command and arguments
  + effective configuration
  + secret and certificate material
  + identity and authorization
  + infrastructure and resource limits
  + network, DNS, proxy, and service discovery
  + database/schema/data state
  + downstream versions and contracts
  + traffic policy and feature flags
```

"The code did not change" does not eliminate a deployment cause. A rebuilt image tag can contain a new base image; a restarted pod can resolve a changed Secret; a ConfigMap can be mounted differently; a service account can lose permission; a flag can expose dormant code.

The first investigation question is therefore: **what effective inputs differ between a healthy old instance and a failing new instance?**

## 1.2 Diff-first incident response

A deployment gives a powerful natural experiment:

```text
same traffic + same environment + old works + new fails
                       |
                       `-> compare old and new before broad dependency blame
```

Capture and compare:

- Image digest, not only tag.
- Build commit, build ID, dependency lockfile/SBOM, base image digest.
- Deployment template, command, arguments, service account, labels, ports.
- Effective non-secret configuration keys, source, and value fingerprints.
- Secret/certificate version and file metadata, never secret values in tickets.
- Node, zone, network policy, resource request/limit, architecture.
- Database driver/runtime version and migration state.
- Feature-flag evaluation context.
- Readiness and endpoint membership.

Change correlation is not proof, but the smallest failing/healthy diff sharply narrows hypotheses.

## 1.3 Immutable artifacts and promotion

Build once and promote the same content-addressed artifact through environments:

```text
registry.example.internal/orders@sha256:4b9...f27
```

A mutable tag such as `release` can point to different bytes across time or nodes. A rollback to that tag may not restore the previous artifact. Record image digest, source commit, build provenance, vulnerability/signature verification, and configuration release identity.

Configuration should also be versioned and reproducible. Runtime secrets can rotate independently, but their version identity and rollout behavior must be observable without revealing content.

## 1.4 Deployment strategies

### Rolling update

Old pods are gradually replaced by new pods. Capacity and failure depend on `maxSurge`, `maxUnavailable`, readiness, termination grace, and traffic draining.

```text
time 1: old old old old
time 2: old old old new
time 3: old old new new
time 4: new new new new
```

Mixed versions coexist, so APIs, messages, cache formats, sessions, and database schemas must be backward/forward compatible. A readiness bug can either send traffic too early or prevent progress.

### Canary

A small percentage, cohort, region, or instance set receives the new version first. Canary comparison is meaningful only when traffic, tenants, cache warmth, and observation windows are considered. A 1 percent canary may never see a rare endpoint.

Automated promotion should compare request rate, error rate, latency, saturation, business metrics, and health to a control. Avoid averaging away the canary.

### Blue-green

Two complete environments exist; traffic switches from blue to green.

```text
blue: old, serving
green: new, validated
switch traffic
```

Rollback can be a fast traffic switch only if database, queue, external side effects, and configuration remain compatible. Long-lived connections, DNS caching, jobs, and asynchronous consumers may keep using the prior environment.

### Recreate

Old stops before new starts. This avoids mixed versions but causes downtime and increases rollback pressure. It may be valid for nonredundant or exclusive resources, but should be explicit.

## 1.5 Desired state versus effective runtime state

A Git manifest shows desired configuration. It does not prove what a process uses. Configuration can be transformed or overridden between source and runtime:

```text
repository values
 -> deployment rendering
 -> platform object
 -> projected environment/file
 -> framework binding and profile
 -> application default/override
 -> dynamic configuration or feature flag
 -> effective behavior
```

Kubernetes ConfigMap/Secret updates illustrate this difference:

- Environment variables are resolved when the container starts and do not update in place.
- Projected volumes are updated eventually, but an application may read only at startup.
- `subPath` mounts do not receive projected updates.
- A rollout annotation or checksum often forces replacement when configuration changes.
- A dynamic config client may cache, poll, stream, or fall back.

Observe effective configuration through a secured diagnostics endpoint that lists safe keys, sources, and fingerprints. Do not expose raw secrets.

## 1.6 Configuration precedence

Framework precedence varies, so document and test it. A conceptual order is:

```text
command-line arguments
  override environment variables
    override profile-specific files
      override packaged files
        override application defaults
```

Remote config, system properties, injected files, and feature flags can appear elsewhere. Common traps include:

- Wrong active profile.
- Environment variable name transformed incorrectly.
- Duplicate keys with different case or separators.
- String `"false"` interpreted as truthy by custom code.
- Unit mismatch such as `30` seconds versus milliseconds.
- Shell quoting or newline in a Secret.
- A value set to an empty string overriding a valid default.
- Configuration loaded before a mounted file appears.

## 1.7 Secrets and certificates

A secret incident is diagnosed by metadata:

```text
expected secret object/version
actual mounted or injected version
service account identity
key name and file path
file owner/mode and process UID/GID
certificate subject/issuer/SAN
notBefore/notAfter and chain
trust-store identity
reload behavior
```

Never compare secrets by printing them. Use approved fingerprinting where policy permits. Certificate failures can arise from expiration, not-yet-valid clocks, missing intermediate CA, hostname/SAN mismatch, trust-store changes, client-certificate selection, algorithm policy, or a process that never reloaded rotated material.

## 1.8 Database migration compatibility

Deployment and schema changes form one compatibility protocol. Use expand-migrate-contract:

```text
1. Expand: add nullable column/table/index or compatible capability.
2. Deploy compatible code: reads old/new safely; writes as planned.
3. Migrate/backfill: bounded, observable, resumable.
4. Switch behavior: feature flag or controlled rollout.
5. Contract: remove old column/behavior only after all readers are gone.
```

During rolling deployment:

- Old code must tolerate the expanded schema and new writes.
- New code must tolerate old rows and partially completed backfill.
- DDL locks and backfill load must be measured.
- Migration ownership and one-run locking must be explicit.
- Rollback must not require restoring already-dropped data.

Destructive rename/drop in the same release can make rollback impossible. Prefer additive schema and dual-read/dual-write only with a clear convergence and removal plan.

## 1.9 Kubernetes lifecycle and traffic

### Startup probe

Startup answers: has initialization completed enough that liveness should begin? While it fails, Kubernetes suppresses liveness and readiness checks for the container. It protects slow starts from premature liveness restarts.

### Readiness probe

Readiness answers: should this instance receive new service traffic now? Failure removes the pod from ready endpoints but normally does not restart it.

Readiness should cover essential ability to serve, not every optional dependency. If all instances make readiness depend on one optional reporting service, that dependency can remove the whole application from traffic.

### Liveness probe

Liveness answers: is the process irrecoverably unable to make progress such that restart is useful? It should not fail merely because a remote dependency is down. Otherwise dependency outage creates restart storms and destroys diagnostic evidence.

### Pod readiness is not the whole path

```text
process listens
 -> startup succeeds
 -> readiness succeeds
 -> pod Ready condition
 -> Service selector matches
 -> EndpointSlice contains ready address
 -> proxy/load balancer programs route
 -> discovery/DNS returns expected service
 -> caller can connect and protocol succeeds
```

Success at one stage does not prove later stages.

## 1.10 Graceful startup and drain

On termination, Kubernetes commonly:

1. Marks the pod terminating.
2. Runs `preStop` if configured.
3. Sends `SIGTERM`.
4. Removes endpoints asynchronously.
5. Waits for the termination grace period.
6. Sends `SIGKILL` if the process remains.

The application should stop accepting new work, become unready, allow endpoint propagation, drain in-flight requests, stop claiming new queue work, finish or safely checkpoint current work, close resources, and exit before the grace period.

Readiness removal and load-balancer propagation are not instantaneous. A short sleep alone is a fragile drain strategy. Align application drain timeout, proxy retry behavior, request deadlines, consumer lease duration, and Kubernetes grace period.

## 1.11 Mixed versions and contracts

During a rollout, new callers can hit old servers and old callers can hit new servers. Messages can remain in a queue for days. Compatibility covers:

- Request/response fields and defaults.
- Enum additions and unknown-value behavior.
- Authentication claims and scopes.
- Event schema and semantic meaning.
- Cache key/value formats.
- Session serialization.
- Database schema and rows.
- Feature-flag evaluation.

Use additive APIs, tolerant readers, explicit versions only for semantic breaks, and mixed-version integration tests. Do not rely on deployment order as the sole compatibility mechanism.

## 1.12 Feature flags

Flags decouple code deployment from feature exposure, but they are production configuration:

- Define owner, purpose, default, targeting, expiry, and removal date.
- Audit changes and evaluate fail-safe defaults.
- Monitor outcomes by flag variant.
- Avoid a combinatorial matrix of interacting flags.
- Test both paths and stale client/cache behavior.
- Use a kill switch for risky optional behavior, not for hiding mandatory correctness.

A flag rollback can be faster and safer than artifact rollback if the fault is isolated to the flagged path. It cannot undo schema or external side effects already produced.

## 1.13 Rollback versus roll-forward

Rollback is favored when:

- The previous artifact/config is known good and immutable.
- Database and messages remain backward compatible.
- The new release has not produced irreversible incompatible state.
- Traffic can be shifted quickly and safely.

Roll-forward is favored when:

- Migration or external effects cannot be reversed.
- Old code cannot read newly written data.
- A narrow fix is understood and can deploy faster.
- Security or data-correction requirements forbid returning to old behavior.

Sometimes the safest sequence is pause rollout, disable a flag, shift traffic to healthy capacity, then roll forward. "Rollback" must name artifact, configuration, schema, flags, and traffic; reverting only code may not restore behavior.

## 1.14 5xx attribution

A 5xx means some server-side layer reported failure, but the body/status alone may not identify which:

```text
client
 -> CDN/WAF
 -> external load balancer
 -> ingress/API gateway
 -> service proxy/mesh
 -> application
 -> downstream
```

Examples:

- Gateway `502`: upstream connect/reset/protocol failure.
- Gateway `503`: no healthy upstream endpoints or overload policy.
- Gateway `504`: upstream deadline exceeded.
- Application `500`: unhandled/internal error.
- Application `503`: deliberate unavailable/overload response.

These are conventions, not universal truths. Attribute using response headers, access-log source, request/trace ID, ingress upstream status, application request count, and downstream spans.

## 1.15 Glossary

| Term | Meaning |
|---|---|
| Artifact | Built deployable unit such as a container image or JAR |
| Image digest | Content-addressed immutable image identity |
| Desired state | Configuration declared to the deployment platform |
| Effective configuration | Values actually resolved and used by the process |
| Rollout | Controlled replacement or exposure of a release |
| Canary | Limited exposure of a new version against a control |
| Blue-green | Parallel old/new environments with a traffic switch |
| Rolling update | Gradual old-to-new instance replacement |
| Rollback | Restore a prior compatible release state |
| Roll-forward | Deploy a corrective newer state |
| Readiness | Eligibility to receive new traffic |
| Liveness | Whether restart is appropriate for lost process progress |
| Startup probe | Startup grace gate before other probes govern |
| EndpointSlice | Kubernetes object listing Service backend endpoints |
| Drain | Stop new work and complete/checkpoint current work |
| Immutable configuration | Versioned configuration promoted without in-place ambiguity |
| Secret rotation | Replacing credential material and transitioning consumers |
| Expand-contract | Add compatibility first, remove old shape later |
| Feature flag | Runtime control that changes behavior without rebuilding |
| Error budget | Allowed unreliability implied by an SLO |
| Upstream | Context-dependent target a proxy calls; verify each product's terminology |

---

# 2. Core evidence and metrics

## 2.1 Release identity

Every request log and metric series should expose bounded labels or fields:

```text
service, environment, region, instance
source_commit, build_id, image_digest
deployment_revision, configuration_revision
safe configuration fingerprint
feature variant
```

Do not use unbounded request IDs as metric labels. Use them in logs/traces.

## 2.2 Golden and rollout metrics

| Signal | Why it matters |
|---|---|
| Request rate by version | Confirms actual exposure and denominator |
| Error rate by version/status/route | Shows regression concentration |
| Latency histogram by version/route | Finds tail regression |
| CPU, memory, throttling, pools, queues | Detects resource/saturation change |
| Pod ready/desired/available | Measures rollout capacity |
| Restart count and exit reason | Separates crash, OOM, probe restart, eviction |
| Startup/readiness/liveness failures | Locates lifecycle gate |
| Ingress upstream status and connect time | Attributes proxy versus app failure |
| Endpoint count and rejected connections | Detects discovery/traffic gap |
| DB connection/query/migration metrics | Detects compatibility and credential failure |
| Consumer lag/job ownership | Detects mixed-version async effects |
| Business success and correctness metrics | Catches failures hidden by HTTP 200 |

Compare rates, not raw counts, and compare equivalent traffic. A canary with 2 errors in 10 requests is worse than stable with 20 in one million.

## 2.3 Kubernetes evidence, safely collected

Concrete read-only examples:

```text
kubectl -n shop-prod rollout status deployment/order-api
kubectl -n shop-prod rollout history deployment/order-api
kubectl -n shop-prod get deployment order-api -o yaml
kubectl -n shop-prod get replicasets,pods -l app=order-api -o wide
kubectl -n shop-prod get pods -l app=order-api -L app.kubernetes.io/version
kubectl -n shop-prod describe pod order-api-7d86f9b56c-r8m4n
kubectl -n shop-prod get events --sort-by=.metadata.creationTimestamp
kubectl -n shop-prod logs order-api-7d86f9b56c-r8m4n --since=30m
kubectl -n shop-prod logs order-api-7d86f9b56c-r8m4n --previous
kubectl -n shop-prod get service order-api -o yaml
kubectl -n shop-prod get endpointslice -l kubernetes.io/service-name=order-api -o yaml
```

YAML can include environment references and sensitive metadata. Restrict storage and redact before sharing. Do not run `kubectl get secret -o yaml` for routine diagnosis.

Interpret evidence:

- `CrashLoopBackOff` is a restart backoff condition, not a root cause; inspect prior exit reason/logs.
- `OOMKilled` and exit code 137 point toward memory enforcement, but verify container status and node events.
- Exit code 143 often means handled `SIGTERM`, not an application crash.
- `Ready=False` plus running process points to readiness or readiness dependency.
- Desired replicas equal current but available is low means capacity is not serving.
- Service selector with zero EndpointSlice addresses explains proxy 503 without an app 5xx.

## 2.4 Old/new comparison table

| Dimension | Healthy old | Failing new | Interpretation |
|---|---|---|---|
| Image digest | digest A | digest B | Artifact/build/base image changed |
| Node/zone | zone A | zone A | Reduces node/zone hypothesis |
| Service account | orders-v1 | orders-v2 | Identity/RBAC/cloud role changed |
| Config fingerprint | cfg-91 | cfg-94 | Resolve key/source differences |
| Secret version | db-17 | db-18 | Credential/format/rotation suspect |
| Ready endpoint | yes | no | Inspect probe and endpoint gate |
| DB TLS result | succeeds | trust error | Trust store/cert/runtime difference |
| Request cohort | same route/tenant | same | Stronger version comparison |

One row suggests a hypothesis; several independent signals establish a causal chain.

---

# 3. Generic deployment investigation and design workflow

## 3.1 Incident workflow

1. **Protect users.** Pause promotion, stop replacing healthy old capacity, or reduce new-version traffic. This limits exposure while preserving the comparison.
2. **Define scope.** Record start time, routes, status codes, regions, versions, cohorts, and business impact. Scope tells whether the change is global, version-specific, or topology-specific.
3. **Preserve release evidence.** Record digest, revision, configuration fingerprint, flags, migration version, events, prior logs, and representative trace before restarting or rolling back.
4. **Locate the 5xx/lifecycle layer.** Determine whether requests reach ingress, Service endpoints, process listener, application handler, and dependency. This avoids debugging app code for gateway-generated errors.
5. **Compare healthy old and failing new.** Hold traffic and environment as constant as possible, then diff artifact and effective runtime inputs.
6. **Check lifecycle and capacity.** Startup, readiness, liveness, restarts, endpoint membership, drain, and resource saturation can explain immediate rollout errors.
7. **Walk the failing dependency path.** Resolve DNS, route, TLS, identity, pool, protocol, and database/schema evidence from inside equivalent runtime context using approved diagnostics.
8. **Correlate with changes.** Code, manifest, config, Secret, certificate, migration, flag, infrastructure, and downstream deploys all belong on the timeline.
9. **Choose the reversible mitigation.** Traffic shift, rollout pause, flag disable, config correction, rollback, or roll-forward must respect schema and side effects.
10. **Verify and monitor.** Confirm endpoint membership, representative requests, error/latency/business metrics, queue health, and no hidden old/new divergence.
11. **Correct and prevent.** Fix the mechanism, automate comparison/validation, add canary gates, compatibility tests, and the earliest actionable alert.

## 3.2 Deployment design workflow

1. Build once and identify artifact by digest.
2. Version configuration; inventory precedence and safe fingerprints.
3. Define Secret/certificate rotation and reload behavior.
4. Design mixed-version API, event, cache, session, and schema compatibility.
5. Choose rollout strategy and capacity parameters.
6. Implement startup, readiness, liveness, and graceful drain based on real semantics.
7. Define canary cohorts, minimum sample, metrics, abort/promote thresholds.
8. Separate migration phases and test rollback at every phase.
9. Define feature-flag ownership, defaults, telemetry, expiry, and kill behavior.
10. Automate post-deploy smoke/business checks from realistic network paths.
11. Record provenance and expose version/config identity in telemetry.
12. Rehearse rollback and roll-forward, including asynchronous workers.

---

# 4. Original interview questions

# 1. Everything was working before deployment, but APIs started failing immediately after deployment. How would you investigate?

## Meaning and invariants/risks

The timing makes the release the leading hypothesis, not proof. "Deployment" includes artifact, config, secret, schema, identity, traffic, infrastructure, and flags. The operational invariant is to retain known-good capacity and avoid making evidence disappear while limiting customer impact.

Risks include rolling all old pods away, unsafe rollback across a destructive migration, retry amplification, restarting away the only comparison, and focusing on code while a gateway has no ready endpoints.

## Failure locations

- Image fails to start or binds the wrong port/interface.
- Startup/readiness/liveness behavior changed.
- Config/profile/secret/certificate is wrong.
- Database migration or contract is incompatible.
- Service selector, route, policy, service account, or DNS changed.
- Resource limits cause throttling or OOM.
- New code throws or calls a failing dependency.
- Feature flag activates an untested path.
- Old pods are terminated before connections drain.

## Detailed causal mechanisms with plain-language example

A release changes readiness path from `/health/ready` to `/actuator/health/readiness`, but the deployment still probes the old path. The process starts and direct local calls work, yet pods never enter Service endpoints. Ingress returns 503 immediately. Application error logs are empty because requests never reach it.

Another release drops a database column before all old pods leave. Old pods return 500 while new pods appear healthy. The deployment is correlated, but the mechanism is mixed-version schema incompatibility.

## Ordered investigation/design reasoning and why each step matters

1. Pause rollout and retain old replicas; this controls impact and preserves the natural A/B comparison.
2. Scope errors by version, route, status, region, and start second; this tests release specificity.
3. Attribute the response layer with ingress/access logs, upstream status, trace, and application request count.
4. Inspect rollout status, pod conditions, restarts, events, endpoints, and prior logs; this finds lifecycle and routing failure before deep code analysis.
5. Compare old/new digests, deployment template, effective config fingerprints, identities, flags, resources, and nodes.
6. Check migration state and old/new compatibility before rollback.
7. Walk one representative failing request through dependencies and compare it with a successful old request.
8. Mitigate through traffic shift, flag, config, rollback, or roll-forward based on proven compatibility.

## Evidence, state transitions, tools, and result interpretation

```text
Deployment:
Progressing -> some New Ready -> error gate breached -> Paused

Request:
client -> ingress 503 -> no ready endpoint
or
client -> ingress -> new pod -> database auth error -> application 500
```

Use rollout history/status, ReplicaSet/pod labels, EndpointSlices, ingress upstream logs, application logs/traces, metrics split by version, config fingerprint, migration table, and audit history.

- Errors only on new digest with same node/traffic implicate new effective inputs.
- Ingress errors with no app request count indicate pre-application failure.
- App 500 plus DB authentication exception indicates the app generated the 5xx from a dependency failure.
- Restarts after liveness failures indicate probe-driven restarts, not necessarily crashes.

## Immediate mitigation/recovery

Pause promotion. Shift traffic to healthy old instances or disable the isolated feature. Restore endpoint capacity. Roll back only after confirming schema/message/config compatibility; otherwise roll forward or correct config. Rate-limit retries and watch queues.

## Permanent correction/design

Add automated canary gates, immutable digests, effective-config comparison, post-deploy smoke tests, compatibility migrations, lifecycle tests, and graceful drain. Require release telemetry to include version and config revision.

## Prevention and alerts

Alert on new-version error ratio versus control, zero/low endpoints, rollout unavailable replicas, restart increase, readiness duration, and business success. Block promotion when thresholds fail.

## Common mistakes

- Immediately restarting all pods.
- Saying "deployment caused it" without locating the layer.
- Comparing tags instead of digests.
- Rolling back code across an incompatible schema.
- Checking only aggregate fleet metrics.
- Disabling probes to force green status.

## Concise interview-ready answer

I pause the rollout, retain old capacity, and scope failures by version, route, status, and time. I first attribute whether the 5xx comes from ingress, routing, application, or a dependency, then compare old and new image digests, effective config, identity, probes, endpoints, resources, flags, and schema state. I preserve evidence and choose traffic shift, flag disable, compatible rollback, or roll-forward, then verify both technical and business metrics.

---

# 2. Only newly deployed instances are failing. What could be wrong?

## Meaning and invariants/risks

This is a high-value controlled comparison. Differences can be artifact-related or can correlate with scheduling, startup time, identity, configuration resolution, cache warmth, or traffic targeting. Keep at least one old and one new instance available for evidence, without exposing customers unnecessarily.

## Failure locations

- Defective code, dependency, base image, architecture, or startup command.
- New config/Secret revision or active profile.
- New service account, RBAC, workload identity, or network policy.
- New pods scheduled only on one bad node/zone/pool.
- Readiness passes before initialization or cache/connection warm-up.
- Resource requests/limits changed.
- New label/port mismatch.
- Mixed-version schema, API, event, session, or cache incompatibility.
- Canary receives a different tenant or route cohort.

## Detailed causal mechanisms with plain-language example

Old pods use Java trust store A embedded in image digest A. New digest B updates the base image and drops an internal CA. Only new pods fail TLS to Payment. The deployment manifest and application commit look unchanged, but the immutable artifact differs.

Alternatively, all new pods land on ARM nodes while a native library supports only x86. The symptom follows new instances because scheduling and release happen together.

## Ordered investigation/design reasoning and why each step matters

1. Confirm failures follow version/digest rather than one node, zone, tenant, or route.
2. Compare representative old/new requests of the same shape to avoid cohort bias.
3. Diff image digest/build provenance, pod template, command, identity, resources, config and secret versions.
4. Compare startup/readiness timing and endpoint admission; new pods may be exposed too early.
5. Compare node/zone/architecture and network path.
6. Compare dependency handshakes, schema expectations, cache/session formats, and feature evaluation.
7. Reproduce through a controlled canary or approved in-cluster diagnostic rather than changing production blindly.

## Evidence, state transitions, tools, and result interpretation

```text
old digest A: 10,000 requests, 0.1% 5xx
new digest B: 500 requests, 18% 5xx
same route/tenant/zone: regression follows digest B
```

Inspect pod `imageID`, labels, service account, node, env key names, mounted-file metadata, probe history, and trace spans. If moving new digest to a known-good node still fails, node is less likely. If old digest with new config fails, configuration is isolated. Use controlled matrices only when safe.

## Immediate mitigation/recovery

Set canary weight to zero or pause rollout, keep evidence pod out of customer endpoints, and serve from old capacity. Correct a narrow config/identity issue or deploy a compatible fix. Ensure old capacity can handle traffic before shifting all load.

## Permanent correction/design

Promote by digest, sign/provenance-check images, test target architecture, compare effective config automatically, gate readiness on essential initialization, test mixed versions, and make canaries representative.

## Prevention and alerts

Split RED and saturation metrics by digest/revision/zone. Alert on statistically meaningful new-versus-control regression and endpoint churn. Validate base-image and trust-store changes in CI.

## Common mistakes

- Assuming code diff is the only diff.
- Deleting failing pods before comparison.
- Ignoring traffic cohort and cold-start bias.
- Declaring all nodes bad because only new pods fail.
- Exposing a diagnostic pod to normal traffic.

## Concise interview-ready answer

I treat old and new as an A/B test and compare equivalent requests. I diff immutable image/build identity, effective config and Secret versions, service account, resources, probes, node/zone/architecture, feature variants, and contract compatibility. I remove new instances from traffic while preserving one for evidence, then isolate whether failure follows digest, config, or placement. The permanent fix adds version-split telemetry and automated canary/config/provenance gates.

---

# 3. Old instances work but new instances fail to connect to the database. What would you check?

## Meaning and invariants/risks

The database is reachable for at least one workload, so compare connection inputs and runtime path. "Cannot connect" can mean DNS, TCP, TLS, authentication, authorization, pool exhaustion, driver protocol, or schema initialization. Do not weaken TLS or print credentials.

## Failure locations

- Wrong URL/host/port/database/profile or precedence.
- Secret key/version, whitespace, encoding, or mount path differs.
- Rotated credentials accepted by old existing connections but rejected for new logins.
- New service account/cloud identity lacks database access.
- NetworkPolicy, security group, firewall, DNS, or IPv6 behavior differs.
- Missing CA/intermediate, hostname mismatch, expired cert, clock skew, or client-cert failure.
- Driver/runtime changes protocol or cipher requirements.
- Pool initialization overwhelms connection limits.
- Migration lock or startup query is mistaken for connectivity.

## Detailed causal mechanisms with plain-language example

The database password rotated. Old pods keep long-lived authenticated connections and continue serving. New pods load the new Secret, but it contains a trailing newline from an incorrect generation step, so authentication fails. Restarting old pods would remove the last healthy pool.

In another case, new pods use a new service account and network policy selector; TCP times out, while credentials are fine. The error phase distinguishes them.

## Ordered investigation/design reasoning and why each step matters

1. Preserve old connections and pause rollout; restarting can turn partial outage into total outage.
2. Identify the failure phase: DNS, connect timeout/refused, TLS, authentication, authorization, pool acquisition, or SQL.
3. Compare sanitized effective JDBC/connection properties and configuration source, including host, port, database, TLS mode, user identity, and driver.
4. Compare Secret object/version/key and mounted metadata without printing the value.
5. Compare service account, network policies, node/zone, DNS resolution, and route from equivalent runtime context.
6. Inspect certificate chain, SAN, validity, trust-store fingerprint, and clock for TLS errors.
7. Inspect database audit/connection logs, limits, role grants, and migration locks.
8. Test the narrow hypothesis with approved read-only diagnostics.

## Evidence, state transitions, tools, and result interpretation

```text
DNS failure       -> no address; inspect name/search domain/resolver
connect timeout   -> route/firewall/policy or silent endpoint
connection refused-> wrong port or no listener
TLS alert         -> chain/SAN/protocol/client certificate/trust
auth rejected     -> credential/identity/account state
permission denied -> role/grant/database ownership
pool timeout      -> capacity/leak/slow connect, not necessarily network
SQL syntax/missing relation -> schema/driver/application compatibility
```

Use sanitized startup logs, effective config source, pod identity, DNS and TCP telemetry, TLS diagnostic metadata, database connection/audit logs, pool metrics, and migration history. A successful TCP connection does not prove TLS or login. A successful login does not prove schema permission.

## Immediate mitigation/recovery

Stop replacing old pods. Correct the Secret/config/identity or restore a compatible credential through approved rotation. Shift new pods out of traffic. Increase connection capacity only if measured and safe; do not mask a connection leak or startup storm. Never set trust-all TLS.

## Permanent correction/design

Use dual-credential rotation or overlapping validity, startup validation with safe error classification, least-privilege identities, controlled pool warm-up, trust-store tests, immutable config references, and pre-deployment connectivity checks from the actual workload identity.

## Prevention and alerts

Alert on new connection failures by phase, pool acquisition time, database connection utilization, certificate expiry, Secret rollout age, and authentication rejection. Rehearse rotation without relying on old pooled connections.

## Common mistakes

- Restarting healthy old pods.
- Printing passwords to compare them.
- Blaming firewall for an authentication error.
- Disabling certificate verification.
- Treating missing table as connectivity.
- Increasing pool size without checking DB connection limits.

## Concise interview-ready answer

I preserve the old pools and classify the exact connection phase. Then I compare old/new sanitized URL and source, Secret version/key, workload identity, network path, DNS, driver, TLS trust and certificate metadata, pool settings, and database audit/grants. Old persistent connections can hide a bad credential rotation, so I avoid restarting them. I correct the narrow input or identity issue and add phased connection telemetry plus rotation and target-identity tests.

---

# 4. A configuration value is correct locally but wrong in production. How would you troubleshoot it?

## Meaning and invariants/risks

Local correctness proves only one precedence chain and environment. The goal is to trace the production value from desired source to effective application behavior. A value may be correct in the repository yet overridden, transformed, cached, or never reloaded.

## Failure locations

- Wrong environment/profile/namespace/release values.
- Template renders unexpected quoting, type, unit, or key name.
- Environment/argument overrides a file.
- ConfigMap or Secret reference/key is stale.
- Environment variable requires pod restart.
- Projected volume update is delayed or blocked by `subPath`.
- Dynamic config client cache, fallback, targeting, or outage.
- Application binds a different prefix or reads only at startup.
- One code path uses a hard-coded/default value.
- Feature flag evaluation differs by identity.

## Detailed causal mechanisms with plain-language example

Repository config says timeout `30s`. Production sets environment variable `PAYMENT_TIMEOUT=30`, and the application interprets the bare number as milliseconds. The value appears "30" in both places but semantics differ.

Another case: ConfigMap changed from `false` to `true`, but the value is injected as an environment variable. Existing pods retain `false` until replaced, while engineers expect live update.

## Ordered investigation/design reasoning and why each step matters

1. Define the observed wrong behavior and the exact key/type/unit; otherwise the team may compare different concepts.
2. Identify affected instances and whether values differ by revision, restart time, tenant, or flag cohort.
3. Read the secured effective-config diagnostic: value for non-sensitive keys or hash/redacted form, source, profile, and load time.
4. Walk precedence backward through arguments, environment, mounted files, remote config, rendered manifest, and repository source.
5. Check update semantics: startup-only, poll, stream, volume refresh, `subPath`, or required rollout.
6. Compare a healthy and failing instance and correlate configuration/audit history.
7. Correct the authoritative source and deploy/reload through the supported mechanism.

## Evidence, state transitions, tools, and result interpretation

```text
Git value -> rendered value -> platform object -> pod projection
-> framework source/precedence -> parsed typed value -> runtime behavior
```

Inspect deployment YAML cautiously, ConfigMap metadata/content when non-sensitive, pod environment key names without dumping all values, mount paths/timestamps, application startup property-source logs, dynamic-config audit, and safe fingerprint endpoints.

- Correct platform object plus old environment value means restart/rollout semantics.
- Correct mounted file plus wrong effective value means precedence, path, or binding.
- Correct effective value plus wrong behavior means code/feature interpretation, not distribution.

## Immediate mitigation/recovery

Pause rollout if the value is harmful. Correct the authoritative production source, trigger the documented reload or controlled rollout, and verify effective value on each instance before returning traffic. Use a feature kill switch if designed. Avoid ad hoc per-pod edits because replacement loses them and creates drift.

## Permanent correction/design

Document precedence and types, validate configuration at startup, reject invalid units/ranges, version config, expose safe source/fingerprint diagnostics, force rollouts using config checksums when needed, and test rendered production-like manifests.

## Prevention and alerts

Alert on config-fingerprint divergence, rejected configuration, dynamic-config fetch failure, and unsafe fallback. Audit changes and require schema validation. Include config identity in incident dashboards.

## Common mistakes

- Comparing source files instead of effective runtime values.
- Dumping all environment variables and secrets.
- Editing a live pod.
- Assuming ConfigMap update reloads every application.
- Ignoring type/unit parsing.
- Restarting until one instance happens to work.

## Concise interview-ready answer

I identify the exact key, type, unit, affected revisions, and observed behavior, then trace production precedence from repository and rendered manifest through platform object, pod projection, framework property source, parsing, and runtime use. I compare safe effective-config fingerprints and sources between instances and check reload semantics. I fix the authoritative source and perform a controlled reload/rollout, then add schema validation, provenance, drift alerts, and production-like rendering tests.

---

# 5. One service instance has different configuration from the others. What could cause this?

## Meaning and invariants/risks

Instance-level drift breaks the assumption that replicas are interchangeable. It can produce intermittent failures proportional to traffic and may disappear on restart, making evidence preservation important.

## Failure locations

- Pods belong to different ReplicaSets or revisions during/after rollout.
- One pod started before a ConfigMap/Secret/environment change.
- Mutable image tag or nonimmutable downloaded config differs.
- Projected volume or dynamic config refresh failed on one pod.
- `subPath`, local file, init container, or sidecar produced stale content.
- Node-local DNS/cache/file/daemon differs.
- Per-instance environment override or manual mutation.
- Feature targeting uses pod/zone/tenant identity.
- Clock skew changes certificate or flag evaluation.

## Detailed causal mechanisms with plain-language example

Four pods use an environment variable from ConfigMap revision 22. The ConfigMap changes in place, then only one pod restarts on another node. That pod reads revision 23; the other three retain revision 22. The deployment name is identical, but effective startup inputs differ.

Or a mutable image tag is resolved to a new digest only on a node without the cached image because `imagePullPolicy` allows cache reuse.

## Ordered investigation/design reasoning and why each step matters

1. Identify the outlier by error rate and remove it from traffic without deleting it; this limits impact and preserves evidence.
2. Compare owner ReplicaSet, pod-template hash, imageID/digest, creation time, restart count, node, and service account.
3. Compare safe configuration fingerprint, active profile, source versions, load time, and feature variants.
4. Inspect mount metadata, refresh-client status, init/sidecar results, and platform events.
5. Check manual changes and audit logs.
6. Determine whether drift is desired mixed-version behavior or unintended nonconvergence.
7. Replace through the controller only after evidence, then verify fleet convergence.

## Evidence, state transitions, tools, and result interpretation

```text
Desired config revision 23
pod A started before change -> effective 22
pod B started after change  -> effective 23
```

Pod-template hash differences indicate different templates. Same template but different imageID indicates mutable-tag or registry/cache issues. Same digest and desired references but different effective fingerprint points to runtime loading/refresh or instance-local state.

## Immediate mitigation/recovery

Mark the outlier unready or remove it through approved traffic controls. Preserve metadata/logs, then recreate it from the controller or correct the refresh mechanism. Verify all replicas' digest and config fingerprint. Do not copy files manually between pods.

## Permanent correction/design

Use immutable digests and versioned configuration names/checksums, automated rollout on startup-only changes, drift detection, restricted mutation rights, and safe effective-config endpoints. Avoid node-local required configuration.

## Prevention and alerts

Alert when one service revision has multiple unexpected config fingerprints or image digests. Monitor refresh failures and projected-file age. Periodically reconcile desired and effective identities.

## Common mistakes

- Deleting the pod before collecting differences.
- Assuming same deployment name means same inputs.
- Manually patching one replica.
- Comparing raw secrets.
- Ignoring feature-targeting context.
- Treating intended rolling mixed versions as drift without checking policy.

## Concise interview-ready answer

I remove the outlier from traffic but preserve it, then compare ReplicaSet/template hash, image digest, start time, node, identity, safe config and Secret revision fingerprints, profiles, mounts, refresh status, and feature targeting. Common causes are mixed revisions, startup-only environment values, failed refresh, mutable tags, or manual/node-local drift. I restore it through the controller and prevent recurrence with immutable identities, config-triggered rollouts, drift detection, and mutation audit.

---

# 6. A service registers successfully but other services cannot discover it. What would you check?

## Meaning and invariants/risks

Registration proves that some registry/control-plane write succeeded. Discovery also requires correct service identity, namespace, health eligibility, endpoint publication, client lookup, network reachability, and protocol agreement. In Kubernetes, pods do not usually self-register; Services/selectors and EndpointSlice controllers publish endpoints.

## Failure locations

- Caller uses wrong service name, namespace, port, scheme, or environment.
- Registration metadata advertises an unreachable host/container-local address.
- Lease expires or health marks instance unavailable.
- Service selector does not match pod labels.
- Pod is not Ready, readiness gate is false, or endpoint is excluded.
- EndpointSlice/controller/proxy update is delayed or unhealthy.
- DNS negative/positive cache is stale; search path differs.
- Mesh sidecar, network policy, firewall, RBAC, or mTLS blocks traffic.
- Client discovery cache, zone filter, version filter, or load-balancer rule excludes it.
- Service exposes named port pointing at wrong target port.

## Detailed causal mechanisms with plain-language example

The Order pod logs "registered" with Consul using address `127.0.0.1`. The registry contains it, but remote callers connect to their own loopback and fail. In Kubernetes, a pod is Running but readiness fails; the Service exists and DNS resolves, yet EndpointSlice has no ready address, so ingress returns 503.

## Ordered investigation/design reasoning and why each step matters

1. Define discovery technology and expected service identity; registry, DNS, Kubernetes Service, and mesh have different evidence.
2. Check the registry/Service record and advertised address/port/scheme.
3. Check health/lease/readiness and current endpoint membership.
4. Resolve from an affected caller and inspect cache/TTL/search domain; control-plane view may differ from client view.
5. Test route/TCP/TLS/protocol from the caller's identity and network path using approved diagnostics.
6. Inspect selectors, labels, named ports, zone/version filters, network policy, mesh config, and mTLS identity.
7. Compare a healthy discoverable instance to isolate metadata or topology differences.

## Evidence, state transitions, tools, and result interpretation

```text
registered -> health eligible -> endpoint published -> DNS/client lookup
-> network reachable -> TLS accepted -> application protocol accepted
```

Use Service and EndpointSlice YAML, pod conditions, DNS metrics/logs, registry health/lease data, mesh proxy configuration/status, NetworkPolicy, and caller traces.

- DNS name resolves but there are zero ready endpoints: readiness/selector/controller.
- Endpoint exists but TCP times out: routing/policy/firewall.
- TCP works but TLS fails: identity/trust/SAN.
- Direct endpoint works but service name fails: DNS/proxy/port mapping.
- Only one caller fails: caller cache, identity, policy, or zone filter.

## Immediate mitigation/recovery

Correct metadata, selector, readiness, port, or route through controlled configuration. Restore healthy endpoints or route to known-good instances. Flush/restart only the affected discovery client if evidence proves stale cache and normal refresh cannot recover. Do not bypass mTLS or hard-code pod IPs.

## Permanent correction/design

Validate advertised endpoints, use readiness-aware registration, set TTL/heartbeat and deregistration policy, test discovery from a real caller identity, standardize service names/ports, and monitor control-plane-to-data-plane convergence.

## Prevention and alerts

Alert on registered-versus-healthy endpoint mismatch, zero endpoints for an active Service, lease expiry, DNS failures, endpoint programming latency, and mTLS rejection. Add deployment smoke tests through discovery, not direct localhost only.

## Common mistakes

- Treating registration success as reachability.
- Testing only from the service itself.
- Hard-coding an IP as a fix.
- Ignoring readiness and selector labels.
- Disabling network policy or TLS.
- Flushing every cache without proof.

## Concise interview-ready answer

I walk the full chain: correct service identity, registered metadata, health/lease, readiness, selector and endpoint publication, caller DNS/cache, route, port, TLS, and protocol. I test from the affected caller's identity and compare a healthy endpoint. Registration alone proves only a control-plane write. I correct the narrow metadata or data-plane fault and add endpoint-convergence and caller-path smoke tests plus zero-endpoint alerts.

---

# 7. Health check is failing after deployment even though the application starts successfully. Why?

## Meaning and invariants/risks

Process start only proves the runtime launched. A probe can fail because it targets the wrong address/path/port/scheme, because initialization is incomplete, because the health implementation reports a dependency, or because timeouts/resources prevent a response. Misusing liveness can turn a recoverable problem into a restart loop.

## Failure locations

- Wrong probe path, port, named port, scheme, host, or authentication.
- Application binds only loopback or a different management port.
- Startup is slower than probe timing.
- Readiness waits on migrations, cache warm-up, leader election, or essential dependency.
- Liveness includes remote database/message broker and restarts on their outage.
- Probe timeout is shorter than scheduling/GC/startup latency.
- Sidecar/mesh interception or network policy changes.
- Health endpoint returns unexpected status/body after framework upgrade.
- CPU throttling or memory pressure delays probe.
- TLS health endpoint has trust/client-certificate mismatch.

## Detailed causal mechanisms with plain-language example

The Java process logs "Started" after opening its main port, but it is still applying a five-minute cache warm-up. Readiness correctly remains false. Kubernetes should keep it out of traffic, not restart it. If the same condition is wired to liveness with a 10-second threshold, the pod is killed repeatedly and can never finish warm-up.

## Ordered investigation/design reasoning and why each step matters

1. Identify which probe fails and its exact failure: HTTP status, timeout, connection refused, TLS, or exec exit.
2. Compare probe configuration with actual listeners, paths, schemes, and startup logs.
3. Run the equivalent read-only check from the pod/network context if approved, not only from a laptop.
4. Inspect pod events, probe timing, restart reason, `--previous` logs, CPU throttling, and GC pauses.
5. Inspect health component details to find the failing essential dependency or readiness gate.
6. Decide whether the component belongs in startup, readiness, liveness, or observability only.
7. Correct semantics/timing rather than simply increasing every threshold.

## Evidence, state transitions, tools, and result interpretation

```text
container started
 -> startup probe succeeds
 -> readiness succeeds -> endpoint receives traffic
 -> liveness continuously proves local progress
```

- Connection refused early: listener not open, wrong port/interface, or startup still active.
- HTTP 404: wrong path/base path.
- HTTP 401/403: probe authentication mismatch.
- HTTP 503 with component detail: readiness deliberately rejects.
- Timeout plus CPU throttling: scheduling/resource issue.
- Repeated `Unhealthy` then exit 143: kubelet terminated due to probe policy.

## Immediate mitigation/recovery

Pause rollout and retain healthy capacity. Correct the probe target or add a startup probe. Remove optional remote dependencies from liveness. Increase timing only from measured startup/pause distributions and within rollout objectives. Do not force-ready an instance that cannot serve safely.

## Permanent correction/design

Separate startup/readiness/liveness endpoints and semantics. Keep liveness local and cheap, readiness bounded to essential serving ability, and expose dependency details securely. Test probes in the built image with production ports, base paths, sidecars, and resource limits.

## Prevention and alerts

Monitor probe failure reason, startup/readiness duration, restarts caused by liveness, endpoint availability, and resource throttling. Canary-gate on readiness stability, not one successful check.

## Common mistakes

- Equating "process started" with "ready."
- Making liveness depend on every downstream.
- Disabling the probe.
- Making readiness always return 200.
- Raising thresholds without finding whether path/port is wrong.
- Ignoring prior container logs.

## Concise interview-ready answer

I identify whether startup, readiness, or liveness fails and classify the exact response. Then I verify probe path, port, scheme, listener, timing, events, prior logs, resources, and health-component details. Starting does not mean ready. I keep liveness local, use startup for long initialization, and readiness for essential serving capability. I correct semantics or measured timing, preserve old capacity, and add probe stability and endpoint alerts.

---

# 8. A deployment causes a sudden increase in 5xx errors. How would you investigate?

## Meaning and invariants/risks

The key questions are which layer emitted the 5xx, which requests and versions are affected, and whether retries or rollout behavior amplify it. Aggregate 5xx can hide a 100 percent canary failure. Error bodies may be rewritten by proxies.

## Failure locations

- CDN/WAF/load balancer rejects or cannot reach ingress.
- Ingress has zero ready endpoints, connect reset, timeout, or protocol mismatch.
- Service mesh denies mTLS or resets during drain.
- New app throws, rejects overload, or times out on dependency.
- Database/schema/config/secret failure becomes app 500/503.
- Old pods terminate active requests.
- New and old versions disagree on API/event/cache/session format.
- Client retries multiply traffic and exhaust pools.

## Detailed causal mechanisms with plain-language example

During rolling replacement, new pods pass shallow readiness before their connection pools are warm. They receive traffic, calls time out, ingress reports 504, clients retry, and both new and old pools saturate. The visible 5xx spike is partly gateway-generated and partly amplified by retry load.

Another rollout sends `SIGTERM` and immediately closes sockets while endpoints still propagate. Ingress sees resets from terminating old pods, so errors correlate with deployment even if new code is correct.

## Ordered investigation/design reasoning and why each step matters

1. Pause rollout and quantify impact by status, route, version, instance, region, and minute.
2. Identify the response generator using headers, ingress upstream status, access logs, app request counts, and traces.
3. Compare new versus old error rates using denominators and equivalent cohorts.
4. Inspect endpoint count/churn, readiness, terminations, restarts, drain timing, and capacity.
5. For requests reaching the app, group exceptions and failing dependency spans rather than reading random logs.
6. Check config/secret/identity/schema/flag and resource differences.
7. Inspect retry rate, queue depth, pool wait, and downstream saturation for amplification.
8. Choose a compatible mitigation, then verify from client through business result.

## Evidence, state transitions, tools, and result interpretation

```text
request ID
client status=503
ingress upstream_status="-", endpoint_count=0
application request absent
=> routing/readiness generated failure before application
```

```text
request ID
ingress upstream_status=500
application status=500, digest B
trace: database span auth failure
=> application generated 500 because new runtime DB identity failed
```

Inspect request-rate/error-rate/duration by version, ingress status/upstream status, EndpointSlices, rollout events, termination logs, traces, exception fingerprints, dependency/pool metrics, and retry headers/counts.

## Immediate mitigation/recovery

Pause/abort canary, restore ready capacity, shift traffic, disable the feature, or correct config. Drain safely. Apply load shedding and retry budgets if amplification threatens recovery. Roll back only if schema/messages/side effects remain compatible; otherwise roll forward.

## Permanent correction/design

Add version-aware SLO gates, realistic readiness, warm-up, graceful drain, compatibility tests, retry budgets, dependency bulkheads, and layer-specific observability. Preserve request IDs across proxies and log upstream status.

## Prevention and alerts

Alert separately on ingress-generated and app-generated 5xx, new-versus-control regression, zero endpoints, connect resets during termination, retry amplification, and business failure. Abort promotion automatically on sustained thresholds with minimum sample size.

## Common mistakes

- Assuming every 5xx is an application 500.
- Looking at counts without request denominators.
- Averaging canary into fleet metrics.
- Retrying 5xx at every layer.
- Rolling back without checking database compatibility.
- Ignoring termination/drain evidence.

## Concise interview-ready answer

I pause the rollout, split 5xx by status, route, version, instance, and region, and use request IDs, ingress upstream status, app request counts, and traces to identify the generating layer. I compare canary with control, then inspect endpoints, probes, restarts, drain, config, identity, schema, dependencies, resources, and retry amplification. I restore healthy traffic with a compatible flag, traffic shift, rollback, or roll-forward and verify business outcomes.

---

# 5. Cross-cutting production operation

## 5.1 Canary decision model

A canary gate needs:

- A healthy control serving comparable requests.
- Minimum request count and observation time.
- Route and business-flow coverage.
- Error, latency, saturation, and correctness thresholds.
- Automatic pause/abort behavior.
- Human-readable evidence.

```text
new error rate - control error rate
new p99 / control p99
new business-success rate / control business-success rate
```

Do not promote merely because pods are Ready. Do not fail a canary on one statistically meaningless event without considering severity; one data-corruption event can be sufficient, while one expected client cancellation is not.

## 5.2 Blue-green switch checklist

Before switching:

- Green uses the intended digest, config, secrets, identity, and schema.
- Smoke and business checks pass through real routing.
- Consumers/jobs are not duplicated unexpectedly.
- Database writes are readable by blue if rollback is planned.
- Capacity is warm.
- DNS/load-balancer/session behavior is understood.

After switching:

- Watch connection resets, old long-lived traffic, queue ownership, error/latency/business metrics.
- Keep blue isolated but available for the documented rollback window.
- Do not let both colors execute singleton jobs without coordination.

## 5.3 Rolling update capacity and drain

Example with 10 replicas:

```text
maxUnavailable = 1
maxSurge = 2
```

At most one desired replica may be unavailable and up to two extra pods may exist, but actual serving capacity can still fall if readiness is false, startup is slow, old pods drain long requests, or resource quotas block surge. Validate pod disruption budgets, cluster capacity, and dependency connection limits.

For long requests:

```text
termination grace
  >= endpoint propagation allowance
   + maximum accepted in-flight drain time
   + shutdown overhead
```

This is a design inequality, not a reason to permit unbounded requests. Enforce deadlines and safe cancellation.

## 5.4 Secret and certificate rotation

A robust credential rotation:

1. Creates a new credential with bounded overlap.
2. Grants least-required access.
3. Deploys consumers that can use/reload it.
4. Verifies new connections, not just old pools.
5. Revokes the old credential after full convergence.
6. Audits versions and access.

For certificates, monitor expiry well before the deployment window, validate the complete chain and SAN, and test actual process reload behavior. A file changing on disk does not prove a TLS client/server reloaded it.

## 5.5 Migration release plan

```text
Release A: add new nullable field and compatible index
Release B: code reads old/new, writes both if required
Job:      resumable backfill with progress and load limits
Release C: switch read path after validation
Release D: stop old writes
Release E: remove old field after rollback window and all old consumers
```

For each phase, write:

- Forward and rollback compatibility matrix.
- Lock/load estimate.
- Validation query or business metric.
- Ownership and abort threshold.
- Recovery for partial completion.

## 5.6 Safe evidence retention

During an incident, retain:

- Artifact digest and provenance.
- Deployment/config revision and sanitized diff.
- Pod status/events and prior logs.
- Request/trace IDs and exception fingerprints.
- Metrics snapshots by version.
- Migration and flag audit history.
- Timeline of mitigations and observed results.

Do not retain unrestricted environment dumps, Secret objects, tokens, private keys, or customer payloads.

---

# 6. Decision trees

## 6.1 Immediate post-deployment failure

```text
Errors begin with rollout
|
+-- Pause rollout and retain old capacity/evidence
|
+-- Do failing requests reach the application?
|   |
|   +-- No
|   |   -> ingress status, endpoints, readiness, selector, port,
|   |      mesh/network policy, drain
|   |
|   `-- Yes
|       -> exception/trace
|           +-- dependency failure -> DNS/TCP/TLS/auth/pool/schema
|           +-- code path -> version/flag/payload compatibility
|           `-- saturation -> resources, queues, retries, warm-up
|
+-- Does failure follow new digest/revision?
|   +-- Yes -> diff old/new effective inputs
|   `-- No  -> inspect shared dependency/traffic/infrastructure
|
`-- Is rollback compatible?
    +-- Yes -> controlled rollback/traffic shift
    `-- No  -> flag/config mitigation or roll-forward
```

## 6.2 Database connection failure

```text
Cannot connect
|
+-- DNS resolves?
|   `-- No -> name, namespace, resolver, cache
|
+-- TCP opens?
|   +-- timeout -> route/policy/firewall
|   `-- refused -> host/port/listener
|
+-- TLS succeeds?
|   `-- No -> chain, SAN, validity, trust, client identity, clock
|
+-- Authentication succeeds?
|   `-- No -> Secret version/format, account, workload identity
|
+-- Authorization succeeds?
|   `-- No -> role/grants/database
|
`-- Query succeeds?
    `-- No -> schema, migration, driver, lock, SQL
```

## 6.3 Health failure

```text
Probe fails
|
+-- startup -> initialization complete within measured policy?
+-- readiness -> can safely receive new work?
`-- liveness -> is restart actually useful?

For each:
target path/port/scheme -> response/error -> resource delay
-> dependency semantics -> correct probe or application behavior
```

## 6.4 Rollback or roll-forward

```text
Known bad release
|
+-- Can previous code read current schema/data/messages?
|   +-- No -> roll-forward or compatibility bridge
|   `-- Yes
|
+-- Have irreversible external effects occurred?
|   +-- Yes -> contain and roll-forward/compensate by business policy
|   `-- No
|
+-- Is failure isolated behind a safe flag/config?
|   +-- Yes -> disable/correct, verify
|   `-- No
|
`-- Is prior immutable release known good and capacity available?
    +-- Yes -> controlled rollback/traffic shift
    `-- No  -> narrow roll-forward
```

---

# 7. Useful extra interview questions

## 7.1 Why can a rollback fail even when the old image was healthy?

The world changed after deployment: schema contracted, new rows use a new format, messages use new semantics, Secrets rotated, or external side effects occurred. The old binary may no longer be compatible. Rollback is a tested multi-dimensional operation, not simply selecting an older tag.

## 7.2 Should readiness check the database?

Only if the instance cannot serve any useful/safe traffic without it and fleet-wide removal will not worsen the outage. Often a bounded local capability plus circuit breaking/load shedding is safer. Optional dependencies belong in detailed health/metrics, not liveness. The answer depends on the service contract.

## 7.3 How do you handle database migrations with multiple replicas?

Prefer a dedicated controlled migration job or a robust migration lock with one owner, explicit timeout, observability, and idempotent/resumable scripts. Avoid every pod racing migrations during startup. Application readiness should reflect whether its required compatible schema exists.

## 7.4 What makes a good post-deployment smoke test?

It runs through real DNS, ingress, identity, and discovery; exercises a safe representative read and controlled write; verifies downstream/business outcome; records version; and cleans up safely. It is not only `GET /health` or localhost.

## 7.5 How do you detect mutable image tags?

Compare pod `imageID` digests for the same declared tag, record registry digest at promotion, enforce digest pinning/admission policy, and retain build provenance. Different digests under one release label are drift.

## 7.6 What is a readiness race?

The pod becomes Ready before essential initialization, routes, cache, connection pools, or sidecars can serve. Traffic then fails during the gap. Make readiness represent actual serving capability and require consecutive stability or warm-up evidence where useful.

## 7.7 How do retries affect deployment incidents?

Retries increase load exactly when capacity may be reduced by rollout. Multiple layers can multiply attempts, saturate pools, and obscure the original error. Use one retry owner, idempotency, bounded attempts, backoff/jitter, deadlines, retry budgets, and load shedding.

## 7.8 How do you compare configuration without exposing secrets?

Expose approved key names, source identity, load timestamp, type, and a keyed or policy-approved fingerprint. Compare Secret object/version and certificate metadata. Restrict diagnostics and never log raw secret values or hashes vulnerable to guessing for low-entropy secrets.

## 7.9 What should happen when SIGTERM arrives?

The process should become unready, stop accepting/claiming new work, drain or checkpoint in-flight work within its deadline, close resources, and exit successfully. The platform's endpoint propagation and termination grace must support that behavior.

## 7.10 How do you test mixed versions?

Run old callers against new servers and new callers against old servers, consume old/new messages, share caches/sessions as production does, execute against expanded/partially backfilled schema, and test rollback after new writes. Include concurrent rolling behavior, not only isolated versions.

---

# 8. Cheat sheets

## 8.1 First 15 minutes

```text
[ ] Pause promotion; retain healthy old capacity
[ ] Record start, impact, status, route, region, version
[ ] Capture image digest, revision, config fingerprint, flag, schema
[ ] Attribute 5xx layer
[ ] Check pod state, events, probes, endpoints, restarts, drain
[ ] Compare one healthy old and failing new instance
[ ] Trace one representative request
[ ] Check dependency phase and retry amplification
[ ] Confirm rollback compatibility before acting
[ ] Mitigate, then verify technical and business outcome
```

## 8.2 Old/new diff checklist

```text
artifact digest and build provenance
base image/runtime/architecture
command, arguments, ports, profiles
deployment template and ReplicaSet
effective config source/fingerprint/load time
Secret/certificate version and metadata
service account/RBAC/cloud identity
node, zone, network policy, mesh
requests/limits and pool sizes
startup/readiness/liveness
feature variants
schema/migration/backfill state
API/event/cache/session compatibility
traffic cohort and cache warmth
```

## 8.3 Probe cheat sheet

| Probe | Question | Failure effect | Keep out |
|---|---|---|---|
| Startup | Has initialization completed? | Suppresses other probes; may restart after threshold | Optional remote health |
| Readiness | Can this instance safely take new work? | Removes from ready endpoints | Irrelevant optional dependencies |
| Liveness | Is local progress irrecoverably lost? | Restarts container | Database/broker availability by default |

## 8.4 5xx attribution cheat sheet

| Evidence | Likely layer |
|---|---|
| CDN/WAF log has rejection; ingress has no request | Edge |
| Ingress 503, no upstream selected, zero endpoints | Discovery/readiness |
| Ingress 502 with connect reset during termination | Drain/process/proxy |
| Ingress 504, app trace continues past deadline | App/downstream latency and cancellation |
| App logs same request ID and emits 500 | Application |
| App emits 503 due to pool/load shed | Application capacity/dependency |
| No app record but sidecar denies mTLS | Mesh/identity |

## 8.5 Safe configuration checklist

```text
[ ] Schema, type, unit, range, and required keys validated
[ ] Precedence documented and tested
[ ] Environment/profile explicit
[ ] Config revision and safe fingerprint observable
[ ] Secret values never logged
[ ] Update/reload semantics known
[ ] Config change triggers rollout when startup-only
[ ] Fleet drift detected
[ ] Change audit and rollback value retained securely
[ ] Feature flags have owner, default, metric, and expiry
```

## 8.6 Release readiness checklist

```text
[ ] Build once; promote digest
[ ] Provenance/signature/SBOM policy passes
[ ] Production-like manifest renders and validates
[ ] Secret/certificate identity and expiry valid
[ ] Expand migration complete and compatible
[ ] Old/new API, event, cache, session tests pass
[ ] Probes reflect semantics and measured timing
[ ] Graceful drain fits termination policy
[ ] Canary has control, sample, thresholds, and abort
[ ] Smoke test uses realistic route and identity
[ ] Rollback and roll-forward plans are compatible
[ ] Dashboards split by revision and business outcome
```

## 8.7 One-minute interview framework

```text
Protect -> scope -> preserve -> attribute -> compare -> trace
-> choose compatible mitigation -> verify -> prevent
```

Say what each step proves. Mention immutable digests, effective configuration, lifecycle/endpoints, mixed-version schema compatibility, and the response-generating layer. End with safe mitigation and a permanent release gate.

## Final principle

A deployment incident becomes manageable when releases are immutable, effective state is observable, old and new can be compared, health reflects real lifecycle semantics, contracts tolerate mixed versions, and rollback is treated as a compatibility decision rather than a reflex.
