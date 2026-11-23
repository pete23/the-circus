# 0015. Events via SSE and webhooks

Status: accepted, 2026-11-23

## Context
Business logic lives above us, so the apps there need to react to what happens to data.

## Decision
Emit events for creates, updates, transitions and model changes, derived from the transaction log. Offer both an SSE stream with a resumable cursor and registered webhooks with retries.

## Consequences
- Webhooks need delivery guarantees, signing and subscription management (open).
- Events are replayable from the log.
