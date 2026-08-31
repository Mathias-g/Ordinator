# Architecture Decision Records

This directory is the single decision log for Ordinator. Every architecturally
significant decision lives here as a numbered, immutable Markdown file.

## Why a central log

- Significant decisions span the whole project (language, definition-file
  shape, control-plane shape, dependency rules, packaging). They have no single
  home in the code.
- Code moves: packages get split, merged, renamed. ADRs are immutable records
  that must outlive the current shape of the code.
- One global sequence means one greppable decision log and compatibility with
  ADR tooling (MADR, adr-tools, log4brains), which all assume one directory.

## Conventions

- Filenames: `NNNN-short-kebab-title.md`, zero padded, e.g.
  `0005-compile-formulas-to-sql-views.md`.
- Numbers are assigned sequentially and never reused.
- ADRs are immutable. To reverse or change a decision, write a new ADR and set
  the old one's status to `superseded by ADR-NNNN`.
- `0000-adr-template.md` is the template. Copy it, do not edit it in place.

## What an ADR records (and what it does not)

An ADR records the decision and its durable rationale, so a future reader can
understand the decision without knowing what the project looked like on the day
it was written. It is not a diary of the moment.

- Do **not** reference the implementation plan's phases or step numbers (for
  example "Phase 3"). PLAN.md is not a durable artifact; it changes as the
  project moves, so a reference to it makes an immutable ADR stale. Anchor the
  decision to the SPEC section it concerns (for example "SPEC: The compiler")
  instead.
- Do **not** describe the current state of the codebase (what is or is not
  built yet). That lives in PLAN.md and SPEC.md's intro, both of which are
  meant to drift.
- Do **not** write an ADR to describe current state, or for a change with no
  contested choice. ADRs are for decisions with real alternatives someone might
  later reverse.

## Status lifecycle

`proposed` -> `accepted` -> (`deprecated` | `superseded by ADR-NNNN`)

## Ordinator-specific fields

- `scope`: the parts of Ordinator the decision touches, e.g. `compiler`,
  `daemon`, `control-plane`, `access-rules`, `events`, `packaging`.
- `interface-impact`: whether the decision changes a public contract, the
  **config file format** (the table and view YAML), the **CLI surface**, or the
  **daemon control protocol**. `none`, `new`, or `breaking`. This is the field
  that matters most, because a public-contract change ripples to every author
  and every tool that uses Ordinator, while an implementation change behind a
  stable interface stays local.

## Enforcement

Where a decision can be checked by tooling, record how in the ADR's
`Confirmation` section. Lean on automated checks (tests, vet, lint,
pre-commit) rather than prose where you can.
