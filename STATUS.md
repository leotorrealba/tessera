# Tessera - Project Status

> **Last updated:** 2026-05-27
> **Version:** v0.1.6 protocol line, v0.1.x additive evolution
> **Owner:** @leotorrealba
> **Repo:** [github.com/leotorrealba/tessera](https://github.com/leotorrealba/tessera)
> **Knowledge base:** SPEC.md, TECHNICAL.md, conformance fixtures, Graphify output
>
> This file is the operational snapshot for Tessera. Update it whenever a goal
> changes state, protocol scope changes, Graphify refreshes, or PR status changes.

---

## Health Snapshot

- `origin/main` is at `469389be3afa0143ebf10e979af2931377f9aa89`.
- The current protocol surface includes project, task, actor lifecycle, and
  attachment verbs through the v0.1.6 line.
- Goal #1 implementation is complete on branch `codex/task-list-lease-contract`.
  Local gates passed: JSON sweep, `git diff --check`, LGTM, QA, adversary-review,
  and retrospect.
- PR #13 is mergeable but blocked remotely by GitHub Actions account billing:
  both required jobs report `The job was not started because your account is
  locked due to a billing issue.`
- Kiln can target Tessera-native queue intake against the published `task.list`
  and `task.claim` contract after the PR lands.
- `graphify-out/` exists locally but is untracked; refresh evidence must be
  reviewed before any tracked Graphify report is claimed.

## Roadmap

- Goal #1: publish the task queue scanning and claim lease protocol.
- Next after Goal #1 lands: update Kiln to consume Tessera-native queue intake
  instead of explicit known-task only intake.
- Follow-up hardening: expired-lease projection fixture, `_auth`/`_clock`
  operation-hash stripping fixture, `task.claim` payload-mismatch fixture, and
  runnable in-repo conformance harness.

## Implemented

- Core resources: actor, project, task, event, operation, attachment, and
  AgentContext companion schema.
- Core verbs: project create/list/get, task create/get/update_status/list/claim,
  actor lifecycle verbs, and attachment upload/list/get/finalize.
- Conformance fixture model with paired request/response examples and canonical
  dependency order.
- v0.1.6 queue/claim protocol: deterministic `task.list`, server-owned
  `task.claim`, replay-safe claim lease events, fixture `_auth`, fixture
  `_clock`, and deterministic 30-minute conformance lease window.

## Remaining

- Resolve the GitHub Actions billing lock, re-run PR #13, then merge the
  Tessera v0.1.6 PR after remote checks pass.
- Follow-up Kiln integration after the Tessera PR lands.

## Technical Debt

- SPEC.md, TECHNICAL.md, README.md, CHANGELOG.md, and the conformance README
  all need to stay aligned with the current v0.1.6 surface as Goal #1 lands.
- No runnable in-repo conformance harness is present; protocol validation is
  mostly schema/fixture review plus downstream implementation tests.
- Expired claim lease projection is not fixture-covered yet.
- `_auth` and `_clock` stripping before operation request hashing is documented
  but not fixture-covered yet.
- `task.claim` operation payload mismatch relies on generic idempotency rules;
  there is no task-claim-specific mismatch fixture yet.

## Notes

- Keep Tessera protocol-first. Do not encode Kiln-specific labels, branches, or
  Forgejo/GitHub provider details into protocol schemas.
- Queue scanning must stay transport-agnostic and implementation-neutral.
- Claim leases must be safe for AI workers: optimistic concurrency,
  idempotency, conflict on active competing claim, and deterministic listing.

## Update Discipline

- Update this file before every PR that changes protocol scope.
- Tie health claims to commits, PRs, command output, or fixture names.
- If Graphify is stale relative to the current commit, say so explicitly.
