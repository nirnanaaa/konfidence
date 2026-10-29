# ADR-0001: Konfidence becomes a sovereign deployment orchestrator

| Field | Value |
| --- | --- |
| Status | Accepted |
| Date | 2026-10-29 |
| Author | nirnanaaa |
| Decision log | [docs/design/decisions.md](../design/decisions.md) |

Konfidence changes course from a product bundled with the vector runtime to
a standalone deployment orchestrator. The runtime becomes a separate product.
Every landscape receives software through one pull-based, offline-capable
transfer model. The design docs in [docs/design](../design/README.md) hold
the details; this record states the change and why.

## Context

The previous konfidence design lives in
`konfidence-project/konfidence-docs` under
`src/docs/extend-customize/decisions/`. It carried the vector runtime in the
same product: runtime API, UI, `Stage` and `Vector` CRDs, and
`VectorPromotion`.

Three requirements no longer fit that shape:

- One control plane must deliver to 500–1000 Kubernetes landscapes
  (D-TOPO-02).
- Some landscapes are air-gapped. Their operators must keep the final say
  over what runs, even against a compromised control plane.
- Another team owns the vector runtime. It needs a stable integration
  surface, not shared Go types.

The prior ADRs on OCM (ADR-0006, ADR-0019), multi-landscape delivery
(ADR-0007, ADR-0027) and promotions (ADR-0032, ADR-0039, ADR-0040) are the
starting point. Gates keep the ADR-0039 model.

## Decision

### Konfidence orchestrates deployments and nothing else

Konfidence is a standalone deployment orchestrator (D-PROD-01). A central
control plane snapshots git commits into releases and builds one signed OCM
bundle per release. It promotes releases in gated waves to attached and
isolated landscapes. Helm charts are the primary content, and Flux is the
only deployment engine on a landscape (D-BUN-01, D-LAND-01).

### The runtime splits off behind three konfidence-owned contracts

The vector runtime leaves konfidence (D-PROD-02). Konfidence imports no
runtime Go types. To konfidence a `Stage` is one more manifest in a chart.
The runtime integrates only through three contracts that konfidence owns and
versions (D-PROD-03):

| Contract | Runtime uses it to | Owner doc |
| --- | --- | --- |
| Builder `builder/v1` | turn a source tree into extra chart resources | [07](../design/07-sources-and-releases.md) |
| Gate (ADR-0039) | hold or clear a promotion through the API | [05](../design/05-promotion-and-rollout.md) |
| Status report | read signed per-resource status from landscapes | [04](../design/04-trust-and-attestation.md) |

The split is done when konfidence deploys and promotes `example-app` across
one attached and one isolated landscape with no runtime installed. That test
is milestone M1 (D-PROD-04).

### Transfer is sovereign by design

The control plane never connects into a landscape (D-PROD-05). The rules
that follow from it:

- **Pull only.** One bridge per landscape opens every connection. Its
  identity key is the landscape's only credential (D-XFER-01).
- **One model online and offline.** gRPC and CTF archives carry the same
  content-addressed blobs. The bridge runs the same verify-and-unpack code for
  both (D-XFER-01, D-XFER-03).
- **Signed declarative deliveries.** Each landscape gets a signed delivery
  listing its full desired set, ordered by a monotonic delivery sequence
  (D-BUN-04, D-ATT-02).
- **Local authority.** The operator's `ApplicationPolicy` decides on the
  landscape, including a manual brake per application. The control plane
  cannot override it (D-POL-01, D-POL-07).
- **Optional reporting.** Reports are signed by the bridge and capped at
  `none`, `health` or `resources` (D-POL-05, D-ATT-01).
- **Trust anchored on the landscape.** Trust is pinned on first use and
  confirmed by an enrollment code on both sides. Recovery from a key
  compromise is a local re-pin (D-TRUST-01 to D-TRUST-03).
- **Nothing sensitive crosses.** Git credentials stay on the control plane,
  and secrets stay on the landscape (D-SRC-01, D-POL-03).

## Consequences

### What gets easier

- The runtime team ships on its own schedule against versioned contracts.
- Air-gapped and connected landscapes run one code path, so the isolated
  path cannot rot.
- A compromised control plane can withhold deliveries but cannot override
  local policy, forge reports or roll counters back.
- Every deployed state is traceable to a release ID, a component digest and
  a signed report.

### What it costs

- Every landscape runs a bridge, and operators upgrade it by hand. The API
  stays additive-only within `v1`, with `buf breaking` in CI (D-API-02).
- Every landscape needs Flux with `ExternalArtifact` enabled and an
  operator-provided image registry (D-LAND-01, D-XFER-04).
- Enrollment and tenant boundaries need human steps on the landscape.
- The UI must show how it knows a state. Isolated landscapes show "attested
  at T", never plain healthy (D-UI-02).
- In v1 the bridge is a single pod. A bridge outage pauses imports and
  reports; running workloads keep running (D-TOPO-01).
- Git means github.com only for now (D-SRC-02).

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| Keep the runtime inside konfidence | Couples two products and two teams through shared Go types (D-PROD-02). |
| Push from the control plane into landscapes | Needs inbound access that sovereign operators refuse and air-gapped landscapes cannot give. |
| Landscapes pull from an OCI registry | A second protocol and credential per landscape; the offline path would need its own code (D-XFER-02). |
| Landscapes sync from git with Flux | Git credentials would reach landscapes, and isolated landscapes cannot reach git (D-SRC-01, D-BUN-02). |

## Links

- [Doc map and reading order](../design/README.md)
- [Decision log](../design/decisions.md)
- [Overview and product boundary](../design/00-overview.md)
- [Roadmap and M1 scope](../design/09-roadmap.md)
