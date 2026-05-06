# Changelog

All notable changes to the **Tessera** protocol are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and Tessera adheres to [Semantic Versioning 2.0.0](https://semver.org/) as
clarified in [`SPEC.md` § Versioning](./SPEC.md#versioning).

Implementations should pin to a specific Tessera version and follow the
upgrade notes when bumping.

---

## [v0.1.5] — 2026-05-06 — Project creation verb

**Status: additive. No breaking changes; the v0.1.0 schema lock holds.**

### Added
- New `project.create` verb (`schemas/verbs/project.create.req.json`,
  `schemas/verbs/project.create.res.json`). Creates a project with a
  stable slug and display name. Optional `repo_path`. Idempotent via
  `operation_id`.
- Four conformance fixtures:
  - `project-create-happy` — successful creation.
  - `project-create-operation-replay` — idempotent replay returns cached project.
  - `project-create-slug-conflict` — 409 `slug_conflict` when slug is taken.
  - `project-create-validation-error` — 400 `validation_error` for invalid slug.

### Error codes
| Code | HTTP | Meaning |
|---|---|---|
| `slug_conflict` | 409 | The requested slug is already taken by another project. |

### Conformance notes
- Implementations claiming v0.1.5 conformance MUST implement `project.create`.
- Implementations NOT implementing `project.create` remain Tessera v0.1.4
  conformant and SHOULD NOT claim v0.1.5.

### Upgrade notes (v0.1.4 → v0.1.5)
- Add `POST /projects` (or equivalent verb endpoint) that accepts
  `{operation_id, slug, display_name, repo_path?}` and returns `{project}`.
- Wire idempotency via `operation_id` — same request body replays the cached
  project; differing body returns 409 `operation_id_conflict`.
- Enforce slug uniqueness — return 409 `slug_conflict` (not a generic 500) when
  the slug is already taken.
- Pass all four `project-create-*` conformance fixtures.

---

## [v0.1.4] — 2026-05-05 — Attachment resource and upload lifecycle

**Status: additive. No breaking changes; the v0.1.0 schema lock holds.**

### Added
- New `attachment` resource (`schemas/resources/attachment.json`). Two-phase
  lifecycle: `pending` (upload slot reserved) → `ready` (binary confirmed).
  Fields: `id`, `task_id`, `filename`, `content_type`, `size_bytes`, `status`,
  `url` (null until ready), `created_by`, `created_at`.
- Four new verbs for the attachment upload lifecycle:
  - `attachment.create_upload` — reserve an upload slot; returns the pending
    attachment and an opaque `upload_url` (local path or presigned cloud URL).
    Idempotent via `operation_id`.
  - `attachment.finalize` — confirm binary upload, transition to `ready`, set
    `url`. Idempotent via `operation_id` and domain-idempotent on already-ready
    attachments.
  - `attachment.get` — fetch a single attachment by id. Does NOT return
    `upload_url`.
  - `attachment.list` — list all non-deleted attachments for a task, ordered by
    `created_at` ascending, in `{attachments: [...]}` envelope.
- Four conformance fixture pairs covering the attachment happy path:
  `attachment-create-upload-happy`, `attachment-finalize-happy`,
  `attachment-get-happy`, `attachment-list-happy`.

### Unchanged
- All v0.1.0–v0.1.3 resources and verbs.
- Actor, task, project, event, and operation schemas.
- AgentContext response shape and 32KB truncation contract.

### Upgrade notes (v0.1.3 → v0.1.4)
- Implementations MUST add the `attachments` DB table and storage backend
  to claim v0.1.4 conformance.
- Implementations NOT implementing attachment verbs remain Tessera v0.1.3
  conformant — attachment support is additive.
- Authorization rule: attachment scope MUST derive from the task's project.
  Cross-project attachment access MUST be rejected.

---

## [v0.1.3] — 2026-04-29 — Agent registration + agent-session lifecycle

**Status: additive. No breaking changes; the v0.1.0 schema lock holds.**

### Added
- `actor.register` now supports both human and agent registration.
  Agent registration requires `agent_runtime` and `parent_actor_id`, and
  `parent_actor_id` must resolve to an active human actor responsible
  for the spawn.
- Two new actor-lifecycle verbs:
  - `actor.heartbeat` — agent-session verb that refreshes server-side
    liveness metadata without rotating the session credential.
  - `actor.deactivate` — agent-session verb that revokes or ends that
    session credential. Idempotent via `operation_id` and
    domain-idempotent when the session is already inactive.
- Additive schema updates for the expanded `actor.register` request /
  response branches and for the paired `actor.heartbeat` /
  `actor.deactivate` req/res files.
- Additive conformance fixtures covering agent registration, agent
  heartbeat, and agent deactivation flows.

### Clarified
- Token redaction on idempotent replay still applies to
  `actor.register` for both human and agent branches: the plaintext
  `token` is returned exactly once for a given `operation_id`, and
  replay returns `{actor}` without `token`.
- `actor.revoke_token` remains the credential-rotation primitive for
  humans, including the `last_admin_protected` guard path.
- `actor.deactivate` is not the human last-admin path; it applies to
  agent sessions only.

### Unchanged
- Existing task/project verbs and actor read verbs.
- The human-only `actor.revoke_token` contract from v0.1.2, aside from
  the clarified division of responsibility between human rotation and
  agent-session deactivation.

---

## [v0.1.2] — 2026-04-29 — Actor lifecycle verbs

**Status: minor. Additive only. No breaking changes; the v0.1.0 schema
lock holds. Implementations conformant to v0.1.0 / v0.1.1 remain
conformant to v0.1.2 without changes — they simply do not implement the
new verbs.**

### Added
- Four actor-lifecycle verbs:
  - `actor.register` — mint a new human actor and return a one-time
    plaintext token. Idempotent via `operation_id`.
  - `actor.list` — read; envelope `{actors: [...]}` with optional `kind`
    filter. Deliberate envelope (not a bare array) so future pagination
    fields can land additively.
  - `actor.get` — read by id.
  - `actor.revoke_token` — invalidate the actor's active credential.
    Idempotent via `operation_id` AND domain-idempotent (re-revoke is a
    no-op returning the same `{actor}` shape).
- Eight new schema files under `schemas/verbs/` (paired req/res for each
  verb).
- Eleven new conformance fixtures under `conformance/fixtures/`:
  - `actor-register-happy`, `actor-register-operation-replay`,
    `actor-register-invalid-kind`, `actor-register-validation-error`
  - `actor-list-happy`, `actor-list-filtered-by-kind`
  - `actor-get-happy`, `actor-get-not-found`
  - `actor-revoke-happy`, `actor-revoke-already-revoked`,
    `actor-revoke-not-found`
- New error code `last_admin_protected` (HTTP 409) on
  `actor.revoke_token` when the call would invalidate the last active
  human credential. Documented in SPEC; not exercised by a fixture in
  this release because it is a system-wide precondition, not a
  request-shape error.

### Token redaction rule (read this carefully)
The plaintext `token` is returned by `actor.register` exactly ONCE — on
the first call for a given `operation_id`. Idempotent replay (same
`operation_id` + same request body) returns `{actor}` ONLY; the `token`
field MUST be omitted. The schema permits both shapes by leaving `token`
out of `required`; the redaction rule is enforced by the paired
fixtures (`actor-register-happy.res.json` MUST contain `token`;
`actor-register-operation-replay.res.json` MUST NOT). There is
intentionally no token-recovery path — if a token is lost, the holder of
an active credential MUST `actor.revoke_token` and `actor.register` a
fresh actor. See [SPEC.md § Actor lifecycle](./SPEC.md#actor-lifecycle-v013).

### Intentional non-additions
- `parent_actor_id` is NOT in the `actor.register` request body. The
  resource schema still carries the field for agents that other code
  paths create, but accepting it on register today would be
  accepted-then-ignored ergonomics — it returns when agent registration
  itself returns, alongside a real capabilities/spawn model.
- `kind: "agent"` is NOT accepted on register in v0.1.2. The enum is
  `["human"]`. This is the deliberate humans-only scope for the verb;
  agent registration is deferred.
- `actor.rotate_token` (single-call revoke + re-register) is NOT in
  v0.1.2. The two primitives compose; implementations may expose a
  rotate convenience as a non-Tessera HTTP extension.
- `actor_events` audit log is NOT in v0.1.2; deferred to v0.2.

### Unchanged
- All v0.1.0 schemas and v0.1.1 fixtures — bytes-identical.
- All previous error codes and HTTP semantics.

---

## [v0.1.1] — Conformance fixture error-code alignment

**Status: patch. Schemas unchanged. Documentation/fixtures only.**

### Fixed
- Conformance fixture `_error.code` strings now match the codes the
  reference implementation (Sprino) actually emits, eliminating a
  contradiction with v0.1.0's strict `_error.code` matching contract:
  - `task-create-operation-conflict`: `operation_payload_mismatch` →
    `operation_id_conflict`
  - `task-update-status-version-conflict`: `version_conflict` →
    `version_mismatch`
  - `task-create-invalid-uuid`: `invalid_request` → `validation_error`
  - `task-create-missing-required-field`: `invalid_request` →
    `validation_error`

These were placeholder strings introduced in v0.1.0 without verifying
against a working implementation. The codes a conformant implementation
needs to emit are the ones the fixtures now declare.

### Unchanged
- All schemas (verbs and resources) — bytes-identical to v0.1.0.
- All happy-path fixtures.
- HTTP statuses on error fixtures (400, 409 still apply as before).

Implementations that already emit the new codes (Sprino does) require
zero changes.

---

## [v0.1.0] — Spec stabilization milestone

**Status: stable schemas. Additive changes only through v0.1.x.**

### Added
- **Versioning policy** ([`SPEC.md § Versioning`](./SPEC.md#versioning)) —
  formal definition of what counts as additive vs breaking, and how
  `v0.x` differs from `v1.x`.
- **Deprecation policy** ([`SPEC.md § Deprecation`](./SPEC.md#deprecation)) —
  90-day minimum notice before any field or verb is removed in the next
  major version, plus a `Tessera-Deprecation` header convention for runtime
  hints.
- **Migration guide template** ([`SPEC.md § Migration guide template`](./SPEC.md#migration-guide-template)) —
  every breaking-version bump must ship a migration guide using this
  template. Future major releases will publish one alongside the spec
  change.
- **Conformance section expansion** ([`SPEC.md § Conformance`](./SPEC.md#conformance)) —
  defines what passing the suite means, what's tested vs out of scope, and
  the reference test harness location.
- New conformance fixtures (paired `req`/`res` JSON):
  - `task-update-status-version-conflict` — 409 on stale `if_match`.
  - `task-create-operation-replay` — same `operation_id` returns the
    original response unchanged.
  - `task-create-operation-conflict` — same `operation_id`, different
    payload → 409.
  - `task-get-truncated` — `agent_context` over 32KB sets `truncated:true`
    with `next_page_tokens`.
  - `task-create-invalid-uuid` — malformed `operation_id` → 400.
  - `task-create-missing-required-field` — missing `title` → 400.

### Changed
- **Schemas locked.** No field renames, type narrowings, or removals are
  permitted in v0.1.x. New optional fields and new verbs MAY be added.
- README and SPEC headers reflect the v0.1.0 stabilization milestone.
- Roadmap reframed from weekly cadence to phase-based milestones for
  clarity.

### Out of scope (still planned for later)
- New verbs: `events.list`, `actor.register`, `actor.list`,
  `project.create`, `project.update`. These are deferred to a later phase
  once the v0.1 surface has been exercised by at least one third-party
  implementation.
- Comments, labels, search, sprints — re-evaluated as universal vs
  implementation-specific in v0.2 design.

---

## [v0.0.2] — 2026-04 — Project scoping + multi-repo

### Added
- `project.list` and `project.get` verbs — read-only project introspection.
- `repo_path` resolution on `task.create` — implementations MAY map a
  client-supplied repo path to a project; response always returns the
  resolved `task.project_id`.
- Frontend project switcher in the reference implementation (Sprino).

### Schemas
- `schemas/verbs/project.list.{req,res}.json`
- `schemas/verbs/project.get.{req,res}.json`
- `task.create.req.json` extended with optional `repo_path`.

---

## [v0.0.1] — First concrete schemas

### Added
- Five core resource schemas plus the `agent-context` companion schema
  under `schemas/resources/`: `actor`, `project`, `task`, `event`,
  `operation`, `agent-context`.
- Three verbs under `schemas/verbs/`: `task.create`, `task.get`,
  `task.update_status`.
- Three happy-path conformance fixtures.
- Idempotency rules (UUIDv7 client-supplied `operation_id`).
- Optimistic concurrency rules (`version` + `if_match`).

---

## [v0.0.0] — Bootstrap

### Added
- Repo scaffold: README founding story, MIT license, `SPEC.md` skeleton.
- No schemas yet — placeholder for protocol design discussion.

---

[v0.1.3]: https://github.com/leotorrealba/tessera/compare/v0.1.2...v0.1.3
[v0.1.2]: https://github.com/leotorrealba/tessera/compare/v0.1.1...v0.1.2
[v0.1.1]: https://github.com/leotorrealba/tessera/compare/v0.1.0...v0.1.1
[v0.1.0]: https://github.com/leotorrealba/tessera/compare/v0.0.2...v0.1.0
[v0.0.2]: https://github.com/leotorrealba/tessera/compare/v0.0.1...v0.0.2
[v0.0.1]: https://github.com/leotorrealba/tessera/compare/v0.0.0...v0.0.1
[v0.0.0]: https://github.com/leotorrealba/tessera/releases/tag/v0.0.0
