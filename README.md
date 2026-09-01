# Ordinator

A self-hosted, headless, agent-first relational spreadsheet. Schema and
formulas live in text files; SQLite holds only the data.

Ordinator is an alternative to Grist. Tables, typed columns, and formulas on
SQLite, the way a spreadsheet works. What sets it apart is how you work with
it: Grist is worked through a GUI, Ordinator through an agent.

Because an agent, not a GUI, is what reads and writes the definition, the whole
definition is a text file: the Board, the structure and behavior of a document
together. An agent opens a document and sees every table, column, and formula
in one place. It is diffable, reviewable, and statically checkable. An agent
authors it, a human reviews the diff in a PR, and a compiler turns the reviewed
YAML into a migrated SQLite schema, with a plan/dry-run step before anything
destructive is applied.

## How it is put together

- **The Boards**: one `board.yaml` per document, holding every table, column
  type, and formula. Access rules live in their own `access.yaml`.
- **The daemon** owns the document folder and its single write connection. It
  hosts the compiler, computes formulas over an in-memory model with a
  dependency graph, and serves data continuously.
- **The CLI** is the control plane: how you drive the daemon. You use it to
  inspect a document, plan and apply schema changes, and operate the running
  system. CI, an agent, and a human drive it the same way.

Ordinator is headless, with no UI of its own. The entire frontend is a separate
project, Cerebror, that depends on Ordinator, never the other way around.

## What it deliberately is not

Ordinator does not own workflow. No automations, schedules, branching, or
outbound logic in its own config; those belong to a workflow tool. Formulas are
pure and synchronous, with no I/O. And Ordinator is independent of Servitor: it
sits well next to Servitor in the same stack, but every integration point is a
published standard, useful to someone who has never heard of Servitor.

## Status

Nothing is built yet; the design is being worked out deliberately before code
starts. Open thinking lives in [IDEAS.md](IDEAS.md), settled behavior in
[SPEC.md](SPEC.md), and the plan in [PLAN.md](PLAN.md).
[AGENTS.md](AGENTS.md) describes how context is kept in this repository.
