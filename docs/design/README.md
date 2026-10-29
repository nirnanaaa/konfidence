# Design docs for the konfidence deployment orchestrator

These docs define the target design of konfidence and the path to it. They
serve a principal engineer who decides and the coding agents who implement.
[ADR-0001](../adr/0001-deployment-orchestrator.md) records why konfidence
became a deployment orchestrator.

## One file per concern

| File | Holds |
| --- | --- |
| [decisions.md](decisions.md) | Every design decision as a stable `D-<AREA>-<NN>` ID. Docs refine it, never contradict it. |
| [00-overview.md](00-overview.md) | Target vision, runtime boundary, the three runtime contracts, non-goals and the glossary. |
| [01-control-plane.md](01-control-plane.md) | Processes, River job kinds and queues, bridge streams, sizing and recovery. |
| [02-domain-model.md](02-domain-model.md) | Every CRD field, enum, reason, Postgres table and the `NOTIFY` channel. |
| [03-bundles-and-transfer.md](03-bundles-and-transfer.md) | Bundle and delivery format, CTF, transfer planning, thin and full images. |
| [04-trust-and-attestation.md](04-trust-and-attestation.md) | Keys, enrollment, compromise recovery, CRL, signed reports and counters. |
| [05-promotion-and-rollout.md](05-promotion-and-rollout.md) | Promotions, waves, gates, `Deployment` transitions, rollback and removal. |
| [06-landscape-bridge-and-policy.md](06-landscape-bridge-and-policy.md) | Bridge, Flux integration, policies and tenant boundaries. |
| [07-sources-and-releases.md](07-sources-and-releases.md) | Git credentials, polling, release IDs and the builder contract. |
| [08-api-and-ui.md](08-api-and-ui.md) | Proto names, auth, roles and the UI. |
| [09-roadmap.md](09-roadmap.md) | Milestones M1 to M3, workstreams and the first five tasks. |

## Read in this order

1. [ADR-0001](../adr/0001-deployment-orchestrator.md) for the change of
   course.
2. [00-overview.md](00-overview.md) for the vision and the glossary.
3. [decisions.md](decisions.md) for the binding rules.
4. [09-roadmap.md](09-roadmap.md) for what ships when.
5. The topic docs your task touches, plus [02](02-domain-model.md) for names.

## Change a decision in decisions.md first

1. Edit the decision in [decisions.md](decisions.md) as its own change. Keep
   its ID; a new decision gets the next free ID in its area.
2. Add one line under "Changes after review" at the end of decisions.md
   with the date.
3. Update every doc that cites the ID, then grep for the ID to check.
4. If the change moves scope between milestones, update
   [09-roadmap.md](09-roadmap.md).

A doc that disagrees with decisions.md is a bug in the doc.

## Rules for agents that implement from these docs

- **Names have one owner.** Take CRD fields, enums, reasons, tables and the
  channel from [02](02-domain-model.md). Take River job kinds and queues from
  [01](01-control-plane.md), and proto names from [08](08-api-and-ui.md). If
  a name is missing, add it to its owner doc first (D-API-06).
- **M-tags set scope.** Build only steps tagged with your milestone. `M1`
  is the acceptance test scope; `M2+` and fields marked `# M2+` wait.
  [09-roadmap.md](09-roadmap.md) decides when the docs disagree.
- **One workstream per task.** Take one workstream ID from the roadmap. Its
  linked sections and Path steps are the spec.
- **Specs stay fixed.** Don't edit a doc or the acceptance test to make code
  pass. A spec change is a separate, stated change.
- **Unverified stays unverified.** Text marked "unverified", "inference" or
  `TODO(<who>): verify` is a hypothesis. Check it before you code against it.
