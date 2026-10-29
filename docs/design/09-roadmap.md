# Roadmap from the acceptance test to 1000 landscapes

This page owns the milestone list and the M1 scope (D-PROD-04). Each
workstream is one task for one coding agent and links the doc sections it
implements. "P3" means step 3 of that doc's Path section. Terms are in the
[glossary](00-overview.md#glossary).

## Target vision

A central control plane turns git commits into signed bundles and promotes
them to 500–1000 Kubernetes landscapes. Attached or air-gapped, each landscape
pulls through one bridge that verifies, applies local policy and signs what
runs. The vector runtime is a separate product that plugs in through
builders, gates and status reports only.

## Three milestones, each with one exit test

| Milestone | Exit test |
| --- | --- |
| M1 | The D-PROD-04 acceptance test passes in CI on every merge to main. |
| M2 | The runtime team ships the vector builder and a gate writer without a konfidence code change. A second project deploys to a shared landscape. |
| M3 | The simulator runs 1000 landscapes within the 01 sizing. Customers with a security office run `CustomerCA` with an HSM. |

## M1 passes the acceptance test on two kind landscapes

M1 is the smallest feature set that passes this test. Nothing in M1 exists
for its own sake, and no doc claims the test passes on a subset.

### The acceptance test deploys and promotes example-app with no runtime

The harness starts three kind clusters: the control plane in `konfidence all`
mode with Postgres, one attached landscape and one isolated landscape. Each
landscape has Flux with `ExternalArtifact=true` and a local registry. The
registry and kind's containerd config are test harness, not konfidence
(D-XFER-06). No runtime component is installed anywhere.

1. Create a project, a `GitHubPAT` credential and `example-app` with one
   source. The built-in Helm builder builds it.
2. Enroll the attached bridge online and the isolated bridge offline. Both
   sides confirm the enrollment code.
3. Approve a `LandscapeGrant` per landscape. The operator writes the
   `Boundary`, deployer manifests and the `default` policy by hand.
4. Push a commit. `example-app@1` builds, signs and passes `ocm verify`.
5. Create a `PromotionConfig` with both landscapes, one wave, completion
   `Attested` and an `explicit-approval` gate. Approve it.
6. The attached bridge pulls the delivery, Flux reports `Ready`, and its
   `Deployment` turns `Attested`.
7. The isolated operator exports a state request file, imports the CTF, then
   carries the report out. That `Deployment` turns `Attested`.
8. Push a second commit and promote `example-app@2` the same way. The UI
   shows "live" for the attached and "attested at T" for the isolated one.
9. Re-import the first CTF on the isolated landscape. The bridge rejects it.

### M1 starts with five parallel workstreams

| ID | Workstream | Implements | Path steps | Needs |
| --- | --- | --- | --- | --- |
| M1-01 | Acceptance test harness: kind clusters, registries, Flux, the test above as a failing CI job | [00 vision](00-overview.md#konfidence-turns-git-commits-into-attested-deployments-on-every-landscape) | 03 P6 harness note | none |
| M1-02 | Spike: chart `.tgz` as `ExternalArtifact` behind `chartRef`; record the helm-controller failure condition | [03 Flux handover](03-bundles-and-transfer.md#the-bridge-verifies-then-unpacks-then-hands-over-to-flux), [06 reasons](06-landscape-bridge-and-policy.md#the-bridge-reports-one-reason-per-application) | 03 P1 | none |
| M1-03 | Go types, CRDs, enums, CEL immutability, migrations, name drift check | [02 all sections](02-domain-model.md) | 02 P1–3 | none |
| M1-04 | Proto module, `buf lint` and `buf breaking`, authz option, interceptor, RPC lint | [08 schema](08-api-and-ui.md#one-schema-two-package-trees-two-listeners), [08 roles](08-api-and-ui.md#roles-exist-at-three-scopes) | 08 P1–2 | none |
| M1-05 | Release ID serializer with golden files | [07 release ID](07-sources-and-releases.md#the-release-id-hashes-a-versioned-line-format) | 07 P1 | none |

### M1 control-plane workstreams

| ID | Workstream | Implements | Path steps | Needs |
| --- | --- | --- | --- | --- |
| M1-06 | Process modes, migration runner, River with five queues and the failure split | [01 processes](01-control-plane.md#five-process-types-share-one-binary-and-one-database), [01 controllers](01-control-plane.md#controllers-decide-workers-execute), [01 jobs](01-control-plane.md#job-kinds-queues-and-unique-keys) | 01 P1–2 | M1-03 |
| M1-07 | `GitHubPAT` credential with grants and probe, poll scheduler, `source.poll`, `source.resolve`, `Release` creation | [07 flow](07-sources-and-releases.md#one-flow-creates-every-release), [07 credentials](07-sources-and-releases.md#three-credential-types-one-secret-each), [07 expiry](07-sources-and-releases.md#expiry-shows-up-as-a-condition-before-syncs-fail) | 07 P2–4, 01 P3 | M1-05, M1-06 |
| M1-08 | Built-in Helm builder, `bundle.build`, signing worker with `secret` signer, NetworkPolicy and re-hash | [03 bundle](03-bundles-and-transfer.md#the-bundle-carries-everything-that-is-equal-on-every-landscape), [03 build and sign](03-bundles-and-transfer.md#one-worker-builds-another-worker-signs), [04 custody](04-trust-and-attestation.md#only-the-signing-worker-holds-the-release-signing-key) | 03 P2, 04 P2, 01 P3 | M1-07 |
| M1-09 | `PromotionConfig` with a landscape list and one wave, `Promotion`, `Deployment` state machine, supersede rules, `ApproveGate` and `RejectPromotion`, completion, `ackTimeout`, reporting-level validation, `cr.prune` | [05 flow](05-promotion-and-rollout.md), [02 retention](02-domain-model.md#retention-keeps-etcd-at-the-sized-budget) | 05 P1, P3, P4, 01 P3 | M1-06 |
| M1-10 | `delivery.prepare`: sequence, desired set, delivery with empty CRL, dispatch fence, `NOTIFY`, ack moves `dispatch_entries` | [01 delivery job](01-control-plane.md#one-delivery-job-per-landscape-builds-fences-and-announces), [03 delivery](03-bundles-and-transfer.md#the-delivery-carries-everything-that-differs-per-landscape), [05 dispatch](05-promotion-and-rollout.md#the-delivery-not-the-deployment-is-what-gets-dispatched) | 01 P5, 03 P3, 05 P2 | M1-08, M1-09 |
| M1-11 | `BridgeService` on the API: `Hello`, `Welcome`, epoch, `LISTEN`, resync, 30 min lifetime, degraded mode, `PlanTransfer`, `FetchBlob` with the 50-stream cap | [01 streams](01-control-plane.md#api-replicas-hold-bridge-streams-without-owning-state), [03 pull](03-bundles-and-transfer.md#the-bridge-pulls-only-the-blobs-it-lacks), [08 bridge protocol](08-api-and-ui.md#bridgeservice-is-the-one-bridge-wire-protocol) | 01 P4–5, 03 P5, 08 P3 | M1-04, M1-10 |
| M1-12 | OIDC login for browser and CLI, sessions, roles, read services, `WatchService`, audited write path for every M1 resource in 08 P5, `GetPlacement` and `GetPath` | [08 sessions](08-api-and-ui.md#sessions-for-people-short-lived-tokens-for-machines), [08 write path](08-api-and-ui.md#the-api-writes-crds-controllers-do-the-rest), [08 lists](08-api-and-ui.md#lists-come-from-cache-history-from-postgres-watch-only-invalidates), [07 queries](07-sources-and-releases.md#release-history-answers-diff-placement-and-path-queries) | 08 P4–5, 07 P5 | M1-03, M1-04 |
| M1-13 | Fleet, Application and Promotion screens with the observation chip | [08 status](08-api-and-ui.md#status-shows-how-we-know-not-only-what-we-know), [08 UI](08-api-and-ui.md#the-ui-is-three-screens-in-m1) | 08 P7 | M1-12 |

### M1 bridge and trust workstreams

| ID | Workstream | Implements | Path steps | Needs |
| --- | --- | --- | --- | --- |
| M1-14 | Bridge identity key, mTLS with SPKI match, online and offline enrollment with `OfflineService.ImportEnrollmentRequest` and `ExportEnrollmentResponse`, both-side code, audited skip | [04 keys](04-trust-and-attestation.md#two-keys-carry-all-trust), [04 enrollment](04-trust-and-attestation.md#enrollment-counts-only-after-both-sides-confirm-the-code), [04 offline enrollment](04-trust-and-attestation.md#offline-enrollment-moves-the-same-messages-as-files) | 04 P1, P3, P4, enrollment RPCs of 08 P6 | M1-03, M1-04 |
| M1-15 | Bridge pod with Lease, blob store, one import routine, verify order, counter floors, admin endpoint, shipped Roles, CLI | [06 pod](06-landscape-bridge-and-policy.md#one-bridge-pod-runs-every-loop), [06 CLI](06-landscape-bridge-and-policy.md#the-bridge-cli-talks-to-an-admin-endpoint-never-through-exec), [03 layout](03-bundles-and-transfer.md#grpc-and-ctf-share-one-content-addressed-blob-layout), [04 counters](04-trust-and-attestation.md#counters-only-move-upward) | 06 P1, 03 P4, 04 P5 | M1-14 |
| M1-16 | Flux adopt mode with probe, application controller, `ExternalArtifact` and `HelmRelease`, values Secret, image post-renderer, `Full` push to the landscape registry | [06 Flux](06-landscape-bridge-and-policy.md#the-bridge-adopts-a-flux-that-passes-a-behavioural-probe), [06 charts](06-landscape-bridge-and-policy.md#each-chart-becomes-one-externalartifact-and-one-helmrelease), [06 ownership](06-landscape-bridge-and-policy.md#ownership-follows-the-application-deletion-is-always-explicit), [03 profile](03-bundles-and-transfer.md#the-transfer-profile-picks-image-blobs-per-application) | 06 P2–3, 03 P6 | M1-02, M1-15 |
| M1-17 | `default` policy: values layers, `secretRef`, `Deny`, `updates: Manual` with `accept`; hand-written `Boundary` by grant ID; admission from metadata; deployer manifests | [06 policy](06-landscape-bridge-and-policy.md#exactly-one-policy-applies-plus-the-default), [06 values](06-landscape-bridge-and-policy.md#values-merge-in-four-layers-last-wins), [06 acceptance](06-landscape-bridge-and-policy.md#the-operator-can-hold-updates-for-local-acceptance), [06 admission](06-landscape-bridge-and-policy.md#admission-uses-the-bundles-build-time-metadata), [06 boundary](06-landscape-bridge-and-policy.md#a-boundary-confines-one-grant-and-tenant-rbac-is-the-hard-guard) | 06 P4–5 | M1-16 |
| M1-18 | Reporter at `None`, `Health`, `Resources`; DSSE reports; `SubmitReport`; `attest.Import`; D-ATT-05 matching | [04 reports](04-trust-and-attestation.md#reports-are-dsse-envelopes-signed-by-the-bridge), [04 import](04-trust-and-attestation.md#one-import-path-serves-stream-and-file), [05 matching](05-promotion-and-rollout.md#a-report-attests-only-the-deployment-it-matches), [06 reporting](06-landscape-bridge-and-policy.md#reporting-level-filters-what-leaves-the-landscape) | 06 P6, 04 P6 | M1-09, M1-15 |
| M1-19 | State request file, `OfflineService` CTF and report RPCs with signed download, `ctf.export`, `ctf import` | [03 offline trip](03-bundles-and-transfer.md#one-state-request-file-goes-out-and-one-ctf-comes-in-per-trip), [01 isolated](01-control-plane.md#one-delivery-job-per-landscape-builds-fences-and-announces), [08 services](08-api-and-ui.md#public-services-group-by-resource-family) | 03 P7, 01 P6, 08 P6 | M1-10, M1-18 |

### M1 leaves out everything the test does not exercise

Selectors, multiple waves, rollback, removal and tombstones are M2. So are
`OverrideGate`, `Thin`, per-application policies, Flux install mode and ESO.
Further M2 items: SSH and GitHub App credentials, multiple sources, webhooks,
image builders, machine auth, revocation and screens beyond three. M1 ships
`ApproveGate` and `RejectPromotion`; 05 and 08 follow this split.

### Unverified facts each have one workstream that settles them

| Fact | Settled by |
| --- | --- |
| helm-controller accepts a packaged `.tgz` as `ExternalArtifact` content | M1-02 |
| helm-controller signals "gave up" as `Stalled` with `RetriesExceeded` | M1-02 |
| OCM requires semver versions; how the Go library writes an `ociImage` into a CTF | M1-08 |
| kustomize `images` rewrites container fields only | M1-16 |
| connect-web sends `Connect-Protocol-Version` on every request | M1-12 |
| GitHub returns `github-authentication-token-expiration` for PATs | M1-07 |
| GitHub git-protocol limits at a few thousand applications | M2-07 |
| The OCM Go library accepts an external `crypto.Signer` | M3-06 |

## M2 makes the runtime contracts usable and opens shared landscapes

Each workstream also enables its `# M2+` fields in 02 (02 P4).

| ID | Workstream | Implements | Path steps |
| --- | --- | --- | --- |
| M2-01 | Image builder runner for `builder/v1`, vector builder as first consumer | [07 builders](07-sources-and-releases.md#builders-turn-a-checked-out-source-into-ocm-resources) | 07 P8 |
| M2-02 | External gate writers: `SetGateState`, report query API (`DeploymentService`), gate-writer library, `soak` gate | [05 gates](05-promotion-and-rollout.md#gates-follow-adr-0039-at-two-scopes) | 05 P7 |
| M2-03 | Machine auth with OIDC trust rules on `Project` and `Landscape`, including transfer-station robots (D-API-04) | [08 sessions](08-api-and-ui.md#sessions-for-people-short-lived-tokens-for-machines) | 08 P8 |
| M2-04 | Selectors, multiple waves, thresholds, chaining, `Rollback`, `Remove` with tombstones, catch-up, `OverrideGate` | [05 rollout](05-promotion-and-rollout.md#rollout-runs-wave-by-wave), [05 rollback](05-promotion-and-rollout.md#rollback-promotes-an-earlier-release), [05 removal](05-promotion-and-rollout.md#removal-is-a-promotion-that-carries-a-tombstone) | 05 P5–6, 03 P8 |
| M2-05 | Shared landscapes: per-application policies, boundary controller, Flux install mode with lockdown flags, ESO bindings | [06 boundary](06-landscape-bridge-and-policy.md#a-boundary-confines-one-grant-and-tenant-rbac-is-the-hard-guard), [06 secrets](06-landscape-bridge-and-policy.md#secret-requirements-bind-to-local-stores) | 06 P7 |
| M2-06 | `Thin` transfer, `requireSignatures` image checks, format-major hold-back | [03 profile](03-bundles-and-transfer.md#the-transfer-profile-picks-image-blobs-per-application), [03 delivery job](03-bundles-and-transfer.md#the-delivery-job-builds-one-delivery-per-landscape-change) | 03 P8, 06 P7 |
| M2-07 | Multiple sources, tree-hash skip, `SSHDeployKey`, `GitHubApp` with webhooks | [07 sources](07-sources-and-releases.md#an-application-has-one-or-more-sources-each-with-its-own-credential), [07 polling](07-sources-and-releases.md#polling-detects-changes-webhooks-only-shorten-the-wait) | 07 P6–7 |
| M2-08 | Deployment, Release with diff, Landscape and Audit screens; `AuditService` | [08 UI](08-api-and-ui.md#the-ui-is-three-screens-in-m1), [07 queries](07-sources-and-releases.md#release-history-answers-diff-placement-and-path-queries) | 08 P8, 07 P7 |
| M2-09 | Bridge key deny list, revocation, `trust repin --cp-key-fingerprint`, restore drill in CI | [04 compromise](04-trust-and-attestation.md#compromise-recovery-runs-on-the-landscape-never-from-the-control-plane), [04 deny list](04-trust-and-attestation.md#a-revoked-bridge-key-goes-on-the-deny-list-m2), [01 recovery](01-control-plane.md#landscapes-keep-running-when-the-control-plane-is-gone) | 04 P7, 01 P8 |
| M2-10 | Bridge simulator: 1000 streams, 150 000 `Deployment`s, 2 replicas, a full-transfer fan-out | [01 sizing](01-control-plane.md#sizing-for-1000-landscapes-fits-three-api-replicas) | 01 P7 |
| M2-11 | `blob.gc` and `history.partition` | [01 stores](01-control-plane.md#crds-hold-desired-state-postgres-holds-everything-that-grows) | none |
| M2-12 | CRDs to `v1beta1`, once the acceptance test passes twice without schema change | [02](02-domain-model.md) | 02 Later |

## M3 hardens scale, the bridge and the trust root

M2-10 results decide M3-01 to M3-03; skip any the simulator does not justify.

| ID | Workstream | Trigger |
| --- | --- | --- |
| M3-01 | `Deployment` as a Postgres row plus one summary CR per wave; API lists read Postgres instead of an informer | simulator shows etcd or informer pressure |
| M3-02 | Separate `konfidence blobs` mode and a fleet-wide transfer budget | simulator shows stream or UI pressure |
| M3-03 | Incremental blob inventory | above about 100 000 blobs per landscape |
| M3-04 | Per-application apply with a per-application delivery sequence in reports | measured head-of-line blocking |
| M3-05 | Active/passive bridge replicas | failover time becomes a requirement (D-TOPO-01) |
| M3-06 | `CustomerCA` mode with offline root and `trust crl import`, plus the `pkcs11` signer | before any customer whose security office approves the trust root |
| M3-07 | Control-plane key transitions and bridge identity rotation | after M3-06 |
| M3-08 | Bridge support window (for example N-2 minors), pre-raise view, admin-set emitted format major | first format-major raise |
| M3-09 | Read replicas for UI history, watch resume tokens | refetch or history load shows |

M3-06 moves ahead of M2 if such a customer arrives first. The `secret`
signer suffices for the acceptance test and for attached-only installs.

## Not planned yet

Each item has a recorded design note but no milestone:

- Customer co-sign through `ApplicationPolicy.spec.requiredSignatures`.
  `Manual` acceptance covers four-eyes review until then.
- A bridge-owned control namespace per `Boundary`. It needs proof first
  that `--no-cross-namespace-refs` permits `targetNamespace`.
- Per-chart target namespaces.
- Dependency mirror rewrites, an extra CA bundle and `HTTPS_PROXY` for build
  workers.
- A GitHub Enterprise Server base URL on `GitCredential`.
- Semver-range refs, and a shared fetch cache if fetch time dominates builds.
- Pause and resume, opt-in automatic rollback, cross-application ordering.
- Custom roles and opt-in four-eyes approval.
- CTF volume splitting and a courier drop folder.
- A reproducible-build check and a TPM-backed bridge identity.
- Vault Transit or cloud KMS signer backends, on demand.
- A multi-region control plane.

## Start with these five tasks tomorrow

1. **M1-01** Write the acceptance test as a failing CI job. It defines
   "done" for M1, so nobody changes it to get green.
2. **M1-02** Run the `ExternalArtifact` spike on kind. Its answer fixes the
   artifact format for M1-08 and M1-16.
3. **M1-03** Generate types, CRDs and migrations from 02, with the drift
   check. Every other workstream compiles against them.
4. **M1-04** Create the proto module with `buf breaking` in CI. Bridge and
   API work starts from it.
5. **M1-05** Build the release ID package. It is fully specified and has no
   dependencies.
