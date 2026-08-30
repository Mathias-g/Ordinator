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

Rough layout, not settled:

```
project/
  config.yaml          # substitutions, packages, includes
  tables/
    customers.yaml     # columns, types, formulas
    invoices.yaml
  views/
    billing.yaml       # widgets and layout
  access.yaml
  seeds/
    tax_rates.csv
  data.sqlite          # not the source of truth for anything but data
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

The known options, none chosen:

**Option A: compile to SQL views.** Each table becomes a base table of input
columns plus a view adding the computed ones, with `INSTEAD OF` triggers routing
writes back to the base. Computed values are never stale, there is no runtime
dependency graph, and a raw SQLite read of the file shows the computed columns.
A `stored: true` flag would promote an expensive formula to a real column
with recompute triggers, as a cost knob rather than a modelling decision.

**Option B: stored columns only,** recomputed by triggers on write. Simpler to
reason about, worse for formulas over aggregates, and puts more logic in
generated trigger bodies.

**Option C: compute in the daemon,** Grist-style, with a dependency graph. Most
flexible language options, but computed values exist only after the daemon
computes them, so a raw SQLite read does not show them unless they are written
to stored columns.

The language is a separate axis from the evaluation strategy:

- SQL expressions compile to views (Option A) and nothing else does. If views
  are the mechanism, the language is largely decided by it.
- JSONata is already pinned in Servitor (ADR-0020) for `transform` and
  `dedupe_key`, so reusing it would give one expression language across the
  stack. But JSONata is a JSON-document language and does not compile to a SQL
  view, so this only works with Option C.
- Some third language (a small expression DSL compiled to SQL) is possible and
  probably not worth it.

A point against feeling obligated to match Python: there is no settled industry
formula language to diverge from. The spreadsheet products do not agree. Excel's
Python is a paid cloud-compute add-on, not open core, and it can only process
worksheet or Power Query data; Google Sheets uses JavaScript (Apps Script), not
Python; LibreOffice uses Basic; only Grist commits to Python as first-class. So
choosing JSONata (or any language that fits the "pure, compiled" constraints)
does not break against a standard the rest of the field honors.

Open questions:

- SQLite's expression vocabulary is thin (no regex, weak date handling, limited
  string functions). Shipping extensions or a UDF registry closes the gap; a
  `python:`-style escape hatch reopens the sandbox and dependency-graph problems
  the design was avoiding. Where to draw that line is unresolved.
- Reference traversal sugar (`customer.tax_rate` for a forward reference,
  `lines.amount` for a reverse one) is what makes this feel like a spreadsheet
  rather than like writing SQL by hand. Whether that is a compiler feature or a
  documented pattern is unresolved.
- If two expression languages end up in the stack, the rule for which is where
  needs to be statable in one sentence, or it will be a permanent source of
  confusion.

A note on the "any SQLite client sees the same thing" idea, since it carried a
lot of weight in earlier drafts and that weight was mostly mistaken. The name
sounds like a generic win, but the consumers it would serve barely exist: the
daemon talks to the file directly, Cerebror reads a feed or the HTTP API, and
Servitor uses a helper or CLI, none of which open the file. What the property
actually buys is a raw read of the file for backup, inspection, and one-off
debugging. That is a real but weak property, it is satisfied by stored data
alone, and it is not worth redesigning the formula strategy to preserve. Derived
values are not something an external SQLite client is owed; anyone who needs
them should read through the daemon like everyone else.

### What the options cost, on the current evidence

The trade is sharper than the option list alone suggests. Grist evaluates
formulas in a sandboxed CPython interpreter, one per open document, over an
in-memory object model, and it does not do so in SQL (its SQLite file is storage
only; the formula engine never touches it). That is why its function surface is
bounded by Python and not by SQLite. Its
supported set is a curated subset of the Excel spec, not the whole thing: in
Stats it ships only the descriptive set (AVERAGE family, COUNT/COUNTA, MAX/MAXA,
MEDIAN, MIN/MINA, STDEV/VAR families) and greys out the distribution and
regression functions. So "Grist supports the full spec" is false, and the honest
baseline is a curated subset, not the full Excel language.

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

Against that baseline, Option A is more viable than the thin-vocabulary worry
suggests. SQLite 3.35+ ships a built-in math library, and a handful of loadable
extensions (regex, ICU, a functions extension for MEDIAN/STDEV) cover most of
Grist's scalar surface. The genuine hard exclusions under Option A are few and
mostly conceptual:

- `ISREF` / `ISREFLIST`: a compiled SQL view has no reference/row-set type, so
  there is nothing for them to test. No extension fixes this, because the type
  does not exist at the SQL layer. It also does not need to exist, because the
  schema is statically typed: "is this a reference" is a compile-time fact.
- `PEEK`: reads a stored value that is not recomputed, which a view has no
  concept of (views are always derived). The `stored: true` mechanism is the
  only way to recover a version of it.
- `SCHEDULE`: generates new rows, which a view cannot do; it conflicts anyway
  with the "no scheduler in Ordinator" rule.

Separately, `SELF_HYPERLINK` is not impossible, it is the wrong layer. It builds
a clickable link to a record or page in the app's UI, which is presentation, not
a data computation: a SQL view computes values from data and has no notion of
"the app's navigation," so there is nothing for it to resolve to at the SQL
layer. If the feature is wanted, it is generated in the frontend (Cerebror) from
the row's id, not as a data formula. It is grouped with the hard exclusions only
in the sense that it does not belong in the data model at all.

For the rest, coverage under Option A splits into three kinds of support. Most
of Grist's scalar surface is reachable, but the split determines what you have
to write yourself:

- **Native to SQLite.** The logical functions (`IF`, `AND`, `OR`, `NOT`, `TRUE`,
  `FALSE` via CASE WHEN); the math library SQLite 3.35+ ships (`ABS`, `CEILING`,
  `FLOOR`, `MOD`, `POWER`, `SQRT`, `EXP`, `LN`, `LOG`, `LOG10`, `SIGN`, `TRUNC`,
  `PI`, `RADIANS`, `DEGREES`, `SIN`/`COS`/`TAN` and inverse/hyperbolic); basic
  aggregates (`SUM`, `AVERAGE`, `COUNT`, `MAX`, `MIN`, `AVERAGE_WEIGHTED` via
  `SUM(w*v)/SUM(w)`); core text (`LEN`, `LOWER`, `UPPER`, `TRIM`, `MID`,
  `LEFT`, `RIGHT`, `REPLACE`, `SUBSTITUTE`, `CONCAT`, `CONCATENATE`, `CHAR`,
  `CODE`, `REPT`, `VALUE`, `T`, `EXACT`); type checks via `typeof()` (`ISNUMBER`,
  `ISTEXT`, `ISNONTEXT`, `ISLOGICAL`, `ISNA`); basic dates via `strftime` /
  `julianday` / datetime modifiers (`DATE`, `DATEADD`, `DATEDIF`, `DAY`, `DAYS`,
  `HOUR`, `MINUTE`, `SECOND`, `MONTH`, `YEAR`, `NOW`, `TODAY`, `EDATE`,
  `EOMONTH`, `WEEKNUM`, `WEEKDAY`, `YEARFRAC`); and the cumulative functions as
  SQL window clauses (`NEXT`/`PREVIOUS` as `LAG`/`LEAD`, `RANK` as `RANK()`).

- **Loadable extensions.** Regex (`REGEXEXTRACT`, `REGEXMATCH`,
  `REGEXREPLACE`, and the `ISEMAIL`/`ISURL` that build on it) via a regex
  extension or SQLite 3.45+'s built-in `regexp`; Unicode handling via
  `sqlite-icu`; and the descriptive stats the ecosystem covers (`MEDIAN`,
  `STDEV`, `STDEVP`) via a functions/statistics extension.

- **Written as Go UDFs.** Everything else, mostly the Excel-specific
  coercions and the non-native scalar helpers that no extension bothers to
  implement: the `*A` stats variants (`AVERAGEA`, `MAXA`, `MINA`, `STDEVA`,
  `STDEVPA`, which treat text as 0 and booleans as 1/0), the combinator
  family (`GCD`, `LCM`, `COMBIN`, `FACT`, `FACTDOUBLE`, `MULTINOMIAL`,
  `MROUND`, `EVEN`, `ODD`, `QUOTIENT`, `RAND`, `RANDBETWEEN`, `ROMAN`,
  `ARABIC`, `NUM`, `SERIESSUM`, `SUMPRODUCT`, `SQRTPI`, `ROUNDUP`,
  `ROUNDDOWN`), text helpers (`PROPER`, `CLEAN`, `FIND`, `SEARCH`, `DOLLAR`,
  `FIXED`, `PHONE_FORMAT`), the remaining date functions (`DATEVALUE`,
  `DTIME`, `ISOWEEKNUM`, `NETWORKDAYS`, `XL_TO_DATE`, `DATE_TO_XL`,
  `MOONPHASE`), and the trivial info helpers (`ISERR`, `ISERROR`, `N`, `NA`).

The Grist object-model API (Record, lookupRecords, lookupOne, find.*, $group,
and relationship traversal like `rec.someRef.col`) is not a minor feature; it
is most of what makes Grist more than a spreadsheet, and it is the single
biggest thing a from-scratch tool must get right. Under Option A it is not a
loss of capability, but it does change character: these stop being callable
functions and become the shape of the compiled query. That is a compiler
burden, not a free translation.

- Forward traversal (`rec.customer.tax_rate`) compiles to a JOIN across the
  reference.
- Reverse traversal and row sets (`lines.amount` for the child rows,
  `lookupRecords`) compile to subqueries filtered back on the parent.
- `$group` and aggregation-over-related compile to GROUP BY.
- `lookupOne` is a keyed lookup, expressible as a JOIN or a filtered subquery.

The researched boundary, stated bluntly. The two capabilities that make Grist's
lookups and row sets powerful cannot be done under Option A. What survives is
only the uninteresting leftover, and calling it support is misleading:

- A lookup key that depends only on the current row's own columns can be
  rewritten as a correlated subquery (SQLite supports these; Django's
  `OuterRef`/`Subquery` is the same pattern). But that is the trivial foreign-key
  case, which is just a JOIN and is not what a lookup function is for.
- A lookup key drawn from context (current user, session, environment) cannot
  be done at all, because a view is a fixed stored SELECT and SQLite has no
  per-request context.
- Arbitrary per-row iteration over an unbounded row set cannot be done; it
  compiles only when rewritten into set-shaped form (join, GROUP BY aggregate,
  window function), which strips the formula of exactly the arbitrary runtime
  behavior that makes it powerful.

So the answer is no. Option A cannot do the lookups and row-set iteration that
Grist's object-model API exists to provide; it only covers the trivial subset
that no one would reach for a lookup function to write. There is no mature
precedent for compiling arbitrary per-row imperative formula iteration into
SQLite views; the tools that compile to SQL (PRQL, Malloy, Logica, LINQ) all
compile set-based queries.

For the formulas that do stay in the set-shaped subset, the ergonomics still
matter. A formula that reads as `customer.tax_rate` or `lines.amount` is
pleasant; the compiler has to know how to expand each traversal into the query
structure, and the difference between "a spreadsheet feel" and "writing SQL by
hand" is the reference-traversal sugar the IDEAS already lists as an open
question. This is the part of the design worth protecting, because it is where
the agent-authoring advantage over Grist's mutable metadata is actually earned.

The shape, sketched in JSONata-style expressions over an invoicing scenario.
This is illustrative, not a settled schema; the point is that most of the
interesting work is the object-model traversal, not the scalar functions:

```yaml
tables:
  customers:
    columns:
      name: text
      email: text
      is_vip: bool
      joined: date
      full_name: {formula: "$uppercase(name)"}                       # Text
      email_ok: {formula: "$matches(email, '@')"}                    # Info / regex
      age_days: {formula: "$date_diff($today(), joined, 'd')"}       # Date
      tag: {formula: "$iif(is_vip, 'vip', 'normal')"}                # Logical
  invoices:
    columns:
      customer: {type: ref, target: customers}
      issued: date
      due: date
      status: {type: choice, values: [draft, sent, paid, void]}
      # Object-model: forward traversal into the referenced row
      customer_name: {formula: "customer.name"}                       # Grist (forward)
      days_overdue: {formula: "$iif(status = 'sent', $max(0, $date_diff($today(), due, 'd')), 0)"}  # Date + Logical + Math
      is_late: {formula: "days_overdue > 0"}                          # Logical
      balance: {formula: "paid_amount"}                               # (data column)
  invoice_lines:
    columns:
      invoice: {type: ref, target: invoices}
      amount: number
      qty: number
      # Object-model: reverse traversal up to the parent
      line_total: {formula: "amount * qty"}                           # Math
  invoice_totals:
    columns:
      invoice: {type: ref, target: invoices}
      # Object-model: reverse row set + aggregation over related children
      sum: {formula: "$sum(invoice.lines.line_total)"}                # Grist (reverse) + Math aggregate
      line_count: {formula: "$count(invoice.lines)"}                  # Stats
      max_line: {formula: "$max(invoice.lines.line_total)"}           # Stats
      mid_line: {formula: "$median(invoice.lines.line_total)"}        # Stats (needs extension/UDF)
      status_word: {formula: "$lookup({'sent':'Outstanding','paid':'Paid','draft':'Draft','void':'Void'}, invoice.status)"}  # Lookup
```

The scalar functions (Math, Text, Date, Logical, Stats) are the easy part and
mostly map to native SQLite, extensions, or small UDFs. The reverse traversal
and aggregation over `invoice.lines` is the part that decides whether this
feels like a spreadsheet or like writing SQL by hand, and it is the part Option
A has to compile into subqueries and GROUP BY.

No off-the-shelf Go library provides the hard part either way: a recalculation
engine with a dependency graph over related tables. The Excel-oriented Go
libraries (excelize, unioffice) evaluate over a cell model, and the generic
expression engines (jsonata-go, expr) evaluate one expression against given
data. Neither maintains a reactive graph, so the graph is always written
in-project regardless of language.

Two practical consequences:

- "Support both JSONata and Python behind a config flag" is mechanically easy
  only under Option C (a shared evaluator interface), and it costs two sandboxes
  with different security models. It also rules out Option A, because Python
  does not compile to a SQL view. The lean is to build the interface seam, ship
  one language (JSONata, already pinned in Servitor) and leave the second as a
  documented future seam rather than two sandboxes on day one.
- If a raw read of the file is a goal at all, the formula strategy and the
  storage cost interact: leaning on a stack of loadable extensions to rescue
  Option A means computed columns depend on extensions an external SQLite client
  does not have loaded, so even the raw-read benefit of Option A only holds for
  clients with the same extensions.

JSONata's most concrete gap is date handling: it has no date type at all, only
strings, and SQLite's own date functions are thin. Since date logic (due dates,
aging, overdue, truncation) is the bread and butter of business data, this needs
a deliberate answer, not a late discovery. The lean is a curated set of
host-registered Go date functions behind fixed JSONata names (`$date_add`,
`$today`, `$date_diff`, `$date_trunc`, `$parse_date`, `$format_date`, and the
richer ones like `NETWORKDAYS`), reusing the registered-function mechanism
already used in Servitor. That keeps formulas pure and safe while closing the
gap. The known cost: a fixed set is not Python's full `datetime` plus `dateutil`,
so anything beyond the curated set (business-day or holiday-aware logic, arbitrary
timezone conversion in a formula) will feel cramped and will keep pulling on
that boundary.

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

Servitor's IDEAS.md already sketches a control plane as a separate project: a
read-only app consuming published data, aggregating several deployments, never
talking to the daemon protocol. Ordinator needs a frontend too. Whether these
are one project or two is open.

The tension: the control plane as described is read-only, feed-consuming,
multi-instance, and tolerant of minutes-old data. A Ordinator frontend needs data
entry, which means a live connection and a write path, and it is per-database.

**Option A: one app, packages per backend.** A single renderer where each backend
is an optional package the operator enables. Enable one and it is a Ordinator app
or a workflow observatory; enable both and the nav holds both. This is the
current leaning, and it fits the independence constraint as long as the packages
are independently enableable and neither backend knows the app exists.
Attractive if the Ordinator frontend never edits config, because then both are
pure renderers of declared YAML and the difference really is only the data
source. It also lets one dashboard mix a widget of overdue invoices with a
widget of last night's failed runs.

**Option B: two apps sharing a component library.** An npm package with design
tokens, table chrome, and the structured-error renderer; separate shells,
routing, auth, and deployment. Avoids one app carrying two auth models and two
staleness assumptions.

**Option C: three surfaces.** An observatory (read-only, multi-instance,
feed-consuming), a grid (live, per-database, write path), and no authoring
surface at all, on the grounds that the agent plus the PR review in the git host
is the authoring UI and it already exists.

### What a package is, if Option A holds

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

## A shared published feed format

Both projects could publish the same shape of artifact, which is what would make
one viewer able to render either:

```
feed/
  meta.yaml           # instance id, kind, version, generated_at
  capabilities/       # servitor: mechanism groups. ordinator: table schemas
  definitions/        # servitor: the Wafers. ordinator: table and view YAML
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

This is the master question: Option A (compile to SQL views), Option B (stored
columns), or Option C (daemon-side graph). Everything else about formulas hangs
off it, and it is currently undecided.

- **How much reverse-traversal and aggregation can a compiler prove statically?**
  The `invoice.lines.line_total` pattern (a reverse row set aggregated per
  parent) is the hard case under Option A. The open question is where the
  compiler's ability to expand a formula into a correlated subquery / GROUP BY
  breaks: a dynamic lookup key, a relationship resolved from a value at runtime,
  an unbounded row set. Find the boundary of what is statically compilable and
  what forces a runtime (Option C) escape.
- **Are dynamic lookups and unbounded row sets non-negotiable?** If they are,
  that alone decides the formula strategy, because Option A cannot do them. The
  trivial case (a key from the current row's own columns) can be rewritten as a
  correlated subquery, but that is just a JOIN and not what a lookup is for. A
  key from context (current user, session) is impossible in a pure view. And
  arbitrary per-row iteration over an unbounded row set only compiles when
  rewritten into set-shaped form, which strips the behavior that makes it
  powerful. So if dynamic, computed, or context lookups and arbitrary row-set
  iteration are load-bearing, the answer is no for Option A and the design is
  Option C.
- **What does the reference-traversal sugar actually compile to?** Whether
  `customer.tax_rate` and `lines.amount` are a compiler feature or a documented
  pattern decides whether Option A feels like a spreadsheet or like writing SQL.
  Research: what the expansion rules are, and what a formula cannot express once
  it is limited to what compiles.
- **Is the `stored: true` mechanism a real escape or a fiction?** It is the
  only recovery for `PEEK`-style behavior under Option A, and a cost knob for
  expensive formulas. Research: how recompute triggers keep stored columns
  correct under writes, and whether that reintroduces the staleness/dependency
  problems Option A was meant to avoid.
- **Which language, and does supporting two work?** JSONata is the lean (already
  in Servitor). The open question is whether a second language (Python) behind a
  config flag is ever worth two sandboxes, and whether the "one sentence rule"
  for which language goes where can actually be stated.

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
- **Where does the date boundary sit?** JSONata has no date type; the lean is
  curated host-registered date functions. The open question is how much date
  logic the curated set covers before it becomes "rebuilding datetime," and
  whether business-day/holiday/timezone logic belongs in formulas at all.

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
  Ordinator wants Grist-style ACLs, which under Option A must become SQL-level
  filtering, but how that compiles from YAML is unresolved, and it cannot be
  enforced for a raw file read anyway (a bare SQLite client bypasses any
  in-database filter).
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
