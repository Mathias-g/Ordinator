---
status: accepted
date: 2026-08-31
decision-makers: [owner]
scope:
  - compiler
  - control-plane
interface-impact: new
---

# ADR-0007: Columns carry stable random ids in the Board

## Context and problem statement

The schema lives in text (the Board) and the data lives in SQLite, so a schema
edit can change the data's shape, and a naive one can destroy data: renaming a
column must not wipe its values, and a dropped column's values are gone. The
compiler must classify each edit as a rename, a drop, an add, or a type change,
and it must do so from the two definitions it can see. Column names alone
cannot do this: a column that disappears while a same-typed column appears
could be
either a rename (carry the data) or a drop-and-add (destroy and start fresh),
and SQLite has no link between the two names. The question was how rename,
type-change, and drop intent is expressed: explicit directives in the Board
(`rename:`/`migrate:`/`drop:`), compiler inference with refusal of anything
ambiguous, or some third channel.

## Decision drivers

- The definition lives in diffable, reviewable YAML files (SPEC: The artifact
  and the build step). A schema change's meaning should be visible in the PR
  diff, not established at apply time.
- The Board should stay clean to author: no machinery the author does not need,
  and no second mini-language for a job the existing machinery can do.
- The compiler must never guess on a destructive change. Silent misclassification
  of a rename as a drop-and-add destroys data; the reverse silently carries
  stale data into a fresh column.
- Authoring is agent-first. Whatever an authoring app, an agent, or a human
  writes must be the same artifact, with no round-trip to a running daemon to
  obtain identity.
- Schema changes can destroy data, so the default must be safe and destruction
  must be loud, confirmed, and backed by a recovery path.

## Considered options

- **Explicit directives in the Board.** The author marks renames, type
  migrations, and drops where they happen.
- **Inference from names, refusing the ambiguous.** The compiler classifies by
  comparing old and new definitions by label, and fails on the cases it cannot
  classify.
- **Stable ids managed by the compiler in a sidecar state file.** The Board
  stays label-only; the compiler records id-to-label mappings in a committed
  companion file and resolves ambiguity by asking at apply time.
- **Stable random ids in the Board.** Each column carries a randomly generated
  id, minted locally by whatever authored it; the compiler classifies every
  change by matching ids across definitions.

## Decision outcome

Chosen option: "Stable random ids in the Board", because it makes every schema
change classify by construction, keeps the truth in the reviewed diff, and
requires no coordination, no side channel, and no apply-time questions.

Every column carries an `id` field: a large random identifier (ULID or UUIDv7
class, 128 bits). The id is the column's identity; the label is what humans and
formulas use. The compiler classifies changes by matching ids between the old
and new definition:

- same id, new label: a rename. Data is carried; never destructive.
- id removed: a drop. Destructive by intent; shown loudly in the plan and
  confirmed before it happens.
- new id: an add. A fresh empty column.
- same id, changed type: a type change (see below).

Randomness and size make the ids coordination-free: any author (an app, an
agent, a person's tooling) mints a new id locally at authoring time, with no
round-trip to Ordinator. Two ids colliding is negligible by construction; the
compiler still treats a duplicate id within a document as a validation error
and refuses to process, so the case costs a check, not a mechanism.

### Type changes

A type change carries data the compiler cannot know the intent of, so intent
is never inferred:

- **Provably safe widening conversions** (for example `int` to `numeric`) are
  applied automatically from a small fixed table of safe conversions. The plan
  shows the conversion.
- **Lossy or unknown conversions are refused** with a structured error. There
  is no `migrate:` directive. The author performs the migration as two
  operations that are each already supported: add a new column whose formula
  converts the old values (a formula, so it is typed, validated, diffable, and
  reviewed like any other), then remove the old column. Both steps go through
  the normal plan/confirm cycle and are visible in the diff.

This decomposition keeps the conversion inside the Board's existing formula
language rather than creating a migration sub-language, and the compiler never
picks a lossy conversion silently.

### Why the other options were rejected

- **Explicit directives** were rejected because they are redundant with the
  diff the compiler already computes: with ids, a rename and a drop are exact
  facts, not choices, so the directive is permanent YAML noise for machinery
  the compiler does not need. Directives also create a failure mode where the
  directive and the actual edit disagree.
- **Name-only inference with refusal** was rejected because the ambiguous case
  (removed column plus same-typed new column) is then unanswerable without a
  channel outside the diff. It forces either an apply-time prompt or flag
  (so a rename is no longer a pure PR affair and CI cannot apply it alone) or
  guessing with confirmation, which violates the refuses-to-guess principle.
- **Compiler-managed ids in a sidecar state file** keep the Board label-only
  but cannot detect a rename at all: the state file maps id to label, and once
  the label changes the mapping no longer matches the new Board, so the
  rename-versus-drop-and-add ambiguity returns and must be resolved by an
  apply-time question or a flag. The Board stops being the single source of
  truth for identity, and a forgotten or hand-edited state file creates
  implausible diffs.

### Consequences

- Good: every schema change is classified by construction, with no ambiguity
  and no guessing. The "refuses to guess" behavior remains exactly where it
  belongs: a lossy type change, where intent is genuinely unknowable.
- Good: a rename is a pure PR affair, visible and reviewable in the diff
  (`id unchanged, label changed`), applicable in CI with no human present.
- Good: the Board is the single source of truth for identity. Reviewers, CI,
  and third-party tools see the same truth with no side channel or state file.
- Good: authoring needs no coordination. New ids are minted locally by the
  authoring app, agent tooling, or `ordinator compile --fix` for hand authors;
  nothing consults the daemon to obtain one.
- Good: copying a column block including its id is a duplicate-id validation
  error, not a silent data hijack; deliberately reusing an id is a rename,
  which is what the author said.
- Bad: the Board carries one machine-written field per column that is
  meaningful to no human. Mitigated by ids being minted by tooling, never
  hand-written, and inert (never read by a person).
- Bad: there is no one-step lossy type change; it costs two routine steps.
  Accepted because lossy type changes are rare and the two steps are each
  already supported, loud, and reviewable.
- Neutral: a raw read of the SQLite file sees opaque ids rather than labels.
  This is accepted; a bare read is a backup and debugging case, and the label
  can appear as a readability suffix in the physical name while the id remains
  the identity.
- Neutral: the physical column naming scheme in SQLite (how the id maps to a
  physical column name) is an implementation detail, not settled here.

### Confirmation

Behavior tests pin: a rename with an unchanged id carries data and never
appears destructive; a removed id plans as a destructive drop and is refused
without confirmation; a duplicate id fails validation; a safe widening applies
automatically and a lossy type change is refused with the two-step guidance.
These become tests when the project has code.

## Interface notes

This is a `new` interface impact: the Board file format gains a stable `id`
field on every column, distinct from the human-readable label, which remains
the name used by formulas, access rules, and the API. Authors never invent ids
by hand: tooling mints them, and the compiler validates uniqueness and refuses
a missing or duplicate id. A `compile --fix`-style command inserts missing ids
for hand authors. The exact physical representation in SQLite is not part of
this contract.

## More information

- ADR-0002 (BSSN): the reasoning that rejected speculative directive machinery.
- ADR-0005, ADR-0006: the formula engine that the two-step type-change recipe
  reuses for conversions.
- SPEC: Schema changes and the apply cycle.
