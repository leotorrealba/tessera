# Tessera, Explained Like You're New Here

This is the no-jargon version. If you want the technical deep-dive,
read [`TECHNICAL.md`](./TECHNICAL.md) and [`SPEC.md`](./SPEC.md).

## What is Tessera?

Tessera is a **specification**. It's a written document — plus a set of
schema files — that says:

> "If you're building a project management tool that wants to work
> well with AI agents, here's the shape your data should have, and
> here are the rules your operations should follow."

That's the whole thing. Tessera doesn't run anywhere. It's not a server.
You don't install it. It's a contract that tools can choose to follow.

The reference implementation that *does* run is called
[Sprino](https://github.com/leotorrealba/sprino). Sprino is a
self-hosted PM tool that follows Tessera. But Tessera could one day be
followed by Linear, Jira, or a tool that doesn't exist yet — Sprino is
just the first one.

## Why does it exist?

Today, AI agents (Claude, Cursor, Copilot, ChatGPT, etc.) do a lot of
software work. But every PM tool that wants AI integration ships its
own integration in its own shape:

- Linear has a Linear-shaped MCP server.
- Jira has a Jira-shaped one.
- Your custom tool has a custom-shaped one.

When an agent wants to "create a task," it has to learn each tool's
particular flavor. There's no shared concept of what a task is, or
what a project is, or how you safely retry an operation when the
network blips.

Tessera says: let's agree on the shape. Then any agent that knows how
to talk to one Tessera-conformant tool knows how to talk to all of
them, including ones that get built next year.

## Has anyone done this before?

Yes — that's how we know it works.

- **LSP (Language Server Protocol)** did this for editors and language
  servers. Before LSP, every editor needed a custom integration for
  every language. After LSP, any LSP-compliant server works in any
  LSP-compliant editor. The protocol unlocked the category.
- **OpenTelemetry** did this for observability. Before OpenTelemetry,
  every monitoring vendor had its own SDK. After OpenTelemetry, you
  emit standard signals and any backend can ingest them.

Tessera is trying to be the LSP of AI-native project management.

## What's in it?

Five things. In plain English:

1. **Actors.** Humans and AI agents are both authors of work. Not
   "users with a robot flag" — actually distinct kinds of participants
   at the schema level. This is the most important opinion in the spec.

2. **Projects.** A scope boundary. Tasks belong to projects. Same
   meaning as "workspace," "team," or "board" in other tools.

3. **Tasks.** The atomic unit of work. Has a status, an assignee, a
   version number (so you can safely update it without overwriting
   someone else's change).

4. **Events.** Every change is recorded as a permanent fact in an
   append-only log. Nothing gets silently overwritten. If you want to
   know "what happened to this task last week," you read the log
   instead of guessing.

5. **Operations.** Every mutating action gets an ID. If you retry the
   same action with the same ID, you get the same result back — not a
   duplicate. This matters because agents retry constantly on flaky
   networks.

There's also a small thing called `agent_context` — a structured chunk
of relevant state that the server gives back when an agent fetches a
task, so the agent doesn't have to re-explain itself every time. That
field is capped at 32 KB so responses don't bloat.

## How do members get added? (v0.1.2)

In v0.1.2 Tessera grew a small set of "lifecycle" verbs for actors:
`actor.register` (mint a new human teammate), `actor.list`,
`actor.get`, and `actor.revoke_token`. Before this, every
implementation had to invent its own way to add a person — usually by
hand-editing a config file and restarting the server. Now there's one
shared shape so an admin tool, a bootstrap script, or a future
governance UI can speak the same language to any Tessera tool.

Two opinions worth knowing:

- **The plaintext token is shown exactly once.** When you register a
  new actor, the response includes the bearer token in cleartext. If
  you call again with the same `operation_id` (an idempotent retry),
  the response no longer contains the token — only the actor record.
  This is enforced by the protocol, not left up to the implementer:
  losing the token means rotating to a new one, not recovering the
  old one. That's a deliberate "secrets shown once" stance.
- **Agent registration is deferred.** v0.1.2 only mints `kind=human`
  actors. AI agents are still declared at deploy time (in env files,
  config, or boot scripts). Agent self-registration needs threat-model
  work — what permissions does a freshly-minted agent get? Who
  authorized it? — that we haven't done yet, and shipping the verb
  half-finished would set a bad precedent.

## Tessera vs MCP — what's the difference?

This question comes up a lot.

- **MCP** (Model Context Protocol) is *how* an AI client talks to a
  tool. The pipes. The wire format. Anthropic published it; lots of
  tools speak it.

- **Tessera** is *what* the data looks like once you're talking. The
  shape. The semantics. The contract.

They compose. A Tessera-conformant tool can expose itself over MCP
(Sprino does), or over plain HTTP, or over gRPC, or over WebSockets.
Tessera doesn't care about transport.

You can think of it like this: MCP is HTTP. Tessera is REST conventions
plus a domain model. They live at different layers.

## Tessera vs Sprino — what's the difference?

This is the second-most-asked question.

- **Tessera** (this repo) is the spec. MIT-licensed. Anyone can
  implement it. Free to use, modify, distribute, sell.
- **Sprino** is one specific implementation that follows Tessera.
  AGPL-licensed. You can self-host it.

Why split them? Because protocols belong to everyone, and
implementations belong to whoever maintains them. If we kept Tessera
inside Sprino's source code, it would be hard for a second tool to
adopt it without forking Sprino. By making it a separate repo with its
own license, we make it possible for completely unrelated tools to
adopt the protocol and stay independent.

## What does "the protocol's job is to outlive any single implementation" mean?

It means: if Sprino disappeared tomorrow — got abandoned, got bought,
got rewritten in a way that broke everything — Tessera should survive.
Other tools should still be able to follow it. Agents that learned
Tessera should still be able to use that knowledge.

This is the LSP test. Visual Studio Code didn't have to die for LSP to
keep working — it kept working because it was a separate spec.

## Who is Tessera for?

Three audiences:

1. **People building AI-native PM tools.** If you're shipping a tool
   today and want AI integration that doesn't require every customer
   to write custom MCP glue, implement Tessera. Pass the conformance
   suite. Now agents that already speak Tessera can use your tool.

2. **People building agents.** If you're shipping a coding agent or
   a planning agent or a research agent that needs to track work,
   target Tessera once and you've covered every Tessera-conformant
   tool — including ones that ship after your agent does.

3. **Curious developers.** If you want to know what an AI-native PM
   data model looks like, the spec is short (one document) and the
   reference implementation is small (a few thousand lines of
   TypeScript). It's a quick read.

## What does the name mean?

A *tessera* is a small mosaic tile. In ancient Rome, soldiers carried
wooden tessera tiles with their unit assignment carved into them. The
metaphor is literal: every task is a tessera, every project is the
mosaic that emerges when many tiles get placed together by people and
agents working in coordination.

## How do I use it?

You don't "use" Tessera directly. You either:

- **Try the reference implementation** —
  [Sprino](https://github.com/leotorrealba/sprino). 30-minute self-host
  walkthrough; runs on Docker.
- **Implement Tessera in your own tool** — read [`SPEC.md`](./SPEC.md),
  pass the [conformance suite](./conformance/), open a PR adding your
  implementation to the README. We'd love to know.
- **Build an agent against it** — point your MCP client at any
  Tessera-conformant server and you get task creation, fetch, and
  status updates with real idempotency and concurrency safety.

## What's the catch?

Honest list:

- **It's young.** The schema is stable within v0.1.x but we haven't
  yet seen a third-party implementation, which is the test that
  proves the spec is buildable.
- **The verb set is small on purpose.** Nine verbs as of v0.1.2 — five
  for tasks/projects, four for the actor lifecycle. No comments, no
  labels, no search, no sprints, no webhooks. Those land in v0.2 once
  we know whether they're truly universal or implementation-specific.
- **No transport is mandated.** Some implementers want a stricter
  contract that includes "must speak HTTP this way." Tessera does not
  do that. Transport is your call. This is a feature for tool
  builders and a confusion for newcomers.

## Where to go from here

- **Read the spec:** [`SPEC.md`](./SPEC.md).
- **Read the engineering deep-dive:** [`TECHNICAL.md`](./TECHNICAL.md).
- **Run the reference implementation:**
  [Sprino → Self-host](https://github.com/leotorrealba/sprino#self-host-30-minutes-end-to-end).
- **Implement Tessera yourself:** start with the [conformance suite](./conformance/).
- **Track changes:** [`CHANGELOG.md`](./CHANGELOG.md).
