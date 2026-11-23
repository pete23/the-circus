# 0016. Query API scope

Status: accepted, 2026-11-23

## Context
With no SQL access (0012), the query API carries every read.

## Decision
- Simple queries through `GET /e/{type}` parameters.
- Structured queries through `POST /e/{type}/query`: JSON in our own vocabulary with filters, following references, sorting, paging, both time axes, and `count` and `group-by` aggregation.
- Queries compile to XTQL. No SQL or XTQL is accepted from callers.
- Unsupported operations are refused clearly.

## Consequences
- We own the query vocabulary and must grow it carefully.
