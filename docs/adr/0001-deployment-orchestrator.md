# ADR-0001: Konfidence becomes a sovereign deployment orchestrator

| Field | Value |
| --- | --- |
| Status | Accepted |
| Date | 2026-10-29 |
| Author | nirnanaaa |
| Decision log | [docs/design/decisions.md](../design/decisions.md) |

Konfidence changes course from a product bundled with the vector runtime to
a standalone deployment orchestrator. Every landscape receives software
through one pull-based, offline-capable exchange. The
[design docs](../design/README.md) hold the details; this record states the
change and why.

## Context

The previous design in `konfidence-project/konfidence-docs` carried the
vector runtime in the same product: runtime API, UI, `Stage` and `Vector`
CRDs, and `VectorPromotion`. Three requirements no longer fit that shape:

- One control plane must deliver to 500–1000 landscapes (D-TOPO-02).
- Some landscapes are air-gapped. Their operators must keep the final say
  over what runs, even against a compromised control plane.
- Another team owns the vector runtime. It needs a stable integration
  surface, not shared Go types.

Prior ADRs 0006, 0007, 0019, 0027, 0032, 0039 and 0040 are the starting
point. Gates keep the ADR-0039 model.

## Decision

### Konfidence orchestrates deployments and nothing else

Konfidence is a standalone deployment orchestrator (D-PROD-01). A central
control plane turns each build-input manifest into a release and one signed
OCM bundle (D-REL-01, D-BUN-01). Gated wave promotions take releases to
attached and isolated landscapes. Flux is the only deployment engine on a
landscape (D-LAND-01).

### The runtime splits off behind three konfidence-owned contracts

The vector runtime leaves konfidence, which imports no runtime Go types
(D-PROD-02). To konfidence a `Stage` is one more manifest in a chart. The
runtime integrates only through three konfidence-owned contracts (D-PROD-03):

| Contract | Runtime uses it to | Owner doc |
| --- | --- | --- |
| Builder `builder/v1` | turn a source tree into extra chart resources | [07](../design/07-sources-and-releases.md) |
| Gate (ADR-0039) | hold or clear a wave through `SetGateState` | [05](../design/05-promotion-and-rollout.md) |
| Statement `statement.v1` | read per-application status from signed bridge statements | [04](../design/04-trust-and-attestation.md) |

The split is done when M1, the acceptance test, passes: konfidence deploys
`example-app` to one attached and one isolated landscape without a runtime
(D-PROD-04).

### One model: intent in CRDs, slots in Postgres, one exchange

- **Intent in CRDs, facts in Postgres.** People and GitOps declare six
  intent CRDs. Releases, promotions, gate states, slots, deliveries and
  statements are Postgres rows (D-TOPO-04, D-TOPO-07).
- **One desired slot per application and landscape.** A wave step writes
  the slot with one compare-and-set on promotion sequence. That predicate is
  the only supersede rule and the only fence (D-PROMO-06).
- **Removal is a slot that points at nothing.** It travels as a tombstone
  through the same gates and waves. Absence never deletes (D-PROMO-04).
- **One exchange, online and offline.** The bridge sends a signed
  statement; the control plane answers with the landscape's newest signed
  delivery, listing all its slots, and the blobs the bridge lacks. Online
  this is `Sync` over gRPC; offline it is a statement file out and a CTF in
  (D-API-05, D-XFER-05).
- **Status is derived** from the slot and the newest statement (D-PROMO-03).

### Transfer is sovereign by design

The control plane never connects into a landscape (D-PROD-05).

- **Pull only.** One bridge per landscape opens every connection. Its
  identity key is the landscape's only credential (D-XFER-01).
- **Local authority.** The operator's `ApplicationPolicy` and `Boundary`
  decide on the landscape (D-POL-01, D-POL-07, D-TEN-01).
- **Trust anchored on the landscape.** Trust is pinned on first use and
  confirmed by an enrollment code on both sides. Recovery from a key
  compromise is a local re-pin (D-TRUST-01 to D-TRUST-03).
- **Nothing sensitive crosses.** Git credentials and secrets stay where
  they are (D-SRC-01, D-POL-03).

## Consequences

### What gets easier

- The runtime team ships on its own schedule against versioned contracts.
- Air-gapped and connected landscapes run one code path, so the isolated
  path cannot rot.
- Supersede, fencing and removal are one predicate on one table.
- A compromised control plane can withhold deliveries but cannot override
  local policy or forge statements.

### What it costs

- Operators upgrade every bridge by hand, so `v1` stays additive-only
  (D-API-02).
- Every landscape needs Flux with `ExternalArtifact` enabled and an
  operator-provided image registry (D-LAND-01, D-XFER-04).
- `kubectl` cannot show promotions or rollout status; only the API can.
- A Postgres restore needs one manual step: restart the delivery sequence
  above the highest one any bridge reports (D-ATT-04).
- Enrollment and boundaries need human steps on the landscape.
- The v1 bridge is one pod; an outage pauses imports, never workloads.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| Keep the runtime inside konfidence | Couples two products and two teams through shared Go types (D-PROD-02). |
| Push from the control plane into landscapes | Needs inbound access that sovereign operators refuse and air-gapped landscapes cannot give. |
| Landscapes pull from an OCI registry | A second protocol and credential per landscape; the offline path would need its own code (D-XFER-01). |
| Landscapes sync from git with Flux | Git credentials would reach landscapes, and isolated landscapes cannot reach git (D-SRC-01, D-BUN-02). |
| Releases, promotions and deployments as CRDs | 100 000s of objects in etcd, a retention rule, status mirrors and a restore split-brain (D-TOPO-07). |
