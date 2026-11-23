# 0025. Named, attachable state machines

Status: accepted, 2026-11-23

## Context
State machines recur across types: email confirmation on a new user, on a login, on any privileged request. Things outside a definition will want to refer to them: subscriptions now ("every confirmation, anywhere"), privilege later. Individual transitions are too small to be the unit; the machine is.

## Decision
- A **named** state machine is an entity of the built-in type `machine`, at `/s/{machine}`, with the same API shape as `/m/{type}`. It declares states, an initial state, terminal states, transitions and its own attributes.
- A **private** machine is written inline in one type's definition, for simple cases. Promoting it to a named machine is additive if identical.
- An entity type **attaches** a machine to a field. Each entity carries its own instance under that field: the state and the machine's attributes. Instances are not reified. Attaching the same machine more than once is allowed.
- A machine's transitions set only its own attributes, and its guards see only its instance. An attachment may add guards over the host, including one aggregate over owned references.
- Additive changes to a named machine apply automatically to every attaching type. Breaking changes to a machine attached to more than one type are refused in v0: fork it under a new name, move each type across with an ordinary breaking change, then retire the old one. Retiring a machine stops changes and new attachments; existing attachments keep running.
- The built-in types `entity` and `machine` both attach the built-in machine `active-retired` as `status`.
- The history feed can follow a machine across every type that attaches it: `/history?machine={name}&transition={name}`.

## Consequences
- One email-confirmation machine, written once, used everywhere, and changed in one place.
- A variable number of instances (N signatories) is modelled as its own entity type, owned by the document, with an aggregate guard.
- Forking keeps v0's one-type-at-a-time rule for breaking changes.
- Nested machines (a transition that waits on another machine) are deferred.
