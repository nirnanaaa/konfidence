# Domain model and name registry

This page is the only definition of the names other docs depend on (D-API-06):
every CRD and its fields, every status enum and reason, the Postgres tables
and the `LISTEN/NOTIFY` channel. Topic docs link here and carry no YAML of
their own. A name not on this page does not exist yet; add it here first.
River job kinds belong to [01](01-control-plane.md), proto names to
[08](08-api-and-ui.md).

CRD enum values are PascalCase; proto enums map 1:1 in the prefixed
UPPER_SNAKE form. Fields marked `M2+` or `M3` ship after the acceptance test
(D-PROD-04); everything else is M1.

## Two API groups, one direction of truth

Control-plane CRDs hold desired state and rollout position. Postgres holds
history, audit, sessions, sequences, and the River queue (D-TOPO-04,
D-TOPO-07). The control plane never reads bridge CRDs; it learns their state
from signed reports only (D-ATT-01). Both groups start at `v1alpha1`.

| Group | Kind | Scope | Writer |
| --- | --- | --- | --- |
| `konfidence.cloud` | `Project`, `Landscape` | cluster | API; status by controllers |
| `konfidence.cloud` | `LandscapeGrant`, `GitCredential` | project namespace | API |
| `konfidence.cloud` | `Application` | project namespace | API; status by controllers and workers |
| `konfidence.cloud` | `Release` | project namespace | release controller; status by workers |
| `konfidence.cloud` | `PromotionConfig` | project namespace | API; status by controller |
| `konfidence.cloud` | `Promotion` | project namespace | config controller or API; gates by gate writers |
| `konfidence.cloud` | `Deployment` | project namespace | promotion controller; status by controllers, workers, API |
| `landscape.konfidence.cloud` | `Application` | cluster | bridge only |
| `landscape.konfidence.cloud` | `ApplicationPolicy`, `Boundary` | cluster | landscape operator |

## Identities and counters have one owner each

| Name | Form | Owner |
| --- | --- | --- |
| landscape ID | `lsc-<random>` in `Landscape.spec.landscapeID`, immutable | API at creation |
| grant ID | `lgr-<random>` in `LandscapeGrant.spec.grantID`, immutable | API at creation (D-TEN-02) |
| release ID | `sha256:…` over `konfidence.cloud/release-id/v1` lines | resolve job (D-REL-01) |
| release label | `<application>@<sequence>` | release controller (D-REL-02) |
| bundle component | `konfidence.cloud/p/<project>/<application>`, version `<sequence>.0.0` | build (D-BUN-01) |
| delivery component | `konfidence.cloud/l/<landscape ID>/delivery`, version `<deliverySequence>.0.0` | delivery job (D-BUN-04) |
| policy version | SHA-256 of the canonical effective policy of one application | bridge (D-POL-06) |
| policy ref | `<policy name>@<metadata.generation>`, display only | bridge |
| `setBy` | session subject, or `<issuer>#<sub>` for machines (M2+, D-API-04) | API |
| release sequence | `Application.status.nextReleaseSequence` | release controller |
| promotion sequence | `Application.status.nextPromotionSequence`, copied to `Promotion.spec.sequence` and `Deployment.spec.promotionSequence` | config controller or API |
| delivery sequence (lseq) | `landscape_sequences.delivery_sequence`; CRD `deliverySequence` fields only mirror it (D-ATT-02) | delivery job |
| report sequence | in every report; last accepted in `landscape_sequences.report_sequence` | bridge |

Ordering, superseding and the dispatch fence use the promotion sequence,
never the release sequence. Names: `Release` `<application>-<sequence>`,
`Promotion` `<promotionConfig>-<promotion sequence>`, `Deployment`
`<promotion>-<landscape>`, `LandscapeGrant` `<landscape>`, bridge
`Application` `<project>.<application>`.

Only the bridge sets `konfidence.cloud/project`, `konfidence.cloud/application`
and `konfidence.cloud/landscape-grant` (grant ID) on landscape objects
(D-POL-04).

## Project owns one namespace and its role bindings

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: Project
metadata: { name: payments }
spec:
  namespace: kden-project-payments          # immutable, defaulted
  roleBindings: { project-admin: [{ session: { memberOf: [payments-admins] } }] }   # viewer | developer | release-manager | project-admin (08)
  trustRules: []                            # M2+, D-API-04
status: { conditions: [Ready] }
```

A finalizer deletes the namespace. Deletion is refused while a project
`Deployment` is a landscape's current desired entry.

## Landscape records one cluster with one bridge

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: Landscape
metadata: { name: de-fra-prod-1, labels: { env: prod, ring: canary } }
spec:
  landscapeID: lsc-7f3a9c                   # minted, immutable
  mode: Attached                            # Attached | Isolated
  trust: { mode: Pinned, caSecretRef: { name: acme-ca } }   # Pinned | CustomerCA (M3)
  roleBindings: { landscape-admin: [{ session: { memberOf: [fra-cluster-ops] } }] }   # landscape-viewer | landscape-operator | landscape-admin (08)
  staleAfter: 30m                           # LIVE vs stale (08)
  linkTemplates:                            # the only deep-link source (D-UI-03)
    - { group: vector.example, kind: Stage, url: "https://runtime.fra/stages/{namespace}/{name}" }
  trustRules: []                            # M2+, D-API-04
status:
  phase: Enrolled                           # Pending | Unconfirmed | Enrolled
  bridge: { version: 1.4.2, capabilities: [transfer.full, tombstone, attestation.v1], bundleFormats: [1], keyFingerprint: SHA256:4f… }
  deliverySequence: 118                     # mirror of landscape_sequences
  connection: { state: Connected, since: 2026-10-29T08:12:00Z }   # Connected | Disconnected
  conditions: [Connected, BridgeOutdated, BridgeStateRegressed, FluxReady, FluxLockdown, FluxManaged]
```

`Unconfirmed` lasts until a `landscape-admin` enters the enrollment code or
skips with a reason; nothing is dispatched or imported before `Enrolled`
(D-TRUST-02). Heartbeats stay in `bridge_connections`.

## LandscapeGrant is a project's approved access to one landscape

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: LandscapeGrant
metadata: { name: de-fra-prod-1, namespace: kden-project-payments }
spec:
  landscapeRef: de-fra-prod-1
  grantID: lgr-01hx3k9q                     # minted, immutable (CEL self == oldSelf)
  request: { namespaces: [payments, payments-jobs], clusterScoped: [{ group: apiextensions.k8s.io, kind: CustomResourceDefinition }],
             namespaced: [{ group: monitoring.coreos.com, kind: ServiceMonitor }] }
status:
  approval: { state: Approved, setBy: alice@acme, setAt: 2026-10-20T10:00:00Z }   # Requested | Approved | Revoked
  boundary: { state: WaitingForBoundaryApproval, pendingKinds: [CustomResourceDefinition] }   # Unknown | WaitingForBoundaryApproval | Ready
```

One grant exists per project and landscape. Promotions target only
landscapes with an `Approved` grant.

## GitCredential points at a Secret and scopes its use

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: GitCredential
metadata: { name: github-pat, namespace: kden-project-payments }
spec:
  type: GitHubPAT                           # GitHubPAT | SSHDeployKey (M2+) | GitHubApp (M2+)
  secretRef: { name: github-pat }           # same namespace; key material never in Postgres
  knownHosts: "github.com ssh-ed25519 AAAA…"   # required for SSHDeployKey (D-SRC-04)
  githubApp: { appID: 123456, installationID: 7890 }   # required for GitHubApp (M2+)
  grants: [{ project: payments, repositories: ["acme/payments-*"] }]   # required, owner included
status: { expiresAt: 2027-01-31T00:00:00Z, conditions: [Valid, ExpiringSoon] }
```

## Application declares sources and chart layout

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: Application
metadata: { name: payments-api, namespace: kden-project-payments }
spec:
  sources:                                  # one in M1; builders M2+, builder/v1 contract in 07
    - { name: app, url: https://github.com/acme/payments-api, ref: { branch: main },   # { branch } | { tag }
        credentialRef: { name: github-pat }, builders: [] }   # add project: for a granted credential
  charts:                                   # crds: Skip | Create | CreateReplace
    - { name: api, source: app, path: deploy/chart, valuesFiles: [deploy/values.yaml], dependsOn: [],
        crds: CreateReplace, overridable: [/replicaCount, /ingress/host] }
  secretRequirements: [{ name: db, keys: [username, password] }]
  pollInterval: 5m
status:
  sources: [{ name: app, commit: 9c1e…, observedAt: 2026-10-29T08:00:00Z }]
  resolved: { releaseID: sha256:e3b0…, at: 2026-10-29T08:00:30Z }
  latestRelease: { name: payments-api-42, sequence: 42 }
  nextReleaseSequence: 43
  nextPromotionSequence: 18
  conditions: [SourcesReady, CredentialsValid]
```

The resolve job writes `status.resolved`. The controller creates a `Release`
only for a release ID the application does not have yet (D-REL-01).
`pollInterval` and `credentialRef` are outside the release ID.

## Release is an immutable snapshot of deployable inputs

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: Release
metadata: { name: payments-api-42, namespace: kden-project-payments, labels: { konfidence.cloud/application: payments-api } }
spec:                                       # immutable (CEL)
  application: payments-api
  sequence: 42
  releaseID: sha256:e3b0…
  sources: [{ name: app, url: https://github.com/acme/payments-api, sha: 9c1e… }]
  dependencies: [{ chart: api, name: postgresql, version: 15.5.0, digest: sha256:aa12… }]
  layout: { … }                             # copy of charts, secretRequirements, builders
status:
  phase: Ready                              # Building | Ready | Failed
  component: { name: konfidence.cloud/p/payments/payments-api, version: 42.0.0, digest: sha256:5d41… }
  conditions: [Built, Signed]
```

## PromotionConfig declares source, targets, waves and gates

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: PromotionConfig
metadata: { name: prod, namespace: kden-project-payments }
spec:
  application: payments-api
  source: { application: payments-api }    # { application } | { promotionConfig } (M2+); omitted = API only
  mode: Auto                                # Auto | Manual
  targets:                                  # landscapes, or selector (M2+), plus values
    - { landscapes: [de-fra-prod-1, de-fra-prod-2], values: { ingress: { host: pay.acme.example } } }
  gates: [{ type: explicit-approval.konfidence.cloud, requiresReporting: Health }]   # Health | Resources
  waves:                                    # one wave in M1; selector, gates M2+; ackTimeout attached only
    - { name: canary, selector: { matchLabels: { ring: canary } }, maxConcurrent: 1, failureThreshold: 0,
        completion: Attested, timeout: 30m, ackTimeout: 15m, gates: [{ type: soak.konfidence.cloud }] }
status:
  lastSucceeded: { release: payments-api-41, promotion: prod-16 }
  conditions: [Valid]
```

`maxConcurrent` and `failureThreshold` take an integer or a percentage;
`completion` is `Attested | Dispatched`. A reporting level too low for
`completion` or `requiresReporting` is a validation error (D-POL-05).

## Promotion is a frozen rollout of one release

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: Promotion
metadata: { name: prod-17, namespace: kden-project-payments }
spec:                                       # snapshot at creation, immutable
  promotionConfig: prod
  sequence: 17                              # promotion sequence
  operation: Deploy                         # Deploy | Remove (M2+)
  release: payments-api-42                  # empty for Remove
  gates: [{ type: explicit-approval.konfidence.cloud }]
  targets: [{ landscapes: [de-fra-prod-1, de-fra-prod-2], valuesDigest: sha256:91f2…, values: { … } }]
  waves: [{ name: canary, landscapes: [de-fra-prod-1], maxConcurrent: 1, failureThreshold: 0, completion: Attested, timeout: 30m, ackTimeout: 15m }]
status:
  state: InProgress
  gates:                                    # scope: promotion | wave/<name>
    - { scope: promotion, type: explicit-approval.konfidence.cloud, state: Satisfied, reason: "", setBy: bob@acme, setAt: 2026-10-29T09:00:00Z }
  rejection: { setBy: "", setAt: null, reason: "" }   # ADR-0040 Reject, terminal
  waves: [{ name: canary, state: Rolling, total: 1, completed: 0, failed: 0, unreachable: 0 }]
```

Values are stored once per target entry, not per landscape.

## Deployment is one promotion on one landscape

```yaml
apiVersion: konfidence.cloud/v1alpha1
kind: Deployment
metadata: { name: prod-17-de-fra-prod-1, namespace: kden-project-payments,
  labels: { konfidence.cloud/application: payments-api, konfidence.cloud/landscape: de-fra-prod-1 } }
spec:
  { promotion: prod-17, promotionSequence: 17, operation: Deploy, release: payments-api-42,
    componentDigest: sha256:5d41…, landscape: de-fra-prod-1, grantID: lgr-01hx3k9q, wave: canary,
    valuesDigest: sha256:91f2… }
status:
  phase: Attested
  reason: Applied
  message: ""
  delivery: { sequence: 118, via: Stream, dispatchedAt: 2026-10-29T09:02:00Z }   # first delivery listing it
  attestation:                              # patched only on a decision-relevant change; at = last change
    { deliverySequence: 118, componentDigest: sha256:5d41…, effectiveValuesDigest: sha256:0a9b…,
      policyVersion: sha256:be11…, policyRefs: [default@3, payments-api@7], reason: Applied,
      health: Healthy, at: 2026-10-29T09:05:11Z }
  conditions: [Dispatched, Attested]
```

Report matching follows D-ATT-05 ([05](05-promotion-and-rollout.md)). UI
freshness comes from `attestations`, never from this CR.

## Bridge Application is the installed form of one application

```yaml
apiVersion: landscape.konfidence.cloud/v1alpha1
kind: Application
metadata:
  name: payments.payments-api
  labels: { konfidence.cloud/project: payments, konfidence.cloud/application: payments-api, konfidence.cloud/landscape-grant: lgr-01hx3k9q }
spec:
  release: { id: sha256:e3b0…, label: payments-api@42, componentDigest: sha256:5d41… }
  deliverySequence: 118
  charts: [{ name: api, artifactDigest: sha256:ab01…, dependsOn: [], crds: CreateReplace }]
  secretRequirements: [{ name: db, keys: [username, password] }]
  overridable: { api: [/replicaCount, /ingress/host] }
status:
  conditions: [{ type: Ready, status: "True", reason: Applied }]
  health: Healthy
  policyVersion: sha256:be11…
  policyRefs: [default@3, payments-api@7]
  boundary: payments-fra
  targetNamespace: payments
  effectiveValuesDigest: sha256:0a9b…
  charts: [{ name: api, helmRelease: payments/payments-api-api, ready: true }]
  pendingUpdate: { componentDigest: sha256:77ee… }     # updates: Manual only (D-POL-07)
  acceptedUpdate: { componentDigest: "", tombstoneSequence: 0, acceptedBy: "", acceptedAt: null }
```

State is the `Ready` condition plus one reason; there is no `phase`. `accept`
writes `status.acceptedUpdate` through the bridge admin endpoint (D-LAND-06).

## ApplicationPolicy is the operator's authoritative say

```yaml
apiVersion: landscape.konfidence.cloud/v1alpha1
kind: ApplicationPolicy
metadata: { name: payments-api }            # the policy named "default" is the landscape default
spec:
  boundaryRef: payments-fra                 # ignored for "default"
  selector: { matchLabels: { konfidence.cloud/application: payments-api } }
  action: Allow                             # Allow | Deny
  updates: Auto                             # Auto | Manual (D-POL-07)
  targetNamespace: payments
  values: { api: { replicaCount: 4 } }
  secretBindings:
    - { requirement: db, secretStoreRef: { name: vault, kind: SecretStore }, remoteKey: payments/db }   # M2+, ESO
    - { requirement: cache, secretRef: { name: cache-creds } }
  images: { transfer: Full, registries: [registry.fra.acme.internal], requireSignatures: true }   # Full | Thin (M2+)
  reporting: Health                         # None | Health | Resources
status: { matchedApplications: [payments.payments-api], conditions: [Valid] }
```

M1 ships the `default` policy only. Per-application policies are M2+.

## Boundary is the tenant fence a landscape admin draws

```yaml
apiVersion: landscape.konfidence.cloud/v1alpha1
kind: Boundary
metadata: { name: payments-fra }
spec:
  grant: lgr-01hx3k9q                       # exactly one grant ID (D-TEN-02)
  namespaces: { allowed: [payments, payments-jobs], template: { labels: { team: payments } } }
  quota: { hard: { requests.cpu: "40", requests.memory: 80Gi } }
  approvedClusterScoped:                    # CRDs, cluster kinds, webhooks
    - { group: apiextensions.k8s.io, kind: CustomResourceDefinition, names: [ledgers.payments.acme] }
  approvedNamespaced: [{ group: monitoring.coreos.com, kind: ServiceMonitor }]   # beyond the base set (D-TEN-03)
status:                                     # serviceAccount = HelmRelease serviceAccountName
  { namespaces: [payments, payments-jobs], serviceAccount: konfidence-payments,
    pendingClusterScoped: [], blockingApplications: [], conditions: [Ready] }
```

In M1 the operator writes the `Boundary` by hand; automating namespaces, Role
and quota is M2+. The finalizer holds while `blockingApplications` is set.

## Reasons form one vocabulary for condition, report and Deployment

The bridge `Ready` condition, the report entry `reason` and
`Deployment.status.reason` all use this table. Adding a reason is additive;
renaming one is breaking.

| Reason | Set by | Deployment phase |
| --- | --- | --- |
| `Applied` | bridge | `Attested` when `Healthy` |
| `Progressing`, `Removing` | bridge | unchanged |
| `Removed` | bridge | `Attested` for `Remove` |
| `Failed` | bridge, Flux gave up | `Failed` |
| `Refused` | bridge | `Refused` from `Dispatched` or `Blocked` |
| `WaitingForBoundaryApproval`, `AmbiguousPolicy`, `UnboundSecret`, `ValuesRejected`, `ImageRuleViolation`, `OutsideBoundary`, `RegistryNotConfigured`, `AwaitingLocalAcceptance`, `FluxUnavailable` | bridge | `Blocked` |
| `BridgeTooOldForBundleFormat` | control plane | `Blocked` from `Dispatching`; back to `Dispatching` after a bridge upgrade |
| `ReportingLevelLowered` | control plane | `Failed` from any non-terminal phase; affected gates go `Error` |
| `Exported` | control plane | `Dispatched` (isolated) |
| `AckTimeout` | control plane | `Unreachable` |
| `WaveTimeout` | control plane | `Failed` from `Dispatching`, `Dispatched` or `Blocked` |
| `NewerPromotion` | control plane | `Superseded` from any non-terminal phase |
| `FencedByHistory` | control plane | `Superseded` by the dispatch fence |

Report import rejections are API errors, not CR reasons:
`EnrollmentUnconfirmed`, `KeyRevoked`, `SignatureInvalid`,
`LandscapeMismatch` and `ReportSequenceRegressed`. A report whose
(landscape ID, report sequence) is in `attestations` with the same
`payload_digest` is an idempotent success. Any other report at or below the
last accepted sequence is regressed and sets `BridgeStateRegressed` (D-ATT-03).

## Every other enum in one table

| Field | Values |
| --- | --- |
| `Deployment.status.phase` | `Pending`, `Dispatching`, `Dispatched`, `Blocked`; terminal: `Attested`, `Failed`, `Refused`, `Unreachable`, `Superseded`. [05](05-promotion-and-rollout.md) owns transitions |
| `health` (report, bridge `Application`, `Deployment`) | `Healthy`, `Progressing`, `Degraded`, `Unknown` |
| `Landscape.status.phase` | `Pending`, `Unconfirmed`, `Enrolled` |
| `Release.status.phase` | `Building`, `Ready`, `Failed` |
| `Promotion.status.state` | `Waiting`, `Ready`, `InProgress`, `Succeeded`, `Failed`, `Rejected`, `Superseded` |
| wave `state` | `Waiting`, `Rolling`, `Complete`, `Failed` |
| gate `state` | `Pending`, `Satisfied`, `Error` |
| `bridge_keys.state` | `Unconfirmed`, `Pinned`, `Retired`, `Revoked` |
| `via` | `Stream`, `CTF` in `dispatches`; `Stream`, `File` in `attestations` |
| `audit_log.actor_kind` | `Session`, `Workload`, `System` |
| `audit_log.outcome` | `Pending`, `Succeeded`, `Failed` |

## Postgres holds history, counters and dispatch records

History is append-only. Columns shown are the ones docs rely on.

| Table | Purpose | Main columns |
| --- | --- | --- |
| `landscape_sequences` | sole owner of both per-landscape counters | `landscape_id` PK, `delivery_sequence`, `report_sequence` |
| `dispatches` | one row per signed delivery | `landscape_id, delivery_sequence, delivery_digest, via, created_at, acked_at`; PK `(landscape_id, delivery_sequence)` |
| `dispatch_entries` | `Deployment`s a delivery covers; fence input | `landscape_id, delivery_sequence, project, application, deployment_uid, promotion_sequence, operation, component_digest, values_digest, grant_id` |
| `releases` | every release, for diffs (D-REL-03) | `project, application, sequence, release_id, component_digest, sources, layout, created_at` |
| `deployment_history` | every `Deployment` change | `deployment_uid, project, application, landscape_id, promotion, promotion_sequence, wave, operation, release_id, component_digest, values_digest, policy_version, phase, reason, delivery_sequence, at` |
| `promotion_history` | promotion state and gate transitions | `project, application, promotion, promotion_sequence, release_id, state, scope, gate_type, set_by, reason, at` |
| `attestations` | every accepted signed report | `landscape_id, report_sequence, delivery_sequence, key_fingerprint, via, received_at, payload, payload_digest`; unique `(landscape_id, report_sequence)` |
| `audit_log` | every mutating API call and override | `id, at, actor, actor_kind, project, action, object_ref, reason, request_digest, outcome`; the CR write carries `id` in annotation `konfidence.cloud/audit-id` |
| `sessions` | OIDC sessions (D-API-03) | `id_hash, subject, issuer, groups, created_at, expires_at, last_seen_at` |
| `enrollment_tokens` | one-time bridge enrollment | `token_hash, landscape_id, created_by, expires_at, used_at` |
| `bridge_keys` | bridge keys and deny list (D-TRUST-05) | `landscape_id, fingerprint, public_key, state, predecessor, confirmed_by, skip_reason, created_at, revoked_at` |
| `bridge_connections` | stream owner and heartbeats | `landscape_id, epoch, api_replica, bridge_version, capabilities, connected_at, last_heartbeat_at` |
| `blobs` | content-addressed blob index | `digest, size, media_type, storage_ref, created_at, last_referenced_at` |
| `component_blobs` | blobs a component version needs | `component_digest, blob_digest` |
| `river_*` | River queue, created by `rivermigrate` | River's own |

No table stores a bridge's blob inventory or transfer progress. The inventory
arrives in full with every `PlanTransfer` and state request file, and
transfers resume by offset.

## Dispatch tables carry the sequence, the fence and the channel

`delivery_sequence` is allocated by `UPDATE landscape_sequences … RETURNING`
in its own committed transaction; gaps are allowed. The transaction that
writes `dispatches` and `dispatch_entries` also sends `NOTIFY` and is the
dispatch fence (D-TOPO-07): for a `deployment_uid` not yet in the
landscape's `dispatch_entries` it refuses a promotion sequence not above the
last one for (application, landscape), or a promotion terminal in
`promotion_history`; listed entries pass. An ack or export of
delivery N moves exactly the `Deployment`s in its `dispatch_entries` to
`Dispatched` (D-BUN-04). After a Postgres restore, `delivery_sequence`
restarts 1000 above the highest known value (D-ATT-04). 01 owns the job.

The one `LISTEN/NOTIFY` channel is `konfidence_events`, payload
`<kind>:<landscape ID>`, kinds `dispatch`, `presence`, `revoke` and
`attested`. 01 owns the listener rules.

## Retention keeps etcd at the sized budget

A maintenance job prunes CRs to one rule (D-TOPO-07). Everything older lives
only in Postgres, which keeps history permanently.

- **`Deployment`**: per (application, landscape) the active, the last
  successful and the last terminal one. A landscape's current desired entry
  is never pruned, or it would drop out of the next delivery untombstoned.
- **`Promotion`**: active ones, and terminal ones that own a retained
  `Deployment`.
- **`Release`**: referenced by a retained `Deployment`, an active `Promotion`
  or a `PromotionConfig.status.lastSucceeded`, plus the 5 highest sequences
  per application. This set is the rollback window.

## Path

1. **M1** Generate Go types, CRDs and enums from this page. CEL keeps
   `Project.spec.namespace`, `Landscape.spec.landscapeID`,
   `LandscapeGrant.spec.grantID`, `Release.spec` and `Promotion.spec`
   immutable.
2. **M1** Create migrations for every table above, plus River's own.
3. **M1** Add a CI check that fails on a table, channel, reason or enum value
   this page does not list.
4. **M2+** Enable the fields marked `# M2+`.

Later: `v1beta1` once the acceptance test passes twice without schema
changes; `ApplicationPolicy.spec.requiredSignatures` for customer co-sign;
`Deployment` as a Postgres row if the M2 simulator at 150k objects shows
etcd pressure.
