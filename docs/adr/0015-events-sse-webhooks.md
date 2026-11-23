# 0015. Events via SSE and webhooks

Status: accepted, 2026-11-23 (revised same day)

## Context
Business logic lives above us, so the apps there need to react to what happens to data.

## Decision
The single writer emits an event for every create, update, transition, repair, erasure and model change. Offer both an SSE stream with a resumable cursor and registered webhooks with retries.

## Consequences
- Webhooks need delivery guarantees, signing and subscription management (open).
- Events are replayable from the log; after an erasure, replayed events simply lack the erased field (0019).
