# Contributing

This repository keeps its own context: why things are the way they are, what
each part does, and how to work on it. A few small automated checks keep that
context honest so the codebase stays understandable as it grows.

You do not need to memorize how it all works. The checks guide you, and the AI
agent does most of the paperwork (writing ADRs, tests, and SPEC updates) for
you. Your job is mostly to follow the setup below once, then answer the agent's
questions as you work.

If you want the full reasoning, read `AGENTS.md`. This file is just how to get
set up and what to do day to day.

## One-time setup (do this once after cloning)

1. **Install pre-commit** (the tool that runs checks when you commit):

       go install github.com/pre-commit/pre-commit@latest
       # or install it any other way you prefer

2. **Turn the checks on for this repo:**

       pre-commit install

That is it. From now on, the checks run automatically every time you commit.

As the project gains code, this section grows the setup for building and
testing (the language toolchain, lint, and the test command). Until then the
only check is the ADR lint.

## The everyday loop

1. **Edit your code.** Ask the agent for help; it knows the rules in
   `AGENTS.md`.
2. **Commit.** When you run `git commit`, the checks run first. If something is
   wrong, the commit stops and tells you what to fix. Fix it and commit again.
3. **Push** your branch to GitHub.
4. **Open a pull request.** GitHub runs the same checks on its servers.
5. **Merge** once the checks are green. With branch protection on (see below),
   the button stays locked until they pass.

The checks on your machine are the fast warning. The checks on GitHub are the
real gate. Same checks, two moments.

## Writing an ADR

When the agent drafts an ADR, follow the conventions in `docs/adr/README.md`,
especially "What an ADR records (and what it does not)": keep it to the decision
and its durable rationale, and do not reference plan phases or current codebase
state, since both drift.

## When a check stops you, here is what it means

You do not need to understand the internals. Read the message, do the fix, try
again.

- **"ADR front matter lint" failed.** A decision record under `docs/adr/` is
  missing a field or has an invalid value. The message says which file and which
  field. Copy `docs/adr/0000-adr-template.md` if you are starting one from
  scratch, or ask the agent to fix the fields.

More checks (tests, vet, lint) are added here as the project gains code.

## If you are setting up the repository (admin, one time)

Before anything else, replace the placeholders left by the initial setup:

- **ADR dates**: fill in the `date:` field in each ADR under `docs/adr/` with
  the date you are formally adopting the decision.
- **ADR decision-makers**: replace `[you]` in each ADR's front matter with the
  actual names or roles.

Then, make the checks a hard gate by configuring GitHub branch protection:

1. Go to **Settings > Branches**.
2. Under **Branch protection rules**, click **Add rule**.
3. Branch name pattern: `main`.
4. Tick **Require a pull request before merging**.
5. Tick **Require status checks to pass before merging**, then select the
   **checks** workflow.
6. Click **Create** (or **Save changes**).

Now no change can reach `main` without passing the checks, no matter who or what
made it. This is the piece that actually enforces the process; everything else is
guidance that makes following it easy.

## Why all of this

The short version: context that lives next to the code and stays accurate is worth
far more than docs that drift. The checks stop the context from drifting. The full
reasoning, and the map of where every kind of context lives, is in `AGENTS.md`.
The decisions behind the setup are in `docs/adr/`.
