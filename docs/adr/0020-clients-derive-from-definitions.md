# 0020. Clients derive enumerations from the definition

Status: accepted, 2026-11-23

## Context
Adding a lifecycle state or enum value keeps every stored entity and every old request valid, but a client with a hard-coded list of states breaks on an unknown value.

## Decision
Lifecycle states, enum values and defaults are part of the definition, and the definition is readable at any time (`GET /m/{type}`, as-of if needed). Clients populate dropdowns, switches and validations from the definition, not from constants. A client that hard-codes enumerations has broken the contract.

## Consequences
- New states and enum values are additive, full stop.
- Generated clients should fetch or embed the definition's enumerations rather than freeze them in code.
- The platform's skills say this to agents explicitly.
