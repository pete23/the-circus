# the-circus: tech spec

Status: draft, 2026-11-23. Decisions are recorded in [`adr/`](adr/); this document describes the system as those decisions add up.

## Overview

A single Clojure service in front of an XTDB2 node. Agents and apps talk to the service over HTTP (and MCP); nothing else talks to XTDB.

```
 builder agents / spec authors      apps, runtime agents
            │                              │
            ▼                              ▼
     meta API  /m/{type}            data API  /e/{type}
            │          MCP server          │
            └──────────────┬───────────────┘
                           ▼
              circus service (Clojure)
   metamodel · validation · lifecycles · change classifier
   JSONata transforms · query compiler · single writer
                           │
                           ▼
                        XTDB2
          (private network; operators only)
                           │
                           ▼
              events: SSE stream, webhooks
```

## Storage (ADR 0001)

- XTDB2, bitemporal and append-only. Every row has valid time and system time.
- Local single node (on-disk log and storage) for development; Kafka log and S3 storage for shared deployments.
- Each entity type is stored in its own XTDB table, keyed by an **internal, stable type ID**, not by its public name. This lets a type gain a new name (ADR 0010) without moving data.
- Entity type definitions are themselves stored in XTDB with valid time, so the definition in force at any point can be read back.
- Each entity version records the schema version it was written under.
- Every transaction writes a provenance row in the same transaction: actor, caller kind, reason (where required) and request ID. (To verify: whether XTDB 2.x transaction user-metadata can replace this row.)

## Metamodel (ADR 0003, 0004)

Our metamodel is canonical. External formats are projections of it; in v0 we **export OpenAPI only**.

Building blocks:

| Block | Purpose |
| --- | --- |
| Entity type | Named, versioned. The name is its URL path segment. |
| Attribute | From a closed set of types (string, integer, decimal, boolean, date, instant, enum, money, quantity-with-unit, ...). Required or optional, with simple constraints (range, length, pattern). |
| Identity | Every entity has a server-minted UUIDv7 ID (ADR 0013). Natural keys are expressed as uniqueness constraints. |
| Reference | To another entity type, with cardinality and ownership (owned vs associated). |
| Lifecycle | A state field with declared states, named transitions and optional guards. Only changeable through transitions. |
| Time semantics | Marks attributes whose valid time is business-meaningful (e.g. an effective date), as opposed to bookkeeping. |
| Sensitivity | Marks personal data, which drives erasure. |
| Provenance | Recorded on every change; not declared per type. |

Definitions are data (EDN internally, JSON at the API), compiled to Malli for validation. Clojure is never exposed to callers.

Domain logic is **declarative only** (ADR 0009): types, required fields, ranges, uniqueness, referential integrity and transition guards as simple predicates. Anything richer lives in the apps built on top.

## Lifecycles (ADR 0004)

- Any status-like field must be declared as a lifecycle: states, an initial state, terminal states and named transitions.
- A transition may declare a guard (a predicate over the entity) and a payload schema.
- `PATCH` cannot change a lifecycle field; only `POST /e/{type}/{id}/t/{transition}` can.

## Changing the model (ADR 0005, 0006)

A change is **additive** if every previously valid entity and every previously valid request is still valid afterwards. Anything else is **breaking**. The service classifies each submitted change and rejects one sent to the wrong API.

### API A: additive changes, `POST /m/{type}/changes`

Examples: a new optional field; a new required field with a server-side default; a new state reachable by a new transition; widening a constraint. Applied immediately; nothing breaks.

### API B: breaking changes, `POST /m/{type}/breaking-changes`

Examples: renaming or removing a field; narrowing a type or constraint; removing a state or transition; adding a guard.

- The request must include a **reason** and a **caller flag**, both recorded with the transaction. This is the first privilege gate, before identity exists.
- It may include a **JSONata transformation** from old entity shape to new.
- We **dry-run** the change: apply the transformation (if any) to every current and future-valid version of every entity of that type, then validate against the new definition. The response reports how many entities were checked, fixed and left invalid, with sample failures.
- The caller can then confirm. On confirmation the definition change and all transformed entities are written in one transaction.
- History is not rewritten: past versions remain valid under the definition in force at their valid time.
- Entities still invalid after the change go onto a **worklist** (`GET /e/{type}/non-conforming`).
- Invalid entities must be fixed first: every write to one must produce a valid entity, and no transition (including to a terminal state) is allowed until it is valid.
- The response also says that clients will break, and how (fields renamed, removed, newly required).
- v0 changes touch one entity type at a time. Transformations across types (splitting an entity, moving a field across a reference) are out of scope.

## API (ADR 0008)

Every entity type has the same API shape. Typed OpenAPI is generated per type.

### Data plane

```
GET    /e/{type}                          simple query (filters, order, paging, ?valid-at=, ?system-at=)
POST   /e/{type}/query                    structured query
POST   /e/{type}                          create (we mint the ID)
GET    /e/{type}/{id}                     read (supports as-of)
PATCH  /e/{type}/{id}                     update (not lifecycle fields)
POST   /e/{type}/{id}/t/{transition}      lifecycle transition
GET    /e/{type}/{id}/history             versions across both time axes
GET    /e/{type}/non-conforming           worklist after breaking changes
```

### Meta plane

```
GET    /m                                 list entity types
GET    /m/{type}                          definition (supports as-of)
POST   /m/{type}/changes                  API A: additive
POST   /m/{type}/breaking-changes         API B: breaking, with dry run and confirm
GET    /m/{type}/openapi.json             generated OpenAPI (supports as-of)
```

- `/e` and `/m` are separate entry points so privilege can be enforced by path before fine-grained rules exist.
- No `/v1` in paths: definitions are versioned in time, so clients pin by as-of.
- Entity type names are lowercase kebab-case and immutable. A "rename" creates a new name; the old path answers `308` to the new one for a declared period (ADR 0010).
- Errors use `application/problem+json`.

### Queries (ADR 0016)

- The structured query is JSON in our own vocabulary: filters (and/or/not, comparisons, `in`), following references (`include`), sorting, paging, both time axes, and `count` and `group-by` aggregation.
- It compiles to XTQL. No SQL or XTQL is ever accepted from callers.
- Unsupported operations get a clear refusal, not a workaround.

### MCP (ADR 0011)

A fixed set of generic tools that take an entity type (`list_types`, `describe_type`, `query`, `create`, `update`, `transition`, `history`, `propose_change`, ...). Each type's generated schema is published as an MCP resource, so the tool count doesn't grow with the model.

## Access (ADR 0012)

Agents and apps use the API only. XTDB's Postgres-compatible protocol is reachable by operators on a private network, never by agents. We revisit this if XTDB can be constrained enough to expose safely.

Identity and privilege are deferred, but every transaction records an actor from day one.

## Writes and integrity (ADR 0007)

XTDB2 has no uniqueness or foreign-key constraints. In v0 a **single writer** process serialises all writes, data and meta: it validates against the current definition (including uniqueness and references), then submits. This also guarantees nothing changes between a breaking change's dry run and its application.

Later: optimistic checks inside transactions (e.g. `ASSERT`), once XTDB support is verified.

## Events (ADR 0015)

Transitions, creates, updates and model changes are emitted as events, derived from the transaction log.

- **SSE stream** with a resumable cursor (`Last-Event-ID`).
- **Webhooks** registered by subscribers, with retries.

## Deployment (ADR 0014)

Single tenant: one circus per deployment.

## Open questions

- `DELETE` semantics: end valid time, or only via lifecycle terminal states?
- Erasure API for sensitive attributes, built on XTDB `ERASE`.
- Can model changes be scheduled for a future valid time?
- How long does a renamed type's `308` last, and who decides?
- Webhook delivery guarantees, signing and subscription management.
- JSONata implementation on the JVM (candidate: dashjoin `jsonata-java`), and limits on time and memory.
- XTDB 2.x specifics to verify: transaction user-metadata, `ASSERT`, and erasure behaviour.
- Atomic application of breaking changes over very large data sets.
