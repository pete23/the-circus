# 0019. Erasure covers what we store

Status: accepted, 2026-11-23

## Context
Attributes marked sensitive can be erased with XTDB `ERASE`. By then the value may have left through webhooks, SSE, dry-run samples or exports.

## Decision
Erasure covers what we store. Copies already delivered downstream are the recipient's obligation. We emit an `erased` event so subscribers can act. Events carry values; subscribers who receive sensitive attributes take on the duty that comes with them.

## Consequences
- Simple and honest; no redaction machinery in v0.
- After erasure, replayed events lack the erased field because the log no longer has it.
- Revisit redaction-by-default if subscribers prove careless.
