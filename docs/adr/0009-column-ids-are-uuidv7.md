---
status: accepted
date: 2026-09-01
decision-makers: [owner]
scope:
  - compiler
interface-impact: new
---

# ADR-0009: Column ids are UUIDv7 strings

## Context and problem statement

ADR-0007 established that every column in the Board carries a stable random id,
minted locally by whatever authors the column (an app, an agent, or compiler
tooling for hand authors), with no round-trip to Ordinator. It sketched the id
as "ULID or UUIDv7 class, 128 bits" but did not settle the exact format. That
format is a public contract: it is written into committed `board.yaml` by
arbitrary authoring tools, appears in every schema-change diff, and must stay
stable so the compiler can classify renames, drops, and adds against the last
applied definition (ADR-0008) indefinitely. The question is which concrete
identifier format to standardize on.

## Decision drivers

- The id is minted locally by whichever tool authors the board (ADR-0007), with
  no round-trip to Ordinator. The format must be generatable by the broadest
  range of tooling, ideally with nothing but that tool's standard library.
- The id is the column's identity across renames, type changes, and drops
  (ADR-0007). The format must be unambiguous and stable so the compiler can
  classify every change by matching ids (ADR-0008).
- The id lives in committed YAML and appears in every schema-change diff, so
  string length adds recurring diff noise; compactness matters, within reason.
- Time-ordering is valuable: it gives index locality in SQLite and stable,
  near-insertion ordering in the snapshot diff (ADR-0008).
- Collisions must be negligible by construction, since minting is
  coordination-free (ADR-0007).

## Considered options

- **UUIDv7 (RFC 9562).** 128 bits: a 48-bit millisecond timestamp plus
  randomness. Canonical form is a 36-character lowercase, hyphenated string
  (`0193d5a3-8b92-7b3c-9f52-1e2d3c4b5a6d`).
- **ULID.** 128 bits: a 48-bit millisecond timestamp plus 80 bits of
  randomness. Serialized as a 26-character, case-insensitive Crockford base32
  string (`01J2KQ7VX9M8N4B3T6R5W2Y1`), no hyphens.
- **KSUID.** 160 bits: a 32-bit timestamp plus 128 bits of randomness, 27
  characters of mixed-case base62.
- **Random hex / nanoid.** Random-only, not time-ordered; smaller entropy unless
  made long.
- **Sequential integers.** A counter, which requires coordination or a central
  source and so violates the no-round-trip requirement of ADR-0007.

## Decision outcome

Chosen option: "UUIDv7 (RFC 9562)", in its canonical 36-character lowercase,
hyphenated string form.

UUIDv7 wins on the driver that shapes this decision most: the coordination-free,
any-author-mints-locally requirement. It is a formal IETF standard and the most
universally available identifier generator across every ecosystem (every
language standard library, `uuidgen`, database `gen_random_uuid()`), so the
broadest range of authoring tools can mint a valid id with nothing extra. It is
128-bit and time-ordered, giving negligible collisions and index locality and
stable ordering in the snapshot diff. The 36-character canonical form is the
one real cost versus ULID's 26, and it is accepted: ids are machine-written and
appear once per column, so the extra ten characters of diff noise are minor
compared with the tooling reach and standard footing.

### Why the other options were rejected

- **ULID** was the closest alternative: more randomness (80 vs 74 bits) and a
  shorter, cleaner diff string. It was rejected because it is a spec without
  formal IETF standardization and, more importantly, requires a specific library
  in authoring tools. Under the any-author-mints-locally requirement, a tool
  that cannot mint the format without a bespoke dependency weakens the
  no-round-trip guarantee. The small diff-noise saving did not outweigh that.
- **KSUID** was rejected for the same tooling-reach problem as ULID, with no
  offsetting standard footing, and a longer mixed-case string.
- **Random hex / nanoid** was rejected because it is not time-ordered, so it
  loses index locality and stable snapshot-diff ordering, and gives no
  timestamp signal.
- **Sequential integers** were already ruled out by ADR-0007: they need a
  coordinating counter, which contradicts coordination-free local minting.

### Consequences

- Good: any authoring tool can mint a valid id with ubiquitous, usually
  standard-library, support. No bespoke generator is needed.
- Good: time-ordering gives index locality in SQLite and stable, near-insertion
  ordering in the snapshot diff (ADR-0008).
- Good: 128-bit and coordination-free, with negligible collision, matching the
  id class ADR-0007 already pinned.
- Bad: the 36-character canonical string is noisier in diffs than ULID's 26.
  Accepted: ids are machine-written, appear once per column, and the standard
  footing and tooling reach matter more than ten characters.
- Neutral: the id format is decoupled from the SQLite physical column naming
  scheme, which ADR-0007 left open as an implementation detail.
- Neutral: the canonical form is lowercase hex with hyphens. Authoring tools
  should emit it exactly; the compiler validates it strictly (see Confirmation).

### Confirmation

Behavior tests once the project has code pin: the compiler accepts a canonical
UUIDv7 id and rejects one that is malformed (wrong length, wrong character set,
missing hyphens, not a UUID); a duplicate id still fails validation (ADR-0007);
a rename with an unchanged id still carries data; and the compiler's own id
minting (the `compile --fix` path from ADR-0007) emits canonical UUIDv7.

## Interface notes

This is a `new` interface impact. The Board file format's `id` field,
established in ADR-0007, is specified as a canonical UUIDv7 string: 36
characters, lowercase, hyphenated, per RFC 9562. Any tooling that authors
Boards must emit this form, and Ordinator's own tooling mints and validates it.
The id remains distinct from the human-readable label, which formulas, access
rules, and the API continue to use. The SQLite physical column naming scheme is
not part of this contract.

## More information

- ADR-0007: the id's role in change classification, and the "ULID or UUIDv7
  class, 128 bits" sketch this ADR settles.
- ADR-0008: the last-applied snapshot whose diff ordering benefits from
  time-ordering.
- SPEC: Schema changes and the apply cycle.
- SPEC: The compiler.
