# the-circus: coding standards

Status: accepted, 2026-11-23. These apply to every contributor, human or agent. `CLAUDE.md` at the repo root points here.

## Namespaces

- The root namespace is `the-circus`. Declare it as `(ns the-circus)` and use it; never `the-circus.core`. No namespace anywhere is called `core`.
- Layout is by concern, not by layer: `the-circus.metamodel`, `the-circus.writer`, `the-circus.http`, `the-circus.feed`. Not `handlers`, `models`, `utils`.
- One namespace per file; the file path mirrors the namespace.

## Tooling

- `deps.edn` and `tools.build`. No Leiningen.
- Formatting: `cljfmt` defaults, 110 columns, spaces. Enforced in CI.
- Linting: `clj-kondo`; warnings are errors.
- Tests: `clojure.test`, run with Kaocha. A test namespace mirrors its source namespace with a `-test` suffix: `the-circus.writer` is tested by `the-circus.writer-test`.

## Data

- Malli everywhere. No `clojure.spec`.
- Domain data uses namespaced keywords: `:application/status`. Transport JSON uses bare keys; the conversion happens at the HTTP boundary and nowhere else.
- Definitions, configuration and the system are data. Prefer a map over a record, a keyword over an enum, a plain function over a protocol, unless there is a measured reason.

## Errors

- `ex-info` with a `:type` key, namespaced: `{:type :the-circus.writer/conflict}`. No custom exception classes.
- At the HTTP boundary, `:type` maps to a `application/problem+json` response. Nothing else catches exceptions to translate them.

## State and the system

- No global mutable state. No `def` of an atom at namespace level, no `defonce` holding a connection.
- Integrant. Configuration is `resources/system.edn`; dependencies are `#ig/ref`.
- `init-key` and `halt-key!` live in the namespace that owns the component, and the key is namespaced to it: `:the-circus.writer/writer` is defined in `the-circus.writer`. Only component namespaces and `the-circus.system` require `integrant.core`.
- The system map exists in exactly two places: `integrant.repl`'s state in development and a local in `-main` in production. Handlers receive their dependencies by injection; nothing reaches for a global.

## Functions

- Every public var has a docstring that says what it returns and what it throws. Private vars may have one.
- Prefer pure functions; push effects to the edges (the writer, the HTTP layer). A function that touches XTDB says so in its name or its namespace.
- No `declare` to dodge ordering; order the file so it reads top-down.

## Comments

- Comment the why, not the what. A comment that restates the code is deleted.
- An ADR number in a comment (`;; ADR 0005`) is the preferred way to say "this is deliberate".

## Commits

- Imperative subject line, under 72 characters. The body says why, not what.
- A commit that implements or changes a decision references its ADR.
- Commit to `main` until agents are spawned; then branches.

## Things we do not do

- No `core` namespaces.
- No `clojure.spec`.
- No Leiningen.
- No global state.
- No Clojure exposed to callers of the platform (ADR 0006): callers see JSON, JSONata and HTTP.
