# Tessera v0.0.2

**Status:** v0.0.2 — project scoping and repo-aware task creation shipped. Breaking changes still expected through v0.1. Implementations should pin to a specific v0.0.x and follow upgrade notes.

## TL;DR for implementers

1. Install your DB schema for the five core resources (`actor`, `project`, `task`, `event`, `operation`). The shapes are defined in [`schemas/resources/`](./schemas/resources/).
2. Implement the v0.0.2 verbs (`project.list`, `project.get`, `task.create`, `task.get`, `task.update_status`). Request/response shapes are in [`schemas/verbs/`](./schemas/verbs/).
3. Make your implementation pass [`conformance/fixtures/`](./conformance/fixtures/). The fixtures form coherent project lookup and create → get → update_status narratives.

## Core resources

Five resources, five JSON Schemas. All schemas use JSON Schema 2020-12.

| Resource | Schema | Purpose |
| --- | --- | --- |
| **Actor** | [`schemas/resources/actor.json`](./schemas/resources/actor.json) | Humans and agents as first-class participants. `kind` discriminates at the schema level — agents are NOT users-with-a-flag. |
| **Project** | [`schemas/resources/project.json`](./schemas/resources/project.json) | Scope boundary. Tasks live in projects. |
| **Task** | [`schemas/resources/task.json`](./schemas/resources/task.json) | Unit of work. `status` is a materialized projection of the latest `status_changed` event. |
| **Event** | [`schemas/resources/event.json`](./schemas/resources/event.json) | Append-only authoritative state changes. Source of truth. MUST NOT be mutated or deleted. |
| **Operation** | [`schemas/resources/operation.json`](./schemas/resources/operation.json) | Server-side idempotency dedup record. 30-day minimum retention. |
| **AgentContext** | [`schemas/resources/agent-context.json`](./schemas/resources/agent-context.json) | Structured response field on `task.create` and `task.get`. Caps at 32KB; over-cap responses set `truncated:true` with `next_page_tokens`. |

## Verbs (v0.0.2)

| Verb | Request | Response | Idempotent? |
| --- | --- | --- | --- |
| `project.list` | [req](./schemas/verbs/project.list.req.json) | [res](./schemas/verbs/project.list.res.json) | N/A (read-only) |
| `project.get` | [req](./schemas/verbs/project.get.req.json) | [res](./schemas/verbs/project.get.res.json) | N/A (read-only) |
| `task.create` | [req](./schemas/verbs/task.create.req.json) | [res](./schemas/verbs/task.create.res.json) | Yes (operation_id) |
| `task.get` | [req](./schemas/verbs/task.get.req.json) | [res](./schemas/verbs/task.get.res.json) | N/A (read-only) |
| `task.update_status` | [req](./schemas/verbs/task.update_status.req.json) | [res](./schemas/verbs/task.update_status.res.json) | Yes (operation_id) + concurrency-safe (if_match) |

`task.create` accepts either `project_id` or `repo_path`. Implementations MAY resolve `repo_path` through a repo-root mapping, a project table, or client-side detection such as `.sprino/project.id`; the response always contains the resolved `task.project_id`.

## Idempotency rules (`operation_id`)

- **Format:** UUIDv7 supplied by the client. Implementations MUST NOT generate operation_ids server-side.
- **Retention:** 30 days minimum. After expiry, retries return `410 Gone`.
- **Replay semantics:**
  - Same `operation_id` + same `request_hash` (SHA-256 of canonical JSON) → return the cached response verbatim.
  - Same `operation_id` + different `request_hash` → return `409 Conflict` with the original cached payload.
- **Server cleanup:** Implementations SHOULD purge expired operations on a daily cadence. The protocol does not mandate a specific schedule.

## Optimistic concurrency (`version`)

- Every mutating verb takes `if_match: <version>`.
- On match: server writes the event, increments the version, returns the new task and event with status `200 OK`.
- On mismatch: server returns `409 Conflict` with the SERVER'S current task body (no new event written, no version increment).
- Reasoning: AI agents tend not to pause to check state. Last-write-wins is a footgun.

## Event log semantics

- Events are append-only and authoritative.
- Mutating operations write the event BEFORE updating materialized state, in the same transaction.
- Materialized state (`tasks.status`, `tasks.assignee_id`) is recomputable by replaying events in `created_at` order. Implementations MAY keep mutable rows for read performance, but events remain the source of truth.
- Implementations MAY expose a future `events.list` verb (out of scope for v0.0.2) that lets agents query event history without polling.

## Transport-agnostic

The protocol does not mandate a transport. Implementations MAY expose verbs over:
- **HTTP REST** — e.g. `POST /api/tasks` (create), `GET /api/tasks/:id` (get), `PATCH /api/tasks/:id/status` (update).
- **MCP** — e.g. `sprino.task.create` as an MCP tool over stdio or HTTP.
- **gRPC, WebSocket, anything else** — as long as the request/response shapes match.

The reference implementation ([Sprino](https://github.com/leotorrealba/sprino)) exposes both HTTP and MCP-over-HTTP from the same Hono process.

## Conformance

A runnable conformance suite lives in [`conformance/`](./conformance/). See [`conformance/README.md`](./conformance/README.md) for how to run it against your implementation.

## Out of scope for v0.0.2 (planned for v0.0.x → v0.1)

- Concurrency conflict scenarios (multiple writers).
- Operation_id reuse with mismatched payload (409 case).
- Operation expiry (410 Gone).
- 32KB agent_context truncation + pagination.
- Multi-actor narratives.
- Event log replay verbs (`events.list`).
- Project mutation verbs (`project.create`, `project.update`).
- Actor lifecycle verbs (`actor.register`, `actor.list`).
- Comments, labels, search, sprints (re-evaluated as universal vs implementation-specific in v0.2 design).

These will graduate into v0.1 once stable.

## Reference implementation

[Sprino](https://github.com/leotorrealba/sprino) — TypeScript + Hono + Drizzle + Postgres + MCP SDK. Self-hostable. Single-tenant for v0.x; multi-tenant cloud product post-graduation.
