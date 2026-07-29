# `task-recurrence` — the closed rule vocabulary and its civil-date arithmetic

Pins the next date every client must compute for a recurring task.

- **Reference implementation:** `dialect/src/recurrence.ts` (vocabulary), `dialect/src/civilDate.ts` (arithmetic) — main `myne` repo
- **Spec anchor:** `markdown-dialect.md` §3.8.7
- **Runner:** `tests/dialect/taskVectors.test.ts` (`pnpm test`) — TS-side, no Rust generator
- **Generator:** `GEN_TASK_VECTORS=1 pnpm vitest run tests/dialect/_genTaskVectors.test.ts`

## What this category pins

A `repeat` field carries a rule from a **closed** three-tier English vocabulary:
plain intervals (`every 3 days`), weekday sets (`every monday and friday`), and
calendar positions in a month (`last friday of every month`, `day 31 of every
month`). Closedness is the point — a general implementation of a recurrence
standard is an *evaluator*, and Myne refuses to ship one.

Given a rule, a current date, and the anchor field(s), a case pins the next
occurrence. Disagreement here means two clients showing a user two different next
dates for the same task, which is worse than either being wrong on its own.

## Case shape

```
cases[]:
  vector_id                  "task-recurrence/v0.1/<slug>"
  case_kind                  "advance" | "unrecognized"
  rule                       the expression, exactly as the file carries it
  expected_rule_recognized   whether the closed vocabulary reads it
  expected_canonical_rule    the rule re-rendered canonically   (advance only)
  expected_mode              "due" | "completion"               (advance only)
  expected_family            "interval" | "calendar"            (advance only)
  today                      the current civil date
  due / remind               the anchor field(s) before advancing
  expected_due / expected_remind   after advancing
```

## The four properties worth stating outright

**1. Arithmetic is over civil year/month/day fields anchored to UTC.** No local
instant is ever formed, so nothing can land in a DST gap or a repeated hour. A
port that builds a local `Date`/`Calendar` instant and adds milliseconds will
disagree on the transition cases — that is exactly what these vectors are for.

**2. Two families, with deliberately different month-end behaviour.** Interval
rules count from the previous date and clamp, so they drift: `every month` from
31 January gives 28 February and then **28 March**. Calendar rules are anchored
and do not: `day 31 of every month` gives 28 February and then **31 March**. Both
halves of each pair are separate cases, because pinning only the clamp would let a
port agree on February and diverge in March. The user chooses which behaviour they
want by choosing the rule; nothing is inferred and no hidden anchor is remembered.

**3. Whole intervals until strictly after today.** A stale rule does not produce
one already-overdue occurrence, and does not produce a thousand catch-up
occurrences either. `advance-stale-yields-one-occurrence` pins a rule three years
stale resolving to a single date, and the runner additionally asserts across the
whole corpus that no result ever lands on or before `today`.

**4. The reminder's gap is carried, not recomputed.** With both fields present the
due date is the anchor and the reminder shifts by the **same number of days**.
`reminder-gap-survives-a-clamp` is the case that proves it: 31 January to 28
February is a 28-day step, so the reminder moves 28 days too. A port that
recomputes "three days before" instead will pass the simple case and fail this one.

## The load-bearing negative

An expression outside the vocabulary must advance **nothing**. The `unrecognized`
cases carry `expected_rule_recognized: false` and **no** expected date. A port that
falls back to a default rule repeats a task the user never asked to repeat; a port
that silently treats the task as non-recurring produces a data-loss-shaped
surprise, where the user believes it repeats, completes it, and finds out months
later that nothing was ever scheduled. Both are failures of the same case.

## How to consume

Parse `rule`. If your parser rejects it, assert `expected_rule_recognized` is
false and that no expected date is present. Otherwise assert the canonical
re-render, the mode, and the family match; then advance the given anchor(s)
against `today` and assert the resulting dates equal `expected_due` /
`expected_remind` exactly.

## What this does NOT cover

The field **syntax** that carries the rule (`[repeat:: …]` versus `🔁 …`) is the
separate `task-inline-format` category. The rule's **localized display** is a
per-client presentation concern and is deliberately not pinned: what must agree
across implementations is the bytes in the file and the date they resolve to.
