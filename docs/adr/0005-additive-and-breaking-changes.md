# 0005. Two change APIs: additive and breaking

Status: accepted, 2026-11-23

## Context
Both agents and humans write definitions. Some changes are safe; others break existing data, existing clients or both. The world above us has decisions to make about breaking changes; our job is to make them explicit.

## Decision
A change is **additive** if every previously valid entity and every previously valid request is still valid afterwards. Anything else is **breaking**.

- `POST /m/{type}/changes` accepts additive changes only.
- `POST /m/{type}/breaking-changes` accepts breaking changes. It requires a reason and a caller flag, may include a transformation (0006), is dry-run before confirmation, and reports what will break for both data and clients.
- Entities still invalid after a breaking change go onto a worklist. They must be fixed before any other write or any transition, including to a terminal state.
- History is not rewritten; past versions stay valid under the definition in force at their valid time.
- v0 changes touch one entity type at a time.

## Consequences
- We must classify changes reliably and reject ones sent to the wrong API.
- Breaking changes are searchable, alertable and, later, privileged by path.
- Fix-first is strict; it may be relaxed if it proves painful in the demo.
