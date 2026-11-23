# 0024. Nothing is written twice

Status: accepted, 2026-11-23

## Context
With the transaction history as the only record (0015), anything a reader needs must be recoverable from it. Comparing entity versions recovers creates, field changes and state changes, but not the transition name (`reject` and `withdraw` may both end in a terminal state), nor whether a change was a repair. Recording the request or an event alongside the data would fix that by writing everything twice.

## Decision
- Every entity version carries two system columns: `_circus/op` (`create`, `update`, `transition`, `repair`) and `_circus/transition`.
- A transition's input lands only on declared attributes: a transition declares the attributes it `sets` (e.g. `reject` sets `rejection-reason`). There are no free-standing payloads.
- The reason for a privileged change (retire, a breaking change) is provenance: one small row per transaction with actor, caller kind, reason and request ID, or XTDB transaction user-metadata if supported. Provenance never copies the request.
- Refusals are not history; they go to the structured application log.

## Consequences
- Every change is fully described by the version it produced plus its transaction's provenance.
- Transition inputs are visible in ordinary entity history, and subject to erasure like any attribute.
- Definitions must declare attributes for anything a transition records.
