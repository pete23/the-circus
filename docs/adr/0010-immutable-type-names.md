# 0010. Entity type names are immutable

Status: accepted, 2026-11-23

## Context
An entity type's name is its URL path segment, so it is part of every client's contract.

## Decision
Names are lowercase kebab-case and immutable. A "rename" adds a new name; the old path answers `308 Permanent Redirect` to the new one for a declared period. Storage is keyed by an internal stable type ID, so no data moves.

## Consequences
- Old clients keep working during the redirect period.
- We must decide how long redirects last and who decides.
