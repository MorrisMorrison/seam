# seam

A Claude Code skill that splits implementation work between you and the
agent: you write the code that is dangerous or that you would regret not
understanding, the agent writes everything else, and both proceed in
parallel.

## Why

AI-assisted development produces comprehension rot — the riskiest code in a
project is often code its author has only read — and attention collapse,
because the wait around an agent run has no shape. seam addresses both by
moving your attention onto real work on the critical path instead of into
the gaps around it.

## How it works

1. **Triage** — every task in a plan is scored on blast radius and
   comprehension debt, then assigned. Nothing starts until you approve the
   manifest.
2. **Seam** — for your units the agent writes a stub, failing tests, and the
   surrounding wiring, then asks whether the tests are the spec you want.
3. **Parallel** — it works its tasks, never touching a file you own. Out of
   work, it stops and reports.
4. **Stall ladder** — hint, bigger hint, or a draft to reshape, offered once.
   Full handoff exists and is yours to invoke.
5. **Integration** — full suite, then reef with hunks labeled by author.

## Install

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\seam" -Target "$env:USERPROFILE\projects\seam\skills\seam"
```

Then invoke with `/seam` in any repo that has an implementation plan.

## Tests

See `tests/README.md`. seam is a markdown protocol, so it is verified by
application tests in fresh sessions rather than unit tests.

## Docs

- Design: `docs/specs/2026-08-08-seam-design.md`
- Plan: `docs/plans/2026-08-08-seam-implementation.md`
