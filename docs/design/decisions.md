# Design decisions

The binding decisions for the konfidence deployment orchestrator, each with a
stable ID `D-<AREA>-<NN>`. The other design docs refine them, never
contradict them. Prior art: ADR-0006, 0007, 0019, 0027, 0032, 0039, and 0040
in `konfidence-project/konfidence-docs`, `src/docs/extend-customize/decisions/`.

## Product boundary

- **D-PROD-01** Konfidence is a standalone deployment orchestrator. A central
  control plane delivers applications to attached and isolated landscapes.
- **D-PROD-02** The vector runtime is another team's product. Its API, UI,
  `Stage` and `Vector` CRDs leave konfidence, which imports no runtime types.
- **D-PROD-03** The runtime integrates only through three konfidence-owned
  contracts: builders, gates (ADR-0039), and signed statements.
- **D-PROD-04** Milestone M1 is the acceptance test: konfidence deploys and
  promotes `example-app` across one attached and one isolated landscape with
  no runtime installed.
- **D-PROD-05** Delivery is pull-based and works offline; reporting is
  optional. The control plane never connects into a landscape.

## Topology and state

- **D-TOPO-01** A landscape is one cluster with one bridge. In v1 the bridge
  is a single replica with `Recreate` strategy, a Lease, and one
  ReadWriteOnce volume. A bridge outage never touches running workloads.
- **D-TOPO-02** Target scale: 500–1000 connected landscapes.
- **D-TOPO-03** API replicas are stateless. Any replica answers any bridge.
- **D-TOPO-04** Postgres holds every fact: releases, promotions, gate states,
  desired slots, deliveries, statements, events, audit, sessions, and River.
  One `NOTIFY` kind, "delivery changed", wakes the held `Sync` calls.
- **D-TOPO-05** One binary runs in the modes `api`, `worker`, and `signer`.
  Every River job commits in the transaction that causes it. Controller-runtime
  only writes intent CRD status. No job moves bytes. Each landscape has one
  delivery job, unique over unfinished states, so changes coalesce.
- **D-TOPO-07** Intent lives in CRDs, facts live in Postgres. The CRDs are
  `Project`, `Landscape`, `LandscapeGrant`, `GitCredential`, `Application`,
  and `PromotionConfig`. Informers read them; Postgres never copies them. An
  ad-hoc promotion is an API call that writes a row.

## Sources and releases

- **D-SRC-01** Only the control plane reads git. No git credential reaches a
  landscape or a bundle.
- **D-SRC-02** GitHub PAT, SSH deploy key, or GitHub App; github.com only.
- **D-SRC-03** A project owns each `GitCredential` and can grant it to other
  projects. Key material stays in a Kubernetes Secret, never in the DB.
- **D-SRC-04** SSH requires a pinned `known_hosts`. PAT expiry shows early.
- **D-SRC-05** Change detection polls. GitHub App webhooks only accelerate.
- **D-SRC-06** An `Application` has 1..n git sources, each with a `credentialRef`.
- **D-REL-01** A release is its build-input manifest: one canonical JSON
  document of commits, dependency digests, and the deploy-relevant
  `Application` spec. The release ID is its SHA-256. A release diff is a
  manifest diff.
- **D-REL-02** The component digest names the bytes, the release ID names the
  inputs. A per-application sequence (`payments-api@42`) is the human label.
- **D-REL-03** The API shows release diffs, which landscape runs which
  release, and each release's path through promotions and waves.

## Bundles, deliveries, and transfer

- **D-BUN-01** A bundle is a signed OCM component version of Helm charts with
  vendored dependencies. Only the build reaches public OCI.
- **D-BUN-02** Every landscape receives bundles from the first deployment on.
  No landscape syncs from git.
- **D-BUN-03** Capabilities are the only compatibility primitive. Statements
  list the bridge's capabilities; bundles declare the ones they require. A
  format major is a capability (`bundle.v1`). One comparison function blocks
  the wave step, holds back the one application (`MissingCapability`) by
  leaving its entry out of the delivery, and drives the outdated-bridge view.
- **D-BUN-04** A delivery is a signed OCM component version per landscape,
  versioned by delivery sequence. It lists every desired slot of the
  landscape and carries the CRL.
- **D-XFER-01** The bridge pulls over gRPC with one credential, its identity.
  CTF archives are the offline path. Both share the blob layout and the
  verify-and-unpack code.
- **D-XFER-02** The blob store is the only control-plane storage for bytes.
  Landscapes never talk to it directly.
- **D-XFER-03** Transfers are content-addressed and resume by offset. Each
  statement carries the blob inventory, or its digest when unchanged. Only
  missing blobs travel.
- **D-XFER-04** Images land in a registry the operator provides on the
  landscape; konfidence ships none. Its push credential never leaves the
  landscape. Policy image mode `Full` (default) transfers and pushes images.
  `Thin` transfers none and checks that every digest exists.
- **D-XFER-05** Offline moves the same two messages as online. The state
  request file holds a statement, and the CTF holds an answer plus its blobs.
  An export without a statement plans every blob.
- **D-XFER-06** A Flux post-renderer pins image references to the landscape
  registry by digest. Konfidence never changes node or containerd config.

## Landscape side

- **D-LAND-01** Flux is the only deployment engine. The bridge serves charts
  as Flux `ExternalArtifact`s. It installs a pinned Flux or adopts one.
- **D-LAND-02** Bridge-side CRDs: cluster-scoped `Bundle`,
  `ApplicationPolicy`, `Boundary`. They are the bridge's versioned API.
- **D-LAND-03** A landscape `Bundle` is the installed form of one application: 1..n charts, each a
  `HelmRelease`, ordered by `dependsOn`. No data flows between charts. No
  cross-application ordering in v1.
- **D-LAND-04** The `Bundle` owns its `HelmRelease`s and
  `ExternalArtifact`s; the `Boundary` owns namespaces. Helm keeps CRDs and
  PVCs on uninstall.
- **D-LAND-05** The bridge recreates a locally deleted `Bundle`. An
  operator refuses an application through policy deny, reported as `Refused`.
- **D-LAND-06** The bridge CLI calls an admin endpoint guarded by TokenReview
  and SubjectAccessReview. Bridge keys never leave the bridge process.

## Policy and tenancy

- **D-POL-01** `ApplicationPolicy` belongs to the landscape operator and wins
  on the landscape. It sets allow/deny, namespace, value overrides, secret
  bindings, image rules, and the reporting level.
- **D-POL-02** Values: chart < application < promotion target < landscape
  policy. Statements carry a digest of effective values, never the values.
- **D-POL-03** Bundles declare secret requirements; policies bind them to
  local stores. Secrets never pass through the control plane.
- **D-POL-04** Policies select by label. Only the bridge sets the
  `konfidence.cloud/` labels. A policy matches only inside its own
  `Boundary`. Two matching policies are an error, never a merge.
- **D-POL-05** Reporting level per application: `None` (name and level only),
  `Health`, or `Resources`. No resource `status` subtree leaves a landscape.
  The level takes effect at runtime, with no check at config save. A gate
  that needs missing data stays pending until the wave timeout.
- **D-POL-06** Release plus policy version identify what runs. The bridge
  computes the policy version. The control plane records it and never
  compares it.
- **D-POL-07** Under `updates: Manual` the bridge stores deliveries but
  applies a changed entry, tombstones included, only after the operator
  accepts it locally. `Manual` on the default policy brakes the landscape.
- **D-TEN-01** A `LandscapeGrant` declares that a project may use a
  landscape. The landscape admin approves it by writing a `Boundary`. Until
  then the bridge stores the project's entries and reports
  `WaitingForBoundaryApproval`. The bridge creates namespaces, quota, and a
  deployer ServiceAccount per tenant. Cluster-scoped kinds, CRDs, and webhooks
  always need admin approval.
- **D-TEN-02** Grant identity is the project UID. The API mints it with the
  `Project` and never changes it. The bridge stamps it as
  `konfidence.cloud/project`, and a `Boundary` names exactly one.
- **D-TEN-03** The deployer ServiceAccount gets a konfidence-defined Role,
  never `edit` or `admin`. The Role excludes Flux and konfidence kinds and
  the `bind`, `escalate`, and `impersonate` verbs.

## Trust and statements

- **D-TRUST-01** Trust on first use, both ways: the bridge pins the control
  plane's signing key, the control plane pins the bridge identity key.
- **D-TRUST-02** Both sides compute an enrollment code from landscape ID and
  both key fingerprints. Neither side trusts the other until a human confirms
  the code. Skipping needs an audited reason.
- **D-TRUST-03** A new control-plane key needs the pinned key's signature.
  After a compromise every operator re-pins locally with an out-of-band
  fingerprint. The control plane can never trigger a re-pin.
- **D-TRUST-04** An optional customer CA replaces pinning. Every delivery
  carries the CRL, and operators can import one locally.
- **D-TRUST-05** Statement import checks a deny list of revoked bridge keys.
- **D-TRUST-06** Only the `signer` mode holds the signing key. It runs the
  isolated `signing` queue and re-hashes every blob before it signs.
- **D-ATT-01** A statement is the bridge's only message to the control plane,
  signed with the bridge identity key. It carries the statement sequence,
  capabilities, inventory, last verified and applied delivery sequence, and
  per application what the reporting level allows.
- **D-ATT-02** Two counters only rise. The control plane allocates the
  delivery sequence and never reuses one; the bridge rejects anything at or
  below its last applied delivery. The bridge allocates the statement
  sequence per bridge key; the highest one under the landscape's pinned key
  counts, older ones become history. Every answer returns the last accepted
  statement sequence and the bridge raises its counter above it, so a bridge
  restored from backup recovers on its next sync.
- **D-ATT-04** After a Postgres restore, the runbook restarts the delivery
  sequence 1000 above the highest one any bridge reports.

## Promotion, rollout, and removal

- **D-PROMO-01** Gates keep ADR-0039 semantics. Every gate write is
  `SetGateState(promotion, wave, type, state, reason)`: approval, external
  writer, audited override, and reject. `Rejected` ends the promotion.
- **D-PROMO-02** A promotion is a release or a removal, plus a snapshot of
  its config's ordered waves. A wave is a landscape selector with gates, a
  failure threshold, and a timeout. Waves are the only concurrency unit.
- **D-PROMO-03** A target is one landscape in one wave. Its status derives
  from the slot and the bridge's current statement. Dispatched means a
  delivery lists the slot. Received, Applied, Healthy, Blocked, and Refused
  come from the statement. Superseded means a higher promotion owns the slot.
  A target succeeds at Healthy, or at Received if the entry reports `None`.
  For a landscape with `spec.statements: None`, Dispatched is final.
- **D-PROMO-04** Removal is a slot that points at nothing. Its wire form is a
  tombstone, a delivery entry with an empty bundle reference. Absence from a
  delivery never deletes. The bridge runs one reconcile per application.
- **D-PROMO-05** Before the wave timeout only a terminal Flux failure fails a
  target. At the timeout, a target without a statement since dispatch ends as
  `NoStatement`, and other open targets end as Failed. Both count toward the
  threshold. The rollout stores only frozen outcomes.
- **D-PROMO-06** Every landscape has one desired slot per application. A wave
  step writes it with one compare-and-set on promotion sequence. That
  predicate is the only supersede rule and the only fence. Rollback promotes
  an older release under a new sequence.

## API and UI

- **D-API-01** One protobuf schema served with Connect: gRPC for bridges,
  HTTP/JSON for browser and CLI.
- **D-API-02** Operators upgrade bridges by hand, so `v1` only grows. A
  stated minimum bridge version and capabilities (D-BUN-03) cover the rest.
- **D-API-03** The API authorizes every call and writes intent CRDs and
  Postgres rows. The UI uses only the public API.
- **D-API-04** (M2+) Machine callers use short-lived OIDC tokens through
  trust rules. No long-lived API tokens.
- **D-API-05** `BridgeService` has three RPCs: `Enroll`, `Sync`, and
  `FetchBlob`. `Sync` takes a statement and holds the call until the
  delivery changes or a timeout passes. Its answer carries the signed
  delivery and the blob plan.
- **D-UI-01** The UI uses Svelte, Skeleton, and `konfidence-design-system`
  (restored from commit `fdff305`).
- **D-UI-02** The UI separates dispatched, attested at T, and live. It never
  shows an isolated landscape as plainly healthy.
- **D-UI-03** Deep links come only from URL templates on the `Landscape`. No
  UI plugin system in v1.

## Changes after simplification (2026-10-29)

- D-PROD-04: milestone ownership and M-tags moved to the README.
- D-TOPO-04: Postgres holds every fact; one `NOTIFY` kind wakes held `Sync` calls.
- D-TOPO-05: one binary with modes `api`, `worker`, and `signer`.
- D-TOPO-06: retired with the bridge streams.
- D-TOPO-07: intent in six CRDs, facts in Postgres; no retention rule or fence exception.
- D-REL-01: release ID is the digest of one build-input manifest.
- D-BUN-03: capabilities replace format majors.
- D-BUN-04: the delivery lists slots; no dispatch records.
- D-XFER-02: the blob store is the only control-plane byte storage.
- D-XFER-03: the statement carries the inventory, as a digest when unchanged.
- D-XFER-05: the state request file holds a statement, the CTF an answer.
- D-POL-04: no `landscape-grant` label.
- D-POL-05: reporting level takes effect at runtime; no config-save check.
- D-POL-07: the operator accepts an entry, tombstones included.
- D-TEN-01: the `Boundary` is the grant approval.
- D-TEN-02: grant identity is the project UID.
- D-TRUST-06: the signer is a mode of the one binary.
- D-ATT-01: the signed statement replaces reports, handshake, and state file.
- D-ATT-02: older statements go to history instead of rejection.
- D-ATT-03: retired; no floors, no regression condition.
- D-ATT-04: reduced to one runbook rule.
- D-ATT-05: retired; entry matching moved to 05-promotion-and-rollout.md.
- D-PROMO-01: one `SetGateState` call; `Rejected` is a gate state.
- D-PROMO-02: waves are the only concurrency unit; no `maxConcurrent`.
- D-PROMO-03: status derives from slot and statement; no completion-rule field.
- D-PROMO-04: removal is a slot that points at nothing.
- D-PROMO-05: `NoStatement` at wave timeout replaces `ackTimeout`.
- D-ATT-02: statement sequence per bridge key; answers return the accepted sequence.
- D-PROMO-03: `Landscape.spec.statements: None` makes Dispatched final.
- D-LAND-02/03/04/05: the bridge-side CR is `Bundle`, not `Application`, so the term is unique.
- D-PROMO-06 (new): desired slot with compare-and-set on promotion sequence.
- D-API-02: capabilities replace handshake negotiation.
- D-API-03: the API writes intent CRDs and Postgres rows.
- D-API-05: three RPCs `Enroll`, `Sync`, and `FetchBlob`.
- D-API-06: retired; name ownership moved to the README.
