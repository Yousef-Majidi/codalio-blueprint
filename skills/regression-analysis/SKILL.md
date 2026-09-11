---
name: regression-analysis
description: Use when the user wants to know what a change could break before they merge or ship it — e.g. "what could this break", "is this safe to deploy", "regression risk of this refactor/migration/API change", "what depends on this". Traces the change's real blast radius through its consumers and ranks findings by how silently they would fail.
---

# Regression Analysis

## Overview

Given a change, determine what it could break elsewhere, and report each risk
with a concrete failure scenario and an honest answer to "what would catch
this today?"

A regression is not a bug in the code that was written. It is a bug in code
nobody opened, caused by an assumption nobody knew was being made. Re-reading
the diff cannot find that — the diff is where the analysis starts, not what it
covers. The work is finding everything that depended on the old behaviour, and
then working out which of those failures would announce themselves and which
would not.

**Announce at start:** "I'm using the regression-analysis skill to trace what this change could break."

## Process Flow

1. **Scope the change.** Establish exactly what changed — a diff, a branch, a
   commit range, or a described plan not yet written. Ask where it deploys:
   one codebase the team controls end to end, or a backend with clients that
   update on their own schedule.
2. **Clarifying questions.** Ask one at a time. Does any other repository,
   client, or job read this same data or contract? Are there released
   versions already in users' hands? Is anything persisted, cached, or
   queued in a shape this change touches?
3. **Check what the change operation itself forbids or costs.** Before
   tracing a single consumer, check the mechanics of the operation being
   run. Schema and migration primitives in particular carry restrictions that
   fail on first apply no matter how sound the design is: what cannot run
   inside a transaction, what cannot be reversed once applied, what rewrites
   an entire table and holds a lock while it does, what value an existing row
   receives and in what order. The surrounding code often documents these in
   a comment explaining why an earlier change was made a different way — read
   it. A flawless consumer analysis is worthless if the operation cannot be
   applied.
4. **Establish the real surface.** For every symbol, column, route, key, or
   configuration value the change touches, enumerate its consumers before
   reasoning about risk. Search for the name, not the file — a re-export
   means the real import site is somewhere else entirely. For a stored field:
   every query, every access rule that reads it, every trigger, every
   generated type, every serialized copy. For a shared helper: every caller,
   and what each one assumes about empty, null, ordering, and error
   behaviour — callers rarely assume the same things. For a UI component:
   every render site, including conditional ones. Search sibling repositories
   too when the change touches a shared schema or contract; two clients on
   one database is the most common place for a regression to land outside the
   repository being edited. **If the consumers were not enumerated, the
   analysis is guesswork and the doc must say so.**
5. **Name the invariants at risk.** List what the surrounding system holds
   true and this change could quietly unhold, in two groups. *Enforced* —
   types, constraints, unique indexes, foreign keys, triggers, tests. These
   break loudly, which is the good case. *Assumed* — anything true only by
   convention: this list is sorted, this table is append-only, this is never
   empty after setup, only one caller reaches this, these two fields never
   contradict. These live in comments, in naming, or in nobody's head, and
   they are where regressions come from. Read the comments around the changed
   code: a comment explaining *why* something is the way it is is an
   invariant with a warning label, and changing that code without addressing
   the comment is the highest-yield place to look.
6. **Work the recurring classes.** For each, either name the finding or state
   it does not apply:
   - **Clients already running.** Software in a user's hands does not update
     when the backend does. Removing a field, narrowing a type, renaming a
     value in an enumeration, or adding a required field with no default
     breaks every copy already in the field.
   - **Silent authorization drift.** A changed access rule returns *fewer
     records*, not an error. The screen renders empty and looks fine. Check
     both directions — what is now hidden that should be visible, and what is
     now visible that should be hidden.
   - **Widened inputs.** A new status, role, or variant that existing branch
     logic does not handle. Find every exhaustive match over that type.
   - **Shape drift across a validation boundary.** A hand-written type and a
     runtime schema that no longer agree still compile, and so does a
     generated type that was never regenerated.
   - **Serialization boundaries.** Persisted state, cached payloads, and
     stored snapshots outlive the code that wrote them; old values must still
     parse.
   - **Cache and invalidation scope.** Narrowing invalidation leaves stale
     screens; widening it only costs performance. Check which way this went.
   - **Error and empty paths.** A changed throw-versus-return-nothing
     convention breaks every caller that branched on the old one, and only on
     the path nobody exercises.
   - **Ordering and identity.** Removing a sort, or changing what an
     identifier is derived from, reorders and remounts things downstream.
7. **Rank by detectability, not by severity alone.** This is what makes the
   analysis worth writing. A crash at startup is not a real risk — it is
   found in thirty seconds. Rank by how long a failure survives undetected:
   (1) silent and permanent — wrong data written, confidentiality lost,
   history corrupted; never self-reveals; (2) silent and conditional — wrong
   only for one role, one device, one locale, one stale client; found by a
   user weeks later and not reproducible; (3) visibly broken at runtime — bad
   but findable; (4) caught by the build, the types, or an existing test —
   mention and move on. For every finding, say what would catch it today, and
   if the honest answer is "nothing", that is the finding.
8. **Write a concrete failure scenario for every finding.** Specific inputs
   or state leading to a specific wrong outcome. "This could cause issues" is
   not a finding; if the scenario cannot be written, its existence has not
   been confirmed — move it to the suspected list rather than dropping it.
   Keep what was confirmed by reading the consuming code separate from what
   is suspected but unverified. Both are worth reporting; conflating them is
   not.
9. **Write the doc** to
   `docs/regression/YYYY-MM-DD-<scope-slug>-regression-analysis.md` in the
   user's project (create `docs/regression/` if it doesn't exist).
10. **Self-review.** Check that consumers were actually enumerated rather
    than assumed, that every ranked finding has a scenario, and that the
    detectability rank is honest — the temptation is to rank by how alarming
    a bug sounds rather than by how long it would hide.
11. **User review gate.** Point the user to the file and ask them to review
    before treating it as final. Close the doc with the shortest possible
    list of things to verify before shipping — short enough that it actually
    gets done.

## Output Template

```markdown
# <Scope> — Regression Analysis

> Generated by the regression-analysis skill on <date>. Review and edit before treating this as final.

## 1. Change Under Analysis
## 2. Constraints of the Change Operation Itself
## 3. Consumers Enumerated (and anything not reachable)
## 4. Invariants at Risk (enforced vs assumed)
## 5. Findings, Ranked by Detectability
## 6. Suspected but Unverified
## 7. Verify Before Shipping
```
