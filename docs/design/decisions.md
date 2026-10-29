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
  `Stage` is just another manifest in a bundle.
- **D-PROD-04** Acceptance test for the split: konfidence deploys and promotes
  `example-app` end to end across one attached and one isolated landscape with
  no runtime installed.
- **D-PROD-05** Sovereign by design: pull-based, offline-capable, reporting
  optional. The control plane never connects into a landscape.

## Topology and scale

- **D-TOPO-01** A landscape is exactly one Kubernetes cluster with exactly one
  bridge (the landscape agent), running active/passive with leader election.
- **D-TOPO-02** Target scale: 500–1000 connected landscapes.
- **D-TOPO-03** Control-plane API replicas are stateless and scale
  horizontally. Any replica can serve any bridge stream. State lives in
  Postgres and Kubernetes CRDs.
- **D-TOPO-04** Postgres is the database: history, audit, auth sessions, job
  queue. Change notification to API replicas uses `LISTEN/NOTIFY`.
- **D-TOPO-05** Heavy work (git fetch, builds, bundle assembly, transfers) runs
  as River jobs (`riverqueue/river`) on stateless workers. Controllers stay
  thin and leader-elected: they decide, workers execute. River unique jobs
  deduplicate per (release, landscape).
- **D-TOPO-06** gRPC servers set `MaxConnectionAge` so long-lived bridge
  streams rebalance after scale-out.
- **D-TOPO-07** CRDs are the source of truth for desired state; reconcile
  decisions never read the DB. History is append-only in the DB. CRDs keep the
  current release plus the previous N per landscape.

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
- **D-REL-01** A release ID is a hash over a canonical serialization of the
  snapshotted commit list (repo URL + path + SHA, sorted by source name) plus
  the resolved dependency digests (chart lock). Canonical serialization is not
  protobuf.
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
- **D-BUN-03** The bundle/CTF format carries its own version, independent of
  the API version.
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
- **D-XFER-04** Transfer content is decided per target at transfer time. If
  the landscape operator configured registry credentials on the bridge,
  transfer is thin (charts and metadata; images from the operator's registry).
  Otherwise transfer is full, including images, which the bridge loads into a
  landscape-local registry (bridge-shipped or operator-provided without
  credentials).
- **D-XFER-05** Offline, the operator exports a **state request file** from
  the bridge: bridge version and capabilities, blob inventory, the
  transfer-relevant part of application policies. Without one, the CTF is a
  full transfer.
- **D-XFER-06** The bridge rewrites image references to where images landed,
  pinned by digest, via a Flux `HelmRelease` post-renderer (kustomize `images`).
  No containerd mirror config.

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
  only by the bridge from the verified bundle identity; such labels in a bundle
  are dropped. A policy matches only applications inside its own `Boundary`.
  At most one policy per application besides the default; two matches is an
  "ambiguous policy" error, never a merge.
- **D-POL-05** Reporting level per application: `none | health | resources |
  full`. Promotion-config validation rejects gates a target landscape cannot
  satisfy at its reporting level. A lowered level at runtime makes gates fail
  closed with an explicit reason.
- **D-POL-06** Deployment identity = release + policy version. History
  records both.
- **D-TEN-01** Landscape sharing: a project requests access, the landscape
  admin defines a `Boundary`, the application policy works inside it, the
  bridge enforces. Automated: namespaces from template, ServiceAccount and RBAC
  per tenant, ResourceQuota, Flux multi-tenancy lockdown
  (`HelmRelease.serviceAccountName`, no cross-namespace refs, default
  ServiceAccount). Always manual admin approval: cluster-scoped kinds, CRDs,
  webhooks. "Waiting for boundary approval" is its own status.

## Trust and attestation

- **D-TRUST-01** Trust on first use by default, mutual: the bridge pins the
  control plane's release-signing key, the control plane pins the bridge
  identity key.
- **D-TRUST-02** At enrollment the bridge shows the key fingerprint; the
  operator compares it with the UI before pinning. Skipping is explicit and
  audited.
- **D-TRUST-03** Rotation: a new key is accepted only if signed by the pinned
  key. Re-pinning without that chain needs a deliberate local operator action
  on the bridge; the control plane can never trigger it.
- **D-TRUST-04** Optional customer CA replaces pinning. Revocation via a CRL
  carried in every bundle, including CTF. Isolated landscapes are only as
  fresh as their last transfer; documented, not hidden.
- **D-TRUST-05** Revoked bridge keys go on a deny list checked at attestation
  import.
- **D-ATT-01** Offline attestation is required. The bridge signs reports with
  its identity key, binding landscape ID, release digest, per-resource status,
  effective values digest, policy version and a monotonic per-landscape
  sequence number. Reports arrive over gRPC or as an imported file and are
  processed by the same code.
- **D-ATT-02** The same monotonic sequence protects bundles: the bridge
  rejects bundles with a lower sequence than the last applied. Freshness never
  relies on wall-clock time.

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
- **D-PROMO-04** Removal: a signed tombstone in the bundle removes a whole
  application from a landscape, carrying the landscape sequence number.
  Tombstones go through the same promotion, gate and wave path. Charts removed
  from an application between releases are removed by diffing releases.

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
- **D-UI-01** The UI is built from the start: Svelte, Skeleton and
  `konfidence-design-system` (product-neutral, restored from commit `fdff305`).
- **D-UI-02** The UI distinguishes *dispatched*, *attested at T* and *live*.
  Isolated landscapes are never shown as plainly healthy.
- **D-UI-03** Runtime resources appear as generic resource status from
  attestations, with deep links to the runtime's own UI. No UI plugin system
  in v1.

## Prior art

The previous konfidence design lives in
`konfidence-project/konfidence-docs` under
`src/docs/extend-customize/decisions/`. Relevant: ADR-0006 (OCM
integration), ADR-0007 (multi-landscape), ADR-0019 (OCM vector assembly),
ADR-0027 (landscape deployment target), ADR-0032 (vector promotions),
ADR-0039 (promotion gates), ADR-0040 (promotion rejection and force
execution).
