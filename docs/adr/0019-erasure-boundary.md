# 0019. Erasure covers what we store

Status: accepted, 2026-11-23 (revised same day)

## Context
Attributes marked sensitive can be erased with XTDB `ERASE`. By then the value may have left through the history feed, dry-run samples or exports.

## Decision
Erasure covers what we store. Copies already delivered are the recipient's obligation. Erasure is itself a change in the history feed, so readers can act on it.

## Consequences
- The feed renders from stored versions, so replaying it after erasure simply lacks the field. No redaction machinery.
- To verify: XTDB 2.x log retention, since the underlying log holds original transaction payloads. Erasure is only real if the log's retention is short and the object store holds the lasting copy.
