# THREATS.md

Open, unsettled attack surfaces and things to investigate. This is the home for
security concerns that are not yet decisions and not yet invariants. It is not
a security-practices guide (that is not this file) and it is not a backlog of
planned features (that is IDEAS.md).

## What lives here

- Attack surfaces we know exist but have not decided how to address.
- Questions that need investigation before they can become a decision, an
  invariant, or a spec change.

## What happens when one is resolved

Where a resolved item goes depends on what kind of thing the resolution turned
out to be, not on the fact that it was a threat:

- **Behavior that can be asserted?** A test that would fail if the behavior
  regressed. Tests are enforced; prose is not, so this wins whenever possible.
- **A change to the product's behavior or interface** (config file format, CLI
  surface, daemon control protocol)? The relevant section of `SPEC.md`, not
  Gotchas.
- **A cross-cutting intent or non-obvious constraint** that no single section
  covers and no test can capture? A line in the Gotchas section of `SPEC.md`.
- **A genuine decision with real alternatives someone might later reverse?** An
  ADR in `docs/adr/`, with the test that pins the behavior as part of recording
  it.
- **Investigated and found to be a non-issue?** Discard it here.

Most resolutions are the first two kinds. An ADR is the exception, not the
default: only a real contested choice needs one.

Until then it stays here, as an open item. Keeping it here does not commit the
project to fixing it; it records that the surface exists and is unresolved.

---

## Open items

### Daemon isolation boundary

The daemon owns the SQLite file and its single write connection, and evaluates
formulas against an in-memory model. If anything in-process were compromised, it
reaches the whole database. Servitor isolates by running each capability as a
subprocess with per-node secret delivery, which contains the blast radius of a
compromised node. Whether Ordinator should borrow anything from that model is
unresolved.

The honest constraints on any transfer:

- **Formula evaluation is safe by construction.** Expr is non-Turing-complete
  and side-effect-free (ADR-0006), so there is no arbitrary user code to sandbox
  the way Grist sandboxes Python. The normal single-document case has no
  untrusted-code surface to contain.
- **Subprocess-per-formula is not viable.** Ordinator computes formulas per row
  on every recalc, thousands to millions of times in the daemon's inner loop.
  Servitor's coarse-grained per-node isolation does not transfer at formula
  granularity.
- **The host functions are the seam.** They are supposed to be pure, but they
  are the place where a bug or a future I/O-capable function could act beyond
  the data. This is the real evaluation-side attack surface.
- **Per-document isolation is where the model could transfer.** If one host
  runs many documents of differing trust (untrusted agents authoring config,
  multi-tenant hosting), a subprocess or container per document would contain
  each document's blast radius, exactly like Servitor contains each node.

Open question: whether per-document subprocess isolation is ever worth it, or
whether the single-daemon + safe-formula-language model is sufficient for the
document trust levels Ordinator actually serves. This is a decision only when
multi-document hosting of untrusted documents becomes real; until then it is an
investigation, not a commitment.

### Data at rest: encryption

How data is stored on disk is undecided. Grist stores data plainly in a SQLite
file (any at-rest encryption is a separate layer), but Ordinator's self-hosted
and agent-first posture may want a different answer. This is a storage-layer
concern, separate from compute isolation above.

Open questions:

- Whether the SQLite file is encrypted at rest by default, and if so with what
  (SQLite encryption extensions, a container layer, OS-level disk encryption the
  operator already has).
- Where the key lives and how it is provisioned. Servitor's secret-resolution
  model (per-node secret delivery, ADR-0035) is a candidate source of the key,
  but whether that is appropriate here is unresolved.
- What a raw read of the file should show once encryption exists. The SPEC
  currently says a raw read sees stored columns; encryption would change what a
  bare SQLite client can see, and that interacts with the stored-columns
  contract (SPEC: Formulas).
- Whether "encrypted at rest" should be a config flag or the default, and how it
  interacts with backup and inspection workflows.
