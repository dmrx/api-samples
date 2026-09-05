# Duel scores — Go vs Deno, 2030 CRM

Weights: reviewer 3, agent 3, simplicity 2, performance 2. Max 50 per round.
Permissions held constant (Postgres column grants + RLS available to both).

Rounds 4-5 were run **unlocked**: each side could swap, drop or add any component.

| | R1 case | R2 cross-exam | R3 design | R4 unlocked | R5 iterate | **total /250** |
|---|---|---|---|---|---|---|
| **deno** | 33 | 34 | 43 | 38 | 38 | **186** |
| go | 28 | 22 | 22 | **38** | 25 | **135** |

### By judge (raw sum across 5 rounds, max 25)

| | reviewer (3) | agent (3) | simplicity (2) | performance (2) |
|---|---|---|---|---|
| deno | **20** | **20** | **18** | **15** |
| go | 12 | 15 | 13 | 14 |

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

### Round 4 — unlocked redesign (**tie**)
| | reviewer | agent | simplicity | performance | weighted |
|---|---|---|---|---|---|
| go | 4 | 4 | 4 | 3 | **38** |
| deno | 3 | 5 | 3 | 4 | **38** |

go dropped `html/template` for **templ** (compiled, type-checked views) and its bespoke generator
for **sqlc** (types generated out of the Postgres schema). That fixed all four faults at once:
`map[string]any` gone, one field-list copy with the compiler proving agreement, generation pointed
the right way, silent empty renders now build errors. Its best round of the series.
deno pushed validation into Postgres constraints, closed its raw-query bypass with a generated
repository, added an `audit.grant_history` event trigger, and compiled to a binary: 95MB → 61MB.

### Round 5 — cross-exam and final iteration
| | reviewer | agent | simplicity | performance | weighted |
|---|---|---|---|---|---|
| go | 3 | 2 | 3 | 2 | **25** |
| deno | 4 | 4 | 4 | 3 | **38** |

The fix got priced. go disclosed **1,200 generated structs** (40 objects × ~6 roles × ~5 queries),
regenerated in full on every weekly schema change — sqlc has no incremental mode — and conceded
templ has a single maintainer and thin likely 2030 corpus coverage. deno found the one real RLS
bypass nobody had named (`SECURITY DEFINER`, which runs as the table owner) and banned it in CI.

**Both converged.** go adopted deno's constraint-as-source design in its final iteration. The two
stacks now differ only in language, and in 15MB of RAM against 11ms of p99 and a tighter loop.
