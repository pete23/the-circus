# 0021. Definitions are entities

Status: accepted, 2026-11-23

## Context
The meta API and the data API grew separately and had drifted: bespoke endpoints for changes, retire and revive; no history or batch on the meta side; dry run only on the meta side. We asked where the two interfaces differ and whether every difference is justified.

## Decision
An entity type definition is an entity of a built-in type named `entity`. `/m/{type}` is a privileged route onto `/e/entity/...`, addressed by name, not a separate API.

- `entity` has a unique attribute `name`, and a lifecycle `active ⇄ retired` with transitions `retire` and `revive`, each taking a `reason` payload.
- Additive changes are `PATCH /m/{type}`. A patch whose diff is breaking is refused with a pointer to the breaking endpoint.
- Retire and revive are `POST /m/{type}/t/retire` and `/t/revive`.
- `GET /m/{type}/history`, `?valid-at=`, provenance, events, `Idempotency-Key` and uniqueness all come from the same engine.
- `/batch` accepts definition operations with temporary IDs, so mutually referencing types can be created in one transaction.
- `?dry-run=true` is accepted on every write, data and meta. The breaking endpoint requires it before confirm.
- The definition of `entity` itself (the metamodel) is fixed by the platform and changes only with a platform upgrade. The recursion stops there.

The two justified differences that remain:
1. A definition write's validation scope is every row of the type, not one row. That is why the breaking endpoint exists, with its transformation, dry run and confirm.
2. `/m` is a separate path prefix so privilege can be enforced by route.

## Consequences
- One engine, two routes. Less code, fewer generators, and the circus runs its own machinery against its own definitions from day one.
- Model-change events are ordinary events on `entity`.
- A rename is data: `name` changes and the old name goes into `aliases`, which drives the `308`.
