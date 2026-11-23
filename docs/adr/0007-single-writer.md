# 0007. A single writer

Status: accepted, 2026-11-23

## Context
XTDB2 has no uniqueness or foreign-key constraints, and breaking changes need a stable view of the data between dry run and application.

## Decision
In v0 one process serialises all writes, data and meta. It validates against the current definition (including uniqueness and references) and then submits.

## Consequences
- Simple and correct, and enough for the demo.
- Throughput is limited to one writer; revisit with optimistic checks inside transactions (e.g. `ASSERT`) once XTDB support is verified.
