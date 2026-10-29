# Members exchange events over HTTP, and the broker stays a provider

This page proposes how [members](../glossary.md#member) of a [vector](../glossary.md#vector) communicate asynchronously.
It is for the engineers choosing the async contract.
After reading, you can decide between the three options below.

## The constraint: no member code may know the broker

Two things must not leak into member code: the broker product and the broker client library.
If either leaks, swapping NATS for Kafka means rewriting members, and that is a lock-in.
The test for any design: change the [broker provider](../glossary.md#broker-provider) and redeploy, with no member code or image change.

## Three options

| Option | How members publish and receive | Lock-in | Equal across runtimes |
|--------|----------------------------------|---------|----------------------|
| A. One mandated broker | Native client of that broker | Broker | Only where the client runs |
| B. Language SDK | An SDK per language hides the broker | SDK per language | No: WebAssembly and functions need their own SDK builds |
| C. Protocol contract | CloudEvents over HTTP to and from the [runtime adapter](../glossary.md#runtime-adapter) | None at member level | Yes: every runtime serves and calls HTTP |

Option C is the recommendation.
Option A is the fallback if the provider model proves too expensive to build.

## Option C in detail

### Members publish by POSTing a CloudEvent to their adapter

A member sends a CloudEvent to a fixed local address the adapter serves.
The adapter checks that the member's vector exports that topic, then hands the event to the broker provider.
The member gets a 2xx once the broker has accepted the event.

### Members receive by serving an HTTP endpoint

The adapter pulls or receives from the broker and pushes each event to the member's subscription endpoint.
A 2xx acknowledges the event.
Any other status, or a timeout, means redelivery with backoff.
Push delivery is the shape function platforms already use, so functions, pods and WebAssembly members behave identically.

### CloudEvents is the envelope

CloudEvents is a CNCF graduated project since 2024-01-25.
It has released protocol bindings (v1.0.2) for HTTP, Kafka, AMQP, MQTT and NATS.
Source: [github.com/cloudevents/spec](https://github.com/cloudevents/spec), checked 2026-10-29.
Those bindings let a broker provider map the envelope to its native format without inventing one.

## Topics belong to the publishing vector

A topic is an [export](../glossary.md#export) of the vector that publishes to it.
The publisher owns the topic's name, event types and schema.
A subscriber declares an [import](../glossary.md#import) of the topic, and the same policy that governs sync calls allows or denies it.
Each subscription owns its dead-letter topic, so one slow consumer never affects another.

## Guarantees are named profiles, and brokers declare which they support

A topic declares one [delivery profile](../glossary.md#delivery-profile).
A broker provider declares the profiles it can serve.
Placement refuses a topic on a broker that cannot serve its profile, the same way runtime capabilities work.

| Profile | Guarantees |
|---------|-----------|
| `at-least-once` | Every accepted event reaches every subscription at least once. No ordering. |
| `ordered-per-key` | `at-least-once`, plus order per partition key. |
| `replayable` | `ordered-per-key`, plus retention, so a new subscription can start from the beginning. |

Exactly-once delivery is not a profile.
Subscribers deduplicate by the CloudEvent `id` and `source`.
Inference, not verified: the CloudEvents specification requires that pair to be unique per event.

## The adapter controls flow, so members do not need to

The adapter limits concurrent deliveries per subscription and retries with backoff.
After a configured number of attempts, it moves the event to the subscription's dead-letter topic.
A member signals overload by answering 429 or 503, and the adapter slows down.

## Prior art does most of this already

Inference, not verified: Knative Eventing delivers CloudEvents over HTTP push through Broker and Trigger objects, and Dapr abstracts brokers behind pub/sub components.
Konfidence differs in scope, not mechanism: topics are vector exports under the same policy as endpoints, and all runtimes pass one conformance suite.
Before writing broker providers, check whether Dapr's components can serve as their implementation.
TODO(fkasper): decide build or reuse after a spike.

## What option C gives up

- Broker-specific features such as Kafka transactions or NATS request-reply stay outside the contract.
- One extra hop through the adapter on publish and on delivery.
- Pull-based consumption with member-side batching. A member that needs it must use a declared escape hatch, which is a capability gap.
