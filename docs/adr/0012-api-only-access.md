# 0012. API-only access for agents

Status: accepted, 2026-11-23

## Context
XTDB speaks the Postgres wire protocol, but we cannot yet constrain what a connected client can read or do.

## Decision
Agents and apps use the-circus API only. The Postgres protocol is reachable by operators on a private network, never by agents.

## Consequences
- The query API must be expressive enough that agents do not need SQL (0016).
- Revisit if XTDB can be constrained enough to expose safely.
