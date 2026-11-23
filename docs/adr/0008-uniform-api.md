# 0008. A uniform API at /e/{type} and /m/{type}

Status: accepted, 2026-11-23

## Context
Entity types are the unit of the contract. Agents and the apps they build benefit from every type looking the same.

## Decision
- Data plane at `/e/{type}`: query, create, read, update, transition (`/t/{transition}`), history, worklist.
- Meta plane at `/m/{type}`: definition, additive changes, breaking changes, generated OpenAPI.
- Same shape for every type; typed OpenAPI and MCP schemas are generated from definitions.
- No version in paths; definitions are versioned in time and clients pin by as-of.

## Consequences
- Privilege can later be enforced by path prefix before fine-grained rules exist.
- Generators stay simple because there is one shape.
