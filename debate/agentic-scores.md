# Scores — greenfield agent-native CRM

Weights: reviewer 3, sdlc-agent 3, simplicity 2, efficiency 2. Max 50 per round.

## Round 1 — Show the stack

| contender | reviewer (3) | sdlc-agent (3) | simplicity (2) | efficiency (2) | **weighted** |
|---|---|---|---|---|---|
| **postgres-first** | 5 | 4 | 4 | 5 | **45** |
| graph-native | 3 | 4 | 4 | 1 | **31** |
| maximal | 3 | 3 | 3 | 3 | **30** |
| event-log | 2 | 2 | 2 | 2 | **20** |

**reviewer** — maximal 3: 7 files, 4 languages, two schemas kept in sync by hand. **postgres-first 5: 6 files, 2 languages, one PR, one honest auth site.** event-log 2: auth enforced twice, unmasked payload leak, "which is truth" postmortem. graph-native 3: one language, but recursive CTEs and double-policy edges hurt clarity.

**sdlc-agent** — maximal 3: thorough but DSL sprawl hurts discoverability; the only stack with UI and security tests both. postgres-first 4: 6 files, 2 languages, one catalog to grep — but concedes no React file, no security test. event-log 2: the most ceremony for one field — new event type, projector, 20-min backfill. graph-native 4: node property correctly modelled, includes React, a single SQL catalog keeps it discoverable.

**simplicity** — maximal 3: kept Cedar and GraphQL+REST while calling both weaknesses; the only real cut is 6 of 17. postgres-first 4: named triggers for everything; Temporal's own trigger (multi-week human-in-the-loop) is user-visible, so principled. event-log 2: 4 boxes hides 6 concepts and the worst pageable count. graph-native 4: zero new boxes, but edge table + recursive CTEs is a real added concept versus a plain relationships table.

**efficiency** — maximal 3: 80ms unexplained on an identical Postgres + pgvector stack. **postgres-first 5: named 45ms pgbench p99, 3 pageable, $900/mo — best latency-to-clarity ratio.** event-log 2: matches p99 but 4 pageables and the priciest at $1,100/mo for the fewest boxes. **graph-native 1: omits cost and pageable count; an unbounded-depth p99 cliff an agent can trigger with one bad query.**

**Standing after R1: postgres-first 45, graph-native 31, maximal 30, event-log 20.**

## Round 2 — Cross-exam

| contender | reviewer (3) | sdlc-agent (3) | simplicity (2) | efficiency (2) | **weighted** |
|---|---|---|---|---|---|
| **postgres-first** | 5 | 5 | 3 | 4 | **44** |
| graph-native | 4 | 3 | 3 | 2 | **31** |
| maximal | 3 | 2 | 2 | 2 | **23** |
| event-log | 2 | 2 | 1 | 2 | **18** |

**reviewer** — maximal 3: cut REST honestly, but admits the GraphQL graph duplicates the schema by hand, unenforced. **postgres-first 5: concrete costs stated; the borrowed CTE design credited, still the simplest diff.** event-log 2: append-only — its core legibility promise — conceded broken; truth now needs replay. graph-native 4: honest 17-policy tax and a scoped edge table, but the node-history gap is admitted.

**sdlc-agent** — maximal 2: the GraphQL graph is a hand-synced duplicate with no codegen guard. **postgres-first 5: one generic CTE, one row filter for entity eight — the round's clearest discover-and-ship story.** event-log 2: every write and read path now also carries crypto-shred keys and tombstone checks. graph-native 3: 17 RLS policies need cross-team sign-off an agent can't give.

**simplicity** — maximal 2: cut REST but keeps Cedar and its 35ms hop for a benefit RLS gets free. postgres-first 3: concepts grew (PERIOD + trigger, relationships CTE) but each answers a real need. **event-log 1: crypto-shredding plus tombstones abandon append-only, the log's whole reason to exist.** graph-native 3: honestly shrank its own scope to one hop-4 case.

**efficiency** — maximal 2: named a permanent 35ms Cedar tax per call; cutting REST buys nothing at runtime. postgres-first 4: gave real ceilings (3 consumers, 5 workflow types) and the 2x storage cost unprompted. event-log 2: an unpriced KMS round trip on every write plus a manual WAL/backup purge cycle. graph-native 2: still no cost or pageable numbers; 17 per-row RLS checks inside recursion untimed.

**Running after R2: postgres-first 89, graph-native 62, maximal 53, event-log 38.**

## Round 3 — Unlocked iteration

| contender | reviewer (3) | sdlc-agent (3) | simplicity (2) | efficiency (2) | **weighted** |
|---|---|---|---|---|---|
| **maximal** | 5 | 4 | 5 | 5 | **47** |
| postgres-first | 4 | 5 | 2 | 3 | **37** |
| event-log | 4 | 3 | 4 | 4 | **37** |
| graph-native | 3 | 4 | 3 | 2 | **31** |

**reviewer** — **maximal 5: one signature yields tool + RPC — a diff reader sees the whole contract, no shadow schema.** postgres-first 4: clean three-schema split, but admits 30+ table RLS is unauditable at a glance. event-log 4: smallest, honest diff — but now just postgres-first's shadow. graph-native 3: generated edge policies hide the real diff; the depth cap is trust, not something the reader can verify.

**sdlc-agent** — maximal 4: tightest single-declare story, but two generated adapters still cost a codegen step. **postgres-first 5: all seven artifact kinds plus the one security test the probe explicitly requires.** event-log 3: fewest files, but no React surface and no security test. graph-native 4: matches postgres-first's discoverability; the generated edge policy erases a sign-off bottleneck.

**simplicity** — **maximal 5: 8 boxes, 7 concepts, $600/mo, deleted Cedar and GraphQL cold; the adapter is codegen, not a concept.** postgres-first 2: kept Temporal day one, now 11 concepts — the highest of all four. event-log 4: the biggest single-round cut, but merely converged onto postgres-first. graph-native 3: dissolved to a rule, 0 extra boxes, but the generated-policy macro is bespoke tooling kept.

**efficiency** — **maximal 5: cheapest ($600/mo), fewest pagers (2).** postgres-first 3: priciest, most pagers, and pays +7ms for defaulting history instead of opting in. event-log 4: same 45ms, history opt-in only when asked. graph-native 2: slowest at 62ms — a full depth-4 CTE on every churn call even when 2 hops would do.

---

## Final tally

| contender | R1 | R2 | R3 | **total /150** |
|---|---|---|---|---|
| **postgres-first** | 45 | 44 | 37 | **126** |
| maximal | 30 | 23 | **47** | **100** |
| graph-native | 31 | 31 | 31 | **93** |
| event-log | 20 | 18 | 37 | **75** |

### By judge (raw sum across 3 rounds, max 15)

| contender | reviewer (3) | sdlc-agent (3) | simplicity (2) | efficiency (2) |
|---|---|---|---|---|
| postgres-first | **14** | **14** | 9 | **12** |
| maximal | 11 | 9 | **10** | 10 |
| graph-native | 10 | 11 | **10** | 5 |
| event-log | 8 | 7 | 7 | 8 |

**postgres-first wins on the series; maximal won the final round with the leanest end state anyone
reached — 8 boxes, 7 concepts, 2 pagers, $600/mo — after deleting six of its own seventeen boxes.**
