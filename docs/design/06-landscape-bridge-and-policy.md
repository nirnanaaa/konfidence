# Landscape: bridge, policy and tenancy

The bridge is the only konfidence component in a landscape. It turns each
entry of a verified delivery into a cluster-scoped `Bundle` and its Flux
objects under operator policy, with one reconcile per application. Flux
deploys; the bridge never runs Helm actions itself. Transfer lives in
[03](03-bundles-and-transfer.md), keys and statements in
[04](04-trust-and-attestation.md), names in [02](02-domain-model.md).

## One bridge pod runs every loop

In v1 the bridge is one replica with `Recreate` strategy, a Lease, and one
ReadWriteOnce volume shared by blob store and artifact server (D-TOPO-01).

| Component | Job |
| --- | --- |
| link | `Sync` long-poll and `FetchBlob`, or state request file out and CTF in ([03](03-bundles-and-transfer.md)) |
| importer | verify and unpack deliveries and bundles, write `Bundle` objects |
| artifact server | serve chart tarballs in-cluster for `ExternalArtifact` URLs |
| flux manager | install or adopt Flux, run the behavioural probe |
| boundary controller | namespaces, deployer ServiceAccount, Role and quota per `Boundary` (M2+) |
| application controller | the one reconcile per application |
| statement builder | the signed statement at each application's reporting level ([04](04-trust-and-attestation.md)) |
| admin endpoint | serve the bridge CLI (D-LAND-06) |

On node loss, reattaching the volume can take minutes. Imports, statements
and reconciles pause; running workloads are unaffected. The link retries with
exponential backoff and full jitter from 1 s to 5 min. Bridge releases ship as
a signed chart plus image set the operator imports; deliveries never carry
them.

## The bridge CLI talks to an admin endpoint, never through exec

The `konfidence-bridge` CLI reaches a ClusterIP admin endpoint through
`kubectl port-forward` or an exposure the operator chooses. The endpoint
checks the bearer token with TokenReview and each verb with
SubjectAccessReview on `bridgeoperations.konfidence.cloud`.

| Shipped Role | Verbs | Used for |
| --- | --- | --- |
| `konfidence-bridge-courier` | `export`, `import` | state request file out, CTF in |
| `konfidence-bridge-admin` | `enroll`, `trust`, `accept` | enrollment, `trust repin`, `trust crl import`, local acceptance |

No operator task needs `exec` or read access to the bridge's Secrets.

## The bridge adopts a Flux that passes a behavioural probe

At start and whenever the helm-controller Deployment changes, the flux
manager picks a mode. **Adopt (M1):** Flux CRDs exist; the bridge never edits
that Flux and reports `FluxManaged=False`. **Install (M2+):** no Flux CRDs
exist; the bridge applies its pinned Flux with the `ExternalArtifact` feature
gate and `--no-cross-namespace-refs=true`, `--default-service-account=default`
and `--no-remote-bases=true`
([multi-tenancy](https://fluxcd.io/flux/installation/configuration/multitenancy/)).

The `ExternalArtifact` default differs between Flux versions
([options](https://fluxcd.io/flux/components/helm/options/)), so the probe
tests behaviour, not flags. The bridge publishes an image-free probe chart as
an `ExternalArtifact` and `HelmRelease` in its own namespace. `Ready` sets
`FluxReady=True`; otherwise every `Bundle` reports `FluxUnavailable`.
Adopted Flux without the lockdown flags gets `FluxLockdown=False` as a
warning only, because tenants have no rights on Flux kinds (D-TEN-03).

## One reconcile per application covers deploy, tombstone, deny and dropped charts

The application controller compares two sets and closes the gap:

- **Wanted charts:** the charts of the entry's bundle. A tombstone or
  `action: Deny` wants none.
- **Owned objects:** the `HelmRelease`s, `ExternalArtifact`s and values
  Secrets the `Bundle` owns through owner references (D-LAND-04).

It writes every wanted chart and deletes every owned object it no longer
wants, `HelmRelease`s in reverse `dependsOn` order, while reporting
`Removing`. With nothing wanted and nothing owned it reports `Removed` for a
tombstone and `Refused` for a deny (D-LAND-05). Helm keeps CRDs and
StatefulSet PVCs. A tombstoned `Bundle` keeps its object.

`Failed` means Flux has given up on a `HelmRelease` after its retries and
remediation; `Progressing` never fails. Inference: helm-controller shows this
as `Stalled` with reason `RetriesExceeded`. TODO(bridge): verify against the
helm-controller API docs.

Only an explicit entry changes what runs. An application missing from a
delivery keeps running, so a faulty delivery never wipes a landscape
(D-PROMO-04). A locally deleted `Bundle` comes back: the finalizer
strips owner references, and the importer recreates it from the last applied
delivery and re-adopts the Flux objects.

## Each wanted chart becomes one ExternalArtifact and one HelmRelease

The `ExternalArtifact` (`source.toolkit.fluxcd.io/v1`) points at the artifact
server; the `HelmRelease` references it by `chartRef`
([ExternalArtifact](https://fluxcd.io/flux/components/source/externalartifacts/)).
Both live in the `targetNamespace`, because `--no-cross-namespace-refs` allows
references only within one namespace. Chart order becomes `dependsOn` within
one `Bundle`, never across bundles (D-LAND-03).

| `HelmRelease` field | Value | Why |
| --- | --- | --- |
| `serviceAccountName` | the `Boundary`'s deployer ServiceAccount | Helm runs with tenant RBAC, never helm-controller's |
| `install.crds`, `upgrade.crds` | `Skip`, or the chart's `crds` once the `Boundary` approves its CRDs | CRDs need approval (D-TEN-01) |
| `valuesFrom` | one Secret with the effective values | values never sit in `HelmRelease` specs |
| `postRenderers` | kustomize `images`, digest-pinned to the landscape registry | image relocation (D-XFER-06) |
| labels | `konfidence.cloud/project`, `konfidence.cloud/application` | selection by verified identity |

With `images.transfer: Full` the bridge pushes the transferred image blobs
into the operator's registry under a per-project path (D-XFER-04). `Thin`
(M2+) checks that every digest exists there. Without a registry, an
application with images stops with `RegistryNotConfigured`.

## Admission uses the bundle's build-time metadata

Before writing a `HelmRelease`, the reconcile checks the bundle's rendered
kinds and `crds/` ([03](03-bundles-and-transfer.md)):

- Unapproved cluster-scoped kinds, CRDs, webhooks, or namespaced kinds outside
  the base set stop the application with `WaitingForBoundaryApproval`.
- Flux kinds, `landscape.konfidence.cloud` kinds, or namespaces outside the
  `Boundary` stop it with `OutsideBoundary`.
- Images outside `images.registries` stop it with `ImageRuleViolation`.

Kinds that appear only with landscape values meet the tenant Role at apply
time and fail as `Failed` with Flux's RBAC message.

## Exactly one policy applies, laid over the default

`BundleDeploymentPolicy` belongs to the landscape operator and wins (D-POL-01). M1
ships the `default` policy only.

1. The bridge sets `konfidence.cloud/project` to the project UID and
   `konfidence.cloud/application` from the verified bundle identity, and drops
   every `konfidence.cloud/` label a bundle carries (D-POL-04).
2. It finds the `Boundary` whose `spec.project` is that UID (D-TEN-02). None
   means `WaitingForBoundaryApproval`; the entry waits stored.
3. It evaluates the selectors of policies with that `boundaryRef` only.
4. Two matches stop the application with `AmbiguousPolicy`. Nothing merges.
5. The effective policy is `default` with the match laid over it, field by
   field.

Without a `default` policy the bridge uses `Allow`, `updates: Auto`, the
`Boundary`'s first namespace, `Full` and `reporting: Health`. It hashes the
effective policy into the policy version (D-POL-06).

## Values merge in four layers, last wins

Chart defaults, then bundle values, then promotion-target values from the
entry, then policy `values` (D-POL-02). The last two may set only keys on the
entry's `overridable` list. The control plane checks layer 3 when it creates
the promotion, the bridge checks layer 4. A key outside the list, or a result
that fails the chart's values schema, stops the application with
`ValuesRejected`. Statements carry only `effectiveValuesDigest`.

## Secret requirements bind to local stores

Each `secretBindings` entry binds one requirement of the bundle (D-POL-03).
`secretRef` names a Secret in the `targetNamespace`; the bridge checks its
keys and never copies Secrets. `secretStoreRef` plus `remoteKey` (M2+) makes
the bridge write an `ExternalSecret` (`external-secrets.io/v1`)
([ESO](https://external-secrets.io/latest/api/externalsecret/)). A missing
binding or key stops the application with `UnboundSecret`.

## The reporting level filters each statement entry at runtime

The statement builder applies the effective `reporting` level every time
(D-POL-05). `None` sends the name and level only. `Health` adds
`entryDigest`, `deliverySequence`, the `Ready` reason, health, policy version
and `effectiveValuesDigest`. `Resources` adds the rendered resources with
their readiness, never a `status` subtree. The control plane treats missing
data as not yet known ([05](05-promotion-and-rollout.md)).

## The operator can hold changes for local acceptance

Under `updates: Manual` (D-POL-07) the bridge stores every delivery but
applies a changed entry, tombstones included, only after:

```sh
konfidence-bridge accept <PROJECT>.<APPLICATION> <ENTRY_DIGEST>
```

Until then the application keeps what it runs, `status.pendingEntry` names
the waiting digest, and the reason is `AwaitingLocalAcceptance`. `accept`
writes `status.acceptedEntry`; a different digest waits again. `Manual` on
`default` holds the whole landscape, also after a control-plane key
compromise (D-TRUST-03). The control plane can never accept.

## A Boundary approves one project, and tenant RBAC is the hard guard

The landscape admin approves a project by writing a `Boundary` with its
project UID (D-TEN-01). Per `Boundary` the bridge sets up the namespaces, one
deployer ServiceAccount for every `HelmRelease`, a konfidence-defined Role in
each namespace, and a ResourceQuota.

The Role covers core workload, config and networking kinds, `apps`, `batch`,
`autoscaling`, Roles, RoleBindings and Leases, plus `approvedNamespaced`. It
never includes Flux or `landscape.konfidence.cloud` kinds or the `bind`,
`escalate` and `impersonate` verbs, and is never `edit` or `admin`
(D-TEN-03). Cluster-scoped kinds, CRDs and webhooks wait in
`status.pendingClusterScoped` until the admin copies them into
`approvedClusterScoped`. In M1 the operator writes the `Boundary` and the
deployer objects from example manifests; the boundary controller automates
them in M2+.

## Path

1. **M1** Bridge pod, admin endpoint and shipped Roles; Flux adopt mode with
   the behavioural probe.
2. **M1** The one reconcile: charts with `dependsOn`, values Secret, image
   post-renderer, `Full` transfer, tombstones, deny, dropped charts, recreate
   on local delete.
3. **M1** `default` policy: values layers, `secretRef`, `updates: Manual`
   with `accept`.
4. **M1** Hand-written `Boundary` matched by project UID; admission.
5. **M1** Statement builder with all three levels.
6. **M2+** Per-application policies, ESO, `Thin`, image signatures, boundary
   controller, Flux install mode.

Later: active/passive bridge replicas, per-chart target namespaces.
