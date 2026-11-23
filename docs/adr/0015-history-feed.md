# 0015. The history feed, not events

Status: accepted, 2026-11-23. Replaces "Events via SSE and webhooks".

## Context
Apps above us need to know what changed: live UIs, reactive business logic, sync jobs, agents waiting on a condition, clients watching definitions. Every one of them asks the same question, "changes since cursor X, for the types I care about". The XTDB transaction history already records everything that occurs; it is the keystone of the system.

## Decision
- No event subsystem and no event records. Changes are read from the transaction history.
- The history query gains two wider scopes: `GET /e/{type}/history?after={tx}` and `GET /history?after={tx}&type=…`, alongside the per-entity one. The cursor is the transaction ID.
- SSE is `Accept: text/event-stream` on the same URLs; long-poll is `?wait=`. Clients hold their own cursors.
- After a commit the writer signals waiting readers to look again. The signal carries nothing, and the writer never waits on readers.
- Webhooks are deferred. If a consumer must be pushed to, a dispatcher is one more reader of the feed with its own cursor.

## Consequences
- Durability, ordering, replay and restart come from XTDB; there is no second record to drift.
- We hold no subscriber state.
- What cannot be derived by comparing versions (operation kind, transition name) is recorded on the version itself (0024).
