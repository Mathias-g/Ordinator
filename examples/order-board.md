# Ordinator example: an order board

Ordinator Boards are YAML, one `board.yaml` per document, holding the whole
definition: every table, with its columns, types, and formulas inline. This is
the document as an agent (or a human) sees it, all in one place, diffable and
reviewable.

A Board has two top-level keys:

- `board`: the document's name.
- `tables`: the tables. Under each table, every column: either a plain typed
  column (just a type) or a formula column (a type plus a `formula`). Columns
  are nested under their table, not top-level fields of their own.

Formulas compute in the daemon over an in-memory model with a dependency graph
(SPEC: Formulas). They are pure and synchronous, with no I/O: a value that
needs an external call is not a formula, it is a plain column a workflow
writes into. The vocabulary is Grist's: `text`, `numeric`, `int`, `bool`,
`date`, `choice`, `ref`, and `reflist`, among others.

Every column also carries a stable random `id`, a canonical UUIDv7 string
(ADR-0009), minted by tooling at authoring time; the name is the human-readable
label that formulas and the API use. The id is the column's identity: the
compiler classifies a rename, a type change, a drop, or an add by matching ids
between the old and new definition, and it refuses a missing or duplicate id
(SPEC: Schema changes and the apply cycle).

## The order document

A small order-tracking document: customers, the orders they place, the lines on
each order, and a rolling total per order.

```yaml
board: main

tables:
  # Customers: the people and companies we sell to.
  customers:
    columns:
      name:
        id: 01a05e78-0378-7301-a0f1-aa8ccacff69e
        type: text                        # display name
      email:
        id: 01a05e78-0378-72ca-8d74-9fbbd03524cd
        type: text                        # contact email
      is_vip:
        id: 01a05e78-0378-7002-8c6e-e373b266617e
        type: bool                        # true for high-value customers
      joined:
        id: 01a05e78-0378-724a-9f4b-edb771193bed
        type: date                        # when they signed up

      full_name:                          # formula: name shown capitalized
        id: 01a05e78-0378-7313-83a3-58a17b6687dd
        type: text
        formula: >-
          upper(name)

      email_ok:                           # formula: whether the email looks valid
        id: 01a05e78-0378-72fc-8d1f-415634b9e213
        type: bool
        formula: >-
          matches(email, '@')

  # Orders: one per purchase. customer points at the customer row.
  orders:
    columns:
      customer:
        id: 01a05e78-0378-738d-af49-f6d96e1b95c5
        type: ref
        target: customers
      placed:
        id: 01a05e78-0378-7095-b49f-aa0667f66422
        type: date                        # order date
      status:
        id: 01a05e78-0378-7125-b890-f4b6cae314e9
        type: choice
        values: [draft, confirmed, shipped, cancelled]

      customer_name:                      # formula: read across the reference
        id: 01a05e78-0378-73a9-bb30-d025e3f3cc07
        type: text
        formula: >-
          customer.name

      is_urgent:                          # formula: a VIP's confirmed order is urgent
        id: 01a05e78-0378-70a9-a997-0cca3201d782
        type: bool
        formula: >-
          status == 'confirmed' && customer.is_vip

      # Formula columns can aggregate their own child rows. Here the order
      # rolls up its lines, and the count of lines, without any extra table.
      total:                              # formula: sum of this order's line totals
        id: 01a05e78-0378-7121-8120-3b4aafa6e542
        type: numeric
        formula: >-
          sum(order_lines, .line_total)

      line_count:                         # formula: how many line items the order has
        id: 01a05e78-0378-7019-818f-9834567740a6
        type: int
        formula: >-
          count(order_lines)

      largest_line:                       # formula: the biggest single line
        id: 01a05e78-0378-727c-9eda-228ff4260e95
        type: numeric
        formula: >-
          max(map(order_lines, .line_total))

  # Order lines: the individual items on an order. order points back at the
  # order row; this is what lets orders aggregate their children above.
  order_lines:
    columns:
      order:
        id: 01a05e78-0378-71f0-bf5a-a9e8a1b93a61
        type: ref
        target: orders                     # which order this line belongs to
      sku:
        id: 01a05e78-0378-716b-a723-254f5c76b523
        type: text                         # product identifier
      qty:
        id: 01a05e78-0378-73c3-85b3-5df8b3ab1b76
        type: numeric                       # how many units
      unit_price:
        id: 01a05e78-0378-71e8-9450-b7d91e1ed59d
        type: numeric                       # price per unit

      line_total:                          # formula: cost of this line
        id: 01a05e78-0378-7197-993b-d7276f228e33
        type: numeric
        formula: >-
          qty * unit_price

      order_status:                        # formula: the parent order's status, read up
        id: 01a05e78-0378-7258-aac7-6a1448fd4257
        type: choice
        values: [draft, confirmed, shipped, cancelled]
        formula: >-
          order.status
```

## What it demonstrates

- **Typed columns and references.** `customer` and `order` are `ref` columns
  pointing at another table. The compiler validates that the target exists and
  that every reference resolves (SPEC: The compiler).
- **Scalar formulas.** `full_name` and `email_ok` are plain text and logic over
  the row's own columns.
- **Forward traversal.** `customer.name` reads across a reference to a parent
  row's column; `order.status` does the same from a child.
- **Reverse traversal and aggregation.** `sum(order_lines, .line_total)`,
  `count(order_lines)`, and `max(map(order_lines, .line_total))` read the rows
  that point back at this order and reduce them. This is the object-model API
  that makes Ordinator a spreadsheet rather than SQL by hand (ADR-0006).
- **Conditional logic.** `is_urgent` combines a `choice` comparison with a read
  across a reference. When a value any of these depends on changes, the daemon
  recomputes the affected formulas (SPEC: Formulas).

## How it fits with the rest of a document

A Board is one file in a document folder, alongside two others:

```
documents/
  main/                           # a Document (one database)
    board.yaml                    # this Board: tables, columns, formulas
    access.yaml                   # access rules
    data.sqlite                   # the data, not the definition
```

A Board is how the document works, not how it looks: it does not describe pages
or widgets, and it has no automation. Workflow (triggers, schedules, branching)
belongs to a workflow tool; Ordinator holds the data and computes the formulas,
and exposes plain, public-standard events that a workflow tool can react to
(SPEC: What this is not).
