# Design decisions

This is the decision log for the konfidence deployment orchestrator. Every
design document in this directory builds on it. A design document may refine a
decision but must not contradict it; changing a decision means changing this
file first, as a separate, stated step.

Decision IDs are stable. Reference them as `D-<AREA>-<NN>`.

## Product boundary

- **D-PROD-01** Konfidence is a standalone, enterprise-scale deployment
  orchestrator. A central control plane delivers applications to attached
  (connected) and isolated (air-gapped) landscapes.
- **D-PROD-02** The vector runtime is a separate product owned by another
  team. Its API, UI, `Stage` and `Vector` CRDs leave konfidence. Konfidence
  imports no runtime Go types.
- **D-PROD-03** The runtime integrates only through three contracts owned and
  versioned by konfidence: **builders** (produce extra bundle resources from a
  repo), **gates** (ADR-0039 model) and **status reports**. To konfidence a
  `Stage` is another manifest in a bundle.
- **D-PROD-04** Acceptance test for the split: konfidence deploys and promotes
  `example-app` end to end across one attached and one isolated landscape with
  no runtime installed. Passing it is milestone M1. `09-roadmap.md` owns the
  milestone list and the M1 scope. Every other design doc tags each v1 step M1
  or M2+. No doc claims the test passes on a subset of M1.
- **D-PROD-05** Sovereign by design: pull-based, offline-capable, reporting
  optional. The control plane never connects into a landscape.

## Topology and scale

- **D-TOPO-01** A landscape is exactly one Kubernetes cluster with exactly one
  bridge (the landscape agent). In v1 the bridge is one Deployment with one
  replica, `Recreate` strategy and a `coordination.k8s.io` Lease, so two
  bridge processes never act at once. Its blob store and artifact server run
  in that pod on one ReadWriteOnce volume. A bridge outage pauses imports,
  reports and Flux reconciles of changed charts; running workloads are
  unaffected. Active/passive replicas come later, if failover time becomes a
  requirement.
- **D-TOPO-02** Target scale: 500–1000 connected landscapes.
- **D-TOPO-03** Control-plane API replicas are stateless and scale
  horizontally. Any replica can serve any bridge stream. State lives in
  Postgres and Kubernetes CRDs.
- **D-TOPO-04** Postgres is the database: history, audit, auth sessions, job
  queue. Change notification to API replicas uses `LISTEN/NOTIFY`.
- **D-TOPO-05** Heavy work (git fetch, builds, bundle assembly, delivery
  preparation, CTF export) runs as River jobs (`riverqueue/river`) on
  stateless workers. Controllers stay thin and leader-elected: they decide,
  workers execute. No job moves bytes; the bridge pulls them from any API
  replica. Delivery preparation and CTF export are one job per landscape,
  unique over non-finished states only (completed excluded). Pending changes
  coalesce, and a completed job never blocks a later change.
- **D-TOPO-06** Bridge control streams have a bounded, jittered lifetime
  (30 min ±10 %), enforced in the Connect handler; Connect on `net/http` has
  no connection-age setting. The server ends the stream with `Reconnect`. The
  bridge then dials a new TCP connection so the load balancer places it
  again. In-flight transfer RPCs are not cut; they finish or resume by offset.
- **D-TOPO-07** CRDs are the source of truth for desired state; reconcile
  decisions never read the DB. History is append-only in the DB. One
  executor fence is the stated exception: the transaction that writes a
  dispatch record refuses a dispatch whose promotion sequence is not higher
  than the last one dispatched for that (application, landscape), or whose
  promotion is terminal in history. It marks that `Deployment` superseded, so
  a restored CRD state never rolls a landscape back. A maintenance job prunes
  CRs to one rule:
  - per (application, landscape): the active `Deployment`, the last
    successful one and the last terminal one;
  - a terminal `Promotion` only while it owns a retained `Deployment`;
  - a `Release` while a retained `Deployment`, an active `Promotion` or a
    `PromotionConfig`'s last succeeded release references it, plus the 5 with
    the highest sequence number per application.

  The retained `Release` set is the rollback window. Everything older lives
  only in Postgres.

## Sources, credentials, releases

- **D-SRC-01** Only the control plane reads git. No git credential reaches a
  landscape or a bundle.
- **D-SRC-02** Supported credentials: GitHub PAT, SSH deploy key, GitHub App.
  github.com only; no GitHub Enterprise Server for now (adding a base URL later
  is additive).
- **D-SRC-03** `GitCredential` CRD, project-scoped, with `type` and a
  `secretRef` to a Kubernetes Secret. Key material never enters the DB.
  Credentials can be granted to other projects.
- **D-SRC-04** SSH requires a pinned `known_hosts`; no TOFU for git hosts.
  PAT expiry is surfaced as a status condition before syncs fail.
- **D-SRC-05** Change detection polls by default; GitHub App webhooks only
  accelerate.
- **D-SRC-06** An `Application` (control plane) consists of one or more git
  sources, each with its own `credentialRef`.
- **D-REL-01** A release ID is a hash over a canonical line serialization
  (format `konfidence.cloud/release-id/v1`, not protobuf) of three inputs:
  - the snapshotted commit list (normalized repo URL + path + SHA, sorted by
    source name);
  - the digests of the vendored dependency archives (OCI manifest digest for
    `oci://`);
  - one `layout` line: the SHA-256 of the RFC 8785 canonical JSON of the
    build- and deploy-relevant `Application` spec (charts, values file paths,
    `dependsOn`, CRD policy, overridable keys, secret requirements, builders
    and their config).

  Equal release ID means equal deployable inputs. The controller creates a
  `Release` only for a release ID the application does not have yet. Builder
  image versions are not inputs; they affect only the component digest
  (D-REL-02).
- **D-REL-02** The component version digest (bytes) is a separate identity
  from the release ID (sources). A per-application sequence number
  (`payments@42`) is the human-facing label.
- **D-REL-03** APIs visualize releases: diff between releases, which
  landscape runs which release (with values and policy digests), the path of a
  release through promotions and waves.

## Bundles and transfer

- **D-BUN-01** A bundle is a signed OCM component version. Primary content is
  Helm charts. Chart dependencies are vendored at build time
  (`helm dependency build`); public OCI is reached only by the build, never by
  a landscape.
- **D-BUN-02** Bundle from the first deployment on. Every landscape, including
  the first connected one, receives a bundle. No direct git sync to landscapes.
- **D-BUN-03** The bundle/CTF format carries its own integer major version,
  independent of the API version. Each bundle is built in exactly one major. A
  control-plane release that introduces a new major keeps building the
  previous one until an admin raises the emitted major. Readability is checked
  per application. A release whose major the bridge cannot read is held back
  for that application only; the landscape's other applications and
  tombstones still deliver.
- **D-BUN-04** A delivery is a small signed OCM component version per
  landscape (`konfidence.cloud/l/<landscape>/delivery`, version
  `<delivery sequence>.0.0`). It lists the landscape's full desired set: per
  application a bundle reference by digest, promotion-target values and
  tombstones, plus the CRL. It is the unit of dispatch. A dispatch record in
  Postgres lists the `Deployment`s each delivery covers. An ack or export of
  delivery N moves exactly those `Deployment`s to Dispatched.
- **D-XFER-01** gRPC is the primary transfer path, initiated by the bridge.
  Each landscape has exactly one credential: the bridge identity. CTF archives
  are the offline path. Both use the same content-addressed blob layout and
  the same verify-and-unpack code on the bridge.
- **D-XFER-02** OCI is not a transfer mechanism. The orchestrator-local
  registry is storage behind the control plane, never a protocol landscapes
  use.
- **D-XFER-03** gRPC transfers are chunked, resumable and content-addressed.
  The bridge reports its blob inventory; only missing blobs travel. The blob
  inventory is mandatory, not an optimization.
- **D-XFER-04** Images land in one landscape registry that the operator
  provides and the nodes already pull from. The operator configures it at
  bridge install: endpoint, optional CA bundle, and a push credential in a
  Secret on the landscape. That credential never leaves the landscape and is
  not a control-plane credential (D-XFER-01). `ApplicationPolicy` image mode
  decides transfer content per application at transfer time:
  - **Full** (default): the transfer carries image blobs, and the bridge
    pushes them under a per-project path in the landscape registry.
  - **Thin**: no image blobs travel; the bridge checks that every digest
    already exists in the landscape registry.

  Without a configured landscape registry the bridge applies no application
  that has images and reports `RegistryNotConfigured`. Konfidence ships no
  registry.
- **D-XFER-05** Offline, the operator exports one **state request file** per
  trip from the bridge. It carries bridge version and capabilities, blob
  inventory, the transfer-relevant part of application policies, the last
  applied delivery sequence and the signed attestation reports the control
  plane has not yet acknowledged. One upload imports those reports and
  requests the next CTF. Without a state request file, the CTF is a full
  transfer.
- **D-XFER-06** The bridge rewrites image references to where images landed
  in the landscape registry, pinned by digest, via a Flux `HelmRelease`
  post-renderer (kustomize `images`). Konfidence never changes node or
  containerd configuration. Nodes resolving, trusting and pulling from the
  landscape registry is the operator's cluster setup, the same setup the
  bridge's own images need.

## Landscape side

- **D-LAND-01** Flux is the only deployment engine. The bridge serves chart
  artifacts in-cluster and publishes them as Flux `ExternalArtifact`
  (`source.toolkit.fluxcd.io/v1`, feature gate `ExternalArtifact=true` on
  helm-controller and kustomize-controller), consumed by
  `HelmRelease.spec.chartRef`. The bridge installs a pinned Flux or adopts an
  existing one after checking the feature gate.
- **D-LAND-02** Bridge-side CRDs: cluster-scoped `Application` (installed
  application), `ApplicationPolicy`, `Boundary`. They are the bridge's API and
  follow versioned-CRD compatibility rules.
- **D-LAND-03** A landscape `Application` is 1..n charts, each a `HelmRelease`
  ordered by `dependsOn` and readiness. CRDs come through the chart `crds/`
  directory with an explicit install/upgrade CRD policy. **No data flow
  between charts** (no outputs feeding values, no cross-chart templating) —
  that is runtime territory. No cross-application ordering in v1.
- **D-LAND-04** Ownership: the cluster-scoped `Application` owns its
  `HelmRelease`s and `ExternalArtifact`s. The `Boundary` owns namespaces,
  never the `Application`. Helm keeps CRDs and StatefulSet PVCs on uninstall;
  this is the documented default.
- **D-LAND-05** If an operator deletes an `Application` CR locally the bridge
  recreates it from the last bundle. Refusing an application is done through
  `ApplicationPolicy` deny, shown to the control plane as "refused by
  landscape".
- **D-LAND-06** The bridge CLI talks to a bridge admin endpoint. The endpoint
  authenticates callers with Kubernetes TokenReview and authorizes each
  operation with SubjectAccessReview on `bridgeoperations.konfidence.cloud`
  verbs. Bridge keys never leave the bridge process. No operator task needs
  `exec` into the bridge or read access to its Secrets.

## Policy and tenancy

- **D-POL-01** `ApplicationPolicy` belongs to the landscape operator and is
  authoritative on the landscape. It controls: what may be deployed
  (allow/deny), target namespace, value overrides, secret bindings, image
  rules (thin/full, registry allow-list, required signatures), reporting
  level. A landscape-wide default plus per-application policies.
- **D-POL-02** Value precedence: chart defaults < application < promotion
  target < landscape policy. Applications declare overridable keys (values
  schema + allow-list). Attestations carry a digest of effective values, never
  the values.
- **D-POL-03** Secrets: bundles declare secret requirements (name, keys);
  policies bind them to local stores (ESO `SecretStore` or plain Secret).
  Unbound requirements block the deployment with a clear status. Secrets never
  pass through the control plane.
- **D-POL-04** Policies select applications by label. Labels under
  `konfidence.cloud/` (`project`, `application`, `landscape-grant`) are set
  only by the bridge, from the verified bundle identity and the delivery
  (D-TEN-02); such labels in a bundle are dropped. A policy matches only
  applications inside its own `Boundary`. At most one policy per application
  besides the default; two matches is an "ambiguous policy" error, never a
  merge.
- **D-POL-05** Reporting level per application: `none | health | resources`.
  - `health`: application state (Applied, Refused, Blocked, Removed) plus one
    aggregated health per application and chart.
  - `resources`: `health` plus, per applied resource, group, kind, namespace,
    name, `Ready` status, reason, message (truncated to 1 KiB) and observed
    generation.

  No resource `status` subtree leaves a landscape. Promotion-config validation
  rejects gates and completion rules a target landscape cannot satisfy at its
  level; gate entries may declare `requiresReporting: health | resources`. A
  lowered level at runtime makes gates fail closed with an explicit reason.
- **D-POL-06** Deployment identity = release + policy version. The policy
  version is the SHA-256 of the canonical effective policy for the
  application, computed by the bridge. Reports also carry the contributing
  policy refs `<name>@<generation>` for display. The control plane records the
  policy version and never compares it against an expected value, because
  policy is authoritative on the landscape.
- **D-POL-07** Update acceptance per application:
  `ApplicationPolicy.spec.updates: Auto | Manual`, default `Auto`. Under
  `Manual` the bridge verifies and stores deliveries. It applies a changed
  release or a tombstone for that application only after the landscape
  operator accepts that exact component digest or tombstone sequence locally
  (`konfidence-bridge accept`). Until then the application keeps its applied
  release and reports `AwaitingLocalAcceptance`. `Manual` on the landscape
  default policy is the landscape-wide brake. The control plane can never
  accept on the operator's behalf.
- **D-TEN-01** Landscape sharing: a project requests access, the landscape
  admin defines a `Boundary`, the application policy works inside it, the
  bridge enforces.
  - Automated: namespaces from template, a deployer ServiceAccount and Role
    per tenant (D-TEN-03), ResourceQuota, and `serviceAccountName` on every
    `HelmRelease` the bridge writes.
  - Flux multi-tenancy lockdown flags (`--no-cross-namespace-refs`,
    `--default-service-account`, `--no-remote-bases`) are set only when the
    bridge installs Flux. On adopted Flux the bridge reports
    `FluxLockdown=False` and keeps rendering, because tenants get no rights on
    Flux kinds.
  - Always manual admin approval: cluster-scoped kinds, CRDs, webhooks, and
    namespaced kinds outside the base set.

  "Waiting for boundary approval" is its own status. It is derived from the
  bundle's build-time metadata; tenant RBAC is the hard guard for anything
  that appears only with landscape values.
- **D-TEN-02** Each `LandscapeGrant` carries a random grant ID, minted by the
  API at creation and immutable. The delivery carries it per application; the
  bundle does not. The bridge stamps it as `konfidence.cloud/landscape-grant`.
  A `Boundary` names exactly one grant ID and matches on it only; project and
  grant names are display only. A deleted and recreated grant or project gets
  a new ID, and the landscape admin must update the `Boundary` again.
- **D-TEN-03** The deployer ServiceAccount is bound to a konfidence-defined
  Role in its `Boundary`'s namespaces. It is never bound to the built-in
  `edit` or `admin` ClusterRoles, which aggregate Flux and third-party kinds.
  The Role covers a fixed base set of namespaced kinds plus namespaced kinds
  the `Boundary` approves. It never includes `*.toolkit.fluxcd.io` or
  `landscape.konfidence.cloud` kinds, or the `bind`, `escalate` and
  `impersonate` verbs. The bridge refuses bundles containing Flux or
  konfidence kinds with `OutsideBoundary`.

## Trust and attestation

- **D-TRUST-01** Trust on first use by default, mutual: the bridge pins the
  control plane's release-signing key, the control plane pins the bridge
  identity key.
- **D-TRUST-02** At enrollment both sides compute an enrollment code: 16
  base32 characters of SHA-256 over landscape ID, control-plane key
  fingerprint and bridge key fingerprint. The bridge pins the control-plane
  key only after the landscape operator confirms the code locally. The control
  plane keeps the bridge key `Unconfirmed` until a `landscape-admin` enters the
  code the bridge printed. Until then it dispatches nothing to that landscape
  and imports no attestation from it. Skipping the comparison is explicit on
  either side, needs a reason, and is audited and shown in the UI.
- **D-TRUST-03** Rotation: a new control-plane key is accepted only if signed
  by the pinned key. This covers planned rotation only, because whoever holds
  a compromised key can sign a transition too. Recovery from a control-plane
  key compromise is a local re-pin on every bridge (`trust repin`); the
  control plane can never trigger it. Bridge enrollment and `trust repin`
  accept an expected control-plane key fingerprint obtained out of band
  (`--cp-key-fingerprint`). With it the bridge pins only that key, and its side
  of the comparison needs no control-plane UI.
- **D-TRUST-04** Optional customer CA replaces pinning. With an offline root
  it is the recommended setup when a security office must be able to revoke
  control-plane keys. Revocation uses a CRL carried in every delivery,
  including CTF, and importable locally on the bridge (`trust crl import`),
  because a compromised control plane can withhold a CRL. Isolated landscapes
  are only as fresh as their last transfer; documented, not hidden.
- **D-TRUST-05** Revoked bridge keys go on a deny list checked at attestation
  import.
- **D-TRUST-06** Signing-key custody: only the signing worker Deployment holds
  the release-signing key. It runs the `signing` River queue, with egress
  limited to Postgres, the blob store and the Kubernetes API. It re-hashes
  every resource blob before it signs a bundle or delivery. Processes that
  fetch git, run helm or builders, or reach public registries never hold the
  key. Enrollment responses are not signed; the enrollment code comparison
  (D-TRUST-02) authenticates them.
- **D-ATT-01** Offline attestation is required. The bridge signs reports with
  its identity key, binding landscape ID, release digest, per-resource status,
  effective values digest, policy version and the report sequence (D-ATT-02).
  Reports arrive over gRPC or as an imported file and are processed by the
  same code.
- **D-ATT-02** Two monotonic per-landscape counters guard freshness, each
  owned by its writer.
  - The delivery sequence (lseq) is allocated only by the control plane, in
    Postgres, in its own committed transaction. A failed attempt leaves a
    gap and never reuses a number; gaps are allowed. CRD fields only mirror
    it. It travels in every signed delivery. The bridge rejects a delivery
    that is not higher than its last applied one. Equal lseq with a different
    digest raises an alarm.
  - The report sequence is allocated by the bridge and travels in every
    report. The control plane rejects a report at or below the last accepted
    one.

  Freshness never relies on wall-clock time.
- **D-ATT-03** Counters recover upward only. Every handshake response and
  every delivery carries the control plane's delivery floor and its last
  accepted report sequence. The bridge sets its delivery floor to the maximum
  of its state Secret, the highest verified delivery in its store and the
  received floor. It raises its report counter above the accepted value and
  never lowers either counter. A bridge restored from backup therefore
  recovers on the next connect or delivery, without re-enrollment. A
  regressed report from a valid key sets the Landscape condition
  `BridgeStateRegressed` instead of failing silently.
- **D-ATT-04** After a Postgres restore, lseq restarts at max(restored value,
  highest lseq reported by any bridge) + 1000. A number signed before the
  restore, including one in a CTF a courier still holds, is never reissued.
- **D-ATT-05** A report's application entry updates the current desired
  `Deployment` for (application, landscape) only if two checks pass: the
  report's delivery sequence is at or above the lseq that first carried that
  `Deployment`, and the reported component digest equals the `Deployment`'s.
  That `Deployment` is Attested when the entry reports Applied and Healthy.
  Entries that match no current `Deployment` go to history only.

## Promotion, rollout, removal

- **D-PROMO-01** Gates follow ADR-0039 (inline `status.gates`, type-agnostic
  controller, external writers, override = setting a gate satisfied with
  audit). `VectorPromotion` becomes `Promotion` over a `Release`.
- **D-PROMO-02** Fan-out: a promotion targets a set of landscapes (list or
  selector), creates one `Deployment` record per landscape and rolls out in
  waves with `maxConcurrent` and a failure threshold. Gates at promotion level
  and optionally per wave.
- **D-PROMO-03** Each wave has an explicit per-landscape completion rule:
  `attested` or `dispatched`. Landscapes at reporting level `none` can only use
  `dispatched`.
- **D-PROMO-04** Removal: a tombstone in the signed delivery (D-BUN-04)
  removes a whole application from a landscape. Tombstones go through the
  same promotion, gate and wave path. Charts removed from an application
  between releases are removed by diffing releases.
- **D-PROMO-05** In a wave, an attached landscape that does not acknowledge
  its delivery within the wave's `ackTimeout` (default 15 min) ends its
  `Deployment` as `Unreachable`. Unreachable frees the `maxConcurrent` slot,
  counts toward `failureThreshold` and is shown separately from `Failed`. The
  release stays desired, and the landscape pulls it on reconnect. A late
  arrival is recorded on the `Deployment` without changing the wave result.
  Only a terminal Flux failure or the wave timeout makes a `Deployment`
  `Failed`; `Progressing` never does.

## API and UI

- **D-API-01** One protobuf schema served with Connect (connectrpc): gRPC for
  bridges, HTTP/JSON for browser and CLI.
- **D-API-02** Compatibility because bridges are upgraded manually by
  operators (no self-update): `buf breaking` in CI, additive changes only
  within `v1`, field numbers never reused, capability negotiation at
  handshake, stated minimum bridge version, outdated bridges visible in the UI.
- **D-API-03** The API authorizes (OIDC session in Postgres, project, grants)
  and writes CRDs. The UI uses only the public API and never the Kubernetes
  API.
- **D-API-04** (M2+) Machine callers of `konfidence.api.v1` authenticate with
  short-lived OIDC tokens. Trust rules on `Project` or `Landscape` map
  (issuer, audience, claim match) to a role. GitHub Actions and Kubernetes
  projected ServiceAccount tokens use the same rule type. The issuer's keys
  are configured statically on the rule or fetched from an issuer URL outside
  any landscape; the control plane never fetches keys from a landscape. No
  long-lived API tokens. A gate's `setBy` is `<issuer>#<sub>`.
- **D-API-05** `konfidence.bridge.v1` has exactly one service,
  `BridgeService`, with five RPCs:
  - `Enroll`: unary, enrollment token.
  - `Connect`: bidirectional control stream for handshake, delivery notices,
    acks and `Reconnect`.
  - `PlanTransfer`: unary, full blob inventory.
  - `FetchBlob`: server stream, resumable by offset.
  - `SubmitReport`: unary, one signed report.

  Files a courier carries go through `konfidence.api.v1.OfflineService`.
- **D-API-06** `02-domain-model.md` is the only definition of names other docs
  depend on: CRD fields, status enums and reason strings, Postgres tables and
  the `LISTEN/NOTIFY` channel. `01-control-plane.md` owns River job kinds and
  queues; `08-api-and-ui.md` owns proto names and the only proto sketch. Topic
  docs reference these and carry no YAML or proto sketches of their own. CRD
  enum values are PascalCase; proto enums use the prefixed UPPER_SNAKE form
  and map 1:1.
- **D-UI-01** The UI is built from the start: Svelte, Skeleton and
  `konfidence-design-system` (product-neutral, restored from commit `fdff305`).
- **D-UI-02** The UI distinguishes *dispatched*, *attested at T* and *live*.
  Isolated landscapes are never shown as plainly healthy.
- **D-UI-03** Runtime resources appear as generic resource status from
  attestations. Deep links come from URL templates per group and kind, set on
  the `Landscape` by its landscape-admin. Bundle content never supplies link
  targets. No UI plugin system in v1.

## Prior art

The previous konfidence design lives in
`konfidence-project/konfidence-docs` under
`src/docs/extend-customize/decisions/`. Relevant: ADR-0006 (OCM
integration), ADR-0007 (multi-landscape), ADR-0019 (OCM vector assembly),
ADR-0027 (landscape deployment target), ADR-0032 (vector promotions),
ADR-0039 (promotion gates), ADR-0040 (promotion rejection and force
execution).

## Changes after review (2026-10-29)

- D-PROD-04: passing the test is M1; `09-roadmap.md` owns the milestone list.
- D-TOPO-01: v1 bridge is a single pod with a Lease; active/passive comes later.
- D-TOPO-05: no job moves bytes; one coalescing delivery job per landscape.
- D-TOPO-06: stream lifetime enforced in the Connect handler, not `MaxConnectionAge`.
- D-TOPO-07: one CR retention rule; dispatch fence in Postgres as the only DB read.
- D-REL-01: release ID adds a `layout` line over the deploy-relevant spec.
- D-BUN-03: format major checked per application.
- D-BUN-04 (new): per-landscape signed delivery is the dispatch unit.
- D-XFER-04: images land in an operator-provided registry; konfidence ships none.
- D-XFER-05: one state request file per trip, carrying unacknowledged reports.
- D-XFER-06: konfidence never changes node or containerd configuration.
- D-LAND-06 (new): bridge CLI uses an admin endpoint with TokenReview and SubjectAccessReview.
- D-POL-04: grant label comes from the delivery, not the bundle.
- D-POL-05: reporting levels capped at `none | health | resources` with fixed contents.
- D-POL-06: policy version defined as a bridge-computed hash, recorded, never compared.
- D-POL-07 (new): per-application manual update acceptance on the landscape.
- D-TEN-01: tenant RBAC replaces Flux lockdown as the hard guard.
- D-TEN-02 (new): grants have minted IDs; one grant per `Boundary`.
- D-TEN-03 (new): konfidence-defined deployer Role without Flux or konfidence kinds.
- D-TRUST-02: enrollment code confirmed on both sides before trust counts.
- D-TRUST-03: compromise recovery is a local re-pin with an out-of-band fingerprint.
- D-TRUST-04: CRL rides in every delivery and is importable locally.
- D-TRUST-06 (new): signing key lives only in an isolated signing worker.
- D-ATT-01: reports carry the report sequence.
- D-ATT-02: two counters, delivery sequence and report sequence, each owned by its writer.
- D-ATT-03 (new): counters recover upward only; handshake and delivery carry floors.
- D-ATT-04 (new): lseq restarts 1000 above the highest known value after a Postgres restore.
- D-ATT-05 (new): rule for matching report entries to `Deployment`s.
- D-PROMO-04: tombstones travel in the delivery.
- D-PROMO-05 (new): unacknowledged attached landscapes end as `Unreachable`.
- D-API-04 (new, M2+): machine callers use short-lived OIDC tokens via trust rules.
- D-API-05 (new): one `BridgeService` with five RPCs.
- D-API-06 (new): name ownership across 02, 01 and 08.
- D-UI-03: deep links come only from landscape URL templates.
