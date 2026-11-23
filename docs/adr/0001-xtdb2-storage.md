# 0001. XTDB2 as the storage engine

Status: accepted, 2026-11-23

## Context
The-circus promises that every change to data and to the model is recorded, attributable and readable as of any point in time.

## Decision
Use XTDB2. It is append-only and bitemporal (valid time and system time), stores data columnar on an object store, and supports erasure (`ERASE`) for personal data.

## Consequences
- History and time travel come with storage, not bolted on.
- XTDB has no uniqueness or foreign-key constraints; we enforce integrity ourselves (0007).
- XTDB accepts loosely typed documents; schema enforcement is ours (0003).
