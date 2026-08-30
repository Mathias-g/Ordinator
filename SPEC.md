# SPEC

The product and behavior spec: what Ordinator is and how it behaves.

The open thinking and the not-yet-settled parts of the design live in
[IDEAS.md](IDEAS.md). This file is the source of truth for behavior once the
design decisions are made, and is updated as decisions turn IDEAS entries into
settled product behavior. As of now one area is settled: formulas. The rest of
the spec is written as decisions are made.

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

