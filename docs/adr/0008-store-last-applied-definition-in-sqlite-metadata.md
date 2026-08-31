---
status: accepted
date: 2026-08-31
decision-makers: [owner]
scope:
  - compiler
  - daemon
interface-impact: none
---

# ADR-0008: Store the last applied definition in SQLite metadata

## Context and problem statement

The compiler classifies every schema change by diffing the Board (the new
definition) against the last applied definition (ADR-0007). That comparison
needs a source for the "old" definition that is guaranteed to describe the
data as it actually is. The question was where that snapshot lives: the daemon
must be able to diff after a restart, from a fresh checkout, and against a
file it did not apply itself.

## Decision drivers

- The diff must be against what was *applied*, not what was *intended*: if the
  Board and the data have drifted (an apply was forgotten, interrupted, or
  never run), the plan must show the true distance, not a guess.
- The compiler runs in the daemon, which can restart at any time; the snapshot
  must survive restarts with no re-derivation step and no extra state to lose.
- SQLite holds no editable and no authoritative definition (SPEC: The artifact
  and the build step). The snapshot is within the constraint's terms: it is a
  derived record of what was applied, written by the compiler in the same
  transaction as the change, never edited directly, and never a second place
  where intent is expressed.
- A raw read of the file is a backup and debugging case; the more the file
  explains itself, the better that case works (the ids in SQLite are opaque
  without the id-to-label mapping, ADR-0007).

## Considered options

- **A snapshot table in the SQLite file's metadata**, written in the same
  transaction as the schema change.
- **Daemon memory only.** The daemon keeps the last applied definition in
  process.
- **A second committed artifact on disk** (a compiled copy or state file next
  to the Board).
- **No snapshot.** The compiler diffs the new Board against the Board as last
  committed, trusting git history.

## Decision outcome

Chosen option: "A snapshot table in the SQLite file's metadata", because it is
the only home that is transactionally bound to the data it describes.

The last applied definition is stored in a metadata table inside `data.sqlite`
and written in the same transaction as the schema change it describes. The
compiler reads it to compute the diff; if it is absent (a brand-new document),
every column is an add and the initial apply writes the first snapshot.

This gives drift detection for free: when the committed Board and the stored
snapshot disagree at startup or at plan time, the daemon knows an apply did not
happen, and `plan` shows exactly what would bring the data in line with the
Board. It also makes the file self-describing: a raw read shows the opaque id
columns and the snapshot that maps them to labels, which is what a backup or
debugging read needs.

### Why the other options were rejected

- **Daemon memory** is lost on every restart. The first diff after a restart
  would either fail or silently fall back to a weaker source, and the fallback
  path would be the bug factory.
- **A second committed artifact** (compiled copy or state file next to the
  Board) re-introduces the failure mode ADR-0007 rejected for column ids: a
  generated file that must be committed in step with the hand-authored one,
  where forgetting to commit it produces implausible diffs and editing it by
  hand corrupts identity. Two artifacts that must agree is one artifact too
  many.
- **No snapshot** makes the compiler trust git history, but git describes what
  was *authored*, never what was *applied*. A definition can be committed
  without being applied, applied from an uncommitted state, or applied and then
  reverted in git. The diff would then be against the wrong baseline
  precisely in the situations where correctness matters most.

### Consequences

- Good: the diff is always against ground truth, in the same file, in the same
  transaction as the change.
- Good: restart-safe with no extra state; the daemon re-reads the snapshot on
  startup like any other page of the file.
- Good: drift between the committed Board and the applied schema is detected
  and surfaced at plan time instead of silently mis-diffing.
- Good: a raw read of the SQLite file is self-describing (ids plus the mapping
  that explains them).
- Bad: the snapshot duplicates the definition's content inside the data file,
  so the SQLite file grows with the Board. Accepted: the snapshot is a derived
  record of the last applied definition, not its history, and documents are
  one database each.
- Neutral: the snapshot's exact representation (the compiled form versus the
  Board YAML) is an implementation detail, not settled here.

### Confirmation

Behavior tests pin: applying a change updates the snapshot in the same
transaction (a failed apply leaves the snapshot matching the previous
definition); a diff after a restart matches a diff taken before it; a Board
edited without an apply is reported as drift at plan time. These become tests
when the project has code.

## Interface notes

No public contract changes. The snapshot is internal to the SQLite file, which
the daemon owns; authors and external tools continue to see only the Board and
the CLI. That a raw read includes the snapshot is a property of the file, not a
contract any consumer is owed.

## More information

- ADR-0007: the column identity the diff relies on, and the sidecar state file
  this snapshot must not become.
- SPEC: Schema changes and the apply cycle.
