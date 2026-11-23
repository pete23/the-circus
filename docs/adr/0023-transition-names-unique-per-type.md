# 0023. Transition names are unique per type

Status: accepted, 2026-11-23

## Context
A type may declare several lifecycles. Two of them could each have a transition called `reject`, and `POST /e/{type}/{id}/t/reject` would be ambiguous.

## Decision
Transition names are unique across all lifecycles of a type, enforced at definition time as a metamodel integrity rule. `/t/{transition}` stays flat; each transition knows which lifecycle it moves.

## Consequences
- The URL is a verb on the entity; callers need not know which field it moves.
- Generated operationIds and event names stay flat.
- An agent wanting the same verb in two machines renames one, or more likely splits the entity. Two lifecycles that must move together are a batch, or a sign they are one machine.
