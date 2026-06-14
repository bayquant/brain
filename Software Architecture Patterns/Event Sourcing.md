---
tags: [software-architecture, event-sourcing, patterns]
---
# Event Sourcing

Event Sourcing is an architectural pattern where state changes are stored as a sequence of immutable events, rather than persisting only the current state. The application state is derived by replaying those events from the beginning (or from a snapshot).

---
## Core Concept

Instead of updating a record in place, every change is appended as a new event to an event log. The current state is always a projection of that log.

---
## Event Envelope vs. Event Payload

> **The key distinction is between the event envelope and the event payload.**

- **eEvent envelope** — the metadata wrapping every event: event ID, event type, timestamp, aggregate ID, version/sequence number, and correlation/causation IDs. The envelope is infrastructure-level; it enables routing, ordering, deduplication, and traceability.
- **Event payload** — the domain-specific data describing *what changed*: the actual fields and values meaningful to the business (e.g., `{ "orderId": "123", "amount": 49.99 }`). The payload is application-level.

Keeping them separate allows the infrastructure (brokers, stores, projectors) to handle envelopes generically without needing to understand domain semantics locked inside the payload.

---
## Benefits

- Full audit trail — every state transition is recorded
- Temporal queries — reconstruct state at any point in time
- Event-driven integration — other systems can subscribe to the event log
- Debugging — replay events to reproduce any past state

---
## Trade-offs

- Increased storage — the log grows indefinitely (mitigated with snapshots)
- Query complexity — current state requires projection or a read model ([[CQRS]])
- Schema evolution — event payloads must be versioned carefully as the domain changes

---
## Related Patterns

- [[CQRS]] — commonly paired with Event Sourcing to separate read and write models
- [[Saga Pattern]] — orchestrates multi-step processes using events
