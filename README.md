# Konfidence as a runtime-only framework

This branch explores one direction for konfidence: a runtime framework without a deployment orchestrator.
It starts empty on purpose and shares no history or code with `main`.
Read it to decide whether this direction is worth building and what the first slice is.

The `main` branch explores the opposite direction, a standalone deployment orchestrator.
The two branches are alternatives.

## Pages

| Page | What you decide after reading |
|------|-------------------------------|
| [Runtime model](docs/explore/01-runtime-model.md) | What a vector is and how runtimes, providers and gateways relate to it |
| [Async communication](docs/explore/02-async-communication.md) | How members talk asynchronously without a broker lock-in |
| [Path and open questions](docs/explore/03-path.md) | The build order, the first slice, and what is still undecided |
| [Glossary](docs/glossary.md) | One definition per term |

## Status

Exploration, not a decision.
Statements about third-party projects are marked as verified (with source) or as inference.
