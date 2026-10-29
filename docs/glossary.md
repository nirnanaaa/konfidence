# Glossary

One definition per term. Pages link the first use of a term to its entry here.

## vector

A named, versioned unit of capability with one identity and one policy boundary.
Consumers address the vector, never its members.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## member

A workload that implements part of a vector and runs on one runtime.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## runtime

Anything that executes a member: a Kubernetes pod, a WebAssembly host, a function platform, a VM.
Used in: [runtime model](explore/01-runtime-model.md), [path](explore/03-path.md).

## runtime adapter

The component that joins members of one runtime to the mesh and fulfils the member contract for them.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## member contract

The fixed set of obligations every member meets, whatever its runtime: identity, registration, health, ingress and egress through the mesh.
Used in: [runtime model](explore/01-runtime-model.md), [path](explore/03-path.md).

## export

A capability a vector offers to others: an endpoint (sync) or a topic (async).
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md).

## import

A vector's declared dependency on another vector's export. Policy allows or denies it.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md).

## provider

A pluggable implementation of one contract slot: gateway, broker, identity or runtime adapter.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## gateway

A provider that turns external traffic into calls on a vector's exports.
Used in: [runtime model](explore/01-runtime-model.md), [path](explore/03-path.md).

## broker provider

A provider that stores and delivers events for topics, for example NATS JetStream or Kafka.
Used in: [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## delivery profile

The named set of delivery guarantees a topic needs: ordering, retention, delivery count, dead-letter handling.
Used in: [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## mesh

All vectors plus the identity, routing and policy fabric between them.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md), [path](explore/03-path.md).

## conformance suite

Tests every provider and runtime adapter must pass. It turns "all runtimes are equal" into a check.
Used in: [runtime model](explore/01-runtime-model.md), [async communication](explore/02-async-communication.md), [path](explore/03-path.md).
