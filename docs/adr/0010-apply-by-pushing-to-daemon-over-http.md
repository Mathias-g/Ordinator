---
status: accepted
date: 2026-09-01
decision-makers: [owner]
scope:
  - daemon
  - control-plane
interface-impact: new
---

# ADR-0010: Apply a definition change by pushing it to the daemon over HTTP

## Context and problem statement

The definition (the Board and the access rules) lives in text files that are
committed and reviewed in git (SPEC: The artifact and the build step). Applying
a change to the schema migrates the live SQLite file, which only the daemon may
write because it owns the document folder and its single write connection
(SPEC: The compiler). The daemon is a server that serves data continuously.
For a change to take effect, someone must hand the new definition to the
daemon and ask it to apply, and that someone must be able to be CI: a merge to
main should be able to move a reviewed Board into production with no human
present (ADR-0007 makes renames applicable in CI). The question is how the
daemon is triggered to apply a definition change, and specifically how a
file-based, CI-driven process hands the new definition to a running daemon.

## Decision drivers

- The definition is authored and reviewed outside the daemon, in git
  (SPEC: The artifact and the build step). The daemon is not a git client and
  does not fetch the definition on its own.
- The daemon is authoritative over validation and apply: it owns the single
  write connection and the live data (SPEC: The compiler). A change must go
  through its plan/apply machinery, never be written behind its back.
- CI, an agent, and a human must drive apply the same way, with no interactive
  session required (ADR-0007, SPEC: Schema changes and the apply cycle).
- Ordinator is independent of Servitor (SPEC: Independence from Servitor). The
  mechanism must be a published standard useful to anyone, not tied to a
  specific client.
- The daemon already serves data continuously and is intended to expose an
  HTTP API for reads and writes (ADR-0005: "Consumers (the CLI, the HTTP API,
  Cerebror) read").

## Considered options

- **Push over the daemon's HTTP API.** A client sends the new definition
  (the Board and/or the access rules) to the daemon over its HTTP API, and the
  daemon validates, plans, and applies it against the live SQLite file. CI is
  one such client; it does not touch the server's filesystem.
- **The daemon pulls from git.** The daemon watches (or fetches on a hook) a git
  repository, and applies the definition when the file changes. CI only merges
  the change to main.
- **SSH / agent-based push.** CI copies the file to the host and runs a command
  (Ansible, `scp` plus a reload), which the daemon then picks up.

## Decision outcome

Chosen option: "Push over the daemon's HTTP API". A client hands the new
definition to the daemon over its HTTP API, and the daemon validates, plans,
and applies it. The daemon is authoritative over whether and how the change is
applied; the client (CI, an agent, a human, or any tool) decides when to send
it. This keeps the text authored and reviewed outside the daemon, keeps the
daemon the sole writer of the live data, and gives CI a plain HTTP call to make
after a merge, no interactive session needed.

### Why the other options were rejected

- **The daemon pulls from git** couples the daemon to a specific repository and
  to git itself, which blurs the boundary that the definition is authored
  outside the daemon and the data lives inside it. It also makes the daemon
  notice changes on its own, which the design has deliberately not required,
  and it re-introduces the file-watching/polling machinery the push model
  avoids. It works, but it makes the daemon depend on an external source of
  truth it does not otherwise have.
- **SSH / agent-based push** requires the CI step to have credentials to the
  host and to write files onto it, and it makes the server a passive recipient
  rather than the authority on validation and apply. It also needs the daemon
  to notice the written file, which reintroduces the same watching problem.

### Consequences

- Good: CI, an agent, a human, and any workflow tool drive apply identically,
  as an authenticated HTTP call to the daemon. No interactive session, no
  host filesystem access, no git dependency in the daemon.
- Good: the daemon remains the sole writer of the live SQLite file and the
  authority on validation, planning, and applying (SPEC: The compiler).
- Good: the mechanism is a published HTTP standard, so it stays within the
  independence constraint: Servitor, or anyone else, is merely one HTTP client.
- Bad: the daemon must expose an apply endpoint and authenticate it. This is
  new surface on the daemon control protocol (see Interface notes).
- Neutral: the client is responsible for deciding *when* to push; the daemon
  does not watch for changes on its own. Whether the daemon proactively
  reports that a Board is ahead of what is applied (drift) is a separate,
  still-open question.

### Confirmation

A behavior test, once the project has code, pins: pushing a definition over the
daemon's HTTP API applies it to the live SQLite file, a push carrying a
destructive change is refused without confirmation, and a Board committed but
not applied is reported as drift at plan time. The exact request and response
shape is not fixed here, only that apply happens by pushing the definition to
the daemon over HTTP.

## Interface notes

This is a `new` interface impact. It establishes that the daemon's HTTP API
accepts a definition change (a Board and/or access rules) from a client and
applies it, rather than the daemon fetching or watching for it. The exact
endpoint, request/response schema, and authentication are not settled here and
are left to the control-protocol design; what is settled is the push model:
the client sends the definition, the daemon is authoritative over applying it.

## More information

- ADR-0007: renames are applicable in CI with no human present, which this
  decision makes possible.
- ADR-0008: the last-applied snapshot the daemon diffs against when applying.
- ADR-0005: the daemon hosts the object model and serves consumers including
  the HTTP API.
- SPEC: The compiler, SPEC: Schema changes and the apply cycle.
