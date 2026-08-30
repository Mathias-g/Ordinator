---
status: accepted
date: 2026-08-30
decision-makers: [owner]
scope:
  - compiler
  - daemon
interface-impact: new
---

# ADR-0005: Compute formulas in the daemon

## Context and problem statement

Ordinator's formulas need to support the object-model API that makes a
Grist-shaped tool more than a spreadsheet: forward and reverse reference
traversal, row sets, and dynamic lookups. We researched whether a compiler could
turn formulas into SQLite views and let SQLite evaluate them, the way some query
languages compile to SQL. The research, summarized here, showed that such a
compiler cannot express the powerful cases, because those require runtime
behavior over a live object model. A decision on where formula evaluation lives
is required before the rest of the design can be settled.

## Decision drivers

- Formulas must support the object-model API that makes the tool feel like a
  spreadsheet: reverse traversal and aggregation over related rows
  (`invoice.lines.line_total`), row sets, and dynamic lookups.
- Formulas must support dynamic lookups and unbounded row-set iteration the way
  Grist does, not just a statically-compilable subset.
- The formula language should stay pure and synchronous, with no I/O, so the
  sandbox stays narrow and the dependency graph stays simple.
- The definition lives in text files; the SQLite file holds only data. The
  compiler that turns YAML into schema objects is the product.

## Considered options

- Compile formulas to SQLite views, with `INSTEAD OF` triggers routing writes
  back to base tables.
- Stored columns only, recomputed by triggers on write.
- Compute formulas in the daemon over an in-memory object model, with a
  dependency graph.

## Decision outcome

Chosen option: compute formulas in the daemon, because it is the only one that
can express the object-model API that makes the tool what it is.

The daemon holds an in-memory model of the tables and a dependency graph, and
evaluates formulas against it, keeping computed values current as inputs change.
The SQLite file stores the data; computed values are written out as stored
columns so they survive restarts and are visible to raw reads (SPEC: Formulas).

### Why the other options were rejected

The compile-to-views option cannot do the lookups and row-set iteration that
Grist's object model exists to provide. The research established the boundary
precisely:

- A lookup key that depends only on the current row's own columns can be
  rewritten as a correlated subquery, which SQLite supports (Django's
  `OuterRef`/`Subquery` is the same pattern in production). But that is the
  trivial foreign-key case, which is just a JOIN and is not what a lookup
  function is for.
- A lookup key drawn from context (current user, session, environment) cannot
  be expressed in a view at all, because a view is a fixed stored SELECT and
  SQLite has no per-request context (its literal grammar has only
  `CURRENT_TIME`/`CURRENT_DATE`/`CURRENT_TIMESTAMP`, no `CURRENT_USER`).
- Arbitrary per-row iteration over an unbounded row set has no SQL form. It
  compiles only when rewritten into set-shaped form (join, GROUP BY aggregate,
  window function), which strips the formula of exactly the arbitrary runtime
  behavior that makes it powerful.

No mature tool compiles arbitrary imperative per-row formula iteration into
SQLite views. The tools that compile to SQL (PRQL, Malloy, Logica, LINQ) all
compile set-based queries, not imperative iteration over a row set. This is
consistent with why Grist itself does not compile to SQL: it runs a sandboxed
Python engine over an in-memory model, with SQLite as storage only.

The stored-columns-only option is dominated: it has the recompute-trigger
machinery of a stored model without the flexibility of a runtime, and it is
worse than compile-to-views on the set-shaped subset it can express and worse
than the daemon on the powerful cases. It was not a live contender.

### Consequences

- Good: the object-model API, reverse traversal, aggregation, and dynamic
  lookups are all expressible, matching what makes Grist feel like Grist.
- Good: the formula language is decoupled from what compiles to SQL, so a
  constrained expression language with host-registered functions (Expr, see
  ADR-0006) is viable rather than being ruled out by the compile-to-view
  constraint.
- Bad: computed values exist only after the daemon computes them. The "any
  SQLite client sees the same computed columns" property, which compile-to-views
  would have given, is not a goal. Research showed it is a weak property: it
  only serves a raw read of the file for backup, inspection, and debugging,
  which stored data satisfies alone. Derived values are not owed to an external
  SQLite client.
- Bad: the daemon must maintain a dependency graph and a recompute discipline,
  which is the hard part no off-the-shelf Go library provides. The Excel-oriented
  Go libraries (excelize, unioffice) evaluate over a cell model, and the generic
  expression engines (jsonata-go, expr) evaluate one expression against given
  data; neither maintains a reactive graph.
- Neutral: there is no precedent to copy in Go, but the architecture mirrors
  Grist, which runs a sandboxed Python engine over an in-memory model with
  SQLite as storage only.

### Confirmation

A behavior test that a formula over a reverse row set and an aggregate recomputes
when a child row changes would pin the daemon-evaluation contract. This is a
behavioral guarantee; it becomes a test when the project has code.

## Interface notes

This is a `new` interface impact: it establishes the storage and evaluation
model for computed values. Consumers (the CLI, the HTTP API, Cerebror) read
computed values through the daemon, which is the same path they read stored data.
Raw reads of the SQLite file see stored columns, including materialized computed
values; a raw client is not owed live derived values.

## More information

- The research is summarized above in "Why the other options were rejected":
  the compile-to-SQL boundary (row-keyed keys compile to correlated subqueries,
  context keys do not, arbitrary unbounded iteration has no SQL form), the
  absence of any tool that compiles imperative formula iteration to SQLite, and
  Grist's own architecture of a sandboxed engine over an in-memory model.
- SPEC: Formulas.
- Related: ADR-0006 (Expr as the formula language). ADR-0020 pins JSONata in
  Servitor, which is the context for why Ordinator's language was a separate
  decision rather than inherited.
