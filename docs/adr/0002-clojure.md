# 0002. Clojure as the implementation language

Status: accepted, 2026-11-23

## Context
XTDB is a JVM system. The metamodel, definitions and queries are all data.

## Decision
Implement the-circus in Clojure, running XTDB in the same process where useful. It is the language the author knows best, and treats code and data alike.

## Consequences
- Malli for validation; definitions as EDN internally.
- Clojure is an implementation detail and is never exposed to callers (see 0006).
- A smaller pool of contributors than Kotlin or TypeScript.
