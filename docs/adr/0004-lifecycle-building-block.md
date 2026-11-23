# 0004. Lifecycle is a building block, and the easy path

Status: accepted, 2026-11-23 (revised same day)

## Context
Business data is full of status fields. Left as plain enums, any code can set any value, and the workflow exists only by convention. We cannot mechanically tell a status from a classification, and agents write the definitions.

## Decision
A lifecycle is a first-class building block: states, an initial state, terminal states and named transitions, each with an optional guard and the attributes it sets. A lifecycle field can only change through `POST /e/{type}/{id}/t/{transition}`, never through `PATCH`.

A plain `enum` attribute also exists and is mutable through `PATCH`. We do not prohibit it. Instead we stack the incentives: lifecycle gets named transitions, guards, transitions named in the history feed, a `history` that reads as a story, and generated per-transition operations in OpenAPI and MCP. An enum gets none of that. The platform's skills teach agents: if it changes and the change has a name, it is a lifecycle.

## Consequences
- Where the data implies a state machine, the easy thing to do is declare one.
- Determined flaunters can hide state in an enum or an integer; we cannot help them and do not try.
- Guidance is part of the product: the MCP server ships skills alongside tools.
