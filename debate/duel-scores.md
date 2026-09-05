# Duel scores — Go vs Deno, 2030 CRM

Weights: reviewer 3, agent 3, simplicity 2, performance 2. Max 50 per round.
Permissions held constant (Postgres column grants + RLS available to both).

| | R1 case | R2 cross-exam | R3 design | **total /150** |
|---|---|---|---|---|
| **deno** | 33 | 34 | 43 | **110** |
| go | 28 | 22 | 22 | **72** |

### By judge (raw sum across 3 rounds, max 15)

| | reviewer (3) | agent (3) | simplicity (2) | performance (2) |
|---|---|---|---|---|
| deno | **13** | **11** | **11** | 8 |
| go | 5 | 9 | 6 | **9** |

### Round 1 — the case with permissions neutralised
| | reviewer | agent | simplicity | performance | weighted |
|---|---|---|---|---|---|
| go | 2 | 4 | 2 | 3 | **28** |
| deno | 4 | 3 | 4 | 2 | **33** |

### Round 2 — cross-exam
| | reviewer | agent | simplicity | performance | weighted |
|---|---|---|---|---|---|
| go | 2 | 2 | 2 | 3 | **22** |
| deno | 4 | 4 | 3 | 2 | **34** |

### Round 3 — design the best stack
| | reviewer | agent | simplicity | performance | weighted |
|---|---|---|---|---|---|
| go | 1 | 3 | 2 | 3 | **22** |
| deno | 5 | 4 | 4 | 4 | **43** |

**Turning points**
1. **R1** — go conceded that Postgres grants make the returned column set vary per role, so it must
   scan into `map[string]any`. Its compiler stops checking field names at the data boundary — the
   exact thing its reviewability case rested on. reviewer: 2.
2. **R2** — asked what keeps a Go form's four copies of the field list in sync, go answered
   "None — there isn't one" and reached for codegen. The founding doctrine ("hand-write it, nothing
   hidden") was abandoned by its own author.
3. **R3** — go's generator points the wrong way: it emits Postgres GRANT/RLS DDL *from* Go struct
   tags, when Salesforce → CDC → Postgres already owns the schema. deno generates in the direction
   the data actually flows. reviewer: go 1, deno 5.
4. **R3** — deno answered go's one real advantage (compiler-enforced uniformity) by making the
   schema generated and CI-diffed: drift becomes a red pipeline, not a lint warning. And it finally
   priced Zod (0.2ms/request), closing the dodge it had been marked down for twice.
