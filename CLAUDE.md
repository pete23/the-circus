# the-circus

A low-code platform for agents: a metadata-driven backend on XTDB2, in Clojure.

Read before changing anything:

- `docs/coding-standards.md`: how code is written here. Non-negotiable.
- `docs/tech-spec.md`: what the system is.
- `docs/adr/`: why. A change that contradicts an ADR needs a new ADR, not a quiet edit.

Conventions that trip people up:

- Root namespace is `(ns the-circus)`. There is no `.core` anywhere.
- Integrant for the system; `init-key` lives in the component's own namespace.
- Malli, not spec. `ex-info` with a namespaced `:type`, not exception classes.
- 110 columns. `cljfmt` and `clj-kondo` are enforced.
- Commit to `main`.
