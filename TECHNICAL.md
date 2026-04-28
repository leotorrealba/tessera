# Tessera — Technical Reference

This is the deep-dive companion to the [README](./README.md) and
[SPEC](./SPEC.md). It's aimed at people implementing Tessera in a new
codebase, or evaluating whether it fits their PM tool.

For the plain-English version, see [`EXPLAINED.md`](./EXPLAINED.md).

---

## 1. What Tessera is

Tessera is an open protocol. It defines the **shape of data and the
semantics of operations** that any AI-native project management tool
needs. It does not define a transport, a storage backend, an auth model,
or a UI.

Concretely, Tessera v0.1.x is:

- **Six JSON Schemas** for the core resources: `actor`, `project`,
  `task`, `event`, `operation`, and `agent_context`.
- **Five verbs** with paired request/response JSON Schemas:
  `project.list`, `project.get`, `task.create`, `task.get`,
  `task.update_status`.
- **A conformance fixture suite** — paired `*.req.json` / `*.res.json`
  files that any conforming implementation MUST replay correctly.
- **A set of semantic rules** (event-log append-only, idempotency via
  UUIDv7, optimistic concurrency via `if_match`, agent_context
  truncation, structured error envelope) documented in [`SPEC.md`](./SPEC.md).
- **A versioning + deprecation policy** so implementations can pin a
  version and trust that additive changes won't break them.

That's the whole protocol. Everything else — HTTP routes, database
schema, MCP tool registrations, frontend — is implementation territory.

---

## 2. Why a protocol (and not just a tool)

Three reasons, in priority order.

### 2a. Protocols outlive implementations

Reference implementations come and go. Linear absorbed several startups.
Jira's 2008 schema is now its own technical debt. If "AI-native PM" is
worth doing at all, it's worth being a category that lives across many
tools — not a feature in one closed product.

LSP did this for editors and language servers. OpenTelemetry did it for
observability. The protocol-first move is what made those categories
durable. Tessera applies the same playbook to project state.

### 2b. Agents need a stable target

An AI agent integrated with `task.create` should not have to re-learn
the world every time it switches tools. If an agent knows how to call
`task.create` against a Tessera-conformant server, it knows how to do
that against every Tessera server forever, including ones that don't
exist yet.

This is the case for shipping schemas, conformance fixtures, and
deprecation rules as a public contract — not as Sprino-internal source
code.

### 2c. Forks tend to drift

If the only "spec" is the reference implementation's source code, every
fork becomes a competing dialect. The conformance suite is what prevents
that: a fork that fails the suite isn't Tessera, regardless of how much
of Sprino's code it shares.

---

## 3. What Tessera is NOT

Explicit non-scope:

- **Not a transport.** HTTP, MCP, gRPC, WebSocket — implementations choose.
- **Not a storage backend.** Postgres, SQLite, in-memory — implementations choose.
- **Not an authentication / authorization model.** Tessera assumes a
  trust boundary above the protocol layer. Bearer tokens, OAuth, mTLS —
  implementations choose.
- **Not a UI specification.** Frontends are free.
- **Not a performance contract.** Tessera defines correctness, not
  latency or throughput.
- **Not MCP.** MCP is *how* an AI client talks to a tool. Tessera is
  *what* a project, task, or event looks like once you're talking. They
  compose; they don't compete.

---

## 4. Why these resources, and not others

Six resources. Why these six?

| Resource | Why it's universal |
| --- | --- |
| **Actor** | Every PM tool has authors of work. Tessera makes `kind: agent` a schema-level distinction so agents are first-class — not users-with-a-flag. This is the single most opinionated decision in the spec, and the one that everything else flows from. |
| **Project** | Every PM tool has scope boundaries. "Workspace," "team," "board" — same idea. Naming it `project` is conventional in dev tooling. |
| **Task** | The atomic unit of work. Variants (story, ticket, issue, item) are renaming, not redesign. |
| **Event** | The append-only log. Every PM tool needs an audit trail; making it the source of truth (rather than a sidecar) means agents can replay history rather than guessing. |
| **Operation** | The idempotency dedup record. Required for safe retries from clients (especially agents) that can't always tell whether their last call succeeded. |
| **Agent Context** | Structured response payload returned on `task.get` so agents start with full context (related tasks, recent events, repo refs) instead of re-explaining. Modeled as its own schema because the truncation contract is non-trivial. |

What's NOT in the v0.1 set:

- **Comments** — universal but adds threading semantics. Deferred to v0.2.
- **Labels / tags** — useful but easy to add additively. Deferred.
- **Sprints / iterations** — under question whether this is universal or
  implementation-specific. Re-evaluated in v0.2 design.
- **Webhooks / search / attachments** — implementation territory until
  there's evidence the shape is universal.

The principle: **a resource lives in Tessera once a second implementer
would also need to make the same modeling decision.** Until then, it
stays in implementation land.

---

## 5. The five rules that matter most

These are the load-bearing semantic rules. Anything Tessera-conformant
MUST honor them. Implementation details are open; these contracts are not.

### 5a. Event log is authoritative; materialized state is a projection

- Mutating verbs MUST write the event row BEFORE updating any
  materialized state, in the same transaction.
- Replaying the event log in `created_at` order MUST reproduce the
  current materialized state.
- Implementations MAY keep mutable rows for read performance (and
  almost certainly should), but the events are the truth. If a row
  drifts from the events, the events win.

This is what lets implementations evolve their projection schema
(adding indexes, denormalized columns, new derived fields) without
losing history.

### 5b. Idempotency: client-supplied UUIDv7 + 30-day retention

- The client supplies `operation_id` (UUIDv7). The server MUST NOT
  generate it.
- Operation rows are retained for at least 30 days. After expiry,
  retries return `410 Gone`.
- Replay semantics:
  - Same `operation_id` + same `request_hash` (SHA-256 of canonical
    JSON) → return the cached response verbatim. No new event.
  - Same `operation_id` + different `request_hash` → return
    `409 Conflict` with the original cached response.

The reason `operation_id` is client-supplied: agents that retry on
network errors need to know in advance that two requests are "the same
attempt." Server-generated IDs would make every retry a new operation
and defeat the point.

### 5c. Optimistic concurrency via `if_match: <version>`

- Every mutating verb takes `if_match` set to the version the client
  last observed.
- On match: write the event, increment version, return new state.
- On mismatch: return `409 Conflict` with the **server's current task
  body** so the client can re-read and retry without an extra round-trip.

This exists because last-write-wins is a footgun when AI agents are
authors. Agents tend not to pause to re-check state; explicit version
tokens force them to.

### 5d. `agent_context` truncation contract

- `task.create` and `task.get` responses include a structured
  `agent_context` field.
- The field is capped at 32 KB.
- When over-cap, implementations MUST set `truncated: true` and
  populate `next_page_tokens` for any sub-collections that were cut.
- The exact content of `agent_context` is implementation-defined
  (Sprino includes recent events, related tasks, project README
  excerpts), but the shape and the cap are protocol-defined.

This protects the calling agent from response bloat without forcing it
to issue a follow-up call for every task fetch.

### 5e. Structured error envelope

When a verb fails, implementations return a structured error object
with a stable, lowercase, snake_case `code` (e.g.
`validation_error`, `operation_id_conflict`,
`version_mismatch`). Codes are part of the protocol surface and
covered by the deprecation policy.

This is what lets agents handle errors without parsing prose.

---

## 6. Conformance

Conformance is the contract between Tessera and an implementation. It
lives in [`conformance/fixtures/`](./conformance/fixtures/) as paired
`*.req.json` / `*.res.json` files.

### What "Tessera-conformant" means

An implementation is **Tessera v0.1.x conformant** when:

1. Every fixture in `conformance/fixtures/` passes. The implementation
   accepts the request and returns a response that structurally matches
   `*.res.json`, modulo only the documented conformance normalizations:
   - **Timestamps** (`created_at`, `updated_at`, `event.created_at`)
     may differ — assert ISO-8601 and correct ordering.
   - **`event.id`** may differ — assert valid UUID and uniqueness within
     the response.
   - The volume of `agent_context.related_tasks`, `recent_events`, and
     `repo_refs` is implementation-dependent (only the truncation
     contract is asserted).

   All other response fields — including `task.id` — must match the
   fixture exactly, even when server-generated. See
   [`conformance/README.md`](./conformance/README.md) for the full rules.
2. All five v0.1 verbs are implemented per [`schemas/verbs/`](./schemas/verbs/).
3. The semantic rules in section 5 are enforced (operation replay,
   `409 Conflict` on `operation_id` payload mismatch, `410 Gone` on
   expired operations, optimistic concurrency via `if_match`/`version`,
   `agent_context` truncation, structured error envelope).

### What conformance does NOT cover

- Transport. Implementations choose.
- Storage. Implementations choose.
- Latency / throughput. Tessera defines correctness only.
- Authentication / authorization. Above the protocol layer.

### Reference harness

[Sprino](https://github.com/leotorrealba/sprino) runs the entire fixture
suite in CI as `apps/server/test/conformance.test.ts`. That harness is
the canonical way to verify a new implementation: clone Tessera as a
sibling directory (or git submodule), point the test runner at it, and
watch every fixture replay against your server.

If you implement Tessera in another language, the simplest path is to
write a small script that walks `conformance/fixtures/`, replays each
`*.req.json` against your server, and structurally diffs the response
against `*.res.json` (with the modulo rules above). Two fixture
conventions to know about — both documented in
[`conformance/README.md`](./conformance/README.md):

- **Error fixtures** wrap the expected error under a top-level `_error`
  envelope (`status`, `code`, optional `details`); the harness compares
  `_error.status` and `_error.code` exactly.
- **`_meta`** may appear at the top level of either `.req.json` or
  `.res.json` files. It documents preconditions in human terms and the
  harness MUST ignore it when validating shape.

Open a PR adding a link to your harness in the README — we want to know
who's out there.

---

## 7. Versioning + deprecation (the honest version)

### What changes how

| Change | Bump |
| --- | --- |
| Add a new optional field on a request or response | MINOR |
| Add a new verb | MINOR |
| Add a new conformance fixture for existing behavior | PATCH |
| Clarify or restate existing rules without changing them | PATCH |
| Rename or remove a field | MAJOR |
| Tighten a type (optional → required) | MAJOR |
| Change a verb's idempotency or concurrency semantics | MAJOR |
| Remove a verb (after deprecation cycle) | MAJOR |

### `v0.x` vs `v1.x`

- **`v0.x`** — schemas may evolve. Breaking changes ship as MINOR bumps
  during this phase (`v0.0.2 → v0.1.0` was breaking-allowed). The
  `v0.1.0` milestone freezes the v0.1 surface so implementers can build
  against a stable target. Within `v0.1.x`, only additive changes ship.
- **`v1.x`** — full semver. Breaking changes require `v2.0.0` and a
  migration guide.

`v1.0.0` ships when a second implementation exists and stays
conformant across one full release cycle. Until then, it would be
premature.

### Pinning

Implementations MUST declare which Tessera version they target — in
their manifest, README, response metadata, or all three. Tooling SHOULD
reject mismatches at the major or minor level rather than silently
downgrading.

### Deprecation lifecycle

1. **Announce.** The next minor release marks the item deprecated:
   `"deprecated": true` in JSON Schema, plus a `### Deprecated` block
   in [`CHANGELOG.md`](./CHANGELOG.md) and a "Deprecated in vX.Y.Z" note
   in [`SPEC.md`](./SPEC.md).
2. **Notice period.** Minimum 90 days AND at least one minor release
   between the deprecation announcement and the removal.
3. **Runtime hint (optional).** Implementations MAY include a
   `Tessera-Deprecation: <field-or-verb>; sunset=<RFC3339-date>` HTTP
   response header so SDKs can warn at runtime.
4. **Removal.** The next major version removes the item. The removal
   MUST be listed in the migration guide (template in
   [`SPEC.md`](./SPEC.md#migration-guide-template)).

---

## 8. Roadmap

| Version | Status | Notes |
| --- | --- | --- |
| `v0.0.0` | ✅ shipped | Bootstrap repo, license, founding-story README. |
| `v0.0.1` | ✅ shipped | Three verbs, JSON Schemas, three happy-path fixtures. |
| `v0.0.2` | ✅ shipped | Project read verbs + repo_path resolution. |
| `v0.1.0` | ✅ shipped | Schema stabilization. Versioning + deprecation policy. Expanded fixtures. |
| `v0.1.1` | ✅ shipped | Fixture error-code alignment with reference impl. |
| `v0.1.x` | open | Additive: `events.list`, `actor.register`, `actor.list`, `project.create`, `project.update`. Land once exercised by a third-party implementation. |
| `v0.2.0` | planned | Comments, labels, search, webhooks, sprints — re-evaluated as universal vs implementation-specific. |
| `v1.0.0` | planned | When a second implementation exists. |

---

## 9. License

MIT. Implementations are free to use, modify, distribute, and sell.
The protocol is a commons; implementations compete on quality. The
reference implementation ([Sprino](https://github.com/leotorrealba/sprino))
is AGPL because we want derivative servers to keep their improvements
open — but you are not required to license your own implementation
under AGPL or anything else.
