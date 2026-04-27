# Tessera

**Tessera** is an open protocol for AI-native project state — tracking work performed by AI agents and humans together, with idempotent operations, structured agent context, and an append-only event log of who did what.

The name comes from Latin *tessera*: a small mosaic tile, and the wooden tile Roman soldiers carried with their unit's assignment and orders. The metaphor is literal — every task is a tessera, and the project is the mosaic that emerges when many of them are placed in coordination.

> **Status: pre-alpha.** Spec is evolving weekly. Breaking changes are expected through v0.1.

## What Tessera defines

A small set of resources that any AI-native project management tool needs in 2026:

- **Tasks** — units of work, with status, assignee, and version (for optimistic concurrency).
- **Projects** — scope boundaries. Tasks live in projects.
- **Actors** — humans and AI agents are first-class participants. Schema-level distinction, not a `user` table with a flag.
- **Events** — append-only log of every state change. Source of truth; materialized state is a projection.
- **Operations** — idempotency keys (UUIDv7) supplied by the client, with 30-day retention and 409-on-payload-mismatch semantics.
- **Agent Context** — structured response field on `task.get` so agents start with full state instead of re-explaining.

Specification: [SPEC.md](./SPEC.md). JSON Schemas land here in week 1 of the v0.0.1 milestone.

## Why Tessera (and not just MCP)

MCP (Model Context Protocol) defines how an AI client talks to a tool. It does not define what a project, task, or agent activity *is*. Today every PM tool that wants AI integration ships its own MCP server with its own data model. Tessera is the layer above MCP that says: if you are an AI-native PM tool, here is the shape your data should have so agents and humans can move work between systems without translation.

LSP did this for editors and language servers. OpenTelemetry did it for observability. Tessera does it for AI-native project state.

## Reference implementation

[Sprino](https://github.com/leotorrealba/sprino) is the reference implementation. Sprino implements Tessera v0.x and ships a working self-hosted PM tool with MCP integration. Tessera is the spec; Sprino is the working code.

The two are intentionally decoupled. Tessera is MIT-licensed because protocols belong to everyone. Sprino is AGPL v3 because reference implementations should keep their improvements in the commons.

## Conformance (coming in v0.0.1)

A `conformance/` directory will hold paired JSON request/response fixtures that any Tessera implementation MUST pass. This is the artifact a second implementer needs. If you want to know whether a tool "supports Tessera," the answer is: does it pass the fixture suite?

## Roadmap

- **v0.0.1** (current week) — three verbs (`task.create`, `task.get`, `task.update_status`), JSON Schemas for the core resources, three conformance fixtures.
- **v0.1.0** (week 5) — stable schemas, deprecation policy, migration guide template.
- **v0.2.0** — comments, labels, search, webhooks, sprints (re-evaluated as universal vs implementation-specific).
- **v1.0.0** — when there is a second implementation.

## Get involved

Open an issue if your AI-native PM tool wants to support Tessera, or if you have feedback on the schema. The protocol's job is to outlive any single implementation — the more variation we observe early, the better the v0.1 freeze.

## License

MIT. Implementations are free to use, modify, distribute, and sell. The protocol is a commons; implementations compete on quality.
