# the-circus: business spec

Status: draft, 2026-11-23

## The problem

Agents now build business software. They design schemas, write migrations, wire up APIs and ship UIs, often guided by humans who are not reading every change. The data layer underneath is usually a general-purpose database with full read/write access. The results are predictable:

- Data models that drift with every prompt, and are inconsistent across features.
- Status fields that any code path can set to any value, so workflows exist only by convention.
- Schema changes that silently break existing data or the clients that depend on it.
- No reliable answer to "who changed this, when, why, and what did it look like before?"
- Destructive operations (`DROP TABLE users;`) that are one bad tool call away.

## What the-circus is

**A low-code platform for agents, not humans.** The-circus is the backend that agents build business applications on. It owns the decisions about how data is modelled; agents (and humans writing specs) describe their domain in its terms, and get back a uniform, typed, audited API.

- **We decide how data is modelled.** Entity types are defined in our metamodel, which captures what standard schema languages leave out: identity, lifecycles, references, time semantics, money and units, sensitivity and provenance.
- **Where the data implies a state machine, we implement it.** Status changes happen only through named transitions that we enforce.
- **Every change is recorded and recoverable.** Storage is append-only and bitemporal: any entity, and the definition it was valid under, can be read as of any point in time. Nothing is ever deleted; a retired entity type is frozen, not dropped.
- **Changing the model is managed too.** Changes to an entity type's definition go through us, and must declare themselves as either additive (nothing breaks) or breaking (data or clients may break). Breaking changes are explicit, explained and leave a worklist of anything that needs fixing.

Agents build whatever they like on top: UIs, business logic, integrations. That is not our concern. Because everything underneath is constrained and well understood, what they build is too.

We do not prohibit bad modelling; we make good modelling the easy path and ship the **skills** that teach agents the idioms. Canalising agents is partly a documentation product.

## What it is not

- **Not an agentic development environment.** We do not generate UIs, host business logic or orchestrate agents.
- **Not a general-purpose database.** No SQL or direct database access for agents. Only our API.
- **Not multi-tenant.** One circus per deployment.
- **Not (yet) an identity and access system.** Callers are recorded from day one; fine-grained privilege comes later.

## Who uses it

| Who | What they do |
| --- | --- |
| Builder agents | Define and change entity types through the meta API. |
| Human spec authors | Same as builder agents, often reviewing or directing them. |
| Apps built on top | Read and write business data through the generated, typed data API. |
| Runtime agents | Operate inside those apps through MCP, under the same rules. |
| Operators and auditors | Ask what happened, when and why; answer erasure requests. |

## Why now, and why not the alternatives

- **Backend-as-a-service (Supabase, Hasura, PocketBase)** gives agents a fast path to a database, and the full power to misuse it. Schemas are whatever the agent writes; workflows are conventions; history is whatever someone remembered to log.
- **Human low-code platforms (Airtable, Salesforce)** have the right idea, a platform-owned metamodel, but are built for people clicking through UIs, not for agents calling APIs.
- **Agents need tighter guardrails than humans, not looser.** They are fast, confident and inconsistent. A narrow, opinionated model with enforced lifecycles and explicit breaking changes turns that speed into an asset.

## The demo: a hiring pipeline

A hiring pipeline has long lifecycles, plenty of references and plenty of personal data. The demo will show:

1. An agent defines the model: job openings, candidates, applications, interviews, offers.
2. An agent builds a throwaway UI on the generated API. Applications move through their lifecycle only by named transitions; the UI's dropdowns come from the definition, not from code. Accepting an offer closes the opening and rejects the other applications in one atomic batch.
3. An **additive change**: a new optional field, or a new state that existing records can ignore. Nothing breaks.
4. A **breaking change with a transformation**: e.g. splitting a candidate's name into given and family names, migrated by a JSONata expression we dry-run before applying.
5. A **breaking change that invalidates data**: e.g. a new rule that offers require right-to-work verification. Existing offers are flagged on a worklist and must be fixed before they can move on.
6. **Time travel**: the pipeline, and the model it ran under, as of last Tuesday.
7. **Erasure**: a candidate's personal data is erased on request, with the rest of the history intact.
8. **The `DROP TABLE` moment**: an agent asks to drop a type. It gets a frozen, fully readable, attributed retirement that needed a stated reason, or nothing at all.

## What success looks like

- A builder agent can go from a plain-language brief to a working, typed backend for the demo domain without touching a database.
- Every change to data and to the model can be explained after the fact: who, when, why and what it replaced.
- No breaking change reaches existing data or clients without being declared as breaking.
- A human reviewer can read an entity type's definition and understand the business rules it enforces.
