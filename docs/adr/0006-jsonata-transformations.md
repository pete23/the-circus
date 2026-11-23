# 0006. JSONata for transformations

Status: accepted, 2026-11-23

## Context
Breaking changes may need to migrate existing data. Callers include humans who do not write Clojure, and exposing Clojure would be a sales error.

## Decision
Transformations are JSONata expressions from the old entity shape to the new one. No other code is accepted from callers.

## Consequences
- JSONata is pure and sandboxable; we still need time and memory limits.
- We need a JVM implementation (candidate: dashjoin `jsonata-java`).
- JSONata is the only exception to declarative-only domain logic (0009).
