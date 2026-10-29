# Design docs for the konfidence deployment orchestrator

These docs define the target design of konfidence and the path to it. They
serve a principal engineer who decides and the coding agents who implement.
[ADR-0001](../adr/0001-deployment-orchestrator.md) records why konfidence
became a deployment orchestrator.

## One file per concern

| File | Holds |
| --- | --- |
| [decisions.md](decisions.md) | Every design decision as a stable `D-<AREA>-<NN>` ID. Docs refine it, never contradict it. |
| [00-overview.md](00-overview.md) | The model in five sentences, the runtime boundary, the three runtime contracts, non-goals and the glossary. |
| [01-control-plane.md](01-control-plane.md) | Modes, River job kinds and queues, the delivery job, held `Sync` calls, sizing and recovery. |
| [02-domain-model.md](02-domain-model.md) | Every CRD field, status enum, reason string, Postgres table, the `NOTIFY` channel and the status function. |
| [03-bundles-and-transfer.md](03-bundles-and-transfer.md) | Bundle and delivery format, blob layout, `Sync` blob plan, images, offline files, Flux handover. |
| [04-trust-and-attestation.md](04-trust-and-attestation.md) | Keys, enrollment, compromise recovery, CRL, signed statements and the two counters. |
| [05-promotion-and-rollout.md](05-promotion-and-rollout.md) | Promotions, waves, gates, the slot compare-and-set, entry matching, rollback and removal. |
| [06-landscape-bridge-and-policy.md](06-landscape-bridge-and-policy.md) | Bridge pod, Flux integration, the one reconcile, policies and boundaries. |
| [07-sources-and-releases.md](07-sources-and-releases.md) | Git credentials, polling, the build-input manifest, release IDs and the builder contract. |
| [08-api-and-ui.md](08-api-and-ui.md) | Proto names and the only proto sketch, auth, roles and the UI. |
| [09-roadmap.md](09-roadmap.md) | Milestones M1 to M3, the M1 scope, workstreams and the first five tasks. |

## Read in this order

1. [ADR-0001](../adr/0001-deployment-orchestrator.md) for the change of
   course.
2. [00-overview.md](00-overview.md) for the model and the glossary.
3. [decisions.md](decisions.md) for the binding rules.
4. [09-roadmap.md](09-roadmap.md) for what ships when.
5. The topic docs your task touches, plus [02](02-domain-model.md) for names.

## Change a decision in decisions.md first

1. Edit the decision in [decisions.md](decisions.md) as its own change. Keep
   its ID; a new decision gets the next free ID in its area.
2. Add one line under "Changes after simplification" at the end of
   decisions.md, with the date.
3. Update every doc that cites the ID, then grep for the ID to check.
4. If the change moves scope between milestones, update
   [09-roadmap.md](09-roadmap.md).

A doc that disagrees with decisions.md is a bug in the doc.

## Every name has one owner doc

| Names | Owner |
| --- | --- |
| CRD fields, status enums, reason strings, Postgres tables, the `NOTIFY` channel | [02-domain-model.md](02-domain-model.md) |
| River job kinds and queues | [01-control-plane.md](01-control-plane.md) |
| Proto names, and the only proto sketch | [08-api-and-ui.md](08-api-and-ui.md) |

Topic docs use these names and carry no YAML or proto sketches of their own.
CRD enums are PascalCase (`Degraded`). Proto enums use the prefixed
UPPER_SNAKE form (`HEALTH_DEGRADED`) and map 1:1 to the CRD enum. If a name
is missing, add it to its owner doc first.

## M-tags set scope

[09-roadmap.md](09-roadmap.md) owns the milestone list and the M1 scope.
Every doc tags each v1 step `M1` or `M2+`; fields carry `# M2+` in 02. No
doc claims the acceptance test passes on a subset of M1. When docs disagree
on a tag, 09 decides.

## Rules for agents that implement from these docs

- **One workstream per task.** Take one workstream ID from the roadmap. Its
  linked sections and Path steps are the spec.
- **Build only your milestone.** Steps and fields tagged `M2+` wait.
- **Specs stay fixed.** Don't edit a doc or the acceptance test to make code
  pass. A spec change is a separate, stated change.
- **Unverified stays unverified.** Text marked "unverified", "inference" or
  `TODO(<who>): verify` is a hypothesis. Check it before you code against it.
