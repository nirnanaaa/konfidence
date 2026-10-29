# A vector is a contract, and every runtime joins it the same way

This page proposes the core model for konfidence as a runtime-only framework.
It is for the engineers deciding whether to build it.
After reading, you can judge the model and argue with its five rules.

## Konfidence owns the contract, providers own the data plane

Konfidence ships a specification, a small control plane and a [conformance suite](../glossary.md#conformance-suite).
It does not ship its own proxy, broker or function host.
Every data-plane piece is a [provider](../glossary.md#provider) that implements one contract slot.

The pattern is the one Kubernetes uses for storage and networking: a stable interface, many implementations, conformance tests.
A runtime framework that writes its own proxy competes with Envoy, Istio and Linkerd.
A framework that defines the vector contract competes with nobody and composes with all of them.

## The five rules

1. **Consumers address a [vector](../glossary.md#vector), never a [member](../glossary.md#member).**
   A vector has one name, one workload identity and one policy boundary.
   Members can move between runtimes without any consumer noticing.
2. **Every [runtime](../glossary.md#runtime) meets the same [member contract](../glossary.md#member-contract).**
   No runtime gets a feature the others cannot get.
   Where a runtime is physically limited, it declares a capability gap instead of getting a special path.
3. **A vector talks to the world only through [exports](../glossary.md#export) and [imports](../glossary.md#import).**
   An export is an endpoint or a topic.
   An import names another vector's export.
   Undeclared traffic does not leave the vector.
4. **Sync and async share one identity and one policy.**
   "Vector A may call B's endpoint" and "vector A may subscribe to B's topic" are the same kind of rule.
5. **[Gateways](../glossary.md#gateway) are providers, not the core.**
   An HTTP ingress, a gRPC gateway and a webhook-to-topic bridge are all ways into a vector's exports.
   Konfidence tells the gateway what to expose; the gateway reports its address back.

## The member contract has four obligations

| Obligation | What the member, or its adapter, provides |
|------------|-------------------------------------------|
| Identity | A workload identity bound to the vector, as a SPIFFE ID with mTLS |
| Registration | The member's vector, version and the exports it serves |
| Health | Readiness per export, so routing skips unready members |
| Mediated traffic | All ingress and egress passes the mesh, so policy applies |

A [runtime adapter](../glossary.md#runtime-adapter) fulfils the contract for members that cannot do it themselves.
A Kubernetes pod gets a node or sidecar proxy.
A function platform gets a proxy in front of the function and a credential the function presents on egress.
A WebAssembly host gets the contract as host functions.
The member code stays the same in all three cases: it serves HTTP or gRPC and calls exports by vector name.

## "Equal" is a test, not a promise

Every runtime adapter runs the same conformance suite.
The suite deploys one reference vector onto the runtime and checks all four obligations.
It checks sync calls in both directions, async publish and subscribe, and a policy denial.

A runtime that fails a check either fixes the adapter or declares the gap as a missing capability.
Vectors declare the capabilities they need, and placement refuses a runtime that lacks one.
Example: a function platform without long-lived connections declares `no-streaming`, and a vector exporting a gRPC stream cannot place a member there.

## Gateways implement exports the control plane describes

The control plane gives a gateway provider a list of exports, each with a protocol, a path or topic, and a policy.
The provider configures its own data plane and writes back an address and a status.

For Kubernetes-based gateways the natural interface is Gateway API: konfidence writes routes, and any conforming implementation serves them.
Inference, not verified: Gateway API also covers east-west traffic inside a mesh (the GAMMA initiative), which would make it the sync data-plane interface too.
TODO(fkasper): verify the GAMMA channel status before committing to it.

## The control plane stays small

It holds the vector registry, the export and import graph, policies, the provider registry, and observed member status.
It computes routing and policy and hands them to providers.
It never sits in the request path, so a control-plane outage stops changes, not traffic.

## What this model gives up

- A single hand-tuned data plane. Performance depends on the providers chosen.
- Runtime-specific shortcuts. A feature only one runtime could offer stays out of the contract.
- Zero-configuration local development. A developer needs at least one runtime adapter and one broker provider running.
