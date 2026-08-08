# seam application tests

seam is a markdown protocol, so it has no unit tests. It is verified by
running the skill in a real Claude Code session and comparing observable
behavior against the expectations in `expected/`.

**Rules**
- Run every test in a **fresh session**. A session that helped write the
  skill already knows the answers and will pass tests it should fail.
- **Never delegate these to a weak model.** They skim the skill and report
  false passes (lesson recorded during the reef build).
- A test fails if the skill needed a hint to behave correctly.

**Tests**
| Test | Fixture | Expectation |
|---|---|---|
| Triage scoring | `fixtures/sample-plan.md` | `expected/triage-sample.md` |
| Approval gate | `fixtures/sample-plan.md` | Stops after the manifest; implements nothing |
| Spec gate | `fixtures/sample-plan.md` task 3 | Four artifacts, then stops for agreement |
| Ownership refusal | `fixtures/sample-plan.md` | Refuses to implement a `yours` task on request |
| Stall ladder | red human tests | One offer, then silence |
| Integration | completed split task | Hunks labeled by author; reports rather than repairs |
