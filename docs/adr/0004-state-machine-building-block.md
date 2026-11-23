# 0004. State machines are a building block, and the easy path

Status: accepted, 2026-11-23 (revised same day)

## Context
Business data is full of status fields. Left as plain enums, any code can set any value, and the workflow exists only by convention. We cannot mechanically tell a status from a classification, and agents write the definitions.

## Decision
A state machine is a first-class building block: states, an initial state, terminal states, named transitions (each with an optional guard and the attributes it sets) and the machine's own attributes. An entity type attaches a machine to a field; the instance can only change through `POST /e/{type}/{id}/t/{field}/{transition}`, never through `PATCH`. Machines may be named and shared, or private and inline (0025).

A plain `enum` attribute also exists and is mutable through `PATCH`. We do not prohibit it. Instead we stack the incentives: a state machine gets named transitions, guards, transitions named in the history feed, a `history` that reads as a story, and generated per-transition operations in OpenAPI and MCP. An enum gets none of that. The platform's skills teach agents: if it changes and the change has a name, it is a state machine.

## Consequences
- Where the data implies a state machine, the easy thing to do is declare one.
- Determined flouters can hide state in an enum or an integer; we cannot help them and do not try.
- Guidance is part of the product: the MCP server ships skills alongside tools.
