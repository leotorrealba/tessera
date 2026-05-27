# Retrospect Evidence - Goal 1 Task List Lease

Date: 2026-05-27
Branch: `codex/task-list-lease-contract`

## Gate Verdict

APPROVED.

Goal #1 can complete and ship as a PR, assuming the untracked v0.1.6 schemas, fixtures, and ODD evidence are included and the pre-existing untracked `graphify-out/` directory remains excluded.

## What Was Built

Tessera v0.1.6 now defines `task.list` and `task.claim` so Kiln can discover queue work without already knowing task IDs, then acquire or renew a server-owned lease safely.

The work adds:

- `task.list` and `task.claim` verb schemas.
- Task claim projection fields: `claim_holder_id` and `claim_expires_at`.
- Replay-safe claim lease event semantics using `assigned` events with `payload.claim_expires_at`.
- Conformance `_auth.actor_id` fixture execution context.
- Conformance `_clock.now` fixture execution clock.
- Deterministic 30-minute conformance lease window.
- Fixtures covering happy path, renewal, replay, stale version, active conflict, mixed queue ordering, and pagination.

## Why The Final Shape Is Better

- Event replay semantics are explicit: claim leases reuse `assigned` events without corrupting `assignee_id`.
- `_auth` makes actor identity executable in fixtures without adding client-controlled actor identity to the protocol payload.
- `_clock` makes lease behavior deterministic and testable instead of dependent on wall-clock time.
- Explicit claim fields make list/claim behavior inspectable and exact-match friendly.
- Replay-before-`if_match` is normative, avoiding false version conflicts after successful retries.

## Missed Edge Cases Found

- Renewal must extend from current active expiry, not from `_clock.now`.
- Version mismatch must be checked before active competing claim for fresh requests.
- Active conflicts must return the server's current task.
- `task.list` needs deterministic mixed ordering of claimable and leased tasks.
- Claim lease `assigned` events must not mutate ordinary assignment.

## Residual Non-Blocking Risks

- Add an expired-lease projection fixture proving expired leases become claimable at `_clock.now`.
- Add fixture coverage that `_auth` and `_clock` are stripped before operation request hashes are computed.
- Add a `task.claim` operation payload mismatch fixture.
- Add a runnable in-repo conformance harness, even if Sprino remains the reference implementation.

## Recommendation

Complete Goal #1 and open the PR. Track the residual risks as follow-up hardening after the base v0.1.6 contract lands.
