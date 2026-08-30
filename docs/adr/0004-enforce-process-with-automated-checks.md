---
status: accepted
date: 2026-08-30
decision-makers: [Mathias]
consulted: []
informed: []
scope:
  - compiler
interface-impact: none
---

# ADR-0004: Enforce the process with automated checks

## Context and problem statement

ADR-0003 records how context lives in the codebase, and `AGENTS.md` describes
the process for keeping it accurate. But a written process that relies on
people, and on AI agents, remembering to follow it will drift. We need the
context system to be enforced, not just described.

## Decision drivers

- The process must hold even when the contributor does not know or recall it.
- Feedback should arrive fast, ideally before a bad change is even committed.
- There must be a gate that cannot be skipped, to actually protect `main`.
- Best Simple System for Now: the least machinery that makes the correct path
  the easy one and blocks the wrong one. No heavyweight policy engine.

## Considered options

- Rely on code review alone
- Rely on `AGENTS.md` and the agent's good behavior alone
- Pre-commit hooks only (local)
- CI checks only (server)
- Both: pre-commit hooks plus CI gated by branch protection

## Decision outcome

Chosen option: both layers. Pre-commit hooks give fast, local feedback (a
warning or a blocked commit at the keyboard). CI re-runs the same checks on
GitHub, and branch protection makes passing them a requirement to merge. The
first check is a small ADR lint under `scripts/checks/` that validates
filenames, front matter, status values, and sequential numbering, plus a
bookkeeping reminder that prints a pointer to the pre-commit checklist in
`AGENTS.md` at the end of every pre-commit run. Code quality checks (tests,
vet, lint) join the same two-layer gate once the project has code; see the
Status section of `AGENTS.md`.

Both layers over the alternatives because review and good intentions do not
scale and are exactly what fails under time pressure; pre-commit alone can be
bypassed (`--no-verify`) and lives only on each machine; CI alone gives slow
feedback and lets a broken change get committed and pushed before catching it.
Together, the local layer keeps most problems from ever being committed, and
the server layer is the unskippable gate.

### Consequences

- Good: the context system is enforced regardless of who or what makes a change;
  the hard gate (branch protection) cannot be skipped.
- Good: fast local feedback reduces failed CI runs.
- Bad: a small amount of setup per contributor (`pre-commit install`) and one
  repository setting (branch protection) that an admin must enable.
- Neutral: the ADR lint is a small script; it can be extended as the repo's
  conventions evolve.

### Confirmation

A `checks` workflow runs in CI on every pull request and is required by branch
protection on `main`. `scripts/checks/` holds the ADR lint and the reminder.
`CONTRIBUTING.md` documents the per-contributor and admin setup.

## More information

- ADR-0003 (the context system this enforces)
- pre-commit: https://pre-commit.com/
- Best Simple System for Now:
  https://dannorth.net/blog/best-simple-system-for-now/
