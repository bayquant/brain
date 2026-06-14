---
tags: [architecture, python, design-patterns, hexagonal]
---
# Ports and Adapters

**Port** — an interface that defines what operations are available, living inside the hexagon. It's the contract your domain exposes — technology-agnostic, no implementation details. Example: `UserRepository` as an interface.

**Adapter** — a concrete class that lives outside the hexagon and connects your app to the external world. It implements a port (if one exists) and contains the actual method bodies. Two flavors:
- Wraps external services/APIs — fetching market data, sending emails, processing payments
- Wraps databases — in which case it's usually called a repository instead

The key job of an adapter is translation — converting external data shapes into your domain model.

**Repository** — technically a specific kind of adapter, but named differently because "repository" is a well-understood pattern meaning data persistence. Use this name when the class talks to a database. Can have a port (interface) or not, depending on whether you need to swap implementations.

| Thing | Name |
|---|---|
| Interface defining operations | Port |
| Wraps a database | Repository |
| Wraps an external API/service | Adapter |

The port is the *what*, the adapter/repository is the *how*.
