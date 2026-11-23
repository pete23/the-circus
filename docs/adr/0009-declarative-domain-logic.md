# 0009. Declarative-only domain logic

Status: accepted, 2026-11-23

## Context
Low-code platforms tend to grow a scripting layer, and then become an application server.

## Decision
Inside the-circus, domain logic is declarative only: types, required fields, ranges, lengths, patterns, uniqueness, referential integrity and transition guards as simple predicates. Richer logic lives in the apps built on top, which react to the history feed (0015). The one exception is JSONata transformations (0006).

## Consequences
- Definitions stay readable by humans and checkable by us.
- Some rules will have to live above us; that is the intended boundary.
