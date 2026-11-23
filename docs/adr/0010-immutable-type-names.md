# 0010. Entity type names are stable, with aliases

Status: accepted, 2026-11-23

## Context
An entity type's name is its URL path segment, so it is part of every client's contract.

## Decision
Names are lowercase kebab-case. A rename sets a new `name` on the `entity` row and appends the old one to its `aliases` (0021); an alias path answers `308 Permanent Redirect` to the current name for a declared period. Storage is keyed by the internal stable type ID, so no data moves.

## Consequences
- Old clients keep working during the redirect period.
- We must decide how long redirects last and who decides.
