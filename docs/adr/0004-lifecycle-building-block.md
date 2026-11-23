# 0004. Lifecycle is a building block

Status: accepted, 2026-11-23

## Context
Business data is full of status fields. Left as plain enums, any code can set any value, and the workflow exists only by convention.

## Decision
A status-like field must be declared as a lifecycle: states, an initial state, terminal states and named transitions, each with an optional guard and payload schema. A lifecycle field can only change through `POST /e/{type}/{id}/t/{transition}`, never through `PATCH`.

## Consequences
- Where the data implies a state machine, we implement and enforce it.
- Transitions are addressable, recorded and emitted as events.
- Spotting status fields that should be lifecycles is a modelling responsibility (ours, via review of definitions), not inference.
