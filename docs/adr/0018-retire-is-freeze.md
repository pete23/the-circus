# 0018. Retiring a type freezes it

Status: accepted, 2026-11-23

## Context
The meta API needs a way to end an entity type. Other types may reference it, and the store is append-only.

## Decision
`POST /m/{type}/retire` is a breaking change. Retire means **freeze**: no creates, updates, transitions or further definition changes, but reads, history, queries and inbound references keep working forever. Nothing is removed and the path stays live.

The change may include a mapping for inbound references (drop the field, or point it elsewhere); then the referencing types are updated in the same transaction. Without one, references keep pointing at the frozen type.

## Consequences
- The circus equivalent of `DROP TABLE` is a frozen, fully readable, attributed retirement.
- No dangling references are ever possible.
- "Reversible" in the business spec is softened to "recoverable"; a revert API is an open question.
