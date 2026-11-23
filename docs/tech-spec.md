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
   metamodel · validation · state machines · change classifier
   JSONata transforms · query compiler · single writer
                           │
                           ▼
                        XTDB2
          (private network; operators only)
```

The XTDB transaction history is the only record of what happened. Nothing is written twice (ADR 0015, 0024).

## Namespaces

A pure core, one stateful layer, and the edges. Integrant components are marked ●. Pure-core namespaces never require state or edge namespaces; clj-kondo enforces this.

```
the-circus                 -main: load config, start the system. Nothing else.
the-circus.system          read system.edn, init/halt; integrant.repl hooks

── pure core (no IO) ───────────────────────────────────────────
the-circus.metamodel       bootstrap `entity` definition; Malli schema of a definition;
                           integrity rules
the-circus.compile         definition → validators, defaults, transition table
the-circus.machine         is this transition legal, what does it set; attachment
                           and instance shape
the-circus.ops             create/update/transition/repair/erase/batch as data;
                           op + definition → tx ops or failures
the-circus.classify        old definition × new definition → additive | breaking, and why
the-circus.transform       JSONata wrapper, time and memory limits
the-circus.query           JSON query → XTDB query
the-circus.openapi         definitions → OpenAPI

── state ───────────────────────────────────────────────────────
the-circus.xtdb        ●   the node; thin read and submit functions
the-circus.registry    ●   compiled definitions, as-of lookup; a projection of the
                           `entity` table, refreshed synchronously by the writer
the-circus.resolve         path segment → id: UUID, path key, alias → 308
the-circus.writer      ●   the single writer: serialise, validate, uniqueness and
                           references, provenance, submit, signal readers
the-circus.migrate         dry run and apply for breaking changes; worklist
the-circus.feed            history at three scopes, rendered from the transaction history

── edges ───────────────────────────────────────────────────────
the-circus.http        ●   reitit on http-kit; routes, JSON ⇄ namespaced keywords,
                           problem+json, SSE and long-poll, MCP routes
the-circus.mcp             generic tools, per-type schemas and skills, as routes
```

A write: `http` → `resolve` → `writer` → (`ops` + `registry`) → `xtdb`, then the writer signals `feed` readers. A meta write takes the same path; the writer also refreshes `registry`, and a breaking change detours through `classify` and `migrate`.

## Storage (ADR 0001)

- XTDB2, bitemporal and append-only. Every row has valid time and system time.
- Local single node (on-disk log and storage) for development; Kafka log and S3 storage for shared deployments.
- Each entity type is stored in its own XTDB table, keyed by an **internal, stable type ID**, not by its public name. This lets a type gain a new name (ADR 0010) without moving data.
- Entity type definitions are themselves stored in XTDB with valid time, so the definition in force at any point can be read back.
- Each entity version records the schema version it was written under.
- Every transaction carries provenance: actor, caller kind, reason (where required) and request ID. One small row per transaction, or XTDB transaction user-metadata if 2.x supports it (to verify). Provenance never copies the request.
- Every entity version carries two system columns, alongside XTDB's own: `_circus/op` (`create`, `update`, `transition`, `repair`) and `_circus/transition` (`{field}/{transition}`, when there is one). With these, every change can be read back from the transaction history without any separate record (ADR 0024).

## Metamodel (ADR 0003, 0004)

Our metamodel is canonical. External formats are projections of it; in v0 we **export OpenAPI only**.

Building blocks:

| Block | Purpose |
| --- | --- |
| Entity type | Named, versioned. The name is its URL path segment. |
| Attribute | From a closed set of types (string, integer, decimal, boolean, date, instant, enum, money, quantity-with-unit, ...). Required or optional, with simple constraints (range, length, pattern) and an optional default. |
| Uniqueness | One or more attributes that must be unique at any instant of valid time (ADR 0013). Natural keys live here; IDs are always minted by us. One single-attribute constraint may be marked as the `path-key` (ADR 0022). |
| Reference | To another entity type, with cardinality and ownership (owned vs associated). |
| State machine | States, an initial state, terminal states, named transitions with guards, and the machine's own attributes. Named and reusable (built-in type `machine`, at `/s/{machine}`), or private and inline. An entity type **attaches** a machine to a field; each entity carries its own instance. Only changeable through transitions. The idiomatic home for anything with states (ADR 0004, 0025). |
| Time semantics | Marks attributes whose valid time is business-meaningful (e.g. an effective date), as opposed to bookkeeping. |
| Sensitivity | Marks personal data, which drives erasure (ADR 0019). |
| Provenance | Recorded on every change; not declared per type. |

Definitions are data (EDN internally, JSON at the API), compiled to Malli for validation. Clojure is never exposed to callers.

**Definitions are entities** (ADR 0021). Every definition is an entity of the built-in type `entity`, which has a unique attribute `name`, an `aliases` list (old names, which drive the `308` on rename) and a `status` field running the built-in machine `active-retired` (transitions `retire` and `revive`). `/m/{type}` is a privileged route onto `/e/entity/...` addressed by name. History, as-of, provenance, the history feed, idempotency and uniqueness come from the same engine. The definition of `entity` itself is the metamodel; it is fixed by the platform and changes only with a platform upgrade.

Definitions are validated as a whole (**metamodel integrity**): transitions reference declared states, transition names are unique within a machine (ADR 0023), a transition sets only its machine's attributes, attachments name existing machines, references point at existing types, uniqueness constraints name declared attributes, at most one is the path key, defaults conform to their attribute's type, and a type cannot be renamed or retired in a way that leaves a reference pointing nowhere.

Domain logic is **declarative only** (ADR 0009): types, required fields, ranges, uniqueness, referential integrity and transition guards as simple predicates, including one aggregate over owned references ("all owned `signature`s are `signed`"). Anything richer lives in the apps built on top.

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
  "machines": {
    "status": {
      "inline": true,
      "attributes": {
        "offered-salary":   {"type": "money"},
        "rejection-reason": {"type": "string"}
      },
      "initial": "applied",
      "states": ["applied", "screening", "interviewing", "offered", "hired", "rejected", "withdrawn"],
      "terminal": ["hired", "rejected", "withdrawn"],
      "transitions": {
        "screen":    {"from": ["applied"],                                  "to": "screening"},
        "interview": {"from": ["screening"],                                "to": "interviewing"},
        "offer":     {"from": ["interviewing"],                             "to": "offered",
                      "sets": {"offered-salary": {"required": true}}},
        "hire":      {"from": ["offered"],                                  "to": "hired"},
        "reject":    {"from": ["applied", "screening", "interviewing", "offered"], "to": "rejected",
                      "sets": {"rejection-reason": {"required": true}}},
        "withdraw":  {"from": ["applied", "screening", "interviewing", "offered"], "to": "withdrawn"}
      }
    }
  }
}
```

A shared machine, defined once at `/s/email-confirmation` and attached to `candidate`:

```json
{
  "name": "email-confirmation",
  "attributes": {
    "address":      {"type": "string", "required": true, "sensitive": true},
    "sent-at":      {"type": "instant"},
    "confirmed-at": {"type": "instant"}
  },
  "initial": "unconfirmed",
  "states": ["unconfirmed", "sent", "confirmed", "expired"],
  "terminal": ["confirmed", "expired"],
  "transitions": {
    "send":    {"from": ["unconfirmed", "sent"], "to": "sent",      "sets": {"sent-at": {"required": true}}},
    "confirm": {"from": ["sent"],                "to": "confirmed", "sets": {"confirmed-at": {"required": true}}},
    "expire":  {"from": ["sent"],                "to": "expired"}
  }
}
```

```json
{"name": "candidate", "machines": {"email": {"use": "email-confirmation"}}, "...": "..."}
```

An instance lives inside its entity: `{"email": {"state": "sent", "address": "…", "sent-at": "…"}}`. Internally the attachment name is the keyword namespace: `:email/state`, `:email/sent-at`. The transition is `POST /e/candidate/{id}/t/email/confirm`.

## State machines (ADR 0004, 0025)

- A machine declares states, an initial state, terminal states, named transitions and its own attributes.
- A **named** machine is an entity of the built-in type `machine`, at `/s/{machine}`, and can be attached to any number of types. A **private** machine is written inline in one type's definition (`"inline": true`). Promoting a private machine to a named one is additive if it is identical.
- An entity type attaches a machine to a field. Each entity carries its own instance under that field: the state plus the machine's attributes. Attaching the same machine twice is fine (`signer-a`, `signer-b`).
- A transition may declare a guard and the attributes it `sets`, each optionally required. A transition sets only its own machine's attributes, so its input is stored in the instance and nowhere else (ADR 0024). A machine's guards see only its instance; an attachment may add guards over the host entity, including the aggregate over owned references. The reason for a privileged change (retire, a breaking change) is provenance, not an attribute.
- Transitions are always addressed `POST /e/{type}/{id}/t/{field}/{transition}`. No short form, even for a type with one machine (ADR 0023).
- `PATCH` cannot change any part of a machine instance; only transitions (and `repair`) can.
- Attaching a machine to an existing type is additive: every entity gets an instance in the initial state, materialised on apply. Detaching is breaking.
- A variable number of instances (N signatories) is modelling, not a feature: make `signature` its own entity type running the machine, owned by the document, and guard the document's transition with the aggregate.
- A plain `enum` attribute is allowed and mutable through `PATCH`. It gets none of the above. A state machine is the easy path, not the only path; the platform's skills teach agents when to use one.
- Nested machines (a transition that waits on another machine) are not in v0.

## Clients and definitions (ADR 0020)

Machine states, enum values and defaults are part of the definition, and the definition is readable at any time (`GET /m/{type}` or `GET /s/{machine}`, as-of if needed). Clients derive enumerations from the definition rather than hard-coding them: a dropdown is populated from `/m/application`, not from a constant. A client that hard-codes states has broken the contract, not us. This is what makes new states and new enum values additive.

## Changing the model (ADR 0005, 0006)

A change is **additive** if it cannot fail and cannot change the meaning of any existing row or any previously valid request. Anything else is **breaking**. The service classifies each submitted change and rejects one sent to the wrong API.

### API A: additive changes, `PATCH /m/{type}`

A patch whose diff is breaking is refused with a pointer to the breaking endpoint. Examples: a new optional field; a new required field with a declared default; a new state or enum value; a new transition; widening a constraint.

Additive changes may write data in one case: a new required field with a default is **materialised** onto every existing entity when the change is applied, as a new version valid from the apply time and attributed to the meta-transaction. A default is applied at write time whenever the field is omitted (on create, and once on apply for existing rows). Changing a default later affects only future writes. Stored data therefore always literally conforms, and queries never need null handling.

### API B: breaking changes, `POST /m/{type}/breaking-changes`

Examples: renaming or removing a field; narrowing a type or constraint; removing a state or transition; adding a guard; retiring the type.

- The request must include a **reason** and a **caller flag** (`Circus-Breaking: acknowledged` header), both recorded with the transaction. This is the first privilege gate, before identity exists.
- It may include a **JSONata transformation** from old entity shape to new.
- **Removing a machine state requires a state mapping** (old state → new state) in the change. This is checked up front, so a machine change can never leave an entity in a state that no longer exists.
- A **dry run** is required first (`?dry-run=true`, available on every write but mandatory here): apply the transformation (if any) to every current and future-valid version of every entity of that type, then validate against the new definition. The response reports how many entities were checked, fixed and left invalid, with sample failures, and a confirmation token.
- The caller can then confirm with the token. On confirmation the definition change and all transformed entities are written in one transaction.
- History is not rewritten: past versions remain valid under the definition in force at their valid time.
- Entities still invalid after the change go onto a **worklist** (`GET /e/{type}/non-conforming`).
- Invalid entities must be fixed first: no `PATCH` or transition is allowed until the entity is valid, including transitions to terminal states. The fix goes through `POST /e/{type}/{id}/repair`, which is allowed only on worklist entities, may set any field including machine instances, must produce a valid entity, and is recorded as a repair with actor and reason.
- The response also says how clients will break (fields renamed, removed, newly required; states removed).
- v0 changes touch one entity type at a time. Transformations across types (splitting an entity, moving a field across a reference) are out of scope.

### Changing a named machine (ADR 0025)

A named machine has the same change endpoints as a type, under `/s/{machine}`.

- **Additive** changes (a new state, a new transition, a new optional attribute) apply automatically to every type that attaches the machine. New required attributes with defaults are materialised into every instance.
- **Breaking** changes to a machine attached to exactly one type follow API B, migrating that type's entities.
- **Breaking** changes to a machine attached to more than one type are refused in v0, because they are a migration across types. Instead, **fork**: create the new machine under a new name, move each type onto it with an ordinary breaking change on that type (with a state mapping), then retire the old machine.
- Retiring a machine freezes it: no changes and no new attachments. Existing attachments keep running.
- Private machines change through their type's own `PATCH` and breaking-change endpoints.

### Retiring a type (ADR 0018)

`POST /m/{type}/t/status/retire` is a transition on `entity`'s `status` machine and a breaking change; its reason is recorded as provenance. Retire means **freeze**: no creates, updates, transitions or further definition changes, but reads, history, queries and inbound references keep working forever. Nothing is removed and the path stays live. The change may include a mapping for inbound references (drop the field, or point it elsewhere), in which case the referencing types are updated in the same transaction; without one, references continue to point at the frozen type.

`POST /m/{type}/t/status/revive` reverses it. The type thaws under the definition it was frozen with, and writes resume. Revive is additive: it cannot fail and changes nothing stored. It does not undo a reference mapping applied at retire time; putting those references back is a separate change on the referencing types. Both retire and revive are recorded with a reason and show in the definition's history.

## API (ADR 0008)

Every entity type has the same API shape. Typed OpenAPI is generated per type.

### Data plane

```
GET    /e/{type}                          simple query (filters, order, paging, ?valid-at=, ?system-at=)
POST   /e/{type}/query                    structured query
POST   /e/{type}                          create (we mint the ID)
GET    /e/{type}/{id}                     read (supports as-of)
PATCH  /e/{type}/{id}                     update (not machine instances)
POST   /e/{type}/{id}/t/{field}/{transition}   state machine transition
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
POST   /m/{type}/t/status/retire          API B: freeze the type
POST   /m/{type}/t/status/revive          thaw a retired type
GET    /m/{type}/history                  definition versions across both time axes
GET    /m/{type}/openapi.json             generated OpenAPI (supports as-of)

GET    /s                                 list named state machines
POST   /s                                 create a machine
GET    /s/{machine}                       definition, and which types attach it (supports as-of)
PATCH  /s/{machine}                       additive; applies to every attaching type
POST   /s/{machine}/breaking-changes      breaking; refused if attached to more than one type
POST   /s/{machine}/t/status/retire       freeze the machine
POST   /s/{machine}/t/status/revive       thaw it
GET    /s/{machine}/history               definition versions
```

`/m/{type}` is a privileged route onto `/e/entity/{id}`, and `/s/{machine}` onto `/e/machine/{id}`; `name` is the path key of both (ADR 0022). Both built-in types attach the built-in machine `active-retired` as `status`. The only meta-shaped things are the breaking endpoints (their validation scope is every row of the type, hence transformation, dry run and confirm) and the fixed bootstrap definitions of `entity`, `machine` and `active-retired`.

- `/e`, `/m` and `/s` are separate entry points so privilege can be enforced by path before fine-grained rules exist.
- No `/v1` in paths: definitions are versioned in time, so clients pin by as-of.
- `{id}` in every data-plane path is either the UUID or, if the type declares a `path-key`, a value of that key (ADR 0022). A UUID-shaped segment is always an ID; path-key values may not be UUID-shaped. With `?valid-at=` a key resolves to its holder at that instant. No current holder gives `404` with who last held it and until when. References always store UUIDs.
- Transitions are addressed by field and name, `/t/{field}/{transition}`, so two machines (or the same machine attached twice) never collide (ADR 0023).
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

A fixed set of generic tools that take an entity type (`list_types`, `describe_type`, `describe_machine`, `query`, `create`, `update`, `transition`, `batch`, `history`, `propose_change`, ...). Each type's generated schema is published as an MCP resource, so the tool count doesn't grow with the model. The server also publishes **skills**: instructions resources that teach agents the idioms (when a field is a state machine, how to classify a change, how to read a definition before writing).

## Access (ADR 0012)

Agents and apps use the API only. XTDB's Postgres-compatible protocol is reachable by operators on a private network, never by agents. We revisit this if XTDB can be constrained enough to expose safely.

Identity and privilege are deferred, but every transaction records an actor from day one.

## Writes and integrity (ADR 0007)

XTDB2 has no uniqueness or foreign-key constraints. In v0 a **single writer** process serialises all writes, data, batch and meta: it validates against the current definition (including uniqueness and references), then submits. Uniqueness is checked against currently-valid and future-valid rows (ADR 0013). The single writer also guarantees nothing changes between a breaking change's dry run and its application.

Later: optimistic checks inside transactions (e.g. `ASSERT`), once XTDB support is verified.

## The history feed (ADR 0015)

There is no event subsystem. Every consumer that wants to know what changed (live UIs, reactive business logic, sync jobs, waiting agents, clients watching definitions) asks the same question: changes since cursor X, for the types I care about. That is a history query, at three scopes:

```
GET /e/{type}/{id}/history?after={tx}     one entity
GET /e/{type}/history?after={tx}          one type
GET /history?after={tx}&type=…            everything, filtered
```

- The cursor is the XTDB transaction ID. Clients hold their own cursors; we hold no subscriber state.
- Each change is rendered from the transaction history: the entity version, `_circus/op`, `_circus/transition`, and provenance.
- Filters: `?op=`, `?transition={field}/{name}`, and on `/history` `?machine={name}&transition={name}`, which follows a machine across every type that attaches it (every email confirmation, anywhere).
- **SSE** is `Accept: text/event-stream` on the same URLs: the same query, held open. `Last-Event-ID` is the cursor.
- **Long-poll** is `?wait=30s` on the same URLs, for agents waiting on a condition.
- Definition changes are the history of `/e/entity`.
- After a commit the writer signals waiting readers to look again. The signal carries nothing; readers would be correct polling without it. The writer never waits on readers.
- **Webhooks are deferred.** If a consumer must be pushed to, a webhook dispatcher is one more reader of the feed holding its own cursor; nothing underneath changes.

## Erasure (ADR 0019)

Attributes marked sensitive can be erased for an entity with XTDB `ERASE`, leaving the rest of its history intact. Erasure covers what we store. Values already delivered through the history feed, dry-run samples or exports are the recipient's obligation. Erasure is itself a change in the history feed, so readers can act on it. Because the feed renders from stored versions, replaying it after erasure simply lacks the field. (To verify: XTDB 2.x log retention, since the underlying log holds original transaction payloads.)

## Deployment (ADR 0014)

Single tenant: one circus per deployment.

## Open questions

- `DELETE` semantics: end valid time, or only via terminal states?
- Nested machines: a transition that waits on another machine (step-up confirmation for privileged transitions).
- A revert API (`revert-to?valid-at=` for entities and definitions), so "recoverable" can become "reversible".
- Can model changes be scheduled for a future valid time?
- How long does a renamed type's `308` last, and who decides?
- Webhooks, if a consumer turns up that cannot pull: delivery guarantees, signing, subscription management.
- JSONata implementation on the JVM (candidate: dashjoin `jsonata-java`), and limits on time and memory.
- XTDB 2.x specifics to verify: transaction user-metadata, `ASSERT`, erasure behaviour and log retention, and XTQL vs SQL as the compile target.
- Refusals (rejected writes) go to the structured application log, not XTDB. Revisit if auditors need them queryable.
- Atomic application of breaking changes and large batches over very large data sets.
