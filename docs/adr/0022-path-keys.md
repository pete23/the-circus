# 0022. Business keys in the path

Status: accepted, 2026-11-23. Amends 0013.

## Context
Every entity is addressable by its minted UUID. `/m/{type}` already addresses an `entity` row by `name`, a unique attribute, so business-key addressing exists for one type and should exist for all.

## Decision
- A definition may mark **one** single-attribute unique constraint as the `path-key`. `entity` marks `name`.
- `/e/{type}/{segment}` accepts either a UUID or a path-key value. A segment that parses as a UUID is an ID; anything else is a key. To keep that airtight, **path-key values may not be UUID-shaped**, enforced at write time.
- The segment is resolved to an ID first; everything hanging off the path (`/t/{transition}`, `/history`, `/repair`) works the same either way.
- Uniqueness holds per instant of valid time (0013), so a key may name different entities at different times. With `?valid-at=` the key resolves to the holder at that instant; without it, to the current holder.
- No current holder: `404` with a problem detail naming who last held the key and until when.
- References always store UUIDs. Responses always carry `id` and a canonical URL.
- Composite uniqueness constraints are not path-addressable; use the query.

## Consequences
- `/m/{type}` is no longer a special case; it is the path-key mechanism applied to `entity`.
- One constraint on the business: no UUID-shaped keys. Judged cheaper than an explicit `/by/{attr}/{value}` form.
- Skills nudge agents towards URL-safe key values.
