# Verdict — five bases, 2030 CRM

**.NET wins, 160 of 200, by 33 points.** It won the two heaviest judges outright — reviewer 17/20
and agent 19/20 — for one reason: it is the only base whose **permission rules are generated
in-compiler from `pg_policy`**, so a security rule that drifts is a build error, not a runtime
surprise or a CI check someone can disable. It also produces the fewest generated lines of anyone
(1,600) and reviews a 60-line template rather than the output. Second through fifth — phoenix 127,
spring 123, deno 122, go 117 — are within 10 points of each other, which is noise.

---

## The three findings that outrank the ranking

**1. Everyone was undercounting concepts, badly.** Pressed for an honest count in the final round,
every base revised upward: go 5→10, deno 8→12, **phoenix 4→12**. The real spread is 10 to 13.
There is no simple stack for a CRM. There are five stacks of roughly equal weight and a choice
about which concepts you would rather hold.

**2. Performance is not a discriminator.** Once phoenix re-ran its benchmark without the covering
index it alone had used, its headline 9ms became 49ms and the field collapsed into one band:
deno 41ms · go 44ms · phoenix 49ms · dotnet 52ms · spring 55ms. **Fourteen milliseconds across five
stacks.** They all talk to the same Postgres, and Postgres is doing the work. Stop choosing on this.

**3. Spring was wrong and said so, and it was the biggest single improvement in the series.**
It opened defending entities-as-model — generation flowing *into* the database — scored 20/50 and
last place, then said *"I was wrong. Entities-as-model caused both the N+1 and the 3-6hr/field cost;
both die when the DB, not the class, is truth."* Dropping JPA, Hibernate, Envers and entities for
jOOQ took its p99 from ~700ms to 55ms and its weekly schema-drift cost from **3-6 hours per field to
20 minutes**. It finished 6 points off second.

---

## What each base is actually best at

| base | its one real win | its one real cost |
|---|---|---|
| **dotnet** | Permission drift is a **build error** — `pg_policy` → `[Authorize]`, compile-checked, one source one consumer. 1,600 generated lines, fewest here. Tests 19s, one framework. | 13 concepts, tied highest. 52ms p99, slowest of a band that doesn't matter. |
| **phoenix** | The form is genuinely stateful and it treats it that way; 1,600 hand-written lines is the least code of anyone. Most rigorous capacity work in the series. | 345MB/pod at 5,000 users → 10 pods and 3.45GB at 50,000, against 58MB stateless pods. Thinnest corpus; its uniformity leans on tooling it wrote itself. |
| **spring** | The only base that started with per-row read audit (Envers), and the largest turnaround. 20min/field drift after jOOQ. | Gave up per-row read audit for pgaudit's column grain to get there. 4m10s native image build. |
| **deno** | **38ms cold start, fastest of all five** — no JIT, no warmup. Only base to *remove* a bespoke tool in the final round. | 14,000 generated lines, most here. Won nothing else outright in four rounds. |
| **go** | Lowest honest concept count (10). Cut generated lines 38,000 → 9,100 by dropping per-role structs. | Slowest loop at the end (47s build, 91s tests), 350ms cold start, and a 150-line bespoke wrapper it admits is a penalty by its own rule. Made no change in the final round — "out of runway". |

---

## What to actually build

The architecture is settled and it is the same in all five languages. Every base ended here, and the
one that started somewhere else reversed and gained 22 points a round doing it.

**Postgres owns the truth.** Salesforce is the system of record, CDC mirrors it into Postgres, and
Postgres holds columns, types, column `GRANT`s, `FORCE ROW LEVEL SECURITY` policies, and the easy
half of validation as domains and CHECK constraints. Everything the application knows is **generated
out of the catalog** — types, base validators, and, decisively, **the permission attributes
themselves**. Only cross-field rules, conditional visibility and drafts are hand-written, because
drafts must save partial rows and cannot live behind constraints.

**Generate the permission rules, don't mirror them.** This is what separated first place from the
rest. dotnet reads `pg_policy` in a compiler-integrated source generator and emits `[Authorize]`;
spring adopted the identical pattern through jOOQ and closed its last gap. Everyone who kept the
rule in two places — hand-mirrored, or checked by a test — was marked down for it every round.

**Ship these before the first screen**, because every base conceded each one fails open:
1. A grants CI check diffing `information_schema.column_privileges` against a checked-in file.
2. A schema-drift gate that regenerates from the catalog and fails the build on any diff — as a
   **required status check with no bypass**, not a job someone can quietly disable.
3. `pgaudit` read logging scoped to sensitive columns. Costs disclosed here: ~6ms p99 (dotnet),
   ~9% write-path (deno), ~12% (go), ~15% CPU (phoenix). Without it, no base could say who read a
   leaked field — only spring could, and only before it dropped Envers.
4. A DDL/grant history trigger, so you can say what access looked like on a past date.

**Pick the language on team and corpus, not on the benchmark.** The p99 spread is 14ms. The concept
spread is three. What actually differs is whether your permission rules are checked by a compiler
(dotnet, spring), by a pipeline (deno, go), or by a test at runtime (phoenix) — and how many people
you can hire who already know the language.
