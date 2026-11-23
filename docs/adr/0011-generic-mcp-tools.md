# 0011. Generic MCP tools with per-type schemas

Status: accepted, 2026-11-23

## Context
One MCP tool per type and operation gives the best typing, but the tool count grows with the model and fills agents' context.

## Decision
Expose a fixed set of generic tools that take an entity type. Publish each type's generated schema as an MCP resource.

## Consequences
- Tool count is constant.
- Agents must read a type's schema before writing to it; tool descriptions should say so.
