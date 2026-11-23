# 0003. Our metamodel is canonical

Status: accepted, 2026-11-23

## Context
OpenAPI, JSON Schema and XSD describe shapes, not business entities. They lack identity, lifecycles, reference ownership, time semantics, money and units, sensitivity and provenance.

## Decision
Define our own metamodel and treat it as the single source of truth. External formats are projections of it. In v0 we export OpenAPI only; no imports.

## Consequences
- We decide how data is modelled; callers work in our terms.
- Exports lose information (e.g. lifecycles become enums with notes); that is acceptable for clients.
- Imports, if added later, will produce incomplete definitions that must be completed before use.
