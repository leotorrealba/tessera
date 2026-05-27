# Adversary Review Evidence - Goal 1 Task List Lease

Date: 2026-05-27
Branch: `codex/task-list-lease-contract`

## First Attack - Socrates

Verdict: REJECTED.

Blocking findings:

1. Claim `assigned` events were replay-ambiguous because `payload.to` could mean `assignee_id` or `claim_holder_id`, and `claim_expires_at` was optional.
2. Competing-actor claim fixtures were prose-only because authenticated actor context was not machine-readable.
3. Existing task response fixtures omitted the new optional claim fields while conformance matching is exact.

Disposition:

- Fixed by Averroes.
- Reviewed and approved by Singer.
- Rechecked by QA.

## Second Attack - Descartes

Verdict: REJECTED.

Blocking finding:

- Claim lease time was not deterministic/executable. The server owned `claim_expires_at`, but fixtures asserted exact timestamps without a fixed clock or lease duration, and active lease fixtures could be expired relative to wall clock/release date.

Disposition:

- Fixed by Planck with `_clock.now`, fixture-time active lease evaluation, and a deterministic 30-minute conformance lease window.
- Reviewed and approved by Hegel.
- Clarified by Avicenna: renewal extends from the current active `claim_expires_at`, not from `_clock.now`.

## Final Attack - Aristotle

Verdict: APPROVED.

Previous blockers closed:

- Claim `assigned` replay ambiguity is resolved by requiring `payload.claim_expires_at` for claim-lease events and projecting those events only to claim fields.
- Competing actor fixtures have executable `_auth.actor_id` context.
- Task response fixtures include explicit `claim_holder_id` and `claim_expires_at`.
- Lease timing is deterministic through `_clock.now` and a fixed 30-minute conformance window.

Residual non-blocking risks:

- Expired lease projection is not fixture-covered.
- `_auth` and `_clock` stripping before operation request hashing is not fixture-covered.
- `task.claim` operation payload mismatch has no task-claim-specific fixture yet.

## Gate Verdict

APPROVED. No PR-blocking adversary findings remain. Residual risks should be tracked as follow-up hardening, not blockers for the v0.1.6 protocol PR.
