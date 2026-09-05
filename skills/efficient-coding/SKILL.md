---
name: efficient-coding
description: Use when the user wants code reviewed for performance — e.g. "why is this slow", "review this for performance", "will this hold up under load", "is this query/list/page efficient". Works the cost dimensions in order of impact and names the dominant cost before proposing anything.
---

# Efficient Coding

## Overview

Review a change — or a named slow path — for what it actually costs, and
report the smallest set of fixes that would change the number that matters.

Efficiency has several dimensions and they are not equal. Working them out of
order is theatre: a carefully memoized component that fires twelve sequential
database round trips is still slow, and now it is harder to read as well. So
the review works them **in order of impact**, names the dominant cost out
loud, and fixes that before touching anything below it.

| # | Dimension | Typical win | Usual smell |
|---|---|---|---|
| 1 | Data access | 10-100x | repeated queries in a loop, waterfalls, over-fetching |
| 2 | Algorithmic | 10-100x at scale | nested scans over the same collection |
| 3 | Render / reactivity | 2-10x felt | re-render storms, state wider than its readers |
| 4 | Bundle / payload | load time | heavy dependencies, wide client boundary |
| 5 | Code volume | maintenance | duplication, abstraction with one caller |

**Announce at start:** "I'm using the efficient-coding skill to review this code for performance cost, worked in order of impact."

## Process Flow

1. **Scope the review.** Ask what to review and what "slow" means here: which
   screen, endpoint, or job, and what the user actually experiences. Ask
   whether anything has been measured. A review of code nobody has timed is a
   review of guesses, and should say so in the doc.
2. **Clarifying questions.** Ask one at a time. How large is the data today,
   and how large in a year? Which paths are hot (every page load) versus cold
   (runs once at startup)? Is there a hard budget — a page-load target, a
   platform timeout, a bill?
3. **Name the dominant cost first.** Before proposing a single fix, state
   which dimension the real cost sits in and why. Everything below that
   dimension waits.
4. **Dimension 1 — data access.** Round trips dominate everything else. One
   query returning many records beats many queries returning one, so any
   fetch inside a loop is the first thing to hoist into a single batched
   query. Independent fetches awaited one after another make the caller wait
   for their *sum* rather than their maximum — run them concurrently. Check
   that every query requests only the fields used, has a bound and an
   explicit order, and that every filtered or sorted field is actually
   indexed; an unindexed filter is a full scan wearing a filter's clothes.
   Check that cache invalidation is as narrow as the write that triggered it.
5. **Dimension 2 — algorithmic.** Only worth attention where the collection
   can grow, but nearly free to get right while writing. Nested loops over
   the same collection are usually a lookup table nobody built yet; repeated
   linear searches inside a render or a request handler are the recurring
   case. State the expected size when judging: quadratic over five items is
   correct and saying so stops someone "fixing" it later.
6. **Dimension 3 — render and reactivity.** A freshly built object, array, or
   function handed down as a prop is a new value every time, which defeats
   any memoization below it — worth fixing where the child is expensive or
   the list is long, and not worth it anywhere else. Check that shared state
   is no wider than the set of readers that care about it, that long or
   growing lists render a window rather than every row, and that list keys
   are stable rather than positional.
7. **Dimension 4 — bundle and payload.** Where the client boundary sits
   decides what ships: everything below it, plus everything it imports. Push
   the boundary as far down as possible and keep data fetching and heavy
   dependencies above it. Weigh any new dependency against what it costs
   installed and against the few lines of platform code that may cover the
   real need — and prefer something already in the tree over something new.
   Data crossing a server-to-client boundary is a network cost too; passing a
   whole record to display one field ships the whole record, including fields
   the viewer should not have.
8. **Dimension 5 — code volume.** Fewest moving parts that solve the actual
   problem. Reuse what the codebase already has before adding a helper; a
   duplicate utility two directories over is the most common form of waste.
9. **Say what not to optimize.** Setup code that runs once, cold paths, and
   micro-optimizations that trade readability for nothing measurable get one
   line each in the doc, under a heading that says they were considered and
   dismissed. This is what keeps the recommendations trustworthy.
10. **State effects in real units, and never invent numbers.** Round trips
    removed, renders avoided, bytes not shipped, scans replaced by lookups.
    Where a change is worth measuring, say precisely what to measure and
    where, rather than estimating a speedup nobody can check.
11. **Write the doc** to
    `docs/performance/YYYY-MM-DD-<scope-slug>-efficiency-review.md` in the
    user's project (create `docs/performance/` if it doesn't exist).
12. **Self-review.** Check that the dominant cost is actually named, that no
    fix in a lower dimension is recommended while a higher one is open, and
    that every claimed cost traces to code actually read rather than a
    pattern assumed to be present.
13. **User review gate.** Point the user to the file and ask them to review
    before treating it as final. Record any deliberate simplification with
    its ceiling ("linear scan, fine below a few hundred rows") so the next
    reader knows it was a choice rather than an oversight.

## Output Template

```markdown
# <Scope> — Efficiency Review

> Generated by the efficient-coding skill on <date>. Review and edit before treating this as final.

## 1. Scope Reviewed & What "Slow" Means Here
## 2. Dominant Cost (which dimension, and why)
## 3. Findings by Dimension (in order of impact)
## 4. Considered and Deliberately Not Changed
## 5. What to Measure
## 6. Not Reviewed / Low Confidence
```
