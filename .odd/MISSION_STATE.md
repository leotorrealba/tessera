# Mission State - Tessera Queue Contract

## Shared Constraints

- Work through PRs; never push directly to `main`.
- Keep Tessera protocol-first and transport-agnostic.
- Preserve existing v0.1.x compatibility by using additive schema changes.
- Do not commit generated `graphify-out/` artifacts unless explicitly promoted
  to tracked evidence.
- Do not revert existing local branches or unrelated files.

## Anti-Patterns Caught

- Do not add Kiln-specific SCM labels, branch names, or Forgejo/GitHub behavior
  to Tessera protocol contracts.
- Do not make queue ownership depend on wall-clock assumptions clients cannot
  verify; server-owned lease timestamps must be explicit in responses.

## Architecture Decisions

- Use `task.list` for deterministic queue scanning.
- Use a dedicated claim/lease verb rather than overloading `task.update_status`.
- Reuse existing event kinds where possible; new event kinds require explicit
  protocol-version justification.

## Broadcast

- Goal #1 exists to unblock Kiln's native Tessera intake. Kiln integration is a
  follow-up after this PR lands.
