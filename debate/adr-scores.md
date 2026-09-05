# Scores — what is the domain layer made of?

Architecture fixed (TypeScript, Postgres SoR, domain layer as single interpretation, GraphQL+MCP,
Temporal, Kafka out, amendments A1-A3). Weights: reviewer 3, agent 3, simplicity 2, performance 2.

## Round 1 — Show the domain layer

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| **thin** | 4 | 5 | 4 | 5 | **45** |
| nestjs | 2 | 3 | 2 | 3 | **25** |
| effect | 3 | 2 | 2 | 1 | **21** |
| codegen | 1 | 3 | 1 | 2 | **18** |

**reviewer** — nestjs 2: the same singleton is real, but it concedes guards fail open silently — A2 saves it, not Nest. thin 4: the check is a bare `if` in 440 total lines, the smallest surface to spot an absence in. effect 3: compiles out an omitted Layer at boot, but a bypassing raw query still typechecks. **codegen 1: a 10:1 generated-to-hand surface it admits reviewers can't read — a strong CI net, not visibility.**

**agent** — nestjs 3: fast enough, but request-scoped DI adds cold-start tax; lint-backed, not compiler-backed. **thin 5: order-of-magnitude loops, `register()` structurally blocks inline logic, best corpus.** effect 2: slowest loop (35s typecheck), thinnest corpus, cryptic `R` errors it admits itself. codegen 3: clean tool-add flow, but `custom: true` reopens a second interpretation and the spec has no corpus.

**simplicity** — nestjs 2: 6 claimed hides the DI container, CASL, request scoping and SWC — the real count is near 10. thin 4: 4 undercounts (Fastify, Zod, lint rule, contract test, context module) but is still the smallest true footprint. effect 2: an honest 6 concepts, but 2,300 lines and the slowest `tsc` make them the densest to hold. **codegen 1: 140 concepts is self-admitted — no one person holds that.**

**performance** — nestjs 3: best MCP dispatch (9ms) but a 1.6s cold start repeats on every ROSA scale event. **thin 5: wins every metric — 38ms p99, 85MB RSS, 210ms cold start, fastest agent-facing MCP call.** **effect 1: 180ms p99 contradicts its own "no runtime tax" — that gap IS the tax, plus +25ms MCP dispatch.** codegen 2: decent p99 and RSS, but 900ms cold start and the worst MCP p99.

**Standing after R1: thin 45, nestjs 25, effect 21, codegen 18.**
