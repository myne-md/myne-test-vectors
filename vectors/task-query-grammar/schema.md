# `task-query-grammar` — the closed query roster, and what it refuses

Pins which task-query clauses every client accepts, which it drops, and which
queries it refuses outright.

- **Reference implementation:** `src/notes/taskQuery.ts` (main `myne` repo)
- **Spec anchor:** `markdown-dialect.md` §3.9
- **Runner:** `tests/dialect/taskVectors.test.ts` (`pnpm test`) — TS-side, no Rust generator
- **Generator:** `GEN_TASK_VECTORS=1 pnpm vitest run tests/dialect/_genTaskVectors.test.ts`

## What this category pins

A ```` ```tasks ```` block carries a **closed** keyword roster. The vectors pin
both halves of that claim, and the second half is the one that matters:

- **what is accepted** — one `parse` case per roster clause;
- **what is refused, and why** — `drop` cases carrying a stable reason.

A parse-only vector set would still pass for a port that *also* accepted
`group by function`. The drop cases are what make such a port fail.

## Case shape

```
cases[]:
  vector_id              "task-query-grammar/v0.1/<slug>"
  case_kind              "parse" | "drop" | "refuse"
  query                  the block body, verbatim (may be empty)
  expected_clause_kinds  [ "done", "dueCompare", … ] in source order
  expected_dropped       [ { text, reason } ]  — text VERBATIM
  expected_refused       boolean
```

## The three things a port most easily gets wrong

**1. A field name may never appear where a value goes.** That one sentence is the
grammar's boundary. `in 3 days` is a *parameterized constant* and is accepted —
the same shape the recurrence vocabulary already admits. `due before scheduled`
reads a field where a value belongs and is refused; `due + 3d` reads a field and
operates on it. Once either is representable, so is comparing two fields, and the
grammar needs an evaluator. `drop-field-in-value-position` and `drop-arithmetic`
pin this.

**2. All three function-valued forms are refused**, not just the filtering one.
The upstream grammar has `filter by function`, `sort by function` and
`group by function`; naming one and silently accepting the others would break the
closedness claim outright, so each has its own case.

**3. An all-dropped query REFUSES rather than widening.** A query whose clauses
were all unreadable must not collapse to "match everything": a misunderstood
query rendering the entire vault looks like a *correct enormous result*, which is
the worst failure available here. A genuinely empty query — no clauses written at
all — is a different case and is legitimately unfiltered. `refuse-all-dropped`
and `empty-is-unfiltered` pin the distinction, and it is a real one: both produce
zero clauses, and only one is `refused`.

## Totality

The parser **never throws** and **never silently swallows**. Every unreadable
clause produces a warning carrying the source text verbatim, and the readable
clauses still produce a working query. A port that throws on an unknown clause,
or that drops one without saying so, disagrees with this category even when it
computes the same rows.

## How to consume

Parse `query`. Assert the clause kinds you produced equal
`expected_clause_kinds` in order, that your warnings equal `expected_dropped`
(text and reason), and that your refusal flag equals `expected_refused`.

## What this does NOT cover

**Evaluation** — which tasks a parsed query matches — is out of scope here; that
depends on a vault. The date arithmetic the comparisons resolve against is pinned
by `task-recurrence`, and the field syntax the tasks themselves carry by
`task-inline-format`.
