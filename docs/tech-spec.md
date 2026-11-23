# the-circus: tech spec

Status: draft, 2026-11-23. Decisions are recorded in [`adr/`](adr/); this document describes the system as those decisions add up.

## Overview

A single Clojure service in front of an XTDB2 node. Agents and apps talk to the service over HTTP (and MCP); nothing else talks to XTDB.

```
 builder agents / spec authors      apps, runtime agents
            │                              │
            ▼                              ▼
     meta API  /m/{type}            data API  /e/{type}, /batch
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
| Attribute | From a closed set of types (string, integer, decimal, boolean, date, instant, enum, money, quantity-with-unit, ...). Required or optional, with simple constraints (range, length, pattern) and an optional default. |
| Uniqueness | One or more attributes that must be unique at any instant of valid time (ADR 0013). Natural keys live here; IDs are always minted by us. One single-attribute constraint may be marked as the `path-key` (ADR 0022). |
| Reference | To another entity type, with cardinality and ownership (owned vs associated). |
| Lifecycle | A state field with declared states, named transitions and optional guards. Only changeable through transitions. The idiomatic home for anything with states (ADR 0004). |
| Time semantics | Marks attributes whose valid time is business-meaningful (e.g. an effective date), as opposed to bookkeeping. |
| Sensitivity | Marks personal data, which drives erasure (ADR 0019). |
| Provenance | Recorded on every change; not declared per type. |

Definitions are data (EDN internally, JSON at the API), compiled to Malli for validation. Clojure is never exposed to callers.

**Definitions are entities** (ADR 0021). Every definition is an entity of the built-in type `entity`, which has a unique attribute `name`, an `aliases` list (old names, which drive the `308` on rename) and a lifecycle `active ⇄ retired` with transitions `retire` and `revive`. `/m/{type}` is a privileged route onto `/e/entity/...` addressed by name. History, as-of, provenance, events, idempotency and uniqueness come from the same engine. The definition of `entity` itself is the metamodel; it is fixed by the platform and changes only with a platform upgrade.

Definitions are validated as a whole (**metamodel integrity**): transitions reference declared states, transition names are unique across all lifecycles of the type (ADR 0023), references point at existing types, uniqueness constraints name declared attributes, at most one is the path key, defaults conform to their attribute's type, and a type cannot be renamed or retired in a way that leaves a reference pointing nowhere.

Domain logic is **declarative only** (ADR 0009): types, required fields, ranges, uniqueness, referential integrity and transition guards as simple predicates. Anything richer lives in the apps built on top.

### Example definition

```json
{
  "name": "application",
  "attributes": {
    "submitted-at": {"type": "instant", "required": true},
    "source":       {"type": "enum", "values": ["referral", "inbound", "agency"]},
    "salary-ask":   {"type": "money"}
  },
  "references": {
    "candidate": {"type": "candidate", "cardinality": "one", "ownership": "associated", "required": true},
    "opening":   {"type": "opening",   "cardinality": "one", "ownership": "associated", "required": true}
  },
  "unique": [["candidate", "opening"]],
  "path-key": null,
  "lifecycles": {
    "status": {
      "initial": "applied",
      "states": ["applied", "screening", "interviewing", "offered", "hired", "rejected", "withdrawn"],
      "terminal": ["hired", "rejected", "withdrawn"],
      "transitions": {
        "screen":    {"from": ["applied"],                                  "to": "screening"},
        "interview": {"from": ["screening"],                                "to": "interviewing"},
        "offer":     {"from": ["interviewing"],                             "to": "offered",
                      "payload": {"salary": {"type": "money", "required": true}}},
        "hire":      {"from": ["offered"],                                  "to": "hired"},
        "reject":    {"from": ["applied", "screening", "interviewing", "offered"], "to": "rejected",
                      "payload": {"reason": {"type": "string", "required": true}}},
        "withdraw":  {"from": ["applied", "screening", "interviewing", "offered"], "to": "withdrawn"}
      }
    }
  }
}
```

## Lifecycles (ADR 0004)

- A lifecycle declares states, an initial state, terminal states and named transitions.
- A transition may declare a guard (a predicate over the entity) and a payload schema.
- `PATCH` cannot change a lifecycle field; only `POST /e/{type}/{id}/t/{transition}` can.
- A plain `enum` attribute is allowed and mutable through `PATCH`. It gets none of the above. Lifecycle is the easy path, not the only path; the platform's skills teach agents when to use it.

## Clients and definitions (ADR 0020)

Lifecycle states, enum values and defaults are part of the definition, and the definition is readable at any time (`GET /m/{type}`, as-of if needed). Clients derive enumerations from the definition rather than hard-coding them: a dropdown is populated from `/m/application`, not from a constant. A client that hard-codes states has broken the contract, not us. This is what makes new states and new enum values additive.

## Changing the model (ADR 0005, 0006)

A change is **additive** if it cannot fail and cannot change the meaning of any existing row or any previously valid request. Anything else is **breaking**. The service classifies each submitted change and rejects one sent to the wrong API.

### API A: additive changes, `PATCH /m/{type}`

A patch whose diff is breaking is refused with a pointer to the breaking endpoint. Examples: a new optional field; a new required field with a declared default; a new state or enum value; a new transition; widening a constraint.

Additive changes may write data in one case: a new required field with a default is **materialised** onto every existing entity when the change is applied, as a new version valid from the apply time and attributed to the meta-transaction. A default is applied at write time whenever the field is omitted (on create, and once on apply for existing rows). Changing a default later affects only future writes. Stored data therefore always literally conforms, and queries never need null handling.

### API B: breaking changes, `POST /m/{type}/breaking-changes`

Examples: renaming or removing a field; narrowing a type or constraint; removing a state or transition; adding a guard; retiring the type.

- The request must include a **reason** and a **caller flag** (`Circus-Breaking: acknowledged` header), both recorded with the transaction. This is the first privilege gate, before identity exists.
- It may include a **JSONata transformation** from old entity shape to new.
- **Removing a lifecycle state requires a state mapping** (old state → new state) in the change. This is checked up front, so a lifecycle change can never leave an entity in a state that no longer exists.
- A **dry run** is required first (`?dry-run=true`, available on every write but mandatory here): apply the transformation (if any) to every current and future-valid version of every entity of that type, then validate against the new definition. The response reports how many entities were checked, fixed and left invalid, with sample failures, and a confirmation token.
- The caller can then confirm with the token. On confirmation the definition change and all transformed entities are written in one transaction.
- History is not rewritten: past versions remain valid under the definition in force at their valid time.
- Entities still invalid after the change go onto a **worklist** (`GET /e/{type}/non-conforming`).
- Invalid entities must be fixed first: no `PATCH` or transition is allowed until the entity is valid, including transitions to terminal states. The fix goes through `POST /e/{type}/{id}/repair`, which is allowed only on worklist entities, may set any field including lifecycle fields, must produce a valid entity, and is recorded as a repair with actor and reason.
- The response also says how clients will break (fields renamed, removed, newly required; states removed).
- v0 changes touch one entity type at a time. Transformations across types (splitting an entity, moving a field across a reference) are out of scope.

### Retiring a type (ADR 0018)

`POST /m/{type}/t/retire` is a transition on the `entity` lifecycle, with a `reason` payload, and a breaking change. Retire means **freeze**: no creates, updates, transitions or further definition changes, but reads, history, queries and inbound references keep working forever. Nothing is removed and the path stays live. The change may include a mapping for inbound references (drop the field, or point it elsewhere), in which case the referencing types are updated in the same transaction; without one, references continue to point at the frozen type.

`POST /m/{type}/t/revive` reverses it. The type thaws under the definition it was frozen with, and writes resume. Revive is additive: it cannot fail and changes nothing stored. It does not undo a reference mapping applied at retire time; putting those references back is a separate change on the referencing types. Both retire and revive are recorded with a reason and show in the definition's history.

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
POST   /e/{type}/{id}/repair              fix a non-conforming entity
GET    /e/{type}/{id}/history             versions across both time axes
GET    /e/{type}/non-conforming           worklist after breaking changes
POST   /batch                             several ops across types, all-or-nothing
```

### Meta plane

```
GET    /m                                 list entity types (= query on entity)
POST   /m                                 create a type (= create on entity)
GET    /m/{type}                          definition (supports as-of)
PATCH  /m/{type}                          API A: additive
POST   /m/{type}/breaking-changes         API B: breaking, with dry run and confirm
POST   /m/{type}/t/retire                 API B: freeze the type
POST   /m/{type}/t/revive                 thaw a retired type
GET    /m/{type}/history                  definition versions across both time axes
GET    /m/{type}/openapi.json             generated OpenAPI (supports as-of)
```

`/m/{type}` is a privileged route onto `/e/entity/{id}`, where `name` is `entity`'s path key (ADR 0022). The only meta-shaped things are the breaking endpoint (its validation scope is every row of the type, hence transformation, dry run and confirm) and the fixed bootstrap definition of `entity`.

- `/e` and `/m` are separate entry points so privilege can be enforced by path before fine-grained rules exist.
- No `/v1` in paths: definitions are versioned in time, so clients pin by as-of.
- `{id}` in every data-plane path is either the UUID or, if the type declares a `path-key`, a value of that key (ADR 0022). A UUID-shaped segment is always an ID; path-key values may not be UUID-shaped. With `?valid-at=` a key resolves to its holder at that instant. No current holder gives `404` with who last held it and until when. References always store UUIDs.
- Transition names are unique per type across all its lifecycles (ADR 0023), so `/t/{transition}` is never ambiguous.
- Entity type names are lowercase kebab-case. A rename sets a new `name` and appends the old one to `aliases`; an alias path answers `308` to the current name for a declared period (ADR 0010).
- Creates accept an `Idempotency-Key` header; a repeated key returns the original result.
- Every write accepts `?dry-run=true`: full validation, nothing written, the response is what the real write would return.
- Errors use `application/problem+json`.

### Batch (ADR 0017)

`POST /batch` takes an ordered list of operations (create, update, transition) across any entity types, including `entity` itself, so mutually referencing definitions can be created together. A create may declare a temporary ID (`"$id": "c1"`) that later operations in the same batch reference. Every operation is validated against the current definitions before anything is submitted, then the whole batch is written in one XTDB transaction. It cannot half-apply. There is no cap on the number of operations; a safety limit of 100 MB compressed applies to the payload, to be revisited once we see what XTDB tolerates.

### Queries (ADR 0016)

- The structured query is JSON in our own vocabulary: filters (and/or/not, comparisons, `in`), following references (`include`), sorting, paging, both time axes, and `count` and `group-by` aggregation.
- It compiles to XTDB's query language (XTQL or SQL; to verify which is the better target in current 2.x). No SQL or XTQL is ever accepted from callers.
- Unsupported operations get a clear refusal, not a workaround.

### MCP (ADR 0011)

A fixed set of generic tools that take an entity type (`list_types`, `describe_type`, `query`, `create`, `update`, `transition`, `batch`, `history`, `propose_change`, ...). Each type's generated schema is published as an MCP resource, so the tool count doesn't grow with the model. The server also publishes **skills**: instructions resources that teach agents the idioms (when a field is a lifecycle, how to classify a change, how to read a definition before writing).

## Access (ADR 0012)

Agents and apps use the API only. XTDB's Postgres-compatible protocol is reachable by operators on a private network, never by agents. We revisit this if XTDB can be constrained enough to expose safely.

Identity and privilege are deferred, but every transaction records an actor from day one.

## Writes and integrity (ADR 0007)

XTDB2 has no uniqueness or foreign-key constraints. In v0 a **single writer** process serialises all writes, data, batch and meta: it validates against the current definition (including uniqueness and references), then submits. Uniqueness is checked against currently-valid and future-valid rows (ADR 0013). The single writer also guarantees nothing changes between a breaking change's dry run and its application.

Later: optimistic checks inside transactions (e.g. `ASSERT`), once XTDB support is verified.

## Events (ADR 0015)

The single writer emits an event for every create, update, transition, repair and erasure. Model changes are ordinary events on the `entity` type.

- **SSE stream** with a resumable cursor (`Last-Event-ID`).
- **Webhooks** registered by subscribers, with retries.

## Erasure (ADR 0019)

Attributes marked sensitive can be erased for an entity with XTDB `ERASE`, leaving the rest of its history intact. Erasure covers what we store. Values already delivered through events, webhooks, dry-run samples or exports are the recipient's obligation; we emit an `erased` event so they can act. After erasure, replayed events simply lack the field.

## Deployment (ADR 0014)

Single tenant: one circus per deployment.

## Open questions

- `DELETE` semantics: end valid time, or only via lifecycle terminal states?
- A revert API (`revert-to?valid-at=` for entities and definitions), so "recoverable" can become "reversible".
- Can model changes be scheduled for a future valid time?
- How long does a renamed type's `308` last, and who decides?
- Webhook delivery guarantees, signing and subscription management.
- JSONata implementation on the JVM (candidate: dashjoin `jsonata-java`), and limits on time and memory.
- XTDB 2.x specifics to verify: transaction user-metadata, `ASSERT`, erasure behaviour, and XTQL vs SQL as the compile target.
- Atomic application of breaking changes and large batches over very large data sets.
