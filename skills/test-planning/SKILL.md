---
name: test-planning
description: Use when the user wants to know what to test before shipping — e.g. "what tests do I need", "what should I test here", "is this covered", "what's the minimum test suite for this". Starts from what would fail silently and expensively rather than from coverage, and picks the cheapest tier that can actually catch each risk.
---

# Test Planning

## Overview

Decide which tests must exist before a change ships, at which level, and which
are not worth writing — then hand back a list short enough that it actually
gets written.

The goal is not coverage. It is the handful of tests that would genuinely have
caught the things most likely to go wrong, each written at the lowest level
that can still fail for the right reason. Most changes deserve two or three
tests, not twelve, and a plan that says what it is *not* testing is worth more
than one that quietly implies everything is covered.

| Tier | Catches | Cost |
|---|---|---|
| 0 | Types, lint, exhaustiveness — shape errors | free |
| 1 | Pure logic with no I/O — calculations, transforms, state machines | seconds |
| 2 | Integration against real infrastructure — access rules, constraints, triggers, migrations | needs a real environment |
| 3 | End-to-end or manual — multi-device flows, native behaviour, real timing | slow, human |

**Announce at start:** "I'm using the test-planning skill to decide what needs testing before this ships."

## Process Flow

1. **Scope the plan.** Establish what changed and what test infrastructure
   exists today — is there a runner, is there a way to run anything at Tier 2
   against real infrastructure, does anything run in CI? A plan that assumes
   tiers the project does not have is not a plan.
2. **Clarifying questions.** Ask one at a time. What would be most damaging
   if it were quietly wrong? Is there an environment safe to test against
   with real infrastructure? Who runs the suite, and when — every commit, or
   never?
3. **Start from the risk, not from the code.** Do not walk the diff
   enumerating functions. Ask what could go wrong and sort by **cost times
   silence**: what would be both damaging and silent — data written wrong,
   the wrong person seeing something, history corrupted — comes first,
   always. Then subtle logic with real branching: boundaries, empty inputs,
   ordering, contradictory states. Anything already guaranteed by a type, a
   database constraint, or the framework gets **no test** — an assertion that
   the compiler already enforces costs maintenance and buys nothing.
4. **Pick the lowest tier that can still fail correctly.** Push each check
   down to the cheapest tier that can catch it, and then stop. The rule that
   decides the tier: **if mocking a dependency removes the failure mode, the
   test must not mock that dependency.** A unit test with a substituted data
   layer cannot catch an access-control bug, because the substitute returns
   whatever its author expected — which is precisely the assumption under
   test. Access rules, constraints, and triggers are only ever provable at
   Tier 2, against real infrastructure; a green Tier 1 suite over that
   surface is worse than no tests, because it manufactures confidence.
5. **Push back the other way too.** If logic needs a whole device or browser
   to exercise, that is usually a signal the logic should be lifted out of
   the component into something testable on its own. Propose the extraction
   as part of the plan rather than writing an expensive test around a shape
   that does not need to be that shape.
6. **Write assertions that actually discriminate.** For each planned test:
   - **Both directions, every time.** "B cannot read A's data" is half a
     test — a rule that denies everyone passes it. Pair it with "A can read
     A's data". This applies to any allow/deny, visible/hidden,
     enabled/disabled rule.
   - **Test the bypass, not the happy path.** For anything guarding a
     boundary, the valuable assertions are the workarounds: reach the data by
     another route, change the value the rule depends on, flip the state that
     unlocks it.
   - **Assert on the outcome, not the call.** Checking that a function was
     invoked tests the wiring; checking the resulting state tests the
     behaviour.
   - **Prove the guard can fail.** A guard exercised only against empty input
     proves nothing, because doing nothing passes. Set up the state the guard
     must reject, then assert it was rejected.
   - **Name each test as the behaviour it protects**, so a future failure
     reads as a sentence about the product rather than a function name.
7. **Say what is deliberately not being tested, and why.** An explicit "not
   worth a test, here is what covers it instead — a type, a constraint, an
   existing higher-tier test" is part of the plan, and it stops the same
   question being reopened at every review.
8. **Handle a missing test setup as a finding, not a task.** If the project
   has no runner and the change needs a test, report that absence and propose
   a concrete minimal setup: the runner, the configuration, the narrowest
   scope that works, mirroring any convention a sibling project already uses
   rather than inventing one. Then **ask before creating any files** —
   adding a runner has lock-file and CI consequences and is a project-level
   decision, not a side effect of a feature change.
9. **Split the list in two.** **Must exist before this ships** and **worth
   adding later**. The first list has to be short enough to actually get
   written; if it has grown past a handful, the ranking in step 3 was not
   applied honestly.
10. **Name the gaps plainly.** If a risk cannot be covered at any tier the
    project has, say so and name the manual verification that substitutes for
    it. An honest gap beats a test that looks like coverage and is not.
11. **Write the doc** to
    `docs/test-plan/YYYY-MM-DD-<scope-slug>-test-plan.md` in the user's
    project (create `docs/test-plan/` if it doesn't exist).
12. **Self-review.** Check that no planned test mocks away the thing it
    exists to catch, that every allow/deny rule is asserted in both
    directions, and that nothing on the must-exist list is already guaranteed
    by a type or a constraint.
13. **User review gate.** Point the user to the file and ask them to review
    before treating it as final. This skill plans tests; writing them is the
    next step, in whatever testing workflow the project already uses.

## Output Template

```markdown
# <Scope> — Test Plan

> Generated by the test-planning skill on <date>. Review and edit before treating this as final.

## 1. Change Being Planned For
## 2. Risks, Sorted by Cost × Silence
## 3. Must Exist Before This Ships

| Test | Tier | Behaviour it protects | Key assertion |
|---|---|---|---|

## 4. Worth Adding Later
## 5. Deliberately Not Tested, and What Covers It Instead
## 6. Gaps No Available Tier Covers (and the manual check that substitutes)
## 7. Test Infrastructure Needed — Requires a Decision
```
