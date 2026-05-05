# Tessera v0.1.4

**Status:** v0.1.4 — additive release. The v0.1.0 schema lock holds; v0.1.4 adds the `attachment` resource and four attachment verbs (`attachment.create_upload`, `attachment.finalize`, `attachment.get`, `attachment.list`) without modifying any v0.1.0–v0.1.3 schema incompatibly. Additive changes (new optional fields, new verbs, new resources) remain permitted in v0.1.x; breaking changes require a v0.2.0 bump and a migration guide. Implementations should pin to a specific v0.1.x and follow [`CHANGELOG.md`](./CHANGELOG.md).

## TL;DR for implementers

1. Install your DB schema for the five core resources (`actor`, `project`, `task`, `event`, `operation`) and the `attachment` resource added in v0.1.4. Support the companion `AgentContext` response schema. The shapes are defined in [`schemas/resources/`](./schemas/resources/).
2. Implement the pinned v0.1.x verb set you claim to support. For `v0.1.4`, the full surface is `project.list`, `project.get`, `task.create`, `task.get`, `task.update_status`, `actor.register`, `actor.list`, `actor.get`, `actor.revoke_token`, `actor.heartbeat`, `actor.deactivate`, `attachment.create_upload`, `attachment.finalize`, `attachment.get`, and `attachment.list`. Request/response shapes are in [`schemas/verbs/`](./schemas/verbs/).
3. Make your implementation pass [`conformance/fixtures/`](./conformance/fixtures/). The fixtures form coherent project/task, actor-lifecycle, and attachment narratives.

## Core resources

Six core resources plus one companion schema. All schemas use JSON Schema 2020-12.

| Resource | Schema | Purpose |
| --- | --- | --- |
| **Actor** | [`schemas/resources/actor.json`](./schemas/resources/actor.json) | Humans and agents as first-class participants. `kind` discriminates at the schema level — agents are NOT users-with-a-flag. |
| **Project** | [`schemas/resources/project.json`](./schemas/resources/project.json) | Scope boundary. Tasks live in projects. |
| **Task** | [`schemas/resources/task.json`](./schemas/resources/task.json) | Unit of work. `status` is a materialized projection of the latest `status_changed` event. |
| **Event** | [`schemas/resources/event.json`](./schemas/resources/event.json) | Append-only authoritative state changes. Source of truth. MUST NOT be mutated or deleted. |
| **Operation** | [`schemas/resources/operation.json`](./schemas/resources/operation.json) | Server-side idempotency dedup record. 30-day minimum retention. |
| **Attachment** (v0.1.4) | [`schemas/resources/attachment.json`](./schemas/resources/attachment.json) | File attached to a task. Two-phase lifecycle: `pending` (slot reserved) → `ready` (binary confirmed). |
| **AgentContext** | [`schemas/resources/agent-context.json`](./schemas/resources/agent-context.json) | Structured response field on `task.create` and `task.get`. Caps at 32KB; over-cap responses set `truncated:true` with `next_page_tokens`. |

## Verbs (v0.1.4 surface)

| Verb | Request | Response | Idempotent? |
| --- | --- | --- | --- |
| `project.list` | [req](./schemas/verbs/project.list.req.json) | [res](./schemas/verbs/project.list.res.json) | N/A (read-only) |
| `project.get` | [req](./schemas/verbs/project.get.req.json) | [res](./schemas/verbs/project.get.res.json) | N/A (read-only) |
| `task.create` | [req](./schemas/verbs/task.create.req.json) | [res](./schemas/verbs/task.create.res.json) | Yes (operation_id) |
| `task.get` | [req](./schemas/verbs/task.get.req.json) | [res](./schemas/verbs/task.get.res.json) | N/A (read-only) |
| `task.update_status` | [req](./schemas/verbs/task.update_status.req.json) | [res](./schemas/verbs/task.update_status.res.json) | Yes (operation_id) + concurrency-safe (if_match) |
| `actor.register` (v0.1.3) | [req](./schemas/verbs/actor.register.req.json) | [res](./schemas/verbs/actor.register.res.json) | Yes (operation_id); replay redacts `token` |
| `actor.list` (v0.1.2) | [req](./schemas/verbs/actor.list.req.json) | [res](./schemas/verbs/actor.list.res.json) | N/A (read-only) |
| `actor.get` (v0.1.2) | [req](./schemas/verbs/actor.get.req.json) | [res](./schemas/verbs/actor.get.res.json) | N/A (read-only) |
| `actor.revoke_token` (v0.1.2) | [req](./schemas/verbs/actor.revoke_token.req.json) | [res](./schemas/verbs/actor.revoke_token.res.json) | Yes (operation_id) + domain-idempotent |
| `actor.heartbeat` (v0.1.3) | [req](./schemas/verbs/actor.heartbeat.req.json) | [res](./schemas/verbs/actor.heartbeat.res.json) | No `operation_id`; repeat calls refresh liveness for the same agent session |
| `actor.deactivate` (v0.1.3) | [req](./schemas/verbs/actor.deactivate.req.json) | [res](./schemas/verbs/actor.deactivate.res.json) | Yes (operation_id) + domain-idempotent |
| `attachment.create_upload` (v0.1.4) | [req](./schemas/verbs/attachment.create_upload.req.json) | [res](./schemas/verbs/attachment.create_upload.res.json) | Yes (operation_id) |
| `attachment.finalize` (v0.1.4) | [req](./schemas/verbs/attachment.finalize.req.json) | [res](./schemas/verbs/attachment.finalize.res.json) | Yes (operation_id) + domain-idempotent |
| `attachment.get` (v0.1.4) | [req](./schemas/verbs/attachment.get.req.json) | [res](./schemas/verbs/attachment.get.res.json) | N/A (read-only) |
| `attachment.list` (v0.1.4) | [req](./schemas/verbs/attachment.list.req.json) | [res](./schemas/verbs/attachment.list.res.json) | N/A (read-only) |

`task.create` accepts either `project_id` or `repo_path`. Implementations MAY resolve `repo_path` through a repo-root mapping, a project table, or client-side detection such as `.sprino/project.id`; the response always contains the resolved `task.project_id`.

## Actor lifecycle (v0.1.3)

The six actor-lifecycle verbs let implementations onboard humans and agent sessions, list participants for assignment UIs, refresh agent liveness metadata, and end or rotate credentials without transport-specific assumptions.

- **`actor.register`** mints a new actor and returns the plaintext credential ONCE. The request supports both human and agent branches. Human registration remains the baseline path. Agent registration MUST provide `agent_runtime` and `parent_actor_id`; the `parent_actor_id` MUST resolve to an active human actor responsible for the agent spawn. Idempotent replay with the same `operation_id` redacts the `token` field for both branches.
- **`actor.list`** returns an envelope object `{actors: [...]}` with an optional `kind` filter. Bare-array responses are not used in Tessera so future pagination fields can land additively. Results MUST be ordered by `created_at` ascending; ties MUST be broken by `id` ascending. This ordering is normative so conformance fixtures can assert exact array position.
- **`actor.get`** fetches a single actor by id. Missing actors return a `not_found` protocol error; HTTP transports commonly map that to `404`.
- **`actor.revoke_token`** invalidates the active credential for a human actor. It remains the credential-rotation primitive for humans. The verb is idempotent both via `operation_id` (replay returns the cached response) and at the domain level (re-revoking an already-revoked human returns the same `{actor}` shape with no error and writes no new event). Implementations MUST guard against revoking the last active human credential and MUST return error code `last_admin_protected` when the call would remove the last active human able to authenticate; HTTP transports commonly map that guard to `409 Conflict`.
- **`actor.heartbeat`** refreshes server-side liveness metadata for an authenticated agent session without rotating or replacing that session credential. The v0.1.3 request identifies the session by `actor_id` only; this verb does not use `operation_id`. Repeated calls refresh liveness metadata and MUST NOT mint credentials or mutate runtime or parent-linkage metadata.
- **`actor.deactivate`** revokes or ends an authenticated agent session credential. It is idempotent via `operation_id` and domain-idempotent when the session is already inactive. This is not the human `last_admin_protected` guard path; that protection remains specific to `actor.revoke_token`.

> **Token recovery: there isn't one.**
>
> The plaintext token is returned exactly once on `actor.register`, whether the registered actor is human or agent. On idempotent replay (same `operation_id`) the response is `{actor}` only — the `token` field is omitted. There is intentionally no recovery path. If a human token is lost, an authenticated holder of an active human credential MUST call `actor.revoke_token` and then `actor.register` a fresh human actor. If an agent session token is lost, the responsible active human MUST end that agent session with `actor.deactivate` and then `actor.register` a fresh agent session. This keeps Tessera's credential model honest: no plaintext-at-rest, no "show me the token again" verb, no parallel reset flow to audit.

`actor.revoke_token` is the in-protocol primitive for human credential rotation. `actor.deactivate` is the in-protocol primitive for ending an agent session credential. A more ergonomic single-call rotate verb is intentionally not part of the v0.1.3 surface; implementations are free to expose one as a non-Tessera extension on top of these primitives.

## Attachment lifecycle (v0.1.4)

Attachments associate files with tasks using a two-phase upload contract that is compatible with both local server storage and cloud presigned-URL storage backends.

- **`attachment.create_upload`** reserves an upload slot and returns the pending attachment record plus an opaque `upload_url`. The `upload_url` is implementation-defined — it MAY be a local server endpoint or a presigned cloud storage URL. Clients MUST PUT the raw file bytes to `upload_url` before calling `attachment.finalize`. Idempotent via `operation_id`.
- **`attachment.finalize`** confirms the binary upload and transitions the attachment from `pending` to `ready`. After finalization `url` is set to an opaque download URL. The verb is idempotent via `operation_id` and domain-idempotent (calling finalize on an already-`ready` attachment with the same `operation_id` is a no-op replay).
- **`attachment.get`** retrieves a single attachment by id. Implementations MUST NOT include `upload_url` in get responses.
- **`attachment.list`** returns all `ready` attachments for a task, ordered by `created_at` ascending, in an `{attachments: [...]}` envelope. Pending attachments MUST NOT appear in list responses; they are only visible via `attachment.get` by id.

Attachment authorization derives from the task's project scope: actors authorized on the task's project MAY create and retrieve attachments. Implementations MUST enforce project-scope isolation; cross-project attachment access MUST be rejected.

The `upload_url` returned by `attachment.create_upload` is valid for a single upload attempt. Implementations MAY set an expiry on presigned URLs; clients SHOULD call `attachment.create_upload` again if the upload slot has expired.

## Idempotency rules (`operation_id`)

- **Format:** UUIDv7 supplied by the client. Implementations MUST NOT generate operation_ids server-side.
- **Retention:** 30 days minimum. After expiry, retries return `410 Gone`.
- **Replay semantics:**
  - Same `operation_id` + same `request_hash` (SHA-256 of canonical JSON) → return the cached response for that operation. By default this is the original payload verbatim; verbs MAY define explicitly documented replay-safe redactions for sensitive fields. `actor.register` is the v0.1.3 example: replay returns `{actor}` with `token` omitted.
  - Same `operation_id` + different `request_hash` → return `409 Conflict` with the original cached payload. For `actor.register`, the cached payload MUST have the `token` field omitted even in the conflict response — the "shown exactly once" guarantee applies to all responses derived from the operation cache, including conflict responses.
- **Server cleanup:** Implementations SHOULD purge expired operations on a daily cadence. The protocol does not mandate a specific schedule.

## Optimistic concurrency (`version`)

- `if_match: <version>` applies only to verbs that mutate an existing versioned task record.
- In the pinned v0.1.3 contract, that means `task.update_status`. It does not apply to `actor.register`, `actor.heartbeat`, or `actor.deactivate`.
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

A runnable conformance suite lives in [`conformance/`](./conformance/). See
[`conformance/README.md`](./conformance/README.md) for the harness contract.

### What "Tessera-conformant" means

An implementation is **Tessera v0.1.x conformant** when:

1. **Every fixture in [`conformance/fixtures/`](./conformance/fixtures/) passes.**
   Fixtures come in paired `*.req.json` / `*.res.json` files. The
   implementation accepts the request and returns a response that
   structurally matches `*.res.json` modulo:
   - Server-generated UUIDs (any field whose schema marks `format: uuid`
     and is not part of the request).
   - Server-generated timestamps (`created_at`, `updated_at`).
   - Implementation-specific projection metadata (e.g. database row IDs)
     that the schemas do not require.

2. **All verbs for the pinned v0.1.x target are implemented** with the
   request/response shapes in [`schemas/verbs/`](./schemas/verbs/). For
   `v0.1.3`, that means `project.list`, `project.get`, `task.create`,
   `task.get`, `task.update_status`, `actor.register`, `actor.list`,
   `actor.get`, `actor.revoke_token`, `actor.heartbeat`, and
   `actor.deactivate`.

3. **Idempotency, concurrency, and event-log rules** from this document
   are enforced (operation replay, 409 on `operation_id` payload
   mismatch, 410 on expired operations, optimistic concurrency via
   `if_match`/`version`).

### What conformance does NOT cover

- Transport (HTTP, MCP, gRPC). The reference implementation exposes both
  HTTP and MCP, but the spec is transport-agnostic.
- Storage backend (Postgres, SQLite, in-memory). Implementations may
  store events however they like as long as the projection rules hold.
- Latency / throughput. Tessera defines correctness, not performance.
- Authentication / authorization. These are implementation concerns;
  Tessera assumes a trust boundary above the protocol layer.

### Reference harness

The reference implementation [Sprino](https://github.com/leotorrealba/sprino)
runs the entire fixture suite in CI as `apps/server/test/conformance.test.ts`.
That harness is the canonical way to verify your implementation: clone
Tessera as a git submodule (or sibling), point the test runner at it,
and watch every fixture replay against your server.

## Versioning

Tessera uses [Semantic Versioning 2.0.0](https://semver.org/) with the
following clarifications.

### What counts as a change

| Change | Bump |
| --- | --- |
| Adding a new optional field on a request or response | PATCH within the frozen `v0.1.x` line; MINOR otherwise |
| Adding a new verb | PATCH within the frozen `v0.1.x` line; MINOR otherwise |
| Adding a new conformance fixture (covering existing behavior) | PATCH |
| Clarifying or restating existing rules without changing them | PATCH |
| Renaming or removing a field | MAJOR |
| Tightening a type (e.g. making an optional field required) | MAJOR |
| Changing a verb's idempotency or concurrency semantics | MAJOR |
| Removing a verb | MAJOR (after a deprecation cycle, see below) |

### `v0.x` vs `v1.x`

- **`v0.x`** — schemas may evolve. Breaking changes ship as MINOR bumps
  during this phase (`v0.0.2` → `v0.1.0` is breaking-allowed). The
  `v0.1.0` milestone freezes the v0.1 surface so implementers can build
  against a stable target. Within `v0.1.x`, only additive changes ship.
- **`v1.x`** — full semver applies. Breaking changes require a `v2.0.0`
  bump and a migration guide.

### Pinning

Implementations MUST declare which Tessera version they target (e.g.,
`"tessera": "0.1.0"` in their manifest, README, or response metadata).
Tooling SHOULD reject mismatches at the major or minor level rather than
silently downgrading.

## Deprecation

A field, verb, or rule may be **deprecated** before removal. The deprecation
lifecycle is:

1. **Announce.** The next minor release marks the item deprecated:
   - In schemas: `"deprecated": true` and `"description"` updated to
     name the replacement (per JSON Schema 2020-12 keyword).
   - In SPEC.md: a "Deprecated in `vX.Y.Z`" note next to the item.
   - In CHANGELOG.md: a `### Deprecated` block.

2. **Notice period.** Minimum **90 days** of wall-clock time AND at
   least one minor release between the deprecation announcement and the
   removal.

3. **Runtime hint (optional).** Implementations MAY include a
   `Tessera-Deprecation: <field-or-verb>; sunset=<RFC3339-date>` HTTP
   response header (or equivalent for non-HTTP transports) when a client
   uses a deprecated item, so SDKs can warn at runtime.

4. **Removal.** The next major version (`vN.0.0`) removes the item. The
   removal must be listed in the migration guide (see template below).

Implementations are NOT required to emit deprecation warnings, but if
they do, they MUST use the `Tessera-Deprecation` header convention so
clients can detect them generically.

## Migration guide template

Every breaking-version bump (any MAJOR, plus the `v0.x → v0.x+1`
breaking releases during the `v0.x` phase) ships a migration guide using
this template. The guide lives at `docs/migrations/vN.Y.Z.md` in this
repo.

```markdown
# Migrating to Tessera vN.Y.Z

**From:** vN.Y-1.Z (or vN-1.x.y)
**To:** vN.Y.Z
**Released:** YYYY-MM-DD
**Migration window:** how long the previous version remains supported

## Summary

One paragraph: what changes and why.

## Breaking changes

For each breaking change:

### `<verb>` / `<resource>` / `<rule>`

- **What changed:** before vs after, with code or schema snippets.
- **Why:** the constraint or design driver.
- **Detection:** how a v(N-1) implementation looks on the wire vs
  vN.Y.Z. (Often: a specific HTTP status code, response field, or
  schema-validation error.)
- **Mechanical fix:** the smallest change to bring an implementation
  forward. Include diffs where useful.

## New additive features

(Optional, if MINOR-additive changes also shipped.)

## Test fixtures

List which `conformance/fixtures/` files cover each breaking change so
implementers can verify their migration.

## Acknowledgments

(Optional.)
```

## Out of scope for v0.1 (planned for later)

These were considered for v0.1.0 and intentionally deferred until the
v0.1 surface has been exercised by at least one third-party
implementation:

- Event log replay verbs (`events.list`).
- Project mutation verbs (`project.create`, `project.update`).
- Audit log of actor-lifecycle events (`actor_events`). Deferred to
  v0.2 alongside the broader event-log expansion.
- Comments, labels, search, webhooks, sprints (re-evaluated as universal
  vs implementation-specific in v0.2 design).

These will land additively in future `v0.1.x` releases (no breaking
change) once the design is validated.

## Reference implementation

[Sprino](https://github.com/leotorrealba/sprino) — TypeScript + Hono + Drizzle + Postgres + MCP SDK. Self-hostable. Single-tenant for v0.x; multi-tenant cloud product post-graduation.
