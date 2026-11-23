# 0013. We always mint entity IDs

Status: accepted, 2026-11-23

## Context
IDs can be minted by us or supplied by callers.

## Decision
We mint every entity ID as a UUIDv7. Natural keys are expressed as uniqueness constraints on attributes.

## Consequences
- IDs are uniform, time-ordered and never chosen by an agent.
- Idempotent creates need another mechanism (e.g. an idempotency key header).
