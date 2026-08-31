# Ideas: Ordinator

Catch-all for a direction that is not yet decided or built. Ordinator is a
self-hosted, agent-first database whose schema, formulas, and views live in YAML
files, built to stand on its own and to sit well next to Servitor in the same
stack. These are possibilities, not commitments. Nothing here is a plan or a
decision, and much of it will be discarded. When an idea becomes a real decision
it gets an ADR (the "why") and its behavior is written into a SPEC (the "what");
until then it lives here so it is not lost.

Where an idea has several plausible shapes, the options are listed rather than
resolved. Where an idea depends on something unbuilt, that dependency is named.

---

## The project

Ordinator is a Grist-shaped tool (tables, typed columns, formulas, views) built on SQLite,
where the definition lives in text files rather than inside the database.
Schema, formulas, views, widget layout, and access rules are YAML; the SQLite
file holds only data. A daemon owns the file and its single write connection, a
CLI is the control plane, and a separate frontend project renders views and
handles data entry.

The shape borrows deliberately from Servitor: the artifact is the file, the CLI
serves humans and agents identically, capability discovery is a first-class
operation, validation errors are structured, and a plan/dry-run step exists
before anything is applied.

### Parity is a baseline, not a constraint

Reference material for parity, where it lives locally:

- **Grist's own docs** are forked and cloned at `~/Development/grist-help`
  (the upstream `gristlabs/grist-help` repo). It is the authoritative source for
  how Grist behaves, including its function reference at
  `help/en/docs/functions.md`, and is where to look when deciding what parity
  means for a given feature. Read it directly rather than keeping a copy in this
  repo, so the reference cannot drift from upstream.

"Grist feature parity" is the target user experience and vocabulary, not a rule
that we copy Grist's answers to every question. Grist's design is the output of
its own priorities (a hosted multitenant SaaS, per-document sandboxed Python,
metadata inside the SQLite file, schemas edited in the UI). Where Ordinator's
priorities differ, the answer can legitimately diverge, and deciding from first
principles rather than copying is the point.

We do copy Grist where the parity is load-bearing: the object-model semantics
(references, lookups, reverse traversal, aggregation over related rows), the
column types, and the user-facing vocabulary (columns, tables, references,
formulas, pages). These are what make it feel like Grist, and users already
know them.

We decide for ourselves where our constraints differ from Grist's:

- **The definition lives in files, not in the database.** Schema, formulas,
  views, and access rules are PR-gated YAML; Grist keeps them as mutable
  metadata rows edited in the UI. This is settled (SPEC: the artifact) and is
  the single biggest divergence.
- **Static typing.** Grist columns are loosely typed (`Any`, per-cell
  narrowing). Ordinator columns are typed from the schema, so formulas cannot
  silently mix types. This is a strength we own.
- **No arbitrary user code.** Grist evaluates sandboxed Python; Ordinator
  evaluates a pure, non-Turing-complete expression language with host-registered
  functions (ADR-0006), decided on security grounds.
- **Security at rest and compute isolation.** How data is stored at rest
  (encryption), and how far to isolate the daemon (whether to borrow Servitor's
  subprocess model for per-document isolation), are ours to decide, not
  Grist's answers.
- **The file shape and project layout.** Whether one document is one folder
  under a `documents/` root, how many documents a deployment runs, and how
  boards and access rules are arranged are all decided on our terms.

Where Grist parity and first-principles conflict, first principles win. Parity
is a vocabulary and a feature target; it is not an argument for copying a
design choice Grist made for reasons that do not apply to Ordinator.

### Structure, mirrored from Servitor

Servitor is elegant in a specific way, and that elegance is worth copying even
though the details do not transfer. The principles, and how they map to
Ordinator:

**One artifact, one obvious home.** In Servitor, a Wafer is a file; a mechanism
is a package in its group's directory; a capability is a self-registering
declaration with its schema next to it. Everything is where you would look for
it, and nothing is discoverable only by reading code. Ordinator mirrors this:
a document is a folder with one `board.yaml` holding the whole definition.

**The definition is declarative and self-describing.** A Servitor capability
declares its name, description, role, delivery, and JSON Schema in one place,
and the schema feeds both validation and the agent-facing example, so the two
cannot drift. Ordinator should do the same: a column declares its type and its
formula together in the Board, and the schema drives both validation and the
derived example an agent sees.

**Flat lists plus a separate layout block.** Servitor's nodes are a flat list
with dependency edges, not nested containers; tabs and grouping are a layout
property. Where Ordinator has lists that could nest (for example tables and
columns, or any future widget layout), it prefers a flat list plus a separate
layout block, which keeps the schema non-recursive and examples renderable
from it.

**Named groups that are families, not taxonomies.** Servitor groups mechanisms
by role (core, webhook, singer, mcp, helper) and a mechanism is a member of a
group, not a leaf in a deep tree. A service reached by several mechanisms
appears in several groups. Ordinator applies the same shape where it groups
things (for example widget types by binding shape: tabular, record, aggregate,
stream, static), so the group is the data contract.

**Deletability without a central list.** Removing a Servitor mechanism's
package removes it, with a blank import the only thing to touch. Ordinator's
widgets should be the same: a widget is its own directory, discoverable by
globbing, not registered in a central import list.

**Derived examples from schemas, not hand-written.** Servitor's agent-facing
example fragments are generated from the schema's `examples` values, so they
cannot drift. Ordinator's Board and widget examples should be generated the
same way, from the schema.

The point of listing these here is that Ordinator's file shape should aim at
the same obviousness: an agent (or a human) opening a document should be able
to see what tables exist, what columns they have, and how it all fits together,
without reading implementation code.

Rough layout, not settled:

```
documents/
  main/                           # a Document (one database)
    board.yaml                    # the Board: the whole definition (tables, columns, formulas)
    access.yaml                   # access rules
    data.sqlite                   # the data, not the definition
```

Why this is attractive:

- Grist keeps schema, formulas, views, and ACLs as rows in `_grist_*` metadata
  tables inside the same SQLite file as the data, mutated through user actions.
  Nothing about that is diffable, reviewable, or statically checkable.
- A file-first definition makes a build step possible: YAML in, SQLite schema
  objects out, with a plan/apply cycle between. The compiler becomes the
  product.
- Two file-first projects in one stack can validate against each other before
  either is applied (see "Cross-validation between projects").

### Definitions

The terms, and where each thing lives. This is the hierarchy the rest of the
design builds on:

- **Document** is a database: one folder holding everything for that database.
  Documents live under a `documents/` root, one folder per document. A
  deployment runs one or more documents, each a separate folder with its own
  board, access rules, and SQLite file. A document is the unit of isolation and
  of a self-contained body of data (the analogue of a Grist document, which is
  one `.grist` file).

- **Board** is a document's complete definition: one `board.yaml` per document,
  holding all tables, each with its columns, types, and formulas inline. It is
  the analogue of Grist's code-view file, which holds the whole document's
  schema and formulas in one artifact. Schema and behavior are not separate;
  a column's type and its formula live together, as they do in Grist.

- **Table** is a set of typed columns and rows, defined inside the Board (as
  tables are defined inline in Grist's code-view). A table is data plus its
  schema and its formulas.

- **View** is not part of the Board. The compile-to-SQL-view approach was
  rejected (ADR-0005). Frontend page/screen concerns are separate from the
  Board, which is not for building frontends.

- **Access rules** are defined in `access.yaml` at the document's root.

Where things live, at a glance:

```
documents/
  main/                           # a Document (one database)
    board.yaml                    # the Board: the whole definition (tables, columns, formulas)
    access.yaml                   # access rules
    data.sqlite                   # the data, not the definition
```

### Schema changes and the apply cycle

Because the schema is text (the Board) and the data is in the SQLite file,
editing the schema is a migration, not a silent rewrite. A schema edit can
destroy data if it is treated naively: renaming a column in the Board must not
wipe the column's values out of every row.

**The compiler is the whole thing.** It parses the Board YAML, validates it
(types, references, formula syntax), diffs the old definition against the new
one, plans the migration, and applies it. The diff-and-apply part is the
compiler's **migration step**: it is the established declarative-schema
migration model (the same idea as Atlas and SchemaHero, where you declare the
desired state and the tool computes the plan, rather than hand-writing each
migration the way Flyway and Liquibase do). "Compiler" is the right name for
the whole thing because it does more than migrate: it parses, validates,
analyzes (including building the formula dependency graph), and emits SQLite
schema objects. The migration is one phase of that.

**The core principle: destructive changes must be explicit and visible; the
default is safe.**

**Column identity is a stable id, not the name.** Columns carry an internal
id; the YAML name is a human-readable label. The data in SQLite is keyed by
the id, not the name, so renaming the label never touches the data. This makes
a rename safe by construction, and it is what makes a rename *detectable*: the
compiler can match columns by id across the old and new definition, so a column
whose id is unchanged but whose label changed is a rename, by construction.

**SQLite names alone cannot tell a rename from a drop-and-add.** If the new
Board says `full_name` where the old said `name`, the SQLite file only knows
the old column `name`; it has no link to `full_name`. Looking at names gives an
ambiguous picture: it could be a rename (carry the data) or a drop-and-add
(delete `name` and its data, create a fresh empty `full_name`). The SQLite file
cannot resolve this on its own. That is why the compiler must not guess.

- **A plain column removal is destructive by intent.** It shows up loudly in
  dry-run ("DROP COLUMN customers.age, 10,000 values") and is confirmed before
  it happens.
- **The ambiguous case is refused, not guessed.** When the compiler cannot
  classify a change (a removed column where a new same-typed column appeared,
  which could be either a rename or a drop-and-add; or a type change with no
  migration), it fails and asks which the author meant, rather than silently
  choosing. This is the "refuses to guess" behavior.
- **Renames are label changes, not data operations.** A rename carries the data
  over. It never looks destructive.
- **Type changes and drops are explicit migrations.** Changing a column's type,
  or removing a column, genuinely changes the data's shape, so it must be
  declared as what it is, not inferred. The compiler does not guess "is this a
  rename or a drop-and-add?"; the author says which.
- **The plan/dry-run step surfaces destruction.** `apply` shows a loud, explicit
  diff before it happens, so an operator or agent sees the cost of a destructive
  change ahead of time.
- **A backup or recovery path is the backstop.** Even with the above, a
  destructive act is preceded by a backup or is reversible, so a mistake has a
  recovery path, not just a warning.

Open questions:

- How a rename, type change, or drop is expressed in the Board: an explicit
  `rename:`/`migrate:`/`drop:` directive, versus the compiler inferring it from
  the old and new definition and refusing anything ambiguous.
- Whether the stable column id lives in the Board (the author sees it) or is
  assigned and carried by the compiler, so the id can match columns across
  edits without the author managing it.
- How the compiler diffs old vs new definition to classify a change as safe, a
  rename, a type migration, or a drop, and where that diff is computed and
  stored.
- Whether `apply` can roll back if a migration fails partway.

---

## Independence from Servitor

A constraint, not an idea: Servitor is not a dependency. This is a separate
project that happens to work extremely well alongside Servitor, and it must be
useful to someone who has never heard of Servitor.

The rule that seems to follow: every integration point this project exposes has
to be a published standard with at least one consumer that is not Servitor. Not
"designed to also work elsewhere" but literally, if a feature only makes sense
because Servitor exists, it does not belong here.

Consequences worth noting:

- Change events go out as a public standard, so any workflow tool can receive
  them and Servitor is merely one of the things that can.
- Bulk movement uses Singer, which is a Meltano-ecosystem artifact, not a
  Servitor node type.
- A curated helper node for this project lives in the **Servitor** repo, the way
  `helper/grist` does. Servitor knowing about this project is fine and
  one-directional. This project knowing about Servitor is what to avoid.

The direction of coupling is the whole point: dependencies point at this
project, never out of it.

Open question: whether the constraint is runtime-only or also build-time. They
are different costs (see "Shared code between projects"), and it is possible to
want the first without the second.

## What it deliberately does not do

Servitor already owns workflow. Anything conditional, scheduled, retried, or
fanned out is a Wafer, not a Ordinator feature. In particular there is no
automations layer, no schedules, no outbound logic, and no branching in
Ordinator's own config; those would be a second, worse Wafer.

The line that seems to hold: Ordinator is pure and synchronous. Formulas
compute over rows with no I/O. A value that requires an external call is not a
formula, it is a plain column that a workflow writes into.

Open question: where "pure" starts to hurt. Some things people expect from a
spreadsheet (a currency conversion, a geocode, an enrichment lookup) are
formula-shaped to a user and workflow-shaped to this design. Whether the
awkwardness is acceptable, or whether some narrow escape hatch is warranted, is
unresolved.

---

## Formulas

The decision is made: formulas are computed in the daemon over an in-memory
object model, with a dependency graph, the way Grist evaluates its formulas
(ADR-0005). This section records what that decision is, why it was made, and
what remains open about the formula language. The rejected alternatives
(compile to SQL views, stored columns only) and their rationale are recorded in
ADR-0005 and are not restated here as live options.

The daemon holds the tables in memory and a dependency graph. When a value a
formula depends on changes, the daemon recomputes the affected formulas. The
SQLite file stores the data; computed values are written out as stored columns
so they survive restarts and are visible to a raw read of the file. Reading and
writing go through the daemon.

Why this shape, and what it costs, came from studying Grist. Grist evaluates
formulas in a sandboxed CPython interpreter, one per open document, over an
in-memory model, and it does not do so in SQL (its SQLite file is storage only;
the formula engine never touches it). That is why its function surface is bounded
by Python and not by SQLite. A daemon-side engine is the only way to get the
object-model API that makes a Grist-shaped tool more than a spreadsheet: forward
and reverse reference traversal, row sets, and dynamic lookups. Compiling
formulas to SQL cannot express those, and the research established there is no
mature precedent for doing so.

### What the object-model API is, and why it forces the daemon

The Grist object-model API (Record, lookupRecords, lookupOne, find.*, $group,
and relationship traversal like `rec.someRef.col`) is not a minor feature; it is
most of what makes Grist more than a spreadsheet, and it is the single biggest
thing a from-scratch tool must get right. It is why formula evaluation lives in
the daemon rather than in SQL.

- Forward traversal (`customer.tax_rate`) reads across a reference.
- Reverse traversal and row sets (`lines.amount` for the child rows,
  `lookupRecords`) read back from the parent to the children.
- `$group` and aggregation over related rows group and reduce a row set.
- Dynamic lookups (`lookupOne`) find rows by a key evaluated at runtime, which
  can come from context (the current user, a session value), not just from the
  current row.

These are runtime operations over a live model. A SQL view is a fixed stored
SELECT with its joins and grouping fixed at compile time, and SQLite has no
per-request context, so a compiler cannot express the context-dependent and
arbitrary-iteration cases. There is no mature tool that compiles arbitrary
per-row imperative formula iteration into SQLite views; the tools that compile
to SQL (PRQL, Malloy, Logica, LINQ) all compile set-based queries. That is the
research-backed reason the daemon is required.

### The function surface, and how far it goes

Studying Grist's supported surface sets a realistic baseline: it is a curated
subset of the Excel spec, not the whole thing. In Stats Grist ships only the
descriptive set (AVERAGE family, COUNT/COUNTA, MAX/MAXA, MEDIAN, MIN/MINA,
STDEV/VAR families) and greys out the distribution and regression functions.
The full supported surface, from Grist's function reference with the greyed-out
(unsupported) entries removed:

| Category | Supported functions |
| --- | --- |
| Grist | `Record`, `$Field` / `rec.Field`, `$group`, `RecordSet`, `find.*`, `UserTable`, `all`, `lookupOne`, `lookupRecords` |
| Cumulative | `NEXT`, `PREVIOUS`, `RANK` |
| Date | `DATE`, `DATEADD`, `DATEDIF`, `DATEVALUE`, `DATE_TO_XL`, `DAY`, `DAYS`, `DTIME`, `EDATE`, `EOMONTH`, `HOUR`, `ISOWEEKNUM`, `MINUTE`, `MONTH`, `MOONPHASE`, `NETWORKDAYS`, `NOW`, `SECOND`, `TODAY`, `WEEKDAY`, `WEEKNUM`, `XL_TO_DATE`, `YEAR`, `YEARFRAC` |
| Info | `ISEMAIL`, `ISERR`, `ISERROR`, `ISLOGICAL`, `ISNA`, `ISNONTEXT`, `ISNUMBER`, `ISREF`, `ISREFLIST`, `ISTEXT`, `ISURL`, `N`, `NA`, `PEEK`, `RECORD` |
| Logical | `AND`, `FALSE`, `IF`, `IFERROR`, `NOT`, `OR`, `TRUE` |
| Lookup | `lookupOne`, `lookupRecords`, `CONTAINS`, `SELF_HYPERLINK`, `VLOOKUP` |
| Math | `ABS`, `ACOS`, `ACOSH`, `ARABIC`, `ASIN`, `ASINH`, `ATAN`, `ATAN2`, `ATANH`, `CEILING`, `COMBIN`, `COS`, `COSH`, `DEGREES`, `EVEN`, `EXP`, `FACT`, `FACTDOUBLE`, `FLOOR`, `GCD`, `INT`, `LCM`, `LN`, `LOG`, `LOG10`, `MOD`, `MROUND`, `MULTINOMIAL`, `NUM`, `ODD`, `PI`, `POWER`, `PRODUCT`, `QUOTIENT`, `RADIANS`, `RAND`, `RANDBETWEEN`, `ROMAN`, `ROUND`, `ROUNDDOWN`, `ROUNDUP`, `SERIESSUM`, `SIGN`, `SIN`, `SINH`, `SQRT`, `SQRTPI`, `SUM`, `SUMPRODUCT`, `TAN`, `TANH`, `TRUNC`, `UUID` |
| Schedule | `SCHEDULE` |
| Stats | `AVERAGE`, `AVERAGEA`, `AVERAGE_WEIGHTED`, `COUNT`, `COUNTA`, `MAX`, `MAXA`, `MEDIAN`, `MIN`, `MINA`, `STDEV`, `STDEVA`, `STDEVP`, `STDEVPA` |
| Text | `CHAR`, `CLEAN`, `CODE`, `CONCAT`, `CONCATENATE`, `DOLLAR`, `EXACT`, `FIND`, `FIXED`, `LEFT`, `LEN`, `LOWER`, `MID`, `PHONE_FORMAT`, `PROPER`, `REGEXEXTRACT`, `REGEXMATCH`, `REGEXREPLACE`, `REPLACE`, `REPT`, `RIGHT`, `SEARCH`, `SUBSTITUTE`, `T`, `TASTEME`, `TRIM`, `UPPER`, `VALUE` |

This table is the baseline for what a formula surface should aim at, not a
commitment to replicate it exactly. The daemon does not need SQLite to grow a
function library to cover these; the functions are evaluated in the daemon, and
the scalar ones (Math, Text, Date, Logical, Stats) are mostly straightforward
regardless of language. A few of Grist's functions do not translate to a pure
relational model and are deliberately out of scope:

- `ISREF` / `ISREFLIST`: there is no reference/row-set type at the value layer;
  a reference is just a value, and the schema is statically typed, so "is this a
  reference" is a compile-time fact, not a runtime question.
- `PEEK`: reads a stored value that is not recomputed. Under a daemon model the
  stored-column mechanism is the way to get a version of it, or it is dropped.
- `SCHEDULE`: generates new rows, which conflicts with the "no scheduler in
  Ordinator" rule.
- `SELF_HYPERLINK`: presentation, not a data concern. It builds a clickable link
  to a record or page in the app's UI, which is generated in the frontend
  (Cerebror), not as a data formula.

### The shape, sketched

Illustrative, not a settled schema; the point is that most of the interesting
work is the object-model traversal, not the scalar functions. Formulas are
written in Expr (ADR-0006), with the object-model and date/finance surface
provided by host-registered functions:

```yaml
tables:
  # Customers: people we sell to.
  customers:
    columns:
      name: text                     # display name
      email: text                    # contact email
      is_vip: bool                   # true for high-value customers
      joined: date                   # when they signed up

      full_name:                     # same as name, shown capitalized
        formula: >-
          upper(name)

      email_ok:                      # whether the email looks valid
        formula: >-
          matches(email, '@')

      age_days:                      # days since they joined
        formula: >-
          date_diff(today(), joined, 'd')

      tag:                           # 'vip' for VIPs, else 'normal'
        formula: >-
          is_vip ? 'vip' : 'normal'

  # Invoices: one per bill we send. customer points at the customer row.
  invoices:
    columns:
      customer: {type: ref, target: customers}
      issued: date                   # invoice date
      due: date                      # payment deadline
      status: {type: choice, values: [draft, sent, paid, void]}

      customer_name:                 # pull the customer's name across the reference
        formula: >-
          customer.name

      days_overdue:                  # days late, only meaningful once sent
        formula: >-
          status == 'sent'
          ? max(0, date_diff(today(), due, 'd'))
          : 0

      is_late:                       # true if we are past the deadline
        formula: >-
          days_overdue > 0

      balance: number                # how much is still unpaid (entered, not computed)

  # Invoice lines: the individual line items on an invoice.
  invoice_lines:
    columns:
      invoice: {type: ref, target: invoices}   # which invoice this line belongs to
      amount: number                 # unit price
      qty: number                    # how many units

      line_total:                    # cost of this line
        formula: >-
          amount * qty

  # Invoice totals: one row per invoice, rolling up its lines.
  invoice_totals:
    columns:
      invoice: {type: ref, target: invoices}
      sum:                           # total of all line totals on the invoice
        formula: >-
          sum(invoice.lines, .line_total)

      line_count:                    # how many lines the invoice has
        formula: >-
          count(invoice.lines)

      max_line:                      # the largest single line total
        formula: >-
          max(map(invoice.lines, .line_total))

      mid_line:                      # the middle line total (host function)
        formula: >-
          median(map(invoice.lines, .line_total))

      status_word:                   # 'status' spelled out for display
        formula: >-
          {'sent': 'Outstanding',
           'paid': 'Paid',
           'draft': 'Draft',
           'void': 'Void'}[status]
```

The scalar functions (Math, Text, Date, Logical, Stats) are the easy part. The
reverse traversal and aggregation over `invoice.lines` is the part that decides
whether this feels like a spreadsheet or like writing SQL by hand, and it is the
part the daemon makes possible.

The hard part of the whole design is the recalculation engine with a dependency
graph over related tables, which no off-the-shelf Go library provides. The
Excel-oriented Go libraries (excelize, unioffice) evaluate over a cell model, and
the generic expression engines (jsonata-go, expr) evaluate one expression against
given data. Neither maintains a reactive graph, so the graph is always written
in-project.

Open questions:

- Reference-traversal sugar (`customer.tax_rate` for a forward reference,
  `lines.amount` for a reverse one) is what makes this feel like a spreadsheet.
  Whether it is a compiler feature or a documented pattern, and how the daemon
  expands it, is unresolved.
- Whether a second language is ever worth two engines with different security
  models. The language is now Expr (ADR-0006); the open question is whether a
  Python escape hatch behind a config flag is ever worth the sandboxing burden
  that ADR-0006 rejected, not whether to switch the core language.
- How far the pure, no I/O boundary holds, and whether the curated date-function
  set is enough before it starts to feel like rebuilding `datetime`.

---

## Change events out

For a workflow tool to trigger on Ordinator changes, Ordinator has to emit them.

**Option A: Standard Webhooks.** Emit `webhook-id`, `webhook-timestamp`, and
`webhook-signature` per the spec. This is a published standard already sent by
OpenAI, Anthropic, Stripe, Twilio, and Supabase, so the events are consumable by
n8n, Windmill, Zapier, or fifty lines of Flask, and Servitor is merely one of
the things that can receive them. Servitor's `standard_webhook` receiver is
already built and verifies signatures, so this happens to need no new mechanism
package there either. `webhook-id` is stable across delivery retries, so it maps
onto `dedupe_key: event.webhook_id` for consumers that have such a concept.

**Option B: a bespoke provider-specific receiver** in Servitor, matching the
shape of the unbuilt `grist_webhook` and `atomic_event` receivers. This puts the
integration on the Servitor side and creates exactly the coupling the
independence constraint is trying to avoid, so it is probably wrong regardless
of its other merits.

**Option C: Servitor reads Ordinator's SQLite file directly** in read-only WAL mode.
Cheap, but couples Servitor to the compiled schema and bypasses the access
rules. Probably a bad idea, kept here as the rejected alternative.

**Option D: a generic HMAC scheme of the project's own**, for producers that do
not want a Standard Webhooks library. Weaker than A on every axis except
familiarity, but cheap to offer alongside it.

Delivery mechanics, if events happen at all: an outbox row written in the same
transaction as the row change (the same non-negotiable-atom reasoning as
Servitor execution model step 8), then a delivery loop that reads, POSTs, and
marks delivered. At-least-once, matching Honker's contract.

Open questions:

- Ordering. A workflow triggered by `row_updated` may read the row and find it
  changed again. Including the changed values in the payload and treating the
  row read as advisory is one answer; a version column and explicit staleness
  detection in the Wafer is another. Promising ordered delivery is a third and
  probably not deliverable.
- Granularity. Per-row events, per-transaction events, or both. Per-row is
  simpler and noisier; a bulk import becomes thousands of events.
- Whether event emission is declared per table, per column, or globally.

---

## Writes in

Three shapes, mirroring Servitor's own vocabulary. They are not exclusive and
could arrive in this order:

- **`shell` node against the Ordinator CLI.** Zero integration work. A filtered
  env with one declared secret, `ordinator query --json` and
  `ordinator write --json -`. Useful for learning what the helper should look
  like before committing to its schema.
- **A curated helper** (`helper/ordinator`), matching the `grist`/`slack`/`github`
  helpers: discrete calls, discrete inputs and outputs, schemas in
  `capabilities`. This lives in the Servitor repo, not this one, per the
  independence constraint. Other workflow tools would write their own
  equivalents against the same HTTP API.
- **Singer tap and target.** `tap-ordinator` with a bookmark on an update cursor,
  `target-ordinator` with upsert by primary key. Both catalogs are mechanical
  transforms of the compiled schema, so they cannot drift from it.

Open questions:

- Concurrency during migration. A Servitor node writing while `apply` runs.
  A lease during apply plus a retryable structured error (`schema_locked`) that
  composes with existing backoff is one option; queueing writes behind the
  migration is another.
- Authentication. Ordinator has its own access rules; a Servitor token maps
  to a role. Whether that role is per-Wafer, per-deployment, or per-node is
  unresolved, and it interacts with Servitor's per-node secret delivery
  (ADR-0033).

---

## Cross-validation between projects

Both projects put their whole definition in git as text and both can emit
machine-readable schemas, so a linter in the pipeline could check one against
the other before either is applied:

- A Wafer writing to table `Leeds` fails with `unknown_table`, `suggestion:
  Leads`.
- A Wafer writing `status: pending` into a choice column declared
  `[draft, sent, paid, void]` fails with `invalid_choice_value`.
- A PR dropping a column fails if a registered Wafer references it.
- A view referencing a dropped column fails the same way.

Neither daemon needs to know about the other; this is a linter over two
committed directories of JSON Schema. It is also the clearest thing this design
buys that adopting Grist would not, since a Wafer cannot be statically checked
against a schema that lives as mutable rows in someone else's database.

Under the independence constraint the linter cannot live in either backend,
since neither may know about the other. Options:

- **In the frontend project**, which already holds adapters for both and
  therefore both halves of the schema.
- **A standalone CI action** reading two committed directories, depending on
  neither project.
- **Not built**, leaving the failure to show up at runtime as a degraded widget
  or a failed node.

Open question: what it does about drift between the committed snapshot and the
live deployment, which is the same freshness problem the committed capabilities
directory already has.

---

## The UI

The frontend is **Cerebror**, a separate app in its own project, not internal to
Servitor or Ordinator. It connects to either backend or both, depending on how
it is configured. Neither backend knows the app exists; each is independently
usable without it. This satisfies the independence constraint: Cerebror depends
on Servitor and Ordinator, never the other way.

Cerebror is one app that renders either backend, or both at once. Each backend
is an optional package the operator enables: enable one and Cerebror is a
Ordinator app or a workflow observatory; enable both and the nav holds both.
Because a backend never edits config through the app, Cerebror is a pure
renderer of declared YAML, and the only real difference between backends is the
data source. This lets one dashboard mix a widget of overdue invoices with a
widget of last night's failed runs.

The thing to keep in mind: a Servitor control plane as sketched is read-only,
feed-consuming, multi-instance, and tolerant of minutes-old data, while a
Ordinator frontend needs data entry, a live connection, a write path, and is
per-database. Cerebror carries both. They share an auth model (both support
OIDC) but differ in staleness tolerance.

### What a package is

A package should be a distribution and enablement unit, not a new taxonomy.
Widget groups stay binding-shaped, and a package contributes widgets into those
existing groups rather than defining its own:

```yaml
packages:
  - ordinator:
      instances:
        - {name: ops, url: http://127.0.0.1:7000, secret: ordinator_token}
  - servitor:
      instances:
        - {name: prod, feed: git@github.com:me/servitor-feed.git}
```

Each package brings an adapter, its backend-specific widgets, nav entries, and
its auth requirements.

The design point that decides how much this pays off: whether the adapter
contract is **data-shaped** ("produce a row set", "produce an event sequence")
or **product-shaped**. If data-shaped, core widgets belong to no package at all
and a Servitor run list renders in the same `grid` widget as a Ordinator table,
leaving only genuinely specific widgets in packages (the Wafer DAG diagram, the
formula dependency graph, the outbox delivery panel). If product-shaped, each
package reimplements a grid.

Open questions:

- Auth. The control plane wants coarse "who can see which runner's payloads"
  (Keycloak or similar was floated). Ordinator wants per-row and per-column
  rules enforced server-side. Whether one identity layer serves both, and
  whether the two authorization models can coexist in one app, is unresolved.
- Whether the "create tier" in the Wafers-as-a-diagram idea and any schema
  editing in the Ordinator frontend should exist at all, given both projects gate
  config changes through a reviewed PR (ADR-0009).
- Deployment. One server or two, and whether a Wails packaging still makes sense
  if a live database connection is involved.

---

## What the UI renders, and what it stores

This section concerns Cerebror (the separate UI app; see "The UI"). It
describes how the app's state is split, so the decisions here are likely to
live in the Cerebror project, not in Ordinator. It is recorded here because it
shapes what Ordinator must expose, but the implementation belongs to Cerebror.

If views and layout are declared YAML and the frontend never writes them, state
splits three ways. The split matters because blurring it is how a GUI ends up
editing config files:

- **Declared in YAML, PR-gated.** Which views exist, which widgets they contain,
  widget config, column sets, formulas, access rules, default widths and sorts.
- **Ephemeral, never persisted.** Scroll position, a filter typed to find one
  row, a temporary sort. Dies with the tab.
- **Per-user, persisted as data.** Saved filters, favorites, last-viewed table, a
  column width someone dragged and wants to keep. Rows in a `_ui_prefs` table,
  written through the same API as any other data.

If that holds, the frontend has no path that writes YAML, and no
comment-preserving YAML serializer is needed anywhere.

Open questions:

- Which of column width, sort, and visible-column-set belong in tier one versus
  tier three. There are defensible answers on both sides and the answer probably
  differs per property.
- Whether `_ui_prefs` is a real table subject to the same access rules, or
  daemon-internal state.

---

## Widgets as mechanism groups and mechanisms

This section concerns Cerebror (the separate UI app; see "The UI"). The widget
registry, how widgets group, and how they are discovered are Cerebror's
concerns, so the decisions here are likely to live in the Cerebror project.
It is recorded here because it shapes what Ordinator must expose, but the
implementation belongs to Cerebror.

The widget registry could take the same shape as Servitor's mechanisms
(ADR-0031, ADR-0045): self-registering, discoverable through a `capabilities`
command that writes files, one JSON Schema per widget type with a derived
example, deletable by removing its directory.

If grouped by binding shape rather than by visual category, the group determines
the data contract:

- `tabular` (grid, pivot, list) binds to a row set
- `record` (form, detail card) binds to one row
- `aggregate` (bar, line, scatter, single stat) binds to a grouped query
- `stream` (activity feed, run history, change log) binds to an event sequence
- `static` (markdown, heading, divider) binds to nothing

A `refresh` tag on each widget would be the analogue of a trigger's `delivery`
tag: `static`, `on_load`, `polled`, `live`. This is not decoration; it says
whether a widget can render against a published feed or needs a live connection,
which makes "this dashboard cannot work against a Servitor feed" a validation
error rather than a runtime surprise.

Open questions:

- A widget is two artifacts, unlike a Go mechanism: a schema the server needs for
  discovery and validation, and a renderer the browser needs. One directory
  emitting two build outputs is the obvious shape, with the schema defined once
  (probably in TypeScript, generating the JSON) so the renderer's props and the
  published schema cannot drift. Unverified.
- Deletability without a central import list. `import.meta.glob` over the widgets
  directory plus lazy imports is a candidate; it makes every widget render async.
- Whether the frontend project needs its own headless CLI to emit its
  capabilities directory, which is a real cost.
- Whether view YAML validates in the frontend project (which has widget schemas
  and reads the Ordinator feed) or somewhere else. Keeping the Ordinator daemon
  ignorant of what a chart is seems worth preserving.

### Flat versus nested

Servitor's nodes are a flat list with dependency edges. Widgets tempt nesting,
because tabs and splits and accordions feel like containers. Nesting makes the
schema recursive, makes examples unrenderable from the schema, and produces
validation paths like `/widgets/0/tabs/2/widgets/1/columns/3`.

The alternative is a flat widget list plus a separate layout block, with tabs as
a layout property naming groups of widget names. Unresolved, but the flat
version preserves the mental model an agent already has from Wafers.

### Widget-to-widget communication

A flat list has no answer for "click a row in the grid, the detail panel and the
chart follow." An event bus in core, with widgets emitting and binding named
values, is one option:

```yaml
widgets:
  - name: invoice_list
    type: grid
    emits:
      on_select: {selected_invoice: "id"}
  - name: invoice_detail
    type: record
    source: {table: invoices, where: "id = ${selected_invoice}"}
```

No widget imports another, bindings are declared and therefore lintable, and an
unbound reference is a validation error. Open: whether the bus is core or a
mechanism, and what happens on first render before anything has emitted.

### Degradation

`apply` will eventually drop a column a view still references, even with the
linter. A renderer that shows a greyed placeholder and a warning badge, with the
rest of the view working, is better than one that white-screens. Worth deciding
early because it affects every widget's contract.

---

## Boards

A Board is a document's complete definition: one `board.yaml` per document,
holding every table with its columns, types, and formulas inline. It is the
analogue of Grist's code-view file, which holds a whole document's schema and
formulas in one artifact. A Board is how the document works, not how it looks:
it is not a frontend page, it does not describe widgets or layout, and it is
not split across `tables/` or `views/` directories. Schema and behavior are not
separate; a column's type and its formula live together, as they do in Grist.

A Board is the same kind of thing as the rest of the definition: a file an
agent authors, that is diffable, reviewable, and statically checkable. It is
separate from access rules, which have their own file. Because a Board is one
file, it is atomic by construction: there is no partial state where a table
references something that was never defined, because the whole definition is
compiled together.

The shape below is a sketch, not a settled schema. The intent is that a Board
reads as a document, not as a data dump: a human (and an agent) can see at a
glance which tables exist, what columns they have, and how the document is put
together.

```yaml
board: main
tables:
  customers:
    columns:
      name: text
      email: text
      is_vip: bool
      joined: date

      full_name:                     # formula: same as name, capitalized
        formula: >-
          upper(name)

  invoices:
    columns:
      customer: {type: ref, target: customers}
      issued: date
      due: date
      status: {type: choice, values: [draft, sent, paid, void]}

      days_overdue:                  # formula: days late, once sent
        formula: >-
          status == 'sent'
          ? max(0, date_diff(today(), due, 'd'))
          : 0
```

### What a Board names

A Board holds tables. The same vocabulary as Grist's column types applies,
since Ordinator is Grist-shaped:

- `text`, `numeric`, `int`, `bool`, `date`, `datetime` for scalar columns.
- `choice` (one of a fixed set), `choicelist` (several of a fixed set).
- `ref` (a single reference to another table) and `reflist` (several
  references).
- `attachments` for files.

Validation checks that every table and column a Board references exists and is
well-typed, so a stale Board fails loudly at apply time instead of silently
misbehaving.

### Formatting rules

The files should be easy to read the way Grist's code-view export is easy to
read. Grist generates its code-view with consistent blank-line spacing between
tables and between logical groups of columns, and Ordinator's Board files
should do the same. These are the conventions:

- **One Board per document.** `board.yaml` is the whole definition, and there
  is no other.
- **Blank line between every section.** Tables and logical groups of columns
  are separated by one blank line, never jammed together.
- **Blank line between tables.** Each table gets its own block with a blank
  line separating it from the next, so a large Board does not read as one wall
  of text.
- **Comments say what things are, not which category they belong to.** A comment
  on a column says what the column does or why it exists, not a Grist-internal
  label like `# Stats` or `# Logical`.
- **Two-space indentation** throughout, matching the access YAML.
- **List one column per line**, and align them where the names are of similar
  length. Do not pack many columns onto one line.
- **No trailing whitespace**, and a newline at the end of the file.
- **Block scalars for long expressions.** A formula longer than a single
  readable line uses `>-` and breaks across lines (see the formula sketch in
  "The shape, sketched" under Formulas).

### Open questions

- Whether a Board names frontend pages or screens at all, given Boards are not
  for building frontends. This is unresolved and separate from the Board's
  definition role.
- How access rules reference the tables and columns a Board defines, and
  whether `access.yaml` is validated against `board.yaml`.
- Whether the whole definition in one file stays workable for very large
  documents, or whether a compiler-level include mechanism is ever needed.

---

## A shared published feed format

Both projects could publish the same shape of artifact, which is what would make
one viewer able to render either:

```
feed/
  meta.yaml           # instance id, kind, version, generated_at
  capabilities/       # servitor: mechanism groups. ordinator: table schemas
  definitions/        # servitor: the Wafers. ordinator: the Board YAML
  history/            # servitor: runs and node outcomes. ordinator: change log, outbox
  health.yaml         # daemon up, queue depth, last publish
```

Signed and redacted, per the existing redaction invariants. Servitor's
publication could be the dogfooding Wafer already sketched in its IDEAS.md;
Ordinator's could be a `publish` command the same pipeline runs.

Why this is attractive: the viewer reads feeds rather than talking to products,
so a third project gets an inspection UI by publishing the format. The
cross-validation linter reads the same artifact.

The independence constraint makes this the most likely place for a Servitor
shape to leak into Ordinator, so the location of the normalization
matters:

- **Adapters in the frontend, backends ignorant.** Servitor already writes a
  capabilities directory and has run data; Ordinator already compiles a
  schema and has a change log. The frontend reads each project's native output
  and normalizes internally. Loosest coupling, and a third tool gets an adapter
  without doing anything itself. Costs a normalizing layer per backend.
- **A neutral feed spec owned by neither**, with a publisher in each project
  targeting it. Cleaner data model in the frontend, but both projects then
  encode a format defined elsewhere, which is a soft dependency on a spec repo
  that also has to be maintained.

Open questions:

- Whether the two projects' history data is similar enough to share a format, or
  whether forcing them into one shape distorts both.
- Freshness. A feed pushed on a cron is minutes stale, which is fine for run
  history and not fine for anything transactional.
- Signing and discovery when there are several instances, which the Servitor
  control-plane idea already lists as open.

---

## Shared code between projects

Candidates, in rough order of how clearly they repeat:

- The structured error shape (`path`, `code`, `message`, `suggestion`,
  `expected`, JSON Pointers, batch-all-errors-at-once) and its renderer.
- The loopback daemon protocol.
- The secret provider interface, so the operator seals secrets once and both
  binaries consume them, and Ordinator inherits the KMS/TPM custody story
  without reimplementing it.
- The capabilities-to-disk writer.

Blocker: ADR-0046 puts components in `internal/`, which cannot be imported
across repos. Sharing means promoting a narrow set to a public module, which is
a deliberate decision with a cost.

**Runtime independence is easy; build independence is what this actually costs.**
If Ordinator imports a module from the Servitor repo, it inherits Servitor's
release cadence and Go version, and a security fix in one drags the other. Three
options, and they can differ per item:

- **A neutral third module** both import. Real, but it is a third repo to
  version and release.
- **Vendored copies**, updated by hand when either side changes.
- **Convergent but independent implementations**, tested against shared golden
  fixtures.

The distinction that may resolve most of this: a *format* can be shared without
a *module* being shared. The structured error shape is roughly a struct
definition and a JSON Pointer helper, so duplicating it in both codebases and
testing both against the same fixtures buys the interoperability without the
coupling. The secret provider is the opposite case, since the KMS and TPM
custody code is not thirty lines, and it deserves its own argument rather than a
general sharing policy.

Open question, and the real risk: a shared module that grows to cover the
subprocess runner and the expression evaluator turns two projects into one
project with extra steps. Where to stop is unresolved, and BSSN (ADR-0002) says
a component with one consumer should not be shared at all.

---

## What standalone use costs

Without Servitor as an assumed dependency, "use a Wafer for that" stops being an
available answer to a standalone user, and two things follow.

**An internal scheduler is needed regardless.** Outbox delivery retries,
dedupe-window cleanup, and stored-column recompute all need one. The discipline
is that it stays internal and never appears in user-facing YAML. The moment a
table file can declare `schedule:`, Ordinator has started rebuilding Servitor
inside itself, and it will be a worse one.

**Reactions are the harder tension.** A standalone user will want "email me when
an invoice goes overdue," and the honest answer under the current framing is
"point a workflow tool at the webhook." Options:

- **Ship nothing.** Ordinator alone is a database with no reactions, which is
  fine for some users and a dealbreaker for others. Keeps the boundary perfectly
  clean.
- **Ship one trivial sink** (an outbound webhook plus, say, email) with no
  conditions, no branching, and no retry policy in user YAML. Useful alone, and
  the start of a slope.
- **Ship a documented recipe** rather than a feature: a ten-line receiver script
  in the docs. No code to maintain, no slope, and worse ergonomics.

Unresolved, and it is the main thing that decides whether the project stands on
its own or reads as a Servitor accessory.

## The append-only event log (a Servitor-side idea)

Prompted by DeepSeek Harness, whose session log records everything the model
sees and where resume, fork, search, and replay all operate on the same event
stream.

Servitor already has run history, node outcomes, suspended continuations, and
rerun modes as four separate data structures. If a run were instead defined as
its event log, with run state a fold over that log, some things currently listed
as open might fall out:

- **Fork** becomes possible: re-run a failed run from the failed node with a
  different Wafer version or a patched input, without discarding the original.
  Today `continue` and `restart` (ADR-0044) are the only shapes because there is
  no cursor to fork from.
- **Multiple concurrent parks per run**, deferred in IDEAS.md because the
  continuation is one-per-run keyed by run id, stops being a special case. Parks
  become cursors rather than one overwritten row.
- **Replay in a viewer**: scrub a run's timeline and watch the DAG resolve,
  which is a better inspection surface than a table of outcomes and needs no new
  publishing format.

Why it is separate: this is a data-model question for Servitor, independent of
Ordinator and of any UI. It would interact with the transactional
atom (execution model step 8) and with Honker's queue semantics, and neither
interaction has been thought through.

Not buildable until: someone works out whether the fold is cheap enough at read
time, and what it does to the `suspended_continuations` design (ADR-0040).

---

## Everything is a plugin, and whether any of it applies

DeepSeek Harness makes the model adapter, the tool registry, the session log,
and the agent loop itself plugins, with "no privileged core to patch."

The argument against adopting the headline for Servitor: the privileged core is
the safety argument. The transactional atom is non-negotiable by design, and the
subprocess env filter is the security boundary precisely because no mechanism
can decline to apply it. A swappable worker loop makes both configurable. There
is also a direct tension with ADR-0002, since adopting a meta-framework to get
composability that a registry and Go packages already provide is what BSSN
exists to prevent. And it is an unstable developer preview.

The argument for looking again anyway: Servitor's mechanism model already is a
plugin architecture, and the difference is only where the floor sits. If the
floor turns out to be in the wrong place (too much in the engine, or a mechanism
that keeps needing engine changes), the comparison is worth revisiting.

For the frontend the calculus differs, since there are no durability or
secret-handling invariants there, and a thin core with everything else pluggable
is more defensible. The blocker is delivery: DSH loads plugins at runtime into a
local developer tool, while a third-party widget loaded at runtime into a
browser app holding an authenticated write connection is script execution
against live business data. Compiled-in and vendored widgets keep the property
that a deployment's widget set is a reviewed artifact.

Open question: whether "the app is a harness" is a useful framing at all, or
just a restatement of the registry model already sketched above.

---

## Licensing

MIT is under reconsideration in light of Grist's open-core creep: Grist moved
self-hosted OIDC/SAML SSO behind a paid activation key, and a self-hosted,
agent-first stack needs real SSO (Cerebror wants one identity layer). AGPLv3 is
the candidate for Servitor, Ordinator, and Cerebror. This is not decided; the
two goals pull against each other and the choice is deferred until the projects
are real.

What the discussion established, not a decision:

- The thing that actually stops a Grist-style rugpull is not the license. A
  license cannot bind its author: the copyright holder can always relicense
  their own work, so "the owner could never close it" is structurally
  impossible to license into existence.
- AGPLv3 does bind everyone else. Anyone who runs a modified version as a
  network service must offer the source of their modifications to that
  service's users. That makes the Grist-style enclosure (bolt a paid SSO/ACL
  layer on, host it, close the door) impossible for a third party, which is the
  realistic rugpull.
- The only way to make it impossible *for the owner too* is ownership
  structure, not license: contributors keep their own copyright (no
  assignment CLA), or a neutral steward holds it. That costs unilateral
  control, which is the price of "bulletproof."
- The stack is already structurally resistant because there is no hosted
  product to upsell: self-hosted binaries, no lock-in, nothing to hold behind a
  door. The enclosure lever barely exists by design.
- MIT's cost is the opposite of AGPL's: it maximizes adoption but cannot stop a
  company from closing a fork. AGPL's cost is that some companies' lawyers
  blanket-block AGPL, which can suppress adoption and contribution early.
- The shape matters. The "library friction" of AGPL barely applies because
  these are standalone daemon + CLI binaries, not embeddable libraries.

Open questions:

- Whether AGPL's anti-enclosure benefit outweighs its adoption cost for a
  young project that lives on contributors and reach.
- Whether the "owner cannot rugpull" guarantee is wanted, since it requires
  giving up unilateral control. Most people who say "bulletproof" do not want
  to pay that price.
- Whether the three projects share one license, and whether that interacts with
  the independence constraint (each must be useful to someone who never heard
  of the others; a license is not an integration point, so it likely does not).

---

## Open questions that block turning this into a spec

The design cannot become a SPEC until these are researched and resolved. They
are grouped by the decision they gate. This is a starting point for research,
not a commitment to any answer.

### The formula strategy

Decided: formulas are computed in the daemon over an in-memory object model with
a dependency graph (ADR-0005). This is no longer an open question; what remains
open is the language and the mechanics below, not whether the daemon computes
the formulas.

- **What does reference-traversal sugar expand to?** Whether `customer.tax_rate`
  and `lines.amount` are a compiler feature or a documented pattern decides
  whether formulas feel like a spreadsheet or like writing SQL by hand. Research:
  what the expansion rules are in the daemon's evaluation.
- **Is the `stored: true` mechanism a cost knob or a modelling decision?** It
  promotes an expensive formula to a stored column. Research: how stored columns
  are kept correct under writes in a daemon model, and whether it reintroduces
  the staleness/dependency problems the daemon was meant to avoid.

### The function surface

- **What does the curated Go UDF library have to cover?** The analysis split
  Grist's supported surface into native, extension, and UDF buckets. The open
  question is the real size of the UDF bucket (the `*A` coercions, combinator
  family, date helpers, `NETWORKDAYS`, `$median`) and whether that library is
  small enough to maintain or large enough to be a product of its own.
- **Does relying on loadable extensions matter?** If a raw read of the file is
  a goal, computed columns that depend on extensions are not visible to a bare
  SQLite client. Since a bare read is mostly a backup/debugging case, this is
  probably a non-issue: stored data reads fine, and derived values are not
  something an external client is owed.
- **Where does the date boundary sit?** Expr has native Go `time.Time` (ADR-0006),
  but not the finance conventions. The open question is how much date logic the
  curated host-registered set (business days, month-end, 30/360, Excel serials)
  covers before it becomes "rebuilding QuantLib," and whether
  business-day/holiday/timezone logic belongs in formulas at all.

### The pure, no I/O boundary

- **Does "pure" hold under real use?** The escape hatch was closed by design,
  but the awkward cases (currency conversion, geocode, enrichment) are
  formula-shaped to users. Research: whether a narrow, declared escape hatch is
  warranted or whether the boundary survives contact with real users.
- **What is lost by excluding `REQUEST`-style I/O from formulas?** Grist allows
  it; this design does not. The open question is how much that decision costs in
  real workflows versus what it buys in sandboxing and dependency-graph
  simplicity.

### Cross-validation and the artifact

- **How does the committed snapshot stay fresh against the live deployment?**
  The linter checks committed JSON Schema, but the live deployment drifts. This
  is the same freshness problem the committed capabilities directory already
  has, and it is unresolved.
- **Where does the linter live?** It cannot live in either backend (independence
  constraint). Frontend, standalone CI action, or not built. Unresolved.

### Access rules and auth

- **How do per-row and per-column access rules get compiled and enforced?**
   Ordinator wants Grist-style ACLs. In a daemon model these are enforced as
   filtering in the daemon's reads, but how that applies on top of the compiled
   schema is unresolved, and it cannot be enforced for a raw file read anyway (a
   bare SQLite client bypasses any filter).
- **Authentication is OIDC, across the stack.** All three projects support
   OpenID Connect against an operator-supplied identity provider, so a
   deployment can connect its own IdP (for example Keycloak). Servitor and
   Ordinator authenticate users through OIDC; neither is its own identity layer,
   each trusts an IdP. Cerebror authenticates to both Servitor and Ordinator
   from the app through OIDC, so the same identity works against whichever
   backend(s) it is configured to reach. Ordinator maps the authenticated user
   to an identity that access rules can reference. Unresolved: which flows to
   support (authorization code, device flow for a CLI), how the IdP is
   configured per document versus per deployment, and how an authenticated
   identity maps to the `user` access-rule variable.
- **How do a Servitor token and a role interact?** Per-Wafer, per-deployment,
   or per-node, and how it composes with Servitor's per-node secret delivery
   (ADR-0033). Unresolved.

### Sequencing

- **Nothing is built until Servitor ships** (decided). The remaining open
  question is what, if anything, can be researched and de-risked in the meantime
  without building, so the spec is ready when the build starts.

---

## Cross-cutting open questions

- Whether the Ordinator daemon should refuse to boot without the Honker
  extension, as Servitor does (ADR-0011), or only when event emission is
  configured. A database with no outbound events is still useful.
- Whether the two daemons and two CLIs are an acceptable operational cost, or
  whether this should have been a Servitor mechanism package after all. The
  counterargument is that Ordinator needs a frontend, a different release
  cadence, and its own write connection.
- Where seeds sit on the config/data boundary. Choice lists, tax rates, category
  tables are arguably both. A `managed: true` flag per table, where the compiler
  owns those rows and reconciles them on apply, is one option.
- Branching. YAML merges; the SQLite file does not. For a single-doc tool this
  is fine, for a team it becomes the main operational question.
- Whether Ordinator is genuinely useful to someone with no workflow tool at
  all, or whether "point a workflow tool at the webhook" is an acceptable answer
  in the README. This is the test of whether the independence constraint is real
  or aspirational.
- Whether any of this is worth building before Servitor itself is finished. The
  Servitor status section lists several unbuilt receivers and helpers, and this
  project depends on none of them but competes for the same attention. Decided:
  none of this is built until Servitor ships. The IDEAS file is where the
  thinking lives in the meantime; it is not a build commitment.
