# the-circus

Software engineering is data processing.

Right now agent swarms guided by well-meaning humans are gluing shiz together, from end-user journey down to PostgreSQL or whatever. This project attempts to add structure to that interaction.

Specifically, by co-opting an append-only bitemporal database, then layering a data model and workflow plugin on top. That should canalise the insanity of the agents, so we get the value of the AI revolution without the `DROP TABLE users;` auditless hellscape.

*the-circus*, after Le Carré: an attempt to bring order to unruly and sometimes doubled agents.

## Docs

- [Business spec](docs/business-spec.md): what the-circus is, who it's for, and the demo.
- [Tech spec](docs/tech-spec.md): architecture, metamodel, APIs and open questions.
- [Decision records](docs/adr/): one file per decision.
