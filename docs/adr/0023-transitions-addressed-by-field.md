# 0023. Transitions are addressed by field and name

Status: accepted, 2026-11-23 (revised same day: was "transition names are unique per type")

## Context
A type may attach several state machines, and may attach the same machine more than once (joint signatories: `signer-a` and `signer-b` both run `signature`). Two attachments can therefore offer a transition with the same name.

## Decision
Transitions are always addressed `POST /e/{type}/{id}/t/{field}/{transition}`. Transition names are unique within a machine, enforced at definition time. There is no short form, even for a type with a single machine.

## Consequences
- One URL shape, with no hidden rule that depends on how many machines a type has.
- Generated operationIds and history-feed entries use `{field}/{transition}`.
- The built-in types use it too: `/m/{type}/t/status/retire`.
