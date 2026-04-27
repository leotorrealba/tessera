# Tessera v0.0.1

**Status:** pre-alpha. Schemas TBD week 1. Breaking changes expected through v0.1.

## Verbs (week 1 deliverable)

- `task.create` — create a task in a project. Takes `operation_id` (UUIDv7, idempotency), `project_id`, `title`, optional `description` and `assignee_id`. Returns the created task plus an `agent_context` block.
- `task.get` — fetch a task with its agent_context (related tasks, recent events, repo refs). Capped at 32KB serialized; pagination flags returned when truncated.
- `task.update_status` — mutate a task's status. Takes `if_match: <version>` for optimistic concurrency; returns 409 on version mismatch with the current task body.

## Core resources (preview)

```
actor      (id, kind 'human|agent', display_name, agent_runtime, parent_actor_id, created_at)
project    (id, slug, display_name, repo_path, created_at)
task       (id, project_id, title, description, status 'todo|doing|done|blocked',
            assignee_id, created_by, version, created_at, updated_at)
event      (id, task_id, actor_id, kind, payload, operation_id, created_at)  -- APPEND-ONLY
operation  (operation_id pk, actor_id, request_hash, response_body, created_at, expires_at)
```

## Idempotency rules (v0.0.1)

- `operation_id` is UUIDv7 supplied by the client on every mutating verb.
- 30-day retention. After expiry, replay returns 410 Gone.
- Same `operation_id` + same `request_hash` → cached response.
- Same `operation_id` + different `request_hash` → 409 Conflict with the original payload.

## Optimistic concurrency

Every mutating verb takes `if_match: <version>`. On mismatch → 409 Conflict with the current task body. Last-write-wins is a footgun for AI agents that don't pause to check state.

## Event log semantics

Events are append-only and authoritative. Materialized state (`tasks.status`, `tasks.assignee_id`) is recomputable by replaying events. Implementations MAY keep mutable rows for read performance, but events are the source of truth.

## JSON Schemas

Coming in week 1 of v0.0.1. Will live in `schemas/`. Each verb will have a request schema and a response schema.

## Conformance

Fixtures land in `conformance/` in v0.0.1. Format: paired `<verb>-<scenario>.req.json` and `<verb>-<scenario>.res.json` files. An implementation passes Tessera if it returns responses that validate against the response schemas for every fixture's request.

## Reference implementation

[Sprino](https://github.com/leotorrealba/sprino).
