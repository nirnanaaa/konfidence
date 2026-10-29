# Build the contract first, then prove equality with two runtimes and two brokers

This page proposes the build order for konfidence as a runtime-only framework.
It is for the engineers planning the first slice.
After reading, you know what to build first and which questions block it.

## The first slice proves the two claims that matter

The direction stands or falls on two claims.
Every runtime joins a [vector](../glossary.md#vector) the same way.
The broker can change without touching member code.
The first slice proves both and nothing else.

## Build order

1. **Write the vector API, v0.**
   Objects: `Vector`, `Member`, `Export`, `Import`, `Provider`.
   Capabilities and [delivery profiles](../glossary.md#delivery-profile) are plain strings.
   Done when the reference vector below can be written in it.
2. **Write the [conformance suite](../glossary.md#conformance-suite) against the API.**
   It deploys a reference vector of two [members](../glossary.md#member).
   One serves an endpoint and publishes a topic; the other calls the endpoint and subscribes.
   It checks identity, registration, health, a sync call, async delivery and one policy denial.
   The suite exists before any adapter passes it.
3. **Build two [runtime adapters](../glossary.md#runtime-adapter): Kubernetes pods and one non-Kubernetes runtime.**
   The second runtime should be a function platform or a WebAssembly host, because a second container runtime proves nothing.
   Done when both pass the suite, and the reference vector also passes with one member on each runtime.
4. **Build two [broker providers](../glossary.md#broker-provider): NATS JetStream and Kafka.**
   Done when the suite passes on both, with identical member images.
5. **Wire one [gateway](../glossary.md#gateway) provider.**
   One Gateway API implementation exposes an endpoint export to external clients.

Policy beyond allow and deny, multi-cluster [mesh](../glossary.md#mesh) federation and more providers come after step 5.

## What the first slice deliberately leaves out

- A UI. The API and the suite status are the interface.
- A proxy of our own. The Kubernetes adapter reuses an existing mesh data plane.
- Packaging and delivery of members. Members arrive by whatever means the runtime uses today.

## Open questions

These change the design, so they need an answer before step 1 ends.

1. **What is a vector to you now?**
   This branch assumes a vector is a capability boundary: one identity, many members, many runtimes.
   If a vector is closer to a deployable artefact or a dataflow graph, the model changes.
2. **Which runtimes are in scope for v1?**
   The candidates here are Kubernetes pods, a function platform, a WebAssembly host and VMs.
3. **Is async inside a vector, between vectors, or both?**
   This branch assumes both, with one policy.
4. **Does sovereignty still constrain the design?**
   If every mesh must run disconnected, broker providers and the control plane run per landscape.
   Federation then becomes a later problem.
5. **Build or reuse the mediation layer?**
   Dapr and Knative Eventing cover much of the async path.
   A spike in step 4 decides whether konfidence wraps them or replaces them.
6. **Who ships members onto runtimes?**
   Without an orchestrator, something else places members.
   Either the runtime's own tooling does it, or the `main` branch's orchestrator delivers them.
