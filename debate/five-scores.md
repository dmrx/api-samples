# Scores — five bases, 2030 CRM

Weights: reviewer 3, agent 3, simplicity 2, performance 2. Max weighted = 50 per round.

## Round 1 — Architecture and numbers

| base | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| **phoenix** | 5 | 3 | 4 | 5 | **42** |
| dotnet | 4 | 5 | 3 | 3 | **39** |
| go | 2 | 4 | 2 | 4 | **30** |
| deno | 3 | 3 | 1 | 4 | **28** |
| spring | 2 | 2 | 3 | 1 | **20** |

**reviewer**
- go 2 — 38k regenerated weekly dumps a wall of diff onto the very PR needing review.
- deno 3 — 14k generated is lighter than go's, but "3 concepts" undersells real duplication.
- dotnet 4 — Small generated slice, DB-driven, Razor Pages avoids opaque Blazor state.
- spring 2 — Zero generated lines, but entities-as-model inverts the schema-truth this series already settled.
- phoenix 5 — 2,400 hand-written lines, zero generated glue, DB wins on conflict — most auditable.

**agent**
- go 4 — Compiler forecloses divergence and a 1.4s incremental loop, but templ's thin corpus breeds first-try mistakes.
- deno 3 — Fast loop (12s/48s) but uniformity rests on a CI gate that fails silently if disabled.
- dotnet 5 — Fastest test loop (22s); decades of convention plus IDE tooling give the tightest uniformity.
- spring 2 — Worst loop by far: 2m10s tests, 4m50s native build, though convention keeps diffs uniform.
- phoenix 3 — One-file-per-screen changeset idiom is structurally uniform, but a thin corpus means hallucinated APIs.

**simplicity**
- go 2 — Eight-ish real concepts, but 38k generated lines regenerate whole on every drift.
- deno 1 — Claims 3 concepts while running eight-plus technologies and a CI gate — overclaim penalised hard.
- dotnet 3 — Honestly counts ~15 concepts, the most of anyone — honesty rewarded despite the heaviest load.
- spring 3 — Most named frameworks, but zero codegen and one language — no trust-unread tax at all.
- phoenix 4 — Fewest layers, and it priced the BEAM/OTP/websocket costs itself — best net count.

**performance**
- go 4 — Consistent 41ms p99 at 200 RPS, k6 named, RSS and cold start disclosed cleanly.
- deno 4 — Matches go's 41ms at 200 VU, honestly volunteers a 380ms unindexed worst case.
- dotnet 3 — 71ms p99 is slowest of the fast tier despite good RSS and 40ms cold start.
- spring 1 — 210ms p99 at just 50 VU: worse latency under a quarter of the load.
- phoenix 5 — 9ms p99 crushes the field, k6 named, and it self-flagged the covering index as not like-for-like.

**Standing after R1: phoenix 42, dotnet 39, go 30, deno 28, spring 20.**

## Round 2 — Cross-exam

| base | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| **dotnet** | 4 | 4 | 2 | 3 | **34** |
| go | 3 | 3 | 3 | 3 | **30** |
| phoenix | 3 | 2 | 2 | 4 | **27** |
| spring | 2 | 2 | 1 | 5 | **24** |
| deno | 2 | 1 | 4 | 3 | **23** |

**reviewer**
- go 3 — Fork-templ plan and an honest audit gap, but "build output" dodges the struct-diff question.
- deno 2 — Total honesty, zero mitigation: no CI alert, no read audit, concepts overclaimed — nothing fixed.
- dotnet 4 — Concedes double-authorship but names a real pgTAP parity test and the missing generator fix.
- spring 2 — Alone on read-audit, but walked back its own p99 3-4x and confessed real per-field labour.
- phoenix 3 — Honest, specific concessions on drain loss and drift, but no mitigation — just named trade-offs.

**agent**
- go 3 — Mechanical 1,200-struct regen reviewed via the query diff holds; templ's single-maintainer risk still looms.
- deno 1 — Conceded its uniformity CI gate fails silently with zero backstop — worst loop failure mode here.
- dotnet 4 — pgTAP parity test is a concrete, automatable check an agent can write and keep green.
- spring 2 — 3-6 hours of manual entity work per field breaks the agent-first loop before code exists.
- phoenix 2 — Conceded no compile-time link catches column drift; changeset errors surface only at runtime.

**simplicity**
- go 3 — Splits diff-reviewed SQL from trusted-as-build-output code — reasonable, still a leap of faith.
- deno 4 — Clean, costless retraction: concedes eight concepts, adds no new machinery to hold.
- dotnet 2 — pgTAP bolts a second test framework and assertion language onto an already 15-concept stack.
- spring 1 — 3-6 hours and five touched artefacts per field is concept load recurring every week, forever.
- phoenix 2 — Round 1's "zero generated glue" quietly grows three recovery mechanisms nobody had to hold before.

**performance**
- go 3 — No new numbers; reused the 41ms figure and deflected the regen concern to diff-stat review.
- deno 3 — Honestly conceded the silent gate and no read-audit, but zero new numbers.
- dotnet 3 — Defended 71ms and proposed pgTAP and a source generator, but no fresh runtime data.
- spring 5 — Unprompted revision to ~700ms p99 at matched load, confirming N+1 hits real screens.
- phoenix 4 — Conceded an uncosted extra write per keystroke batch for draft persistence.

**Running after R2: dotnet 73, phoenix 69, go 60, deno 51, spring 44.**

## Round 3 — Unlocked iteration

| base | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| **dotnet** | 5 | 5 | 4 | 4 | **46** |
| spring | 4 | 4 | 5 | 4 | **42** |
| deno | 3 | 4 | 2 | 4 | **33** |
| go | 4 | 3 | 2 | 3 | **31** |
| phoenix | 3 | 3 | 1 | 5 | **30** |

**reviewer** — go 4: cut generated 38k→9,100 and shipped the promised read-audit with real numbers. deno 3: closed the silent CI gate with a nightly re-assert; generated volume unchanged. **dotnet 5: the source generator eliminates the double-authored permission rule entirely.** spring 4: reversed its architecture stance but still hand-mirrors RLS/`@PreAuthorize` unchecked. phoenix 3: conceded "zero glue", now generates 2,000 lines — the smallest move.

**agent** — go 3: slowest tests (91s) and build of the round. deno 4: fastest honest loop, self-healing CI gate. **dotnet 5: 19s one-suite tests, and a compile-time generator makes drift a self-correcting build error.** spring 4: real repeated-job win, 3-6h → 20min, but the mirror stays ungated. phoenix 3: by its own admission Dialyzer can't catch Ecto's casts — containment is partial.

**simplicity** — go 2: claims 5, omits pgaudit, goose, pgx, testcontainers and the wrapper it now owns. deno 2: "honest" 8 ignores three additions this round. dotnet 4: genuine net removal — pgTAP and the double-authored rule gone. **spring 5: dropped JPA, Hibernate, Envers and entities-as-model for one library — real simplification.** **phoenix 1: drafts table, auto-recover, localStorage, pgaudit, Dialyzer, a bespoke generator — and the count still says 4.**

**performance** — go 3: p99 rose to 44ms, disclosed why. deno 4: real memory win, best cold start. dotnet 4: genuine root-cause fix cut p99 71→52ms. spring 4: largest improvement in the series (700→58ms) with the mechanism explained. **phoenix 5: fastest p99 by far, rigorous draft-write capacity math, honest about the non-like-for-like index.**

**Running after R3: dotnet 119, phoenix 99, go 91, spring 86, deno 84.**

## Round 4 — Final rebuttal and iteration

| base | reviewer (3) | agent (3) | simplicity (2) | performance (2) | **weighted** |
|---|---|---|---|---|---|
| **dotnet** | 4 | 5 | 3 | 4 | **41** |
| deno | 3 | 3 | 5 | 5 | **38** |
| spring | 5 | 4 | 3 | 2 | **37** |
| phoenix | 4 | 2 | 2 | 3 | **28** |
| go | 3 | 1 | 4 | 3 | **26** |

**reviewer** — go 3: honest recount, no fix; the 150-line wrapper stays a self-admitted penalty. deno 3: trimmed 12→11 by trusting GitHub's own gate, but 14,000 generated lines is still the most to trust. dotnet 4: folded Dapper in, 14→13, permissions stay one-source and compile-checked. **spring 5: closed its last reviewer gap — `@PreAuthorize` now generated from `pg_policy`, no hand-mirroring left.** phoenix 4: two honest reversals against interest, but the permission story is unchanged.

**agent** — **go 1: slowest build and tests (47s/91s), unparallelised, and "out of runway" — no improvement offered.** deno 3: fastest compile but 48s tests trail dotnet; no compiler-enforced permission drift. **dotnet 5: fastest tests (19s), permission drift is a build error — the tightest self-correcting loop here.** spring 4: matched dotnet's generator, but the 4m10s native image taxes the loop. phoenix 2: slower loop, and uniformity leans on its own bespoke generator plus tooling.

**simplicity** — go 4: honest recount to 10, the lowest count, but confesses the wrapper is still bespoke. **deno 5: the only base that actually removed something — a bespoke checker swapped for GitHub's built-in gate.** dotnet 3: consolidated to 13 and argues that's a floor; credible, but ties spring for the highest. spring 3: fixed its hand-mirrored security, but 13 ties for the most. **phoenix 2: biggest undercount reveal — 12 concepts, mostly scaffolding propping up a thin ecosystem, not serving the CRM.**

**performance** — go 3: tied p99 but worst cold start (350ms) and second-highest audit tax. **deno 5: best p99 (41ms), fastest cold start of all five (38ms), lowest audit tax (9%).** dotnet 4: solid across the board, cheapest quantified audit cost. spring 2: worst native memory, and a 4m10s build undercuts the 0.06s cold-start claim. phoenix 3: honest restatement credited, but worst memory scaling and highest audit CPU tax.

---

## Final tally

| base | R1 | R2 | R3 | R4 | **total /200** |
|---|---|---|---|---|---|
| **dotnet** | 39 | 34 | 46 | 41 | **160** |
| phoenix | 42 | 27 | 30 | 28 | **127** |
| spring | 20 | 24 | 42 | 37 | **123** |
| deno | 28 | 23 | 33 | 38 | **122** |
| go | 30 | 30 | 31 | 26 | **117** |

### By judge (raw sum across 4 rounds, max 20)

| base | reviewer (3) | agent (3) | simplicity (2) | performance (2) |
|---|---|---|---|---|
| **dotnet** | **17** | **19** | 12 | 14 |
| phoenix | 15 | 10 | 9 | **17** |
| spring | 13 | 12 | 12 | 12 |
| deno | 11 | 11 | 12 | 16 |
| go | 12 | 11 | 11 | 13 |

**dotnet wins by 33 points.** Second through fifth are within 10 of each other — effectively a tie
for the places.
