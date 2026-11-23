# 0005. Two change APIs: additive and breaking

Status: accepted, 2026-11-23 (revised same day)

## Context
Both agents and humans write definitions. Some changes are safe; others break existing data, existing clients or both. The world above us has decisions to make about breaking changes; our job is to make them explicit.

## Decision
A change is **additive** if it cannot fail and cannot change the meaning of any existing row or any previously valid request. Anything else is **breaking**.

- `POST /m/{type}/changes` accepts additive changes only. A new required field with a default is additive: the default is **materialised** onto every existing entity on apply, as a new version attributed to the meta-transaction. Defaults are applied at write time when a field is omitted; changing a default later affects only future writes. Stored data always literally conforms.
- `POST /m/{type}/breaking-changes` accepts breaking changes. It requires a reason and a caller flag, may include a JSONata transformation (0006), is dry-run before confirmation, and reports what will break for both data and clients.
- **Removing a lifecycle state requires a state mapping** in the change, checked up front. A lifecycle change can never leave an entity in a state that no longer exists.
- Entities still invalid after a breaking change go onto a worklist. They must be fixed before any other write or any transition, including to a terminal state. The fix is `POST /e/{type}/{id}/repair`: allowed only on worklist entities, may set any field including lifecycle fields, must produce a valid entity, recorded with actor and reason.
- History is not rewritten; past versions stay valid under the definition in force at their valid time.
- v0 changes touch one entity type at a time.

## Consequences
- We must classify changes reliably and reject ones sent to the wrong API.
- Breaking changes are searchable, alertable and, later, privileged by path.
- Read-time defaults were rejected: they force null handling into every query, and changing a default would silently change the meaning of existing rows while being filed as additive.
- New states and enum values are additive because clients derive enumerations from the definition (0020).
