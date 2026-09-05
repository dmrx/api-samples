# Stack debate — a stack for agentic development, 2030+

## The question
Not "what ships this screen." **What stack do you standardize on for the next decade, when
almost all code is written by agents and the scarce resource is human review?**

Judge everything by three things and nothing else:
- **Simplicity** — fewer concepts to hold in one head. Less to reason through, not less to type.
- **Performance** — real latency, memory, throughput numbers. Not benchmarks; per-pod cost.
- **Human reviewability** — one person reads every diff end to end and knows what will happen.

Two things lose automatically:
- **Magic** — anything that acts at a distance: decorators, macros, codegen, runtime wiring,
  DI containers, convention-over-configuration, build output you don't read.
- **Sprawl** — repo count, layer count, language count, dependency count, concept count.

## Principles (12)
1. One screen, one verb, one outcome.
2. If a customer wouldn't say it, keep it but bury it.
3. Simplicity = less to reason through, not less to type.
4. Boring, stable, readable end to end.
5. Server is the sea; JS only as islands.
6. Add a layer only when a user-visible problem demands it.
7. Migrate in tiny daily steps, each deletable in an hour.
8. Code is agent-written; optimize for human review.
9. Tests are the spec.
10. Everything is text in one repo.
11. Small blast radius, reversible in one command.
12. Human writes the why, reads every diff.

## The workload used as a test case
A quotes screen: list, live status over SSE, data from Salesforce mirrored via CDC into
Postgres, on ROSA. It is the *probe*, not the subject. Argue about the stack.

## Contenders
- **incumbent**: Angular MFE + Module Federation, Node BFF, Java Spring core services
- **go**: Go + net/http + html/template + HTMX + Preact/htm islands + Postgres
- **rust**: Rust + Axum + Askama + HTMX + Postgres
- **deno**: Deno or Bun + plain TypeScript, no bundler + HTMX + Postgres

## Judges (weight, total 10 — max 50 per round)
- **reviewer (3)**: can one human read every diff end to end; how much is hidden; how many
  places must they look; what must they take on trust.
- **agent (3)**: tightness of the write-run-fix loop for an AI agent — build time, test time,
  does the error name the exact line and cause, how much of this stack is in the 2030 corpus,
  how uniform is the idiomatic answer.
- **simplicity (2)**: concept count. Layers, languages, repos, indirections. How long to explain
  the whole thing to a new person. Magic is a direct penalty.
- **performance (2)**: p50/p99 latency, memory per pod, cold start, throughput. Real numbers only.

## Style
Everyman-dense. Lead with the answer. No hedges, no preamble, no summary. Concrete numbers.
