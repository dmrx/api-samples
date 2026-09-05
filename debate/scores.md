# Scores

Weights: everyman 1, reviewer 2, agent 2, operator 1. Max weighted = 30.

## Round 1 — Opening

| contender | everyman (1) | reviewer (2) | agent (2) | operator (1) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 2 | 2 | 2 | 2 | **12** |
| go | 4 | 4 | 5 | 4 | **26** |
| rust | 3 | 3 | 3 | 5 | **20** |
| deno | 2 | 5 | 4 | 3 | **23** |

**everyman**
- incumbent 2 — Admits full-tree re-render on every SSE tick for 500 rows; zero mention of reconnect.
- go 4 — Concrete: HTMX swaps rows over SSE, no client router, minimal JS — screen stays lean and fast.
- rust 3 — No GC pause/OOM under CDC load helps stability, but dodges bad-network/reconnect behavior.
- deno 2 — Mostly sells build simplicity; barely touches how the list renders or survives a dropped connection.

**reviewer**
- incumbent 2 — Zone.js patches callbacks into change detection, remoteEntry.json — runtime wiring hidden from diffs.
- go 4 — Plain pgx SQL and html/template read top to bottom; Preact island adds minor client magic.
- rust 3 — Axum extractor traits and Askama macro-generated code hide logic outside the diff.
- deno 5 — No bundler, no macros, no framework layer — TypeScript and HTMX fragments are all there is to read.

**agent**
- incumbent 2 — Ignores build/test loop entirely; three toolchains, Module Federation webpack builds slow and opaque.
- go 5 — Sub-second builds, plain SQL, stdlib — directly nails build time and corpus familiarity.
- rust 3 — Compile-time bug-catching is real, but dodges Rust's own slow build times.
- deno 4 — Zero build step, no-config tests — strong loop, but skips error-clarity claim.

**operator**
- incumbent 2 — Three runtimes (JVM+Node+Angular) to patch, heaviest pod footprint, Angular-major churn yearly.
- go 4 — Static binary, stdlib-only, one pod — but CDN-fetched Preact island sneaks in a runtime dependency.
- rust 5 — One ~20MB binary, one Cargo.lock, no GC/OOM kills — cheapest pod, cleanest 2031 bet.
- deno 3 — Binary-swap upgrades sound cheap, but zero pod/memory numbers — dodges the actual question.

**Standing after R1: go 26, deno 23, rust 20, incumbent 12.**

## Round 2 — Scenario (layout, data path, numbers)

| contender | everyman (1) | reviewer (2) | agent (2) | operator (1) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 2 | 1 | 2 | 1 | **9** |
| go | 4 | 4 | 5 | 5 | **27** |
| rust | 4 | 3 | 3 | 3 | **19** |
| deno | 3 | 5 | 4 | 4 | **25** |

**everyman**
- incumbent 2 — Two-hop SSE relay (core→BFF→browser) hides a silent failure point; drop-handling still unaddressed.
- go 4 — Single-process, direct EventSource+HTMX, fewest hops — fast, but never says what a drop does.
- rust 4 — Same tight single-hop path as go, native EventSource resilience — still silent on reconnect/replay.
- deno 3 — Single LISTEN connection is a named single point of failure, no reconnect story given.

**reviewer**
- incumbent 1 — 765 lines, 3 repos, plus hidden Kafka hop and zone.js runtime wiring outside the diff.
- go 4 — 640 lines, 1 repo, fully visible embed, no macros, longest but nothing hidden.
- rust 3 — 460 lines but admits hidden Askama macro-generated code, needs `cargo expand` to see.
- deno 5 — 214 lines, 1 repo, zero build artifacts or macros, plainest end-to-end diff.

**agent**
- incumbent 2 — 52s cold, 3 toolchains' stack traces to juggle, no test number given at all.
- go 5 — 0.9s cold / 0.2s incremental, one binary, plain SQL, clearest error surface.
- rust 3 — 51s cold is real pain; incremental cheats via `cargo check`, not a run.
- deno 4 — 2.1s cold, 140ms incremental, 0.4s test — honest, tight, but thinner corpus.

**operator**
- incumbent 1 — 3 repos, Kafka + Debezium + Spring + Node BFF — most parts to page you at 2am.
- go 5 — 1 repo, stdlib LISTEN/NOTIFY, static binary, no Kafka, no CDN — least to break or patch.
- rust 3 — Clean data path but 38 crates on cold build and Askama macros add churn surface.
- deno 4 — Zero package.json, single binary, LISTEN/NOTIFY direct — lean, but runtime younger than Go's.

**Running total after R2: go 53, deno 48, rust 39, incumbent 21.**

## Round 3 — Cross-exam

| contender | everyman (1) | reviewer (2) | agent (2) | operator (1) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 1 | 3 | 1 | 1 | **10** |
| go | 3 | 4 | 4 | 3 | **22** |
| rust | 4 | 5 | 5 | 5 | **29** |
| deno | 5 | 4 | 3 | 4 | **23** |

**everyman**
- incumbent 1 — Admits total loss: no gap-fill, up to 500 updates vanish silently, no owner.
- go 3 — Backfill works but exposes a new bug: 3 replicas triple-render every update.
- rust 4 — `Lagged()` caught explicitly, backfill via updated_at, no silent loss.
- deno 5 — Native Last-Event-ID backfill, smallest code (229 lines), pods stay synced.

**reviewer**
- incumbent 3 — Admits 500-row Kafka loss and unimplemented backfill, but never prices a lines-fixed number.
- go 4 — Prices fix at +60 lines, then admits an unpriced 3x redundant query problem across replicas.
- rust 5 — Precise +15 lines, reused updated_at, explicit Lagged detection — cleanest, most consistent numbers.
- deno 4 — Prices fix at +15 lines exactly, smallest total, but dodges the multi-pod query-load question.

**agent**
- incumbent 1 — Admits silent Kafka drops, no offsets, backfill unimplemented — worst error clarity, biggest unplanned fix.
- go 4 — Clean idiomatic fix, +40-60 lines, idempotent re-SELECT reasoning — but no round-3 timing numbers.
- rust 5 — Honest 4.8s/6.3s correction plus typed `Err(Lagged(n))` — exactly the explicit-error signal agents need.
- deno 3 — Tiny diffs, but the multi-pod LISTEN gap needed extra probing before the NOTIFY fan-out logic held up.

**operator**
- incumbent 1 — Admits Kafka is pure overhead, no offset, up to 500 rows silently vanish.
- go 3 — Small backfill fix but flags leader election — new dependency, more moving parts to patch.
- rust 5 — Native Lagged detection, no new deps, 15-line backfill reusing an existing column — leanest fix.
- deno 4 — No broker needed, NOTIFY reaches every pod directly — backfill stays under 15 lines.

**Running total after R3: go 75, deno 71, rust 68, incumbent 31.**

## Round 4 — Stress (off-by-one in the 30-day expiry rule)

| contender | everyman (1) | reviewer (2) | agent (2) | operator (1) | **weighted** |
|---|---|---|---|---|---|
| incumbent | 1 | 1 | 2 | 1 | **8** |
| go | 5 | 4 | 5 | 5 | **28** |
| rust | 3 | 3 | 4 | 4 | **21** |
| deno | 4 | 4 | 5 | 4 | **26** |

**everyman**
- incumbent 1 — Three copies across two languages, no compiler catch, 110s to red, customer finds it first.
- go 5 — Two copies, one test, red in under a second — fewest places the rule can drift.
- rust 3 — Single computed bool is clean but three files touch the number and it's 6.3s to red.
- deno 4 — Three copies of "30" but fastest honest test loop at 0.4s, no template layer hiding it.

**reviewer**
- incumbent 1 — 3 copies, 2 languages, split across MFE and core-svc — no shared constant, most hidden.
- go 4 — Only 2 spots, one language, SQL literal plus a Go bool — easy single-pass review.
- rust 3 — Hedges "also db.rs if" — reviewer can't be sure how many files hide the number.
- deno 4 — 3 spots but one language, one repo, no template layer obscuring the check.

**agent**
- incumbent 2 — 15-110s to red, weakest catch (reviewer/ticket), duplicated in 3 places.
- go 5 — Under 1s to red (0.2s build + 0.3s test), single named table-test failure.
- rust 4 — Honest 6.3s full loop (not the 1.6s type-check cheat), named assertion.
- deno 5 — 0.4s real number, explicitly flags the 140ms typecheck as misleading.

**operator**
- incumbent 1 — Fix crosses 3 repos, ~110s pipeline, JVM+Node+Angular churn — worst blast radius.
- go 5 — One repo, static binary, red in <1s, stdlib-heavy — cheapest CI, one-command rollback.
- rust 4 — One repo but 51s cold build, 38 crates to track — still single-binary rollback.
- deno 4 — One repo, no build step, 0.4s to red — cheapest CI, but younger runtime risks 2031 churn.

---

## Final tally

| contender | R1 | R2 | R3 | R4 | **total /120** |
|---|---|---|---|---|---|
| **go** | 26 | 27 | 22 | 28 | **103** |
| deno | 23 | 25 | 23 | 26 | **97** |
| rust | 20 | 19 | 29 | 21 | **89** |
| incumbent | 12 | 9 | 10 | 8 | **39** |

### By judge (raw sum across 4 rounds, max 20)

| contender | everyman (1) | reviewer (2) | agent (2) | operator (1) |
|---|---|---|---|---|
| go | 16 | 16 | **19** | 17 |
| deno | 14 | **18** | 16 | 15 |
| rust | 14 | 14 | 15 | 17 |
| incumbent | 6 | 7 | 7 | 5 |

**Winner: go (103/120).** deno won the reviewer (least hidden, fewest lines). rust won round 3
outright and ties go on operator. incumbent lost every judge in every round.
