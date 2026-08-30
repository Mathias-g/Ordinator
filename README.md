# Ordinator

A self-hosted, agent-first database. Schema, formulas, and views live in text
files; SQLite holds only data.

Ordinator is a Grist-shaped tool (tables, typed columns, formulas, views) built
on SQLite, where the definition lives in text files rather than inside the
database. A daemon owns the file and its single write connection, a CLI is the
control plane, and a separate frontend project renders views and handles data
entry.

The idea is not near done. The open thinking lives in [IDEAS.md](IDEAS.md); the
product behavior spec, once it exists, lives in [SPEC.md](SPEC.md). See
[AGENTS.md](AGENTS.md) for how context is kept in this repository.
