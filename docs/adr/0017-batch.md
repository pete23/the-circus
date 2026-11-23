# 0017. Multi-entity writes through /batch

Status: accepted, 2026-11-23

## Context
The data API is one entity per call. Business operations span types ("accept offer → close opening → reject other applications"), and seeding a graph needs each create's ID before the next. Without atomicity, apps are pushed into writing sagas on day one.

## Decision
`POST /batch` takes an ordered list of create, update and transition operations across any entity types. A create may declare a temporary ID that later operations reference. Every operation is validated against the current definitions before anything is submitted; the whole batch is written in one XTDB transaction through the single writer. It cannot half-apply.

There is no cap on the number of operations. A safety limit of 100 MB compressed applies to the payload, to be revisited once we see what XTDB tolerates. Large batches are for authorised callers once privilege exists.

## Consequences
- XTDB's multi-table transactions do the heavy lifting.
- A second write shape for generators (OpenAPI, MCP) to support.
- Eventual consistency via events remains available but is not the only option.
