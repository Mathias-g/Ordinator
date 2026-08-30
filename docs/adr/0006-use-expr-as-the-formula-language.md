---
status: accepted
date: 2026-08-30
decision-makers: [owner]
scope:
  - compiler
  - daemon
interface-impact: new
---

# ADR-0006: Use Expr as the formula language

## Context and problem statement

Ordinator needs a formula language for the column formulas it computes in the
daemon (ADR-0005). Grist, the tool Ordinator is shaped after, uses sandboxed
Python, and Grist feature parity is a goal. Research into the language choice
established that a full general-purpose language in a Go daemon carries a real
security burden: sandboxing arbitrary code has a demonstrated escape history
(for example a critical Grist sandbox-escape vulnerability), and real Python in
a Go process means CGo plus an OS or WASM sandbox. We ruled Python out for that
reason. The candidate then became a constrained expression language plus
host-registered functions, and the question was which one. The language choice
is a real decision with genuine alternatives, so it is recorded here rather
than folded into the daemon-evaluation decision.

## Decision drivers

- The daemon must build a dependency graph from formulas: it needs to know, for
  each formula, which columns and tables it reads, so it can recompute affected
  formulas when an input changes (SPEC: Formulas). A language whose AST is
  exposed makes this static analysis cheap; one whose AST is hidden forces
  dynamic tracking or a fork.
- Formulas must work over typed columns (Int, Numeric, Date, Bool, Choice, Ref,
  RefList). Dates matter (payment deadlines, aging, business days, month-end),
  so the language should not force dates through strings.
- Formulas are pure and synchronous, with no I/O (SPEC: Formulas). The language
  must be safe to embed in a Go daemon with no non-termination or side-effect
  surface.
- Formulas should feel spreadsheet-like, so the tool is approachable the way
  Grist is. This is a product goal, not just a taste question.
- The language should be one that is actively maintained and widely adopted, so
  it does not strand the project later.

## Considered options

- JSONata, via gnata. Already pinned in Servitor (ADR-0020) for `transform` and
  `dedupe_key`, which would give one expression language across the stack.
- Expr. A Go-native, always-terminating, side-effect-free expression language.
- CEL. Google's non-Turing-complete, cost-limited expression language
  (Kubernetes, Firebase, Envoy).
- Starlark. A deterministic, execution-step-limited, Python-flavored embeddable
  language from Bazel.
- Full Python. The Grist model; ruled out on security grounds.

## Decision outcome

Chosen option: "Expr", because it is the best fit for the drivers, particularly
the dependency graph, typed dates, and authoring ergonomics.

Expr wins on the requirement that most shapes this decision, the dependency
graph. Its AST is exported and documented, with `IdentifierNode` and
`MemberNode` for column and reference reads and `CallNode` for function calls,
so the daemon can extract each formula's dependencies statically at compile
time. JSONata's Go implementation (gnata) hides its AST in an `internal/`
package and exposes only a fast-path subset, so static dependency extraction
would require forking or reimplementing; CEL and Expr expose walkable ASTs.

Expr also handles the typed-column and date requirements best. It is Go-centric
and type-aware: a typed row can be exposed as a Go struct, integers are native
`int64` (no JSON 2^53 rounding for money and IDs), and dates are native Go
`time.Time` with arithmetic and comparison built in. CEL has native timestamp
and duration types but no date-only type and no aggregation in its core.
JSONata has no date type at all and a JSON-only value model.

For authoring ergonomics, Expr reads closest to spreadsheet prose of the
constrained candidates: `Qty * Price`, `sum(Lines, .LineTotal)`,
`Status = "sent" ? max(0, ...) : 0`, and `Customer.Name`. CEL is the farthest
from spreadsheet ergonomics (C-like, no `if`, no core `sum`), and JSONata's
`$`-function prefix and epoch-millisecond date math add noise.

Expr is also a low-risk choice on the longevity axis, with a caveat. It has a
stable v1 API, continuous fuzzing via OSS-Fuzz, and broad production adoption
(Google, Uber Eats, CoreDNS, OpenTelemetry Collector, Argo, WunderGraph).
However, its development has slowed since 2026: the maintainer (effectively a
single person) has shifted focus elsewhere, and while the project is not
abandoned, upstream activity is low. For a formula engine that Ordinator embeds,
this is acceptable because Expr is stable at a pinned version and is
MIT-licensed, so Ordinator can vendor or fork it and own fixes if upstream stops
responding. gnata, the JSONata candidate, is young (v0.2.x, single-company,
restricted issue tracker) and its hidden AST pushes toward maintaining a fork.

The object-model API that makes the tool feel like Grist (reverse row-set
lookups, dynamic lookups, aggregation over related rows) is not native to Expr,
any more than it is to the other constrained candidates. It is provided by
host-registered functions that the daemon exposes, which matches the daemon
architecture already settled in ADR-0005.

### Why the other options were rejected

- JSONata (gnata) was the incumbent candidate because Servitor already uses it.
  It was rejected for Ordinator because its AST is internal and unimportable, so
  the dependency graph could only be built by forking gnata, writing a parallel
  parser, or falling back to dynamic tracking. Its JSON-only value model forces
  typed rows to be serialized to JSON-safe values at the boundary, it has no
  date type, and the gnata implementation is too young and narrowly maintained
  to bet the formula engine on.
- CEL was rejected on ergonomics: it is a policy language, not a data or
  spreadsheet language. It has no aggregation in its core and no `if`, so
  ordinary formulas become verbose, and it would require building a large domain
  library. Its institutional backing is strong, and it was reconsidered on that
  basis when Expr's maintenance slowed, but its fit here is the weakest and its
  authoring ergonomics would fight the spreadsheet-shaped product goal every day.
- Starlark was rejected because, although it is safe to embed and
  Python-flavored (closest to the Grist authoring feel), it is a full language:
  static dependency analysis requires a resolver and is weakened by dynamic
  member access, and the date/finance layer would still be host code.
- Full Python was rejected on security. Sandboxing a general-purpose language in
  a Go daemon means CGo plus an OS or WASM sandbox, and the class of
  sandbox-escape bugs is real and demonstrated. Expr's non-Turing-complete core
  has no such surface.

### Consequences

- Good: the dependency graph can be built by static analysis of each formula's
  AST, which is simpler and more robust than Grist's runtime observation.
- Good: typed columns, `int64` precision, and native Go dates are first-class, so
  the finance/date layer sits on a real date type rather than strings.
- Good: the language is safe to embed and pure, with no non-termination or
  side-effect surface; the registered host functions are the only I/O and are
  curated (SPEC: Formulas).
- Good: mature and widely adopted, with a stable v1 API and continuous fuzzing.
  Because Expr is MIT-licensed and stable at a pinned version, the low upstream
  activity is mitigated by vendoring or forking it if fixes are needed; Ordinator
  owns its formula engine's dependency surface either way.
- Good: the formula language is kept behind a narrow seam. The daemon treats
  "compile a formula, extract its dependencies, evaluate it" as one interface,
  and Expr is an implementation behind it rather than a type the rest of the
  daemon depends on directly. This keeps the option open to swap in a different
  language later without rewriting the daemon, which is the fallback if Expr ever
  becomes unmaintained in a way that matters. The seam is worth the small
  indirection because it removes the "trapped by the language" failure mode.
- Bad: Expr's function names are not spreadsheet-named by default. Ordinator
  registers Grist and Excel-style names (`SUM`, `DATEADD`, `NETWORKDAYS`, ...)
  as host functions so formulas read the way Grist users expect.
- Bad: reverse row-set lookups, dynamic lookups, and aggregation over related
  rows are not language features; they are host functions the daemon provides,
  so the daemon must materialize or serve row sets.
- Bad: there are no user-defined functions written in the language; every
  reusable function is a host-registered Go function.
- Neutral: this makes the stack two-language, alongside Servitor's JSONata. The
  single-expression-language coherence that JSONata would have given is traded
  for a better fit in Ordinator.

### Confirmation

A behavior test that walks a compiled formula's AST and asserts the extracted
dependencies (columns, tables, and functions read) matches the formula, and a
behavior test that a formula over a reverse row set and an aggregate recomputes
when a child row changes, pin both the dependency-analysis contract and the
daemon-evaluation contract. These become tests when the project has code.

The maintenance caveat is reviewed when the project gains code: before
committing to Expr as a pinned dependency, check that it remains stable or plan
the vendor/fork path. This is a review check, not an automated gate.

## Interface notes

This is a `new` interface impact: it establishes the formula language as part of
the config file format. Formulas in the table YAML are written in Expr, with
Grist and Excel-style functions available as host-registered names. The exact
shape of a formula entry in the YAML is still to be settled (SPEC: The
compiler); what is settled here is that the expression language is Expr and that
the object-model and date/finance surface is provided by registered host
functions.

## More information

- ADR-0005: formulas are computed in the daemon; this ADR settles the language
  it evaluates.
- SPEC: Formulas.
- ADR-0020 (JSONata pinned in Servitor): the reason JSONata was the incumbent
  candidate, and the reason the stack is now two-language.
