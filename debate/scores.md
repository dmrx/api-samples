# Scores — a stack for agentic development, 2030+

Weights: reviewer 3, agent 3, simplicity 2, performance 2. Max weighted = 50 per round.

## Round 1 — Opening

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 1 | 2 | 1 | 2 | **15** |
| go | 5 | 5 | 4 | 4 | **46** |
| rust | 3 | 4 | 3 | 3 | **33** |
| deno | 4 | 3 | 5 | 2 | **35** |

**reviewer**
- incumbent 1 — Module Federation runtime wiring, Spring DI, decorators — the diff alone can't show behaviour.
- go 5 — Stdlib only, no DI/macros/bundler; source is the whole story, nothing to expand.
- rust 3 — Compiler helps but derive/Askama macros generate unread code between diff and binary.
- deno 4 — No build step, single language both tiers; npm-compat shims are the only trust gap.

**agent**
- incumbent 2 — Huge corpus but Maven+Angular+webpack slows the loop; DI errors surface at runtime, not compile.
- go 5 — 2s builds, file:line errors, stdlib forecloses choices — near-identical diffs from ten agents.
- rust 4 — Compiler rejects bugs before review, but 45s incremental builds and macro codegen slow the loop.
- deno 3 — Zero build step is fast, but no-framework TS invites divergent idioms across agents at scale.

**simplicity**
- incumbent 1 — Three languages, three runtimes, MF manifests, DI containers — nobody holds it all.
- go 4 — One language, stdlib only, no build step; islands add a sliver of JS.
- rust 3 — One language but derive/proc-macros hide codegen; lifetimes add real head-load.
- deno 5 — One language everywhere, zero build step, diff is the binary — fewest concepts.

**performance**
- incumbent 2 — Only "40K req/s" — unsourced, implausible for stock Spring, no latency/memory/cold-start.
- go 4 — Most complete: sub-ms p50, 15MB RSS, <50ms cold start — though unsourced.
- rust 3 — 8ms p99 with no p50 is odd; RSS identical to Go's — suspicious, no throughput.
- deno 2 — Cold start and binary size only; 90MB binary isn't runtime memory, no latency at all.

**Standing after R1: go 46, deno 35, rust 33, incumbent 15.**

## Round 2 — The whole thing a human must review

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 1 | 1 | 1 | 3 | **14** |
| go | 4 | 5 | 4 | 5 | **45** |
| rust | 2 | 3 | 2 | 2 | **23** |
| deno | 5 | 4 | 5 | 4 | **45** |

**reviewer**
- incumbent 1 — 350 opaque lines plus Kafka and zone.js, 11 files, worst trust burden by far.
- go 4 — Zero generated code, 340 lines, 7 files — but two languages/renderers and a Kafka hop.
- rust 2 — Honest 2,180-line `cargo expand` disclosure hurts it most, though the output is mechanical.
- deno 5 — Zero trusted lines, 195 total, no build step, simplest disclosed path — no broker at all.

**agent**
- incumbent 1 — Save-to-red 18s+ vs seconds elsewhere; template errors surface at runtime, not compile.
- go 5 — 0.9s loop, 0 generated lines, stdlib forecloses alternatives — strongest uniformity claim yet.
- rust 3 — 13s real save-to-red; `cargo expand` exposes 2,180 generated lines behind the "no codegen" claim.
- deno 4 — Fastest loop (0.4s, zero build) but a lint-enforced golden path is convention, not language-level uniformity.

**simplicity**
- incumbent 1 — Honestly reports 15 including Kafka — real sprawl, not hidden, but still worst raw count.
- go 4 — 9 checks out; Kafka and both rendering models disclosed forthright.
- rust 2 — Omits Kafka/Debezium from its own list; real total nearer 15-16, not the claimed 13.
- deno 5 — The only stack that actually deletes the broker rather than hiding it; 7 checks out clean.

**performance**
- incumbent 3 — Honest 40K retraction, real JVM numbers, but vague "TechEmpower-class" sourcing.
- go 5 — Best sourced: `wrk -c200`, pod spec named, numbers match its own stated load exactly.
- rust 2 — k6 500rps is far lighter than go's 200 concurrent conns — flattering numbers under a softer load.
- deno 4 — No tool named, but volunteering 3-8ms GC jitter unprompted is the round's most honest disclosure.

**Running total after R2: go 91, deno 80, rust 56, incumbent 29.**

## Round 3 — Cross-exam

**reviewer**
- incumbent 2 — Concedes bean overrides leave "no line of code to blame" — the worst kind of unreviewable trust.
- go 5 — Typed CDC decode names the exact failing field; rendering-model split resolved as fixed, nothing hidden.
- rust 3 — Deterministic-from-struct is a real distinction from autoconfig, but the 2,180 lines still aren't in any diff.
- deno 2 — Dodged the dedupe question outright, then admitted an untested `any` cast ships silently past both checks.

**agent**
- incumbent 1 — No defence offered; autoconfig trust compounds unaudited across a decade of upgrades.
- go 4 — Tight loop intact; two-model charge resolved by fixed convention, not compiler enforcement.
- rust 3 — Compiler-enforced uniformity is real, but conceded a 51s cold build every sandbox cycle.
- deno 2 — Admits uniformity is convention-only — one ungrepped agent forks a pattern, only lint/review catch it.

**simplicity**
- incumbent 1 — Admits the Node BFF is a whole unearned layer atop Kafka/Postgres — pure added concept, zero value.
- go 2 — Concedes a fan-in layer must be added later at scale — same penalty as a layer present now.
- rust 4 — Derive macros defended as mechanical, 1:1 with reviewed structs — no real extra concept added.
- deno 3 — Forked patterns caught only by lint/review, not the compiler — concepts can silently multiply.

**performance**
- incumbent 3 — Honest 3.2s×N-pod tax and BFF-latency concessions, but ducked deno's Hibernate-upgrade p99 question.
- go 5 — Only one to concede a real cap — LISTEN-per-pod low hundreds — plus a concrete typed-decode fix.
- rust 2 — Zero new numbers; retreated to "derives are deterministic," contributing nothing to the scaling question.
- deno 1 — Non-answer to jitter ("replicas don't share GC" ≠ the per-pod p99 charge) and dodged NOTIFY fan-out dedupe.

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 2 | 1 | 1 | 3 | **17** |
| go | 5 | 4 | 2 | 5 | **41** |
| rust | 3 | 3 | 4 | 2 | **30** |
| deno | 2 | 2 | 3 | 1 | **20** |

**Running total after R3: go 132, deno 100, rust 86, incumbent 46.**

## Round 4 — The 2030 stress test

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 1 | 1 | 1 | 1 | **10** |
| go | 3 | 3 | 3 | 4 | **32** |
| rust | 4 | 2 | 2 | 3 | **28** |
| deno | 5 | 4 | 5 | 5 | **47** |

**reviewer**
- incumbent 1 — Three copies, two languages, 350 unread autoconfig lines, no test — infinite time to catch.
- go 3 — Two copies, but 2,000 honest lines are the haystack burying a one-char bug.
- rust 4 — Three copies but only ~50 lines matter; expand's 2,180 lines never touch the diff.
- deno 5 — Two copies, grep finds both in one pass; 1,978 of 2,000 lines are true noise.

**agent**
- incumbent 1 — Infinite red confirmed; three copies, two languages, 350 trusted lines crowd out testing.
- go 3 — 0.9s red is the fastest real number, but verbose ceremony admittedly buries the hunk.
- rust 2 — Three copies again; honest 64s clean-sandbox loop is the slowest real signal.
- deno 4 — 0.4s red, flat grep finds both copies fast, but two copies not one.

**simplicity**
- incumbent 1 — Worst drift: 3 copies, 2 languages, and dodges duplication by blaming autoconfig.
- go 3 — Ties fewest copies (2) but its decade answer ignores duplication, naming untested edges instead.
- rust 2 — 3 copies despite one language; honest decade answer names SQL/template drift directly.
- deno 5 — Fewest copies (2) plus the most honest accumulation answer: duplication is the pattern itself.

**performance**
- incumbent 1 — 95s cold spans three services, and concedes zero red signal ever exists untested.
- go 4 — 0.9s to red if the test exists; cheapest compiled loop, honest about the no-test gap.
- rust 3 — Honestly revised 51s→64s with no sccache; rewarded for honesty, still costliest loop.
- deno 5 — Zero build, 0.4s to red; cheapest feedback economics of all four.

---

## Final tally

| contender | R1 | R2 | R3 | R4 | **total /200** |
|---|---|---|---|---|---|
| **go** | 46 | 45 | 41 | 32 | **164** |
| deno | 35 | 45 | 20 | 47 | **147** |
| rust | 33 | 23 | 30 | 28 | **114** |
| incumbent | 15 | 14 | 17 | 10 | **56** |

### By judge (raw sum across 4 rounds, max 20)

| contender | reviewer (3) | agent (3) | simplicity (2) | performance (2) |
|---|---|---|---|---|
| go | **17** | **17** | 13 | **18** |
| deno | 16 | 13 | **18** | 12 |
| rust | 12 | 12 | 11 | 10 |
| incumbent | 5 | 5 | 4 | 9 |

**Winner: go (164/200)** — first on the two heaviest judges and on performance.
**deno (147)** wins simplicity outright and takes round 4 by a mile; it loses on the agent judge,
where its uniformity is a lint rule rather than the language.
**rust (114)** never recovered from disclosing 2,180 generated lines in round 2 — then argued in
round 4 that they never enter a diff, which reviewer accepted a round too late.
**incumbent (56)** lost every judge in every round and conceded its own Node BFF is unearned.
