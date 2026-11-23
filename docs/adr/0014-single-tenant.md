# 0014. Single tenant

Status: accepted, 2026-11-23

## Decision
One circus per deployment. There is no tenant in paths or hosts.

## Consequences
- Simpler model, storage and privilege.
- Isolation between organisations is a deployment concern.
