# Changelog

All notable changes to the **Tessera** protocol are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and Tessera adheres to [Semantic Versioning 2.0.0](https://semver.org/) as
clarified in [`SPEC.md` § Versioning](./SPEC.md#versioning).

Implementations should pin to a specific Tessera version and follow the
upgrade notes when bumping.

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
- Six resource schemas under `schemas/resources/`: `actor`, `project`,
  `task`, `event`, `operation`, `agent-context`.
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

[v0.1.1]: https://github.com/leotorrealba/tessera/compare/v0.1.0...v0.1.1
[v0.1.0]: https://github.com/leotorrealba/tessera/compare/v0.0.2...v0.1.0
[v0.0.2]: https://github.com/leotorrealba/tessera/compare/v0.0.1...v0.0.2
[v0.0.1]: https://github.com/leotorrealba/tessera/compare/v0.0.0...v0.0.1
[v0.0.0]: https://github.com/leotorrealba/tessera/releases/tag/v0.0.0
