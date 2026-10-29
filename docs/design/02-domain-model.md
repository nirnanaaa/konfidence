# Domain model and name registry

This page defines every name the other docs use: CRDs and their fields,
Postgres tables, the one `NOTIFY` channel, status values, and reasons. A name
missing here does not exist yet; add it here first. River job kinds belong
to [01](01-control-plane.md), proto names to [08](08-api-and-ui.md).

Enum values are PascalCase; proto enums map 1:1 in prefixed UPPER_SNAKE.
Fields marked `# M2+` ship after the acceptance test (D-PROD-04). A CI check
fails on any table, channel, reason or enum value this page does not list.

## Intent lives in CRDs, facts live in Postgres

Informers read the intent CRDs; Postgres never copies them (D-TOPO-07).
Every other fact is a Postgres row (D-TOPO-04). Bridge CRDs stay on the
landscape and reach the control plane only as signed statements (D-ATT-01).
Both groups start at `v1alpha1`; the YAML below omits `apiVersion`.

| Group | Kind | Scope | Writer |
| --- | --- | --- | --- |
| `konfidence.cloud` | `Project`, `Landscape` | cluster | API; `Landscape` status by controller |
| `konfidence.cloud` | `LandscapeGrant`, `GitCredential`, `Application`, `PromotionConfig` | project namespace | API; `GitCredential` status by controller |
| `landscape.konfidence.cloud` | `Application` | cluster | bridge only |
| `landscape.konfidence.cloud` | `ApplicationPolicy`, `Boundary` | cluster | landscape operator |

## Identities and counters have one owner each

| Name | Form | Owner |
| --- | --- | --- |
| project UID | `prj-<random>` in `Project.spec.uid`, immutable | API at creation (D-TEN-02) |
| landscape ID | `lsc-<random>` in `Landscape.spec.landscapeID`, immutable | API at creation |
| application key | (project UID, `Application` name) | API |
| release ID | `sha256:` of the canonical build-input manifest | `release.build` (D-REL-01) |
| release label | `<application>@<release seq>` | `release.build` (D-REL-02) |
| promotion sequence | `promotions.seq`, a global identity column | Postgres |
| delivery sequence | Postgres sequence `delivery_seq`, global, gaps allowed | delivery job (D-ATT-02) |
| statement sequence | in every statement; the highest per landscape on its current `Pinned` key counts | bridge (D-ATT-02) |
| bundle component | `konfidence.cloud/p/<project>/<application>`, version `<release seq>.0.0` | build (D-BUN-01) |
| delivery component | `konfidence.cloud/l/<landscape ID>/delivery`, version `<delivery seq>.0.0` | delivery job (D-BUN-04) |
| entry digest | SHA-256 of one delivery entry, tombstones included | delivery job |
| policy version | SHA-256 of one application's effective policy | bridge (D-POL-06) |
| capability | dotted string, M1 set: `bundle.v1`, `statement.v1` | bridge lists, bundle requires (D-BUN-03) |
| `setBy` | session subject, or `<issuer>#<sub>` for machines (M2+, D-API-04) | API |

Supersede and fence use the promotion sequence only. One global delivery
sequence serves every landscape because a bridge needs only an upward one.
After a Postgres restore, the runbook runs
`ALTER SEQUENCE delivery_seq RESTART WITH <highest reported + 1000>` (D-ATT-04).
Only the bridge sets the labels `konfidence.cloud/project` (project UID) and
`konfidence.cloud/application` on landscape objects (D-POL-04).

## Project owns one namespace and its role bindings

```yaml
kind: Project
metadata: { name: payments }
spec:
  uid: prj-01hx3k9q                         # minted, immutable (CEL self == oldSelf)
  namespace: kden-project-payments          # immutable, defaulted; the API creates it
  roleBindings: { project-admin: [{ session: { memberOf: [payments-admins] } }] }   # roles in 08
  trustRules: []                            # M2+, D-API-04
```

## Landscape records one cluster with one bridge

```yaml
kind: Landscape
metadata: { name: de-fra-prod-1, labels: { env: prod, ring: canary } }
spec:
  landscapeID: lsc-7f3a9c                   # minted, immutable
  mode: Attached                            # Attached | Isolated; display and UI basis only
  statements: Expected                      # Expected | None; None makes Dispatched final (D-PROMO-03)
  trust: { mode: Pinned }                   # Pinned | CustomerCA (M3) with caSecretRef
  roleBindings: { landscape-admin: [{ session: { memberOf: [fra-cluster-ops] } }] }
  staleAfter: 30m                           # live vs stale in the UI
  linkTemplates: [{ group: vector.example, kind: Stage, url: "https://runtime.fra/stages/{namespace}/{name}" }]   # D-UI-03
status: { enrollment: Enrolled, keyFingerprint: "SHA256:4f…" }   # controller, from bridge_keys
```

Nothing is delivered or imported before `Enrolled` (D-TRUST-02). Bridge
version, capabilities and liveness come from the latest statement.

## LandscapeGrant lets one project use one landscape

```yaml
kind: LandscapeGrant
metadata: { name: de-fra-prod-1, namespace: kden-project-payments }
spec: { landscapeRef: de-fra-prod-1 }
```

One grant exists per project and landscape. Waves select only granted
landscapes. The landscape admin approves by writing a `Boundary` for the
project UID (D-TEN-01). The API refuses to delete a project or grant while
it holds a slot that points at a bundle; remove the applications first.

## GitCredential points at a Secret and scopes its use

```yaml
kind: GitCredential
metadata: { name: github-pat, namespace: kden-project-payments }
spec:
  type: GitHubPAT                           # GitHubPAT | SSHDeployKey (M2+) | GitHubApp (M2+)
  secretRef: { name: github-pat }           # same namespace (D-SRC-03)
  knownHosts: "github.com ssh-ed25519 AAAA…"   # required for SSHDeployKey (D-SRC-04)
  grants: [{ project: payments, repositories: ["acme/payments-*"] }]
status: { expiresAt: 2027-01-31T00:00:00Z, conditions: [Valid, ExpiringSoon] }
```

## Application declares sources and charts

```yaml
kind: Application
metadata: { name: payments-api, namespace: kden-project-payments }
spec:
  sources:                                  # one in M1; builders M2+ (07)
    - { name: app, url: https://github.com/acme/payments-api, ref: { branch: main },
        credentialRef: { name: github-pat }, builders: [] }
  charts:                                   # crds: Skip | Create | CreateReplace
    - { name: api, source: app, path: deploy/chart, valuesFiles: [deploy/values.yaml],
        dependsOn: [], crds: CreateReplace, overridable: [/replicaCount, /ingress/host] }
  secretRequirements: [{ name: db, keys: [username, password] }]
  pollInterval: 5m
```

`sources`, `charts` and `secretRequirements` enter the build-input manifest;
`pollInterval` and `credentialRef` do not (D-REL-01).

## PromotionConfig declares ordered waves

```yaml
kind: PromotionConfig
metadata: { name: prod, namespace: kden-project-payments }
spec:
  application: payments-api
  source: { application: payments-api }     # { application } | { promotionConfig } (M2+)
  mode: Auto                                # Auto | Manual
  values: { ingress: { className: nginx } }
  landscapeValues: { de-fra-prod-1: { ingress: { host: pay.fra.acme.example } } }
  waves:                                    # one wave in M1
    - { name: canary, selector: { matchLabels: { ring: canary } }, failureThreshold: 0, timeout: 30m,
        gates: [{ type: explicit-approval.konfidence.cloud }] }
```

`failureThreshold` takes an integer or a percentage. A gate that must hold
before any rollout sits on the first wave. A promotion copies `values`,
`landscapeValues` and `waves` at creation; editing the config never changes
a running promotion.

## Bridge Application is the installed form of one slot

```yaml
kind: Application
metadata: { name: payments.payments-api,   # <project>.<application>
  labels: { konfidence.cloud/project: prj-01hx3k9q, konfidence.cloud/application: payments-api } }
spec:                                       # release empty and charts [] for a tombstone
  release: { id: sha256:e3b0…, label: payments-api@42, bundleDigest: sha256:5d41… }
  deliverySequence: 118
  entryDigest: sha256:c7d2…
  charts: [{ name: api, artifactDigest: sha256:ab01…, dependsOn: [], crds: CreateReplace }]
  secretRequirements: [{ name: db, keys: [username, password] }]
  overridable: { api: [/replicaCount, /ingress/host] }
status:
  conditions: [{ type: Ready, status: "True", reason: Applied }]
  health: Healthy
  policyVersion: sha256:be11…
  policyRefs: [default@3]                   # <policy>@<generation>, display only
  boundary: payments-fra
  targetNamespace: payments
  effectiveValuesDigest: sha256:0a9b…
  charts: [{ name: api, helmRelease: payments/payments-api-api, ready: true }]
  pendingEntry: sha256:77ee…                # updates: Manual only (D-POL-07)
  acceptedEntry: { digest: "", by: "", at: null }   # `accept` via the admin endpoint (D-LAND-06)
```

State is the `Ready` condition plus one reason. A tombstoned application
keeps its object with reason `Removed`.

## ApplicationPolicy is the operator's authoritative say

```yaml
kind: ApplicationPolicy
metadata: { name: payments-api }            # "default" is the landscape default, the only one in M1
spec:
  boundaryRef: payments-fra                 # ignored for "default"
  selector: { matchLabels: { konfidence.cloud/application: payments-api } }
  action: Allow                             # Allow | Deny
  updates: Auto                             # Auto | Manual
  targetNamespace: payments
  values: { api: { replicaCount: 4 } }
  secretBindings: [{ requirement: db, secretRef: { name: db-creds } }]   # secretStoreRef + remoteKey M2+
  images: { transfer: Full, registries: [registry.fra.acme.internal] }   # Full | Thin (M2+)
  reporting: Health                         # None | Health | Resources
status: { matchedApplications: [payments.payments-api], conditions: [Valid] }
```

## Boundary is the tenant fence and the grant approval

```yaml
kind: Boundary
metadata: { name: payments-fra }
spec:
  project: prj-01hx3k9q                     # exactly one project UID (D-TEN-02)
  namespaces: { allowed: [payments, payments-jobs], template: { labels: { team: payments } } }
  quota: { hard: { requests.cpu: "40", requests.memory: 80Gi } }
  approvedClusterScoped: [{ group: apiextensions.k8s.io, kind: CustomResourceDefinition, names: [ledgers.payments.acme] }]
  approvedNamespaced: [{ group: monitoring.coreos.com, kind: ServiceMonitor }]   # beyond the base Role (D-TEN-03)
status: { serviceAccount: konfidence-payments, pendingClusterScoped: [], conditions: [Ready] }
```

In M1 the operator writes the `Boundary` by hand (M2+ automates namespaces,
Role and quota).

## Postgres tables hold every fact

Columns shown are the ones docs rely on. `events`, `statements` and `audit_log` are append-only.

| Table | Holds | Main columns |
| --- | --- | --- |
| `releases` | every release (D-REL-01) | `release_id` PK, `project_uid, application, seq, manifest, bundle_digest, required_capabilities, state, created_at`; unique `(project_uid, application, seq)` |
| `promotions` | one row per promotion, ad-hoc or from a config | `seq` identity PK, `project_uid, application, config, release_id` (NULL = removal), `waves, values, landscape_values, state, created_by, created_at, ended_at` |
| `rollout_targets` | one landscape in one wave; frozen outcome only | `promotion_seq, landscape_id, wave, outcome, reason, decided_at`; PK `(promotion_seq, landscape_id)` |
| `gate_states` | the one gate write (D-PROMO-01) | `promotion_seq, wave, type, state, reason, set_by, set_at, override`; PK `(promotion_seq, wave, type)` |
| `desired` | one slot per landscape and application (D-PROMO-06) | `landscape_id, project_uid, application` PK, `promotion_seq, release_id, bundle_digest` (NULL = tombstone), `values, required_capabilities, first_delivery_seq` |
| `deliveries` | every signed delivery | `landscape_id, delivery_seq` PK, `component_digest, created_at, signed_at` |
| `statements` | every accepted bridge statement | `landscape_id, key_fingerprint, statement_seq` PK, `bridge_version, capabilities, verified_delivery_seq, applied_delivery_seq, payload, payload_digest, received_at, last_seen_at` |
| `events` | history of every state change, for UI timelines | `id, at, project_uid, landscape_id, promotion_seq, kind, data` |
| `audit_log` | every mutating API call and override; commits with the row write, or after the API server accepts a CRD write | `id, at, actor, actor_kind, project_uid, action, object_ref, reason, request_digest` |
| `sessions` | OIDC sessions | `id_hash, subject, issuer, groups, created_at, expires_at, last_seen_at` |
| `bridge_keys` | enrollment tokens, pinned keys, deny list (D-TRUST-05) | `id, landscape_id, token_hash, token_expires_at, fingerprint, public_key, state, predecessor, confirmed_by, skip_reason, created_at, revoked_at` |
| `blobs` | the blob store index (D-XFER-02) | `digest` PK, `size, media_type, created_at, last_referenced_at` |
| `component_blobs` | blobs a bundle or delivery needs | `component_digest, blob_digest` |
| `river_*` | River queue, created by `rivermigrate` | River's own |

A wave step writes a slot with one statement. Zero affected rows mean a
higher promotion owns the slot, and the target freezes as `Superseded`. The
same transaction inserts the landscape's delivery job.

```sql
INSERT INTO desired AS d (landscape_id, project_uid, application, promotion_seq, release_id,
                          bundle_digest, values, required_capabilities)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
ON CONFLICT (landscape_id, project_uid, application) DO UPDATE
  SET (promotion_seq, release_id, bundle_digest, values, required_capabilities, first_delivery_seq) =
      (excluded.promotion_seq, excluded.release_id, excluded.bundle_digest, excluded.values, excluded.required_capabilities, NULL)
  WHERE d.promotion_seq < excluded.promotion_seq;
```

The delivery job takes `nextval('delivery_seq')`, lists every slot whose
capabilities the bridge has, and sets `first_delivery_seq` on slots listed
for the first time. `Enroll` stores statement 0, so every enrolled landscape
has capabilities on record. A bridge reuses its statement sequence until the
content changes; a repeat updates `last_seen_at` only. An older statement is
stored but never becomes the latest. No table stores a blob inventory.

## One NOTIFY kind wakes held Sync calls

The channel is `konfidence_delivery`, the payload the landscape ID. The
transaction that sets `deliveries.signed_at` sends it. An API replica that
holds a `Sync` for that landscape answers with the new delivery (D-API-05).
[01](01-control-plane.md) owns the listener.

## Rollout status is a function of slot and latest statement

For target `t` of promotion `p` on landscape `l`, the first matching row
wins. `s` is the slot, `st` the latest statement of `l`, and `e` the entry in
`st` that matches `s` ([05](05-promotion-and-rollout.md) owns matching).

| # | Condition | Status |
| --- | --- | --- |
| 1 | `t.outcome` is set | the outcome |
| 2 | `s.promotion_seq > p.seq` | `Superseded` |
| 3 | no slot, or `s.promotion_seq < p.seq` | `Pending` |
| 4 | `s.required_capabilities` ⊄ `st.capabilities` | `Blocked(MissingCapability)` |
| 5 | `s.first_delivery_seq` is NULL | `Pending` |
| 6 | `l.spec.statements` is `None` | `Dispatched`, final |
| 7 | `st.verified_delivery_seq < s.first_delivery_seq` | `Dispatched` |
| 8 | no `e`, level `None`, or `Progressing` or `Removing` | `Received` |
| 9 | `e.reason` is `Refused` | `Refused` |
| 10 | `e.reason` is in the blocking set | `Blocked(<reason>)` |
| 11 | `e.reason` is `Failed` | `Failed` |
| 12 | `Applied` and `Healthy`, or `Removed` for a tombstone | `Healthy` |
| 13 | `e.reason` is `Applied` | `Applied` |

A target succeeds at `Healthy`, at `Received` with level `None`, and at
row 6. The wave freezes the outcome when it counts the target (D-PROMO-05).
A golden test covers every row.

## Reasons form one vocabulary

The bridge `Ready` condition, statement entries and `rollout_targets.reason`
use this table. Adding a reason is additive; renaming one is breaking.

| Reason | Set by | Status or outcome |
| --- | --- | --- |
| `Applied` | bridge | `Applied`, `Healthy` when healthy |
| `Progressing`, `Removing` | bridge | `Received` |
| `Removed` | bridge | `Healthy` for a tombstone |
| `Failed` | bridge, Flux gave up | outcome `Failed` |
| `Refused` | bridge, policy `Deny` | outcome `Refused` |
| `WaitingForBoundaryApproval`, `AmbiguousPolicy`, `UnboundSecret`, `ValuesRejected`, `ImageRuleViolation`, `OutsideBoundary`, `RegistryNotConfigured`, `AwaitingLocalAcceptance`, `FluxUnavailable` | bridge | `Blocked` (the blocking set) |
| `MissingCapability` | control plane | `Blocked`; the entry stays out of the delivery |
| `NoStatement` | control plane, wave timeout without a statement since dispatch | outcome `Failed` |
| `WaveTimeout` | control plane, wave timeout with any other open status | outcome `Failed` |

Statement import rejections are API errors, not reasons: `EnrollmentUnconfirmed`,
`KeyRevoked`, `SignatureInvalid`, `LandscapeMismatch`.

## Every other enum

| Field | Values |
| --- | --- |
| `promotions.state` | `InProgress`; terminal: `Succeeded`, `Failed`, `Rejected` |
| `rollout_targets.outcome` | `Succeeded`, `Failed`, `Refused`, `Superseded` |
| `gate_states.state` | `Pending`, `Satisfied`, `Error`; terminal: `Rejected` ends the promotion |
| gate `type` | `explicit-approval.konfidence.cloud` (M1), `soak.konfidence.cloud` (M2+), external types (05) |
| `releases.state` | `Building`, `Ready`, `Failed` |
| `health` | `Healthy`, `Progressing`, `Degraded`, `Unknown` |
| `Landscape.status.enrollment` | `Pending`, `Unconfirmed`, `Enrolled` |
| `bridge_keys.state` | `Invited`, `Unconfirmed`, `Pinned`, `Retired`, `Revoked` |
| reporting level | `None`, `Health`, `Resources` |
| `audit_log.actor_kind` | `Session`, `Workload`, `System` |
