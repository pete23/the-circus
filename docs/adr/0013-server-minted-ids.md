# 0013. We always mint entity IDs; uniqueness is scoped to valid time

Status: accepted, 2026-11-23 (revised same day)

## Context
IDs can be minted by us or supplied by callers. Natural keys (email, employee number) still need enforcing, and in a bitemporal store "unique" needs a time scope.

## Decision
- We mint every entity ID as a UUIDv7. There is no separate identity block in the metamodel; natural keys are uniqueness constraints on attributes.
- A uniqueness constraint holds **at any instant of valid time**: two entities cannot share the value during overlapping valid periods. Reuse after one period ends is allowed. The single writer checks every write against currently-valid and future-valid rows.
- Idempotent creates use an `Idempotency-Key` header.

## Consequences
- IDs are uniform, time-ordered and never chosen by an agent.
- A future-dated write cannot create a collision that only appears later.
- Unique-across-all-time was rejected: people reuse email addresses, and erasure would make it incoherent.
