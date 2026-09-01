# SPEC

The product and behavior spec: what Ordinator is and how it behaves.

The open thinking and the not-yet-settled parts of the design live in
[IDEAS.md](IDEAS.md). This file is the source of truth for behavior once the
design decisions are made, and is updated as decisions turn IDEAS entries into
settled product behavior.

## The artifact and the build step

The definition lives in text files, and the data lives in SQLite. The
definition (the Board, the access rules) is authored outside Ordinator,
committed, and PR-gated, so it is diffable, reviewable, and statically
checkable in exactly one place.

The SQLite file holds the data. It holds no editable and no authoritative
definition: intent is expressed only in the Board and the access rules, never
in the database. The data file does contain derived records, which the
compiler writes in the same transaction as the change that produced them:
stored computed columns (ADR-0005) and the snapshot of the last applied
definition (ADR-0008). Nothing but the compiler ever writes these, and neither
is an authoring surface; each can only record what a reviewed change already
expressed.

This is what rules out Grist's model, where the definition is mutable metadata
rows inside the same SQLite file as the data, edited through the running
product: nothing there is diffable, reviewable, or statically checkable, and
the database becomes a second place where decisions are made.

## Formulas

Ordinator computes formulas in the daemon, over an in-memory object model, with
a dependency graph (ADR-0005). The formula language is Expr (ADR-0006); the
object-model and finance/date surface is provided by host-registered functions,
keeping formulas pure and synchronous.

The daemon holds the tables in memory and the dependency graph between computed
values. When a value a formula depends on changes, the daemon recomputes the
affected formulas. The SQLite file stores the data; computed values are written
out as stored columns so they survive restarts and are visible to a raw read of
the file. All reads and writes go through the daemon.

What this means for behavior:

- Formulas are computed by the daemon, not by the SQLite engine. The SQLite file
  is storage only.
- Formulas support the object-model API: forward and reverse reference
  traversal, row sets, aggregation over related rows, and dynamic lookups.
- Formulas are pure and synchronous, with no I/O. A value that requires an
  external call is not a formula; it is a plain column a workflow writes into.
- Computed values are current as long as the daemon is running and recompute
  when their inputs change. A raw SQLite read sees stored columns, including
  materialized computed values; it is not owed live derived values.

## Schema changes and the apply cycle

Every column carries a stable random id: a canonical UUIDv7 string (ADR-0009).
The id is the column's identity; the human-readable label is what formulas,
access rules, and the API use. Ids are minted locally by whatever authors the
column (an app, an agent, or compiler tooling for hand authors), with no
round-trip to Ordinator, and a duplicate id is a validation error. Because the
definition is text and the data
is in SQLite, a schema edit that changes the data's shape is a migration: it
goes through the plan/apply cycle and is never a silent rewrite. Some edits
(a rename, an add) touch no existing data and are near no-ops at the data
layer; the classification below says which is which.

The compiler is the whole of that machinery. It parses the Board
YAML, validates it (types, references, formula syntax), diffs the old
definition against the new one, plans the migration, and applies it. The
diff-and-apply step follows the declarative-schema migration model (declare the
desired state, the tool computes the plan), not hand-written migrations.
The compiler diffs the Board against the last applied definition, stored as a
snapshot in the SQLite file's metadata and written in the same transaction as
the schema change (ADR-0008). A Board that has been committed but not applied
is reported as drift at plan time.
The daemon hosts the compiler: the daemon owns the document folder and its
single write connection and serves data continuously, while the compiler is the
gated pipeline it runs to change the schema. "Compiler" is the right name for
the whole thing because it does more than migrate: it parses, validates,
analyzes (including building the formula dependency graph), and emits SQLite
schema objects.

The compiler classifies every schema change by matching ids between the old and
new definition. No change is ever guessed:

- **Rename** (same id, new label): data is carried; never destructive.
- **Drop** (id removed): destructive by intent; shown loudly in the plan and
  confirmed before it happens.
- **Add** (new id): a fresh empty column.
- **Type change** (same id, new type): provably safe widening conversions are
  applied automatically from a small fixed table. Lossy or unknown conversions
  are refused with a structured error; the author performs the migration in two
  supported steps, adding a converting formula column and then removing the old
  column, so the conversion is a validated, reviewable formula rather than a
  migration directive. There are no `rename:`, `migrate:`, or `drop:` directives
  in the Board.

Destruction is explicit and visible, and the default is safe:

- The plan/dry-run step surfaces destruction before it happens. A plain column
  removal appears as a loud, explicit diff (for example "DROP COLUMN
  customers.age, 10,000 values") that is confirmed before it is applied.
- A destructive act is preceded by a backup or is otherwise reversible, so a
  mistake has a recovery path, not just a warning.

Whether `apply` can roll back a migration that fails partway is open
(IDEAS: Schema changes and the apply cycle), as is how the plan is rendered for
an authoring app (same anchor).

