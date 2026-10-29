# Roadmap from the acceptance test to 1000 landscapes

This page owns the milestone list and the M1 scope (D-PROD-04). Each
workstream is one task for one coding agent and links the doc sections it
implements. "P3" means step 3 of that doc's Path section. Terms are in the
[glossary](00-overview.md#glossary-of-terms-every-doc-uses).

## Three milestones, each with one exit test

| Milestone | Exit test |
| --- | --- |
| M1 | The D-PROD-04 acceptance test passes in CI on every merge to main. |
| M2 | The runtime team ships the vector builder and a gate writer without a konfidence code change. A second project deploys to a shared landscape. |
| M3 | The simulator runs 1000 landscapes within the [01 sizing](01-control-plane.md#sizing-for-1000-landscapes-fits-three-api-replicas). Customers with a security office run `CustomerCA` with an HSM. |

## M1 is exactly what the acceptance test exercises

M1 is the smallest feature set that passes this test. Nothing in M1 exists
for its own sake, and no doc claims the test passes on a subset of M1.

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
3. Create a `LandscapeGrant` per landscape. Each operator approves it by
   writing a `Boundary` that names the project UID, the deployer manifests
   and the `default` policy by hand.
4. Create a `PromotionConfig` with `mode: Auto` and one wave whose selector
   matches both landscapes and an `explicit-approval.konfidence.cloud` gate.
5. Push a commit. `example-app@1` builds, signs, passes `ocm verify`, and
   starts a promotion. Approve its gate through `SetGateState`.
6. The attached bridge's held `Sync` returns the delivery. Flux reports
   `Ready`, the next statement reports it, and the target turns `Healthy`.
7. The isolated operator exports a statement file, uploads it through
   `OfflineService`, imports the CTF, then carries the next statement file
   out. That target turns `Healthy`.
8. Push a second commit and promote `example-app@2` the same way. The UI
   shows "live" for the attached landscape and "as of T" for the isolated
   one.
9. Re-import the first CTF on the isolated landscape. The bridge rejects it,
   because its delivery sequence is below the last applied one.

### Five workstreams start in parallel

| ID | Workstream | Implements | Path steps | Needs |
| --- | --- | --- | --- | --- |
| M1-01 | Acceptance test harness: kind clusters, registries, Flux, the test above as a failing CI job | [00 vision](00-overview.md#konfidence-turns-git-commits-into-signed-deliveries-on-every-landscape) | none | none |
| M1-02 | Spike: chart `.tgz` as `ExternalArtifact` behind `chartRef`; record the helm-controller failure condition | [03 Flux handover](03-bundles-and-transfer.md#the-bridge-verifies-then-unpacks-then-hands-over-to-flux) | 03 P1 | none |
| M1-03 | Go types for the six intent CRDs and the three bridge CRDs, enums, CEL immutability, Postgres migrations, name drift check | [02 all sections](02-domain-model.md) | none | none |
| M1-04 | Proto module, `buf lint` and `buf breaking`, authz option, interceptor and its lint | [08 schema](08-api-and-ui.md#one-schema-serves-bridges-and-people-on-two-listeners) | 08 P1 | none |
| M1-05 | Manifest builder and release ID hasher with golden files | [07 release ID](07-sources-and-releases.md#the-release-id-is-the-digest-of-the-build-input-manifest) | 07 P1 | none |

### Control-plane workstreams

| ID | Workstream | Implements | Path steps | Needs |
| --- | --- | --- | --- | --- |
| M1-06 | Modes `api`, `worker`, `signer`, `all`; migration runner; River queues and the failure split; status writers for `GitCredential` and `Landscape` | [01 modes](01-control-plane.md#one-binary-runs-in-three-modes), [01 jobs](01-control-plane.md#every-state-change-inserts-its-job-in-the-same-transaction) | 01 P1–2, P6 | M1-03 |
| M1-07 | `GitHubPAT` with grants and the credential check; `source.poll` and `release.build` with the Helm builder | [07 flow](07-sources-and-releases.md#one-chain-of-jobs-creates-every-release), [07 credentials](07-sources-and-releases.md#three-credential-types-one-secret-each) | 07 P2–3 | M1-05, M1-06 |
| M1-08 | Secret key backend in `signer` mode, its NetworkPolicy, the re-hash, `bundle.sign` | [03 signer](03-bundles-and-transfer.md#the-signer-signs-what-the-blob-store-holds), [04 keys](04-trust-and-attestation.md#two-keys-carry-all-trust) | 03 P2, 04 P2 | M1-07 |
| M1-09 | Promotions: `PromotionConfig` with one wave, `promotions` rows, ad-hoc promotions, the wave step with the slot compare-and-set, the status function with its golden test, entry matching, frozen outcomes, `NoStatement`, `SetGateState` with `Rejected` | [05 all sections](05-promotion-and-rollout.md), [02 status](02-domain-model.md#rollout-status-is-a-function-of-slot-and-latest-statement) | 05 P1–4, 01 P3 | M1-06 |
| M1-10 | `delivery.build`: advisory lock, capability check, signed delivery with empty CRL, `first_delivery_seq`, `NOTIFY` | [01 delivery job](01-control-plane.md#one-delivery-job-per-landscape-builds-signs-and-announces), [03 delivery](03-bundles-and-transfer.md#the-delivery-carries-what-differs-per-landscape) | 01 P4 | M1-08, M1-09 |
| M1-11 | `BridgeService` (`Enroll`, held `Sync`, `FetchBlob` with its cap) and `OfflineService` with the signed download | [01 held calls](01-control-plane.md#api-replicas-hold-sync-calls-without-owning-state), [03 Sync](03-bundles-and-transfer.md#sync-takes-a-statement-and-answers-with-the-newest-delivery), [08 bridge service](08-api-and-ui.md#the-bridge-service-moves-one-statement-up-and-one-answer-down), [08 offline](08-api-and-ui.md#the-offline-service-takes-a-statement-file-and-returns-a-ctf) | 01 P5, 03 P4, 08 P2 | M1-04, M1-10 |
| M1-12 | OIDC login, sessions, roles, read services, audited writes, placement and path queries | [08 sessions](08-api-and-ui.md#sessions-for-people-short-lived-tokens-for-machines), [08 writes](08-api-and-ui.md#the-api-writes-intent-to-crds-and-facts-to-rows), [07 queries](07-sources-and-releases.md#release-queries-read-manifests-slots-and-statements) | 08 P3–4, 07 P4 | M1-03, M1-04 |
| M1-13 | Fleet, Application and Promotion screens with the observation chip | [08 status](08-api-and-ui.md#status-shows-how-we-know-not-only-what-we-know), [08 UI](08-api-and-ui.md#the-ui-is-three-screens-in-m1) | 08 P5 | M1-12 |

### Bridge and trust workstreams

| ID | Workstream | Implements | Path steps | Needs |
| --- | --- | --- | --- | --- |
| M1-14 | Bridge identity key, mTLS with SPKI match, online and offline enrollment, both-side code, audited skip | [04 enrollment](04-trust-and-attestation.md#enrollment-counts-only-after-both-sides-confirm-the-code) | 04 P1, P3 | M1-03, M1-04 |
| M1-15 | Bridge pod with Lease, local blob store, verify-and-unpack, admin endpoint, shipped Roles, CLI, Flux adopt mode with the probe | [06 pod](06-landscape-bridge-and-policy.md#one-bridge-pod-runs-every-loop), [06 CLI](06-landscape-bridge-and-policy.md#the-bridge-cli-talks-to-an-admin-endpoint-never-through-exec), [03 layout](03-bundles-and-transfer.md#one-blob-layout-serves-grpc-and-ctf) | 06 P1, 03 P3 | M1-14 |
| M1-16 | The one reconcile per application: charts with `dependsOn`, values Secret, image post-renderer, `Full` push, tombstones, deny, dropped charts, recreate on local delete | [06 reconcile](06-landscape-bridge-and-policy.md#one-reconcile-per-application-covers-deploy-tombstone-deny-and-dropped-charts), [06 charts](06-landscape-bridge-and-policy.md#each-wanted-chart-becomes-one-externalartifact-and-one-helmrelease), [03 images](03-bundles-and-transfer.md#images-go-to-the-operators-registry) | 06 P2 | M1-02, M1-15 |
| M1-17 | `default` policy with values layers, `secretRef` and `updates: Manual`; hand-written `Boundary` by project UID; admission | [06 policy](06-landscape-bridge-and-policy.md#exactly-one-policy-applies-laid-over-the-default), [06 acceptance](06-landscape-bridge-and-policy.md#the-operator-can-hold-changes-for-local-acceptance), [06 boundary](06-landscape-bridge-and-policy.md#a-boundary-approves-one-project-and-tenant-rbac-is-the-hard-guard) | 06 P3–4 | M1-16 |
| M1-18 | Statement builder at all three levels, DSSE signing, one import path for `Sync` and the statement file | [04 statement](04-trust-and-attestation.md#a-statement-is-the-bridges-signed-account-of-its-landscape), [04 counters](04-trust-and-attestation.md#two-counters-only-move-upward), [06 reporting](06-landscape-bridge-and-policy.md#the-reporting-level-filters-each-statement-entry-at-runtime) | 06 P5, 04 P4 | M1-09, M1-15 |

### M1 scope rulings where topic docs could read either way

- The wave `selector` with `matchLabels` ships in M1; the test needs it to
  pick both landscapes. Several waves and thresholds above zero are M2.
- The one capability comparison (D-BUN-03) ships in M1. It drives the wave
  step, the delivery job and the outdated flag on the Fleet screen.
- The bridge handles tombstones in M1, inside the one reconcile. Removal
  promotions that write them are M2.

### Unverified facts each have one workstream that settles them

| Fact | Settled by |
| --- | --- |
| helm-controller accepts a packaged `.tgz` as `ExternalArtifact` content | M1-02 |
| helm-controller signals "gave up" as `Stalled` with `RetriesExceeded` | M1-02 |
| OCM requires semver versions; how the Go library writes an `ociImage` into a CTF | M1-08 |
| kustomize `images` rewrites container fields only | M1-16 |
| connect-web sends `Connect-Protocol-Version` on every request | M1-12 |
| GitHub returns `github-authentication-token-expiration` for PATs | M1-07 |
| GitHub git-protocol limits at a few thousand applications | M2-05 |
| The OCM Go library accepts an external `crypto.Signer` | M3-04 |

## M2 makes the runtime contracts usable and opens shared landscapes

Each workstream also enables its `# M2+` fields in
[02](02-domain-model.md).

| ID | Workstream | Implements | Path steps |
| --- | --- | --- | --- |
| M2-01 | Image builder runner for `builder/v1`, the vector builder as first consumer | [07 builders](07-sources-and-releases.md#builders-turn-a-checked-out-source-into-ocm-resources) | 07 P6 |
| M2-02 | Machine auth with OIDC trust rules (D-API-04), external gate writers, override, `soak.konfidence.cloud` | [05 gates](05-promotion-and-rollout.md#gates-follow-adr-0039-through-one-write), [08 sessions](08-api-and-ui.md#sessions-for-people-short-lived-tokens-for-machines) | 05 P7, 08 P6 |
| M2-03 | Several waves, thresholds above zero, `source: {promotionConfig}` chaining, rollback, removal promotions, the `Application` delete check | [05 threshold](05-promotion-and-rollout.md#a-wave-succeeds-within-its-failure-threshold), [05 rollback](05-promotion-and-rollout.md#rollback-promotes-an-earlier-release), [05 removal](05-promotion-and-rollout.md#removal-is-a-slot-that-points-at-nothing) | 05 P5–6 |
| M2-04 | Shared landscapes: per-application policies, boundary controller, Flux install mode with lockdown flags, ESO bindings, `Thin`, image signature checks | [06 boundary](06-landscape-bridge-and-policy.md#a-boundary-approves-one-project-and-tenant-rbac-is-the-hard-guard), [06 secrets](06-landscape-bridge-and-policy.md#secret-requirements-bind-to-local-stores) | 06 P6, 03 P5 |
| M2-05 | Multiple sources, the unchanged-path check, `SSHDeployKey`, `GitHubApp` with webhooks | [07 sources](07-sources-and-releases.md#an-application-has-one-or-more-sources-each-with-its-own-credential), [07 polling](07-sources-and-releases.md#polling-detects-changes-webhooks-only-shorten-the-wait) | 07 P5–6 |
| M2-06 | Release with diff, Landscape and Audit screens; `AuditService` | [08 UI](08-api-and-ui.md#the-ui-is-three-screens-in-m1) | 08 P6, 07 P6 |
| M2-07 | Bridge key deny list, revocation, `trust repin --cp-key-fingerprint`, Postgres restore drill in CI | [04 compromise](04-trust-and-attestation.md#compromise-recovery-runs-on-the-landscape-never-from-the-control-plane), [01 recovery](01-control-plane.md#landscapes-keep-running-when-the-control-plane-is-gone) | 04 P5, 01 P8 |
| M2-08 | Bridge simulator: 1000 landscapes against 2 replicas, a full-transfer fan-out | [01 sizing](01-control-plane.md#sizing-for-1000-landscapes-fits-three-api-replicas) | 01 P7 |
| M2-09 | `blob.gc` and `statement.prune` | [01 recovery](01-control-plane.md#landscapes-keep-running-when-the-control-plane-is-gone) | none |
| M2-10 | Intent CRDs to `v1beta1`, once the acceptance test passes twice without a schema change | [02](02-domain-model.md) | none |

## M3 hardens scale, the bridge and the trust root

M2-08 results decide M3-01 and M3-02; skip any the simulator does not
justify.

| ID | Workstream | Trigger |
| --- | --- | --- |
| M3-01 | A fleet-wide transfer budget for `FetchBlob` | simulator shows egress or replica pressure |
| M3-02 | Incremental blob inventory | above about 100 000 blobs per landscape |
| M3-03 | Per-application apply, with a per-application delivery sequence in statements | measured head-of-line blocking |
| M3-04 | `CustomerCA` mode with offline root and `trust crl import`, plus the PKCS#11 signer | before any customer whose security office approves the trust root |
| M3-05 | Control-plane key transitions and bridge identity rotation | after M3-04 |
| M3-06 | Active/passive bridge replicas | failover time becomes a requirement (D-TOPO-01) |
| M3-07 | A bridge support window for raising the minimum bridge version | first raise of the minimum version |
| M3-08 | Read replicas for UI history | history load shows in Postgres |

M3-04 moves ahead of M2 if such a customer arrives first. The secret key
backend suffices for the acceptance test and for attached-only installs.

## Not planned yet

Each item has a design note in its topic doc but no milestone:

- Customer co-sign through `ApplicationPolicy.spec.requiredSignatures`.
  `Manual` acceptance covers four-eyes review until then.
- Per-chart target namespaces.
- Dependency mirror rewrites, an extra CA bundle and `HTTPS_PROXY` for build
  workers.
- GitHub Enterprise Server, semver-range refs, a shared fetch cache.
- Pause and resume, opt-in automatic rollback, cross-application ordering.
- Custom roles and opt-in four-eyes approval.
- CTF volume splitting and a TPM-backed bridge identity.
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
