# Tessera Conformance Tests

This directory holds paired request/response fixtures that any Tessera
implementation MUST pass to claim Tessera support. The pinned target
version is in [`SPEC.md`](../SPEC.md) (currently v0.1.0).

## Layout

```
conformance/
└── fixtures/
    # Happy path — read & write narrative
    ├── project-list-happy.{req,res}.json
    ├── project-get-by-repo-path-happy.{req,res}.json
    ├── task-create-happy.{req,res}.json
    ├── task-get-happy.{req,res}.json
    ├── task-update-status-happy.{req,res}.json
    # Idempotency
    ├── task-create-operation-replay.{req,res}.json     # same op_id, same payload → same response
    ├── task-create-operation-conflict.{req,res}.json   # same op_id, different payload → 409
    # Optimistic concurrency
    ├── task-update-status-version-conflict.{req,res}.json  # stale if_match → 409
    # Agent context
    ├── task-get-truncated.{req,res}.json               # large agent_context → truncated:true + next_page_tokens
    # Validation errors
    ├── task-create-invalid-uuid.{req,res}.json         # malformed operation_id → 400
    └── task-create-missing-required-field.{req,res}.json  # omitted title → 400
```

Each fixture is a pair: a `*.req.json` file with the verb input, and a
`*.res.json` with the expected response.

## Fixture conventions

### Happy-path response shape

A `*.res.json` for a happy path is the **body** of the successful response
(2xx). It contains the verb's response payload directly.

### Error response shape

A `*.res.json` for an error case wraps the expected error in a top-level
`_error` envelope:

```json
{
  "_error": {
    "status": 400 | 409 | 410,
    "code": "invalid_request" | "version_conflict" | "operation_payload_mismatch" | ...,
    "details": { /* implementation-defined diagnostic fields */ }
  }
}
```

Implementations MAY return the same status code with a different
top-level body shape (the spec is transport-agnostic). The harness
compares `_error.status` and `_error.code` exactly; `_error.details` is
informational unless the fixture asserts a specific subfield.

### `_meta` field

Both `*.req.json` and `*.res.json` MAY include an `_meta` field at the
top level. It documents preconditions and expected behavior in human
terms. The harness MUST ignore `_meta` when validating shape — it is
documentation, not protocol.

## How to use them

1. **Seed your implementation's database** with the actor and projects
   the fixtures reference:
   - Actor: `018c3e7a-0001-7000-8000-000000000001` (kind=`human`)
   - Project: `018c3e7a-0002-7000-8000-000000000001` (`sprino`)
   - Project: `018c3e7a-0002-7000-8000-000000000002` (`tessera`)

2. **Run fixtures in dependency order.** Some fixtures depend on prior
   state (see `_meta.preconditions` in each `*.res.json`). The canonical
   order is:

   ```
   project-list-happy
   project-get-by-repo-path-happy
   task-create-happy                          # creates v1 of task ...0001
   task-get-happy                             # reads v1
   task-create-operation-replay               # same op_id → same response, no new event
   task-create-operation-conflict             # same op_id, different payload → 409
   task-create-invalid-uuid                   # 400, no state change
   task-create-missing-required-field         # 400, no state change
   task-update-status-happy                   # v1 → v2
   task-update-status-version-conflict        # if_match=1 against v2 → 409

   # Independent narrative — separate task with large agent_context
   task-get-truncated                         # truncated:true + next_page_tokens
   ```

3. **Compare actual response to expected** with these allowed
   normalizations:
   - **Timestamps** (`created_at`, `updated_at`, `event.created_at`) may
     differ — assert ISO-8601 and correct ordering.
   - **`event.id`** may differ — assert valid UUID, distinct from other
     event ids in the response.
   - **`task.id`** in happy-path fixtures that are referenced by later
     requests MUST match the fixture value exactly. The conformance
     suite does not define a harness-side ID-substitution mechanism for
     dependent requests, so implementations that generate task ids
     server-side MUST do so deterministically for the conformance
     inputs — i.e. `task-create-happy` MUST return the fixture-matching
     id used by subsequent `task-get-*` and `task-update-*` requests.
   - **For `task-get-truncated`**: `agent_context.related_tasks`,
     `recent_events`, `repo_refs` content is implementation-dependent in
     volume. The corresponding collection MAY be empty even if a
     pagination token is set (a token signals that the implementation
     has more data available; it does not guarantee the current page is
     non-empty). Total response body MUST stay under 32KB.
   - All other structural fields MUST match exactly.

## What "passes" means

An implementation passes Tessera v0.1.0 if:

- All happy-path fixture inputs validate against
  `schemas/verbs/<verb>.req.json`.
- All happy-path fixture outputs validate against
  `schemas/verbs/<verb>.res.json`.
- Error fixtures may intentionally use requests that do **not**
  validate against `schemas/verbs/<verb>.req.json`; the implementation
  MUST reject such requests, with actual response status code matching
  `_error.status` and a documented error code matching `_error.code`.
- All happy-path responses match the expected fixtures modulo the
  normalizations above.

## What's still NOT covered (planned for v0.1.x additive)

- Multi-actor narratives (agent activity attribution across multiple
  actors in one project).
- `events.list` replay verb (event-sourced rebuild).
- Operation expiry (after 30 days → 410 Gone). Long-running scenario,
  not easily expressible in a sub-second test fixture.
- Project mutation verbs (`project.create`, `project.update`).
- Actor lifecycle verbs (`actor.register`, `actor.list`).

These will land additively in `v0.1.x` minor releases.

## Reference implementation

[Sprino](https://github.com/leotorrealba/sprino) is the reference
implementation. Its CI runs these fixtures against a real Postgres on
every PR via `apps/server/test/conformance.test.ts`. If Sprino's
response shape ever drifts from these fixtures, that's a Tessera-protocol
bug — not an implementation bug — and gets fixed here first.
