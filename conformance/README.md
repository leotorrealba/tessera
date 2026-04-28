# Tessera Conformance Tests

This directory holds paired request/response fixtures that any Tessera implementation MUST pass to claim "Tessera v0.0.2 support."

## Layout

```
conformance/
└── fixtures/
    ├── task-create-happy.req.json       # input
    ├── task-create-happy.res.json       # expected output
    ├── task-get-happy.req.json
    ├── task-get-happy.res.json
    ├── task-update-status-happy.req.json
    ├── task-update-status-happy.res.json
    ├── project-list-happy.req.json
    ├── project-list-happy.res.json
    ├── project-get-by-repo-path-happy.req.json
    └── project-get-by-repo-path-happy.res.json
```

Each fixture is a pair: a `*.req.json` file containing the verb input, and a `*.res.json` file containing the expected response.

## How to use them

1. **Seed your implementation's database** with the actor and project the fixtures reference:
   - Actor: `018c3e7a-0001-7000-8000-000000000001` (kind=`human`)
   - Project: `018c3e7a-0002-7000-8000-000000000001` (`sprino`)
   - Project: `018c3e7a-0002-7000-8000-000000000002` (`tessera`)
2. **For each `*.req.json` file**, send it as input to the corresponding verb (over HTTP `/api/...`, MCP `/mcp/...`, or any transport your implementation chooses).
3. **Compare the actual response to the matching `*.res.json` file**, with these allowed normalizations:
   - **Timestamps** (`created_at`, `updated_at`) may differ — assert they are valid ISO-8601 datetimes and ordered correctly relative to other events.
   - **`event.id`** may differ — assert it is a valid UUID, distinct from other event ids in the response.
   - All other fields including `task.id`, `actor_id`, `version`, `status`, `payload` contents, and structural shape MUST match exactly.
4. **Run task fixtures in sequence in a single test**: create → get → update_status. The task fixtures form a coherent narrative; running them out of order will fail because each assumes the prior state.

## What "passes" means

An implementation passes Tessera v0.0.2 if:
- All fixture inputs validate against the corresponding `schemas/verbs/<verb>.req.json`.
- All fixture outputs validate against the corresponding `schemas/verbs/<verb>.res.json`.
- The actual responses match the expected fixtures (modulo the normalizations above).
- Idempotent retry of `task-create-happy.req.json` returns the same response with no new events.
- Calling `task-update-status-happy.req.json` again with `if_match=1` after the first call (when version is now 2) returns 409 Conflict with the server's current task body.

## What's NOT covered yet (v0.0.3+)

- Concurrency conflict scenarios (two writers, optimistic-concurrency 409).
- Operation_id reuse with mismatched payload (request_hash mismatch → 409).
- Operation expiry (after 30 days → 410 Gone).
- 32KB agent_context truncation with `next_page_tokens`.
- Multi-actor narratives (agent activity attribution).
- Event log replay / event-sourced rebuild.

These will land as v0.0.x increments and graduate into v0.1 once stable.

## Reference implementation

[Sprino](https://github.com/leotorrealba/sprino) is the reference implementation. Its CI runs these fixtures against a real Postgres on every PR. If Sprino's response shape ever drifts from these fixtures, that's a Tessera-protocol bug — not an implementation bug — and gets fixed here first.
