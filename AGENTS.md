# AGENTS.md

This file tells an agent how to work in this repository. Read it before doing anything.

## What this is

Ordinator is a self-hosted, agent-first database: a Grist-shaped tool (tables,
typed columns, formulas, views) built on SQLite, where the definition lives in
text files rather than inside the database. Schema, formulas, views, widget
layout, and access rules are YAML; the SQLite file holds only data. A daemon
owns the file and its single write connection, a CLI is the control plane, and
a separate frontend project renders views and handles data entry. The compiler
that turns YAML into SQLite schema objects is the product.

This file governs how context about the codebase (why things are the way they
are, what each part does, how to work on it) lives in the codebase itself, and
how you, the agent, read it before editing and write it after.

## The docs

- **SPEC.md**: the full product and behavior spec: what Ordinator is, the
  file-first definition, the independence constraint, what it does and does not
  do. The source of truth for what to build and why.
- **PLAN.md**: the implementation plan: build phases in order, dependencies,
  and what "done" means for each. Follow this when building.
- **docs/adr/**: the decision log. Each significant decision, with its
  alternatives and rationale, recorded as a numbered, immutable ADR. This is
  where the "why" of the design lives.
- **IDEAS.md**: a scratchpad for promising directions that are not yet decided
  or built. Not commitments, and not a reference: do not rely on it as durable
  documentation. Ideas move to an ADR (and then the SPEC/PLAN) when they become
  real decisions, and are discarded when not. Once an idea is decided, its
  context, alternatives, and rationale live in the ADR itself and the entry is
  deleted from `IDEAS.md`; never point back at IDEAS.md.
- **THREATS.md**: open, unsettled attack surfaces and things to investigate.
  Not decisions and not invariants yet; when one is resolved it moves to a
  test, the relevant SPEC section, Gotchas, or an ADR depending on what the
  resolution is, and is discarded when found to be a non-issue.
- **README.md**: what Ordinator is and how to get started.

The docs are deliberately plain-language. Keep them that way.

## Current state

Nothing is built. This is a deliberate sequencing decision, recorded in
IDEAS.md: nothing is built until Servitor ships. The open design questions are
researched in IDEAS.md, not resolved. Do not invent code or decisions that have
not been made; the IDEAS file is where thinking lives until a decision turns it
into a SPEC section and an ADR.

## Guiding principle: Best Simple System for Now (BSSN)

Build the simplest thing that meets the need right now, written to an
appropriate standard, with no speculative future-proofing. See ADR-0002 for the
rationale.

In practice:

- Do not record a decision you have not made. No ADR for something still open.
- Do not add a layer where the code is self-evident. An obvious change needs no
  ADR.
- Do not future-proof the context itself: no speculative fields, no scaffolding
  "just in case."
- When something can be taken away and the system still works for now, take it
  away.
- This is why most of Ordinator's design is still open in IDEAS.md: there is no
  concrete need yet to force the answers (ADR-0002).

## Decisions already made (recorded, not locked)

The decision log is `docs/adr/`. Every ADR there records a decision already
made, with its alternatives and rationale. They are not open by default, but
they are not off-limits either: the ADR is where any challenge starts. If you
think one of these is wrong, or that a new decision is needed, raise it. Read
the ADR's rationale first, then make the case. A change is not a routine edit;
it is a new decision, recorded as a new ADR that supersedes the old one.

Read the log by the `scope` field rather than whole:

    grep -rl "<area>" docs/adr/

Structural decisions that are settled but have not (yet) been given ADRs,
because they are being treated as constraints of the design rather than choices
to relitigate. They are recorded in SPEC.md. If one becomes contested, it earns
an ADR:

- **The definition lives in text files; SQLite holds only data.** (SPEC: The
  artifact and the build step)
- **Ordinator is independent of Servitor.** Every integration point is a
  published standard with at least one consumer that is not Servitor.
  (SPEC: Independence from Servitor)
- **Ordinator does not own workflow.** No automations, schedules, branching, or
  outbound logic in its own config. (SPEC: What this is not)

## The context layers

Several homes, each owning one kind of context. Most context is in the
product/behavior spec and the decision log; the rest sits with the code.

| Layer                         | Owns                                              | Mutable? | Enforced? |
|-------------------------------|---------------------------------------------------|----------|-----------|
| `SPEC.md`                     | What the product is and how it behaves            | yes      | review |
| `THREATS.md`                  | Open, unsettled attack surfaces and things to investigate | yes | review |
| `docs/adr/`                   | Decisions with real alternatives, and their rationale | append-only | linter (front matter) |
| Exported identifiers + package docs | A package's public interface               | yes      | review |
| Tests (per package)           | How the package behaves and how to call it        | yes      | CI |
| Package `README.md` / docstring | Why the package is shaped this way: intent, invariants | yes   | no |
| Commit message / PR body      | What a specific change did                         | immutable | no |

IDEAS.md is deliberately not a context layer. It is a scratchpad for not-yet-decided
thinking; it owns no durable context, so nothing a future reader needs depends on
it. PLAN.md is the same kind of artifact: it is derived from the current state of
the codebase versus what the SPEC says the product should do, so it is a tracking
layer, not a context layer, and owns no durable context either. Anything that
does not belong to one of these layers does not go in the codebase.

## Reading before editing

Load context in this order before touching any package:

1. **What the product is and how it behaves:** `SPEC.md`. This is the source of
   truth for behavior.
2. **Why a decision was made (with alternatives):** `docs/adr/`, filtered to
   the package or area rather than read whole (see below).
3. **What a package exposes:** its exported identifiers and package docs.
4. **How a package behaves and how to call it:** the package's tests. They are
   working examples guaranteed current, because CI fails the moment code and
   test disagree.

For what a specific change did, also check the commit message or PR description.

### Finding the decisions that touch an area

Do not read the whole ADR log. Query it by the `scope` field:

    grep -rl "<area>" docs/adr/

This returns the handful of decisions touching that area out of however many
total.

## Routing what you learned

How to route what you learned this session into the codebase, and what to
discard. Most session material is not durable; if a thing does not clearly match
one of these, discard it rather than inventing a home for it.

- **Made a choice with real alternatives someone might later reverse?** ADR. If
  the choice changes behavior, the test that pins the new behavior is part of
  recording it.
- **Changed the product's behavior or interface (config file format, CLI,
  daemon protocol)?** Update `SPEC.md` and, if it was a genuine decision, write
  an ADR.
- **Established or changed how a package is supposed to behave** (an edge case,
  an input/output guarantee, a regression you just fixed)? A test. This is
  preferred over prose whenever the behavior can be asserted.
- **Fixed a bug** (a regression, an edge case, a behavior change)? A test pins
  the new behavior, and the fix and its rationale go in the commit message. Do
  not add a PLAN phase for it: PLAN.md tracks build phases, not bug fixes. A
  bug fix rises to an ADR or a SPEC change only if it altered a product
  contract or required a contested decision.
- **Found an open, unsettled attack surface or something to investigate?**
  `THREATS.md`. It is not a decision and not an invariant yet; when resolved it
  moves to a test, the relevant SPEC section, Gotchas, or an ADR depending on
  what the resolution is, and is discarded when found to be a non-issue.
- **Found a task that cannot be built yet because it depends on another idea in
  `IDEAS.md`** that is not yet in the SPEC/PLAN? Break it out as its own small
  task in the phase it belongs to in `PLAN.md`, marked with the blocking idea.
  Do not silently drop it or fold it into a completed task; it becomes buildable
  when the blocking idea is worked into the SPEC/PLAN.
- **Found a promising direction that is not yet decided or built?** `IDEAS.md`.
  It is not a commitment; it becomes an ADR and a SPEC section when it becomes a
  real decision. When it does, the ADR carries the full context, alternatives,
  and rationale, and the idea is deleted from `IDEAS.md`; do not leave the
  decided entry behind and do not point future readers back at it. It is a
  scratchpad, not a reference.
- **Learned a durable gotcha or invariant that no assertion can capture**,
  something about intent or rationale rather than behavior? Package README or
  docstring, or the Gotchas section of `SPEC.md` for cross-cutting operational
  lessons.
- **Just describes what this diff does?** Commit or PR body.
- **Exploration that concluded nothing durable?** Discard. Do not write it
  anywhere.

When something could live as either a test or a prose line, the test wins. It
is enforced; the prose is not.

### Resolving an open question (the IDEAS loop)

Open design questions live in `IDEAS.md`, grouped under the area they gate.
When a session resolves one, work the loop in this order and finish it in the
same change; a half-landed decision is the drift this exists to prevent:

1. **Decide in conversation with the developer.** Explore the options, make a
   recommendation, get a clear answer. Do not resolve an open question as a
   routine edit.
2. **Draft the ADR** (next number, `status: proposed`). It must be
   self-contained: context, alternatives, rejection reasons, rationale. Nothing
   in it points back at `IDEAS.md`.
3. **Write or extend the SPEC section** so it carries the full settled
   behavior, including durable context that was in the IDEAS entry (the SPEC
   section is the reference; IDEAS is not).
4. **Update `IDEAS.md`.** Delete the decided entry; if a whole section is
   decided, remove the section, not just the question. Do not leave a pointer
   back at what was deleted; the ADR and SPEC anchor are the record. Any
   remaining open sub-questions stay in IDEAS, under the same area, so the
   scratchpad keeps being the list of what is still open.
5. **Flip the ADR to `accepted`** once the developer confirms the draft.
6. **Verify no context was lost.** For moved prose, compare word counts
   against the previous commit (the IDEAS delta should reappear in SPEC or the
   ADR). Anything genuinely discarded, say so plainly in the session summary.

## Working with the developer

This file is guidance you follow while working; it catches most process gaps in
conversation. The hard guarantee comes from the automated gates under "What is
enforced vs trusted." Use both: you guide while the work happens, the gates
block what slips through.

Assume the developer may be junior or moving fast and may not know the
vocabulary. The system works only if you carry the process, not them.

How to behave:

- **Do the bookkeeping yourself.** When a change needs an ADR, a test, or a
  SPEC update, draft it and ask the developer to confirm. Do not tell them to
  go write it. Make the correct path the easy one.
- **Explain the term as you use it.** When you say the compiler, the outbox, or
  the independence constraint, add a one-line plain explanation. Do not assume
  the vocabulary is known.
- **One question at a time.** Do not interrogate. Ask the single most important
  confirmation, act on the answer, move on.
- **Prefer the simplest action (BSSN).** Ask before adding structure.

### Stop and confirm before

- **Changing a public interface** (the config file format, the CLI surface, the
  daemon control protocol, or a package's exported surface). Say plainly what
  depends on it and what changes. If it is a genuine contract change, set
  `interface-impact`, draft the ADR, update `SPEC.md`.
- **Making a decision with real alternatives.** Offer to record a short ADR so
  it is not relitigated later. Draft it if yes. Drop it if it is not actually
  significant (BSSN).
- **Anything destructive or hard to reverse:** deleting or renaming a package
  or a public name, moving a dependency boundary.
- **Adding a dependency.** Ask whether it is needed now or whether a few lines
  do the job for now.
- **Resolving one of the open design questions in IDEAS.md.** These are
  deliberately not decided; resolving one is a decision that needs an ADR and a
  SPEC change, not a routine edit.
- **Committing or pushing.** Do not commit or push on your own. Stage and leave
  changes in the working tree, and ask the developer before committing or
  pushing. The developer decides when, how (message style, branch), and whether
  to push.

### Remind, but do not block, when

- A behavior changed and no test was added for it.
- A public interface changed and no ADR is linked.
- A package's README or docstring now contradicts the code.

### Pre-commit checklist

The automated checks enforce structure. This covers what they cannot:

1. **Behavior added or changed, a test asserts it.** If you changed what a
   package does or fixed a bug, there should be a test that would fail without
   that change.
2. **A real decision was made, ADR drafted, or explicitly declined.** A real
   decision is one with genuine alternatives that someone might later reverse.
3. **SPEC / README still matches the code.** If the public surface or the
   intent changed, check that the prose still accurately describes it. Stale
   prose is worse than no prose.

## Conventions

### ADRs

- Location: `docs/adr/`, a single global numbered sequence.
- Template: copy `docs/adr/0000-adr-template.md`. Do not edit it in place. It
  is MADR 4.0 adapted with the `scope` and `interface-impact` fields.
- Filenames: `NNNN-short-kebab-title.md`, zero padded.
- Numbers are sequential and never reused.
- ADRs are immutable. To reverse a decision, write a new ADR and set the old
  one's status to `superseded by ADR-NNNN`.
- Status lifecycle: `proposed` -> `accepted` -> (`deprecated` | `superseded`).
- A change that breaks the config file format, CLI surface, or daemon protocol
  requires an ADR with `interface-impact: breaking`.
- ADRs are for decisions. Do not write one to describe current state, and do
  not write one for a change that involved no contested choice.
- An ADR records the decision and its durable rationale, not the moment in time
  it was made. Do not reference the implementation plan's phases or step
  numbers (for example "Phase 6"), the current state of the codebase, or any
  other thing that will drift as the project moves. Use the SPEC section the
  decision concerns (for example "SPEC: The compiler") as the anchor instead. A
  future reader of an ADR should understand the decision without knowing what
  the plan looked like on the day it was written.
- **ADRs are self-contained.** An ADR must carry everything a future reader
  needs to understand the decision: the context, the alternatives, and the
  rationale. Do not point back at IDEAS.md, research notes, or session material
  as the source of that context. IDEAS.md is a scratchpad and is not durable;
  it cannot be relied on as a reference. If the research and reasoning behind a
  choice matter enough to be recorded, they belong in the ADR itself.

### PLAN.md

- **PLAN.md is append-only.** Phases are numbered sequentially and never
  renumbered, and an existing phase is never overwritten or replaced to
  describe new work. When new work does not belong in an existing phase, add it
  as a new phase with the next number (for example, if the last phase is 12,
  the new phase is 13).
- An earlier phase may be superseded by a later one, but the superseded phase
  stays in place as a record of what was built; the later phase records the
  change. Do not reach back and rewrite or delete the earlier phase.
- The only exception is when the developer explicitly asks for phases to be
  reordered or merged.
- A change to a task inside an existing phase (marking it done, splitting out a
  blocked task) is fine; renumbering, deleting, or reworking a whole phase is
  not.
- A partially-finished task is split into a done part (`[x]`) and a not-done
  part (`[ ]`), or the `[x]` line is annotated with what is deferred. Do not
  leave a task half-done with no marker of what remains; a `[x]` means "its
  intended scope is done" and a `[ ]` means "not done", with the text saying
  exactly what is left.

### Tests

- Live per package. Run them with the project's test command once it exists
  (see "Building, testing, and releasing").
- Assert the contract and documented behavior, not implementation internals. A
  test pinning an incidental detail breaks on every refactor.

### Prose

- **No em dashes in docs** (the user dislikes them). Use commas, colons, or
  parentheses instead.
- Keep language plain and easy to understand. Avoid jargon where a clear phrase
  works.
- Write docs in a way that a person with no prior context can follow.
- When moving content in the docs, do it non-destructively: leave a pointer to
  where the content went, and verify no text was lost (compare word counts
  against the previous commit).

## Building, testing, and releasing

Not applicable yet: there is no code. The structural checks that exist are the
ADR lint and the pre-commit bookkeeping reminder (ADR-0004). When the project
gains code and a test command, this section records how to build, test, and
release.

## Guardrails

- The product/behavior contract lives in `SPEC.md`. Never let it drift silently
  from what the code does.
- Behavioral guarantees live in tests, not in prose. If it can be asserted,
  assert it.
- Raw session narrative does not go in the codebase. Only the distilled, routed
  artifacts above.
- Keep the prose layer to slow-changing things: purpose, invariants,
  non-obvious constraints. The more you push into prose, the more drift you buy.
- Do not resolve an open design question (formula strategy, access rules,
  frontend shape) as a routine edit. It is a decision: ADR plus SPEC change.

## What is enforced vs trusted

Two layers hold the process together. The agent behavior above is the soft
layer: it catches gaps in conversation, while the work happens. The gates below
are the hard layer: they block non-compliant changes regardless of who or what
made them.

Hard gates (block the change):

- The decision log lint (`scripts/checks/`): front matter parses, statuses are
  valid, numbering is sequential and unique.
- Tests, static analysis, and lint in CI, once the project has code.
- CI plus branch protection on `main`: the unskippable server-side gate. See
  CONTRIBUTING.md for the one-time setup.

Trusted (convention, made easy but not blocked):

- Good commit and PR messages, the discard discipline, SPEC and README
  accuracy. Distillation quality cannot be fully enforced; this file and the
  agent behavior make the correct path the easy one.
- `SPEC.md` accuracy is enforced only by review; the product spec is prose by
  nature.
