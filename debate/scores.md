# Scores — a stack for building a CRM with agents, 2030+

Weights: reviewer 3, agent 3, simplicity 2, performance 2. Max weighted = 50 per round.

## Round 1 — Opening

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 3 | 3 | 2 | 1 | **24** |
| go | 5 | 5 | 4 | 1 | **40** |
| rust | 4 | 4 | 3 | 2 | **34** |
| deno | 2 | 3 | 5 | 1 | **27** |

**reviewer**
- incumbent 3 — Concentrated `@PreAuthorize` is auditable in 40 files, but enforcement isn't visible in the screen diff.
- go 5 — Every screen's permission check is inline and greppable — no generator, no trust required, a breach is visible.
- rust 4 — Compile-time reachability beats human vigilance, but macro expansion itself is unauditable magic.
- deno 2 — Hand-written but admits enforcement leans on a lint rule — weakest guarantee a breach gets caught.

**agent**
- incumbent 3 — Uniform by generator, not by agents; three-toolchain loop slows the write-run-fix cycle badly.
- go 5 — Fast loop, huge corpus, gofmt-enforced idioms make ten agents converge — hand-writing costs an agent nothing.
- rust 4 — Best schema-drift propagation (sqlx compile errors) and clearest errors, but 51s+ builds tax every iteration.
- deno 3 — Fastest loop, no build step, but self-admits uniformity rests on lint not compiler — real risk at 120 screens.

**simplicity**
- incumbent 2 — Three runtimes plus builder + annotations + Envers — abstraction cost claimed free, never priced.
- go 4 — One language, zero hidden abstraction; pays honestly in volume, not in extra concepts to hold.
- rust 3 — Types kill duplication but tax every reviewer with traits, derive macros, 51s+ builds.
- deno 5 — Single language end to end, no DTO layer, and it names its own weak spot outright.

**performance**
- incumbent 1 — Zero numbers. Envers/derived-query claims, no p50/p99, memory, cold start, or query time.
- go 1 — Zero numbers. Grep and file-count claims only, nothing measurable about list/report queries.
- rust 2 — Only concrete figure all round (51s+ builds), and that's compile time, not runtime.
- deno 1 — Zero numbers. Zero-build-step claimed as a virtue, no latency/memory/query data.

**Standing after R1: go 40, rust 34, deno 27, incumbent 24.**

## Round 2 — Price the whole CRM

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 3 | 3 | 3 | 2 | **28** |
| go | 2 | 3 | 2 | 4 | **27** |
| rust | 1 | 2 | 2 | 4 | **21** |
| deno | 5 | 5 | 4 | 5 | **48** |

**reviewer**
- incumbent 3 — 9.6k lines but the rule is split across JSON + annotation + builder; three places, two languages.
- go 2 — Greppable in 120 files, but concedes silent empty-render on a missing field — permission check included.
- rust 1 — 222k lines total; a forgotten role-struct field compiles clean and is silently invisible to reps.
- deno 5 — One SQL query audits all 120 screens; the rule is never in code, missing fields fail safe by omission.

**agent**
- incumbent 3 — Uniform JSON-driven screens, huge corpus, but three-toolchain build and QA-checklist-only drift catch.
- go 3 — Sub-second builds and greppable uniformity, but drift touches 18 files and template failure is silent.
- rust 2 — sqlx's loud errors don't offset a 4m10s cold build wrecking the agent loop at CRM scale.
- deno 5 — Fastest loop; drift touches 1 file with generic screens picking it up — best fit for the weekly job.

**simplicity**
- incumbent 3 — One rule, one method, but three runtimes/languages plus RLS layered on top.
- go 2 — Zero abstraction, rule in 120 places — most concepts held, least removed.
- rust 2 — Removes duplication via 120 role-structs, but adds derive-macro combinatorics itself.
- deno 4 — Moved the rule into Postgres, one declaration site — but RLS/grants/SET LOCAL/db.d.ts are new concepts.

**performance**
- incumbent 2 — Honest about matview vs live aggregate, but 1.38GB and 4.1s cold start dwarf every rival.
- go 4 — Best memory, fast cold start; volunteered the unindexed 1.9s GROUP BY, most honest number of the round.
- rust 4 — Strong p99 40ms but slowest cold start of the fast tier; "200rps" isn't "200 concurrent", naming mismatch.
- deno 5 — Best p99 38ms, fastest cold start 40ms, fastest live GROUP BY 640ms.

**Running total after R2: deno 75, go 67, rust 55, incumbent 52.**

## Round 3 — Cross-exam

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 3 | 3 | 2 | 5 | **32** |
| go | 2 | 2 | 1 | 4 | **22** |
| rust | 4 | 2 | 3 | 1 | **26** |
| deno | 5 | 4 | 4 | 3 | **41** |

**reviewer**
- incumbent 3 — Concrete numbers, but ArchUnit only flags missing annotations; the JSON gap is caught by QA, not CI.
- go 2 — Twice conceded fail-open on the exact breach axis, then pitched an AST lint that doesn't exist yet.
- rust 4 — Landed fail-closed vs fail-open decisively; a real safety distinction, though its own CI gate is unbuilt too.
- deno 5 — Beat two coordinated attacks with real mechanisms, then honestly conceded two gaps without oversold fixes.

**agent**
- incumbent 3 — ArchUnit bytecode rule is real and simple, but the same "QA not CI" gap as go's JSON drift.
- go 2 — Needs a bespoke AST linter built from scratch, plus a loose 20-year claim dents credibility.
- rust 2 — Nightly diff is simple, but 40 structs and a full human day per new role kills loop tightness.
- deno 4 — Single-file SQL grant is the simplest check by far, even after conceding two real CI gaps.

**simplicity**
- incumbent 2 — Three runtimes plus two new tools: ArchUnit bytecode scan and 120 golden-screenshot tests.
- go 1 — Invents a whole new AST-parsing linter from scratch just to patch its own silent-fail hole.
- rust 3 — One more nightly diff job, but stacked on 120 structs and 180k already-generated lines.
- deno 4 — One mechanism (grants + RLS) for reads and writes; gaps are a trigger and a CI diff, not new tools.

**performance**
- incumbent 5 — Only decade-scale number given: Envers ~20GB vs 5GB, 25min restore, plus rollback/CI timings.
- go 4 — Concrete p99 ~150ms at 5M rows by routing reports around RLS via an indexed materialized view.
- rust 1 — Gave zero performance numbers this round — pure silence on the metric.
- deno 3 — Query-plan pushdown claim is technically sound but no concrete p99 for the actual 5-table join.

**Running total after R3: deno 116, go 89, incumbent 84, rust 81.**

## Round 4 — The CRM breach test

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 2 | 2 | 2 | 4 | **24** |
| go | 1 | 1 | 1 | 2 | **12** |
| rust | 3 | 3 | 3 | 3 | **30** |
| deno | 5 | 5 | 5 | 3 | **46** |

**reviewer**
- incumbent 2 — 14 lines, but a same-shaped lie hides among 30 correct sibling edits.
- go 1 — 200-300 lines of absence; a diff cannot show a missing wrapper call.
- rust 3 — ~40 lines, visible added field, but only if the reviewer knows the pattern.
- deno 5 — One line, no screen-side rule to audit — smallest true review surface.

**agent**
- incumbent 2 — Moderate nightly grant-diff needed from scratch; ArchUnit exists but doesn't check correctness.
- go 1 — Bespoke AST parser must be built and maintained forever just to catch one bare template field.
- rust 3 — Extends the existing nightly diff to flag extras too — moderate, but 4m10s cold builds slow the loop.
- deno 5 — One trivial SQL query against a checked-in file, 0s build, covers all 120 screens at once.

**simplicity**
- incumbent 2 — Duplication trap: rule spans ~14 JSON files, trusted twice, plus a 1,400-line builder.
- go 1 — Worst: the rule is presence/absence at ~470 call sites, no single place to check.
- rust 3 — 120 structs, but each field-role fact lives once; greppable, though extras go unchecked.
- deno 5 — One grant, one place, zero screen-side rule; global blast radius but a one-line revert.

**performance**
- incumbent 4 — 6min revert; Envers uniquely proves exactly which partners saw discount, and when.
- go 2 — 2min revert but no field-access log at all; conceded, can't answer the regulator.
- rust 3 — 38s rebuild is fastest recovery, but no access log, only loose request-log bounds.
- deno 3 — Seconds-fast REVOKE, but no grant-history table; can't say who, or since when.

---

## Final tally

| contender | R1 | R2 | R3 | R4 | **total /200** |
|---|---|---|---|---|---|
| **deno** | 27 | 48 | 41 | 46 | **162** |
| rust | 34 | 21 | 26 | 30 | **111** |
| incumbent | 24 | 28 | 32 | 24 | **108** |
| go | 40 | 27 | 22 | 12 | **101** |

### By judge (raw sum across 4 rounds, max 20)

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) |
|---|---|---|---|---|
| deno | **17** | **17** | **18** | **12** |
| rust | 12 | 11 | 11 | 10 |
| incumbent | 11 | 11 | 9 | **12** |
| go | 10 | 11 | 8 | 11 |

**Winner: deno (162/200)** — first with every judge, and the only stack that removed the rule from
the screens instead of repeating it.
**go (101) finishes last**, having won both previous debates. It led round 1 at 40 and fell to 12.
Hand-writing 120 screens is honest at one screen and fatal at 120: 66,000 lines, the rule repeated
at ~470 call sites, and a breach that is an *absence* no diff can show.
