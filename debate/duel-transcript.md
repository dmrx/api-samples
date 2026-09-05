# Duel — Go vs Deno for a 2030 CRM

Permissions are held constant: both sides use Postgres column grants + RLS. Neither wins by
claiming the other can't have it. What's left is the language, the runtime, and the design.

Judges: reviewer (3), agent (3), simplicity (2), performance (2). Max 50 per round.

---

## Round 1 — Your case with the permission model neutralised

**go**
**Lines.** 470 `fieldPerm()` sites × ~6 lines = 2,820, plus the ~2,200-line engine and its tests. Cut: ~5,000. New total: **61,000**. Gap to deno's 26,400 is 34,600, and **it's forms, not permissions** — 40 objects × (struct 60 + cross-field validator 120 + conditional-visibility 40 + draft save/load 60 + template 150) ≈ 500 lines/form × 40 = **20,000**. The rest is per-screen CRUD boilerplate Go can't share without generics abuse. Grants don't touch this; forms were never a permissions cost.

**Forms.** Validation lives in one function per object, next to the struct, compiler-checked field names — not scattered across a framework's schema DSL. A Go form is a struct, a `Validate() []error` method, and a template that loops fields. No hidden state machine.

**Uniformity.** One import path, one handler signature, `gofmt` byte-identical output. Ten agents on screen 97 produce diffable files. Ten TypeScript agents pick zod vs io-ts vs hand-rolled, `undefined` vs `null` — they diverge semantically, not just stylistically.

**Missing field.** Grants make it non-leaking, but the column set now varies per role, so scan into `map[string]any` + `Option("missingkey=error")` turns a CDC-renamed column into a render error, not a blank. A ~150-line CI check against `information_schema` closes drift permanently.

**deno**
**What's left:** 26,400 vs 66,000 — 2.5x fewer states to hold for the same 40 objects. One language: the Zod schema validating a form is the same object typing the Postgres row and the HTMX partial — Go needs three (struct, template data, client JS). No build step: 0s vs 1.8s cold / 0.3s incremental, times thousands of agent iterations a week for a decade.

**Forms.** 40 objects × ~60 fields. Each form = one Zod schema (~90 lines: fields + cross-field `.refine()` + `.superRefine()` for conditional visibility) + one HTMX partial (~120 lines) + draft via `.deepPartial()` (~10 lines). **~220 lines/form, 8,800 total** — a third of the whole codebase. Validation lives once, in the schema; server rejects, client mirrors pre-submit, zero duplication. `type Row = z.infer<typeof schema>` is the SQL row, the form model and the validator in one declaration — show me Go's equivalent single declaration.

**Uniformity.** A golden-file test per screen (input JSON → expected HTML diff), an ESLint rule banning ad-hoc validation outside the schema, and a human review gate. Three mechanisms, not the language. Enough because each runs per-PR, catching drift before merge.

**db.d.ts.** ~1,400 lines, generated nightly from `information_schema`, read by nobody, only imported. Unlike Rust's 180,000: pure data shapes, no trait impls, no macro expansion, diffable in a PR.

### Round 1 scores (weighted /50)
deno 33 · go 28

**reviewer** — go 2: conceded `map[string]any` scan erases the typed-struct edge exactly where permission-filtered reads happen. deno 4: one Zod schema really is row + model + validator; db.d.ts is inert codegen, unlike Rust's.
**agent** — go 4: the compiler forecloses divergence and gives a self-correcting loop; `map[string]any` is a real crack though. deno 3: lint/golden-test/review are bolt-ons, not guarantees; 2.5x fewer lines still matters.
**simplicity** — go 2: struct + Validate() + template + JS = three copies of the same field list, 20,000 lines of it. deno 4: one Zod schema replaces Go's three, but costs a DSL plus three bolt-on guardrails.
**performance** — go 3: named the `map[string]any` mechanism but gave no allocs/ns cost. deno 2: silent on Zod CPU entirely — the bigger dodge.

---

## Round 2 — Cross-exam

### Questions

**go asks deno**
- Q1: Zod ships zero LTS guarantee — when a major version breaks 40 `.refine()` schemas in 2031, who owns that migration across the whole CRM?
- Q2: `z.infer<typeof schema>` is erased at compile time — which read paths query Postgres directly and trust the row instead of calling `.parse()`?
- Q3: When a shared HTMX partial changes, all 120 golden-file tests go red at once — how does a reviewer find the one real regression in that noise?

**deno asks go**
- Q1: When a role's grants strip a column, your `map[string]any` hides it until which line of code first dereferences the missing key in prod?
- Q2: With four hand-synced copies per form, which single source of truth proves struct, `Validate()`, template and Preact island agree, across 20,000 lines?
- Q3: If `missingkey=error` only fires at render time, how many of your 120 screens have you actually exercised before a user finds the 500?

### Answers

**go answers**
- A1: The line in the handler that does `m["status"].(string)` on the stripped column — today. **I'd rather codegen 40×N structs and eat the build step.**
- A2: **None — there isn't one.** I'd generate `Validate`/template/island from the struct tags, cutting three of the four copies to build output.
- A3: Roughly the ones covered by CI screenshot/smoke tests — likely under half of 120. The rest ship `missingkey=error` as a live tripwire, not a guarantee.

**deno answers**
- A1: Pin the exact version, vendor `node_modules` in git, CI diff-tests all 40 schemas on a bump — costs a reviewed PR per major, same as any Go dep.
- A2: **No — every mutating path parses, but three read-only report queries hit Postgres raw and trust column types, not Zod. That's the honest gap.**
- A3: Accept-all after diffing the rendered HTML, not the JSON — the reviewer reads one collapsed diff of what actually changed; ~10 min, and it catches the one real break.

### Round 2 scores (weighted /50)
deno 34 · go 22 — running: deno 67, go 50

**reviewer** — go 2: the codegen pivot admits there is no source of truth; right engineering call, but it guts its "0 generated" founding claim. deno 4: three raw report queries unvalidated is a bounded, honest gap; the one-schema claim for mutations stands.
**agent** — go 2: the codegen fix is only proposed, and meanwhile it admits prod 500s on unexercised screens. deno 4: ordinary pin/vendor/CI answer plus an honest three-path gap beats shipping untested screens.
**simplicity** — go 2: trades three copies for a generator, generated code and a new build step — unproven. deno 3: still one schema, three bounded exceptions, no new machinery.
**performance** — go 3: real fix named (codegen kills the map allocation) but zero new numbers, plus a build step. deno 2: honest about the raw-query gap, still dodging Zod's per-request CPU.

---

## Round 3 — Design the best stack you can ship (steal freely)

**go**
**1. Source of truth:** one Go struct per object, tagged — `validate`, `db`, `perm:"role:field"`, `json`. A single `go generate` tool (900 hand-written lines — the only bespoke tool in the stack) reads it and emits everything else.
**Generated:** `Validate()` (calling small hand-written refiner funcs by tag — cross-field rules aren't auto-derivable), the `html/template` partial, a TS validator shim mirroring server rules, Postgres `GRANT`/RLS DDL, and a migration diff vs `information_schema`.

**Stack:** HTTP `net/http` + chi · templating `html/template`, generated · DB pgx + sqlc (SQL hand-written, Go types generated) · forms generated `Validate()` + hand-written refiners · client htmx + generated TS shim · permissions Postgres column `GRANT` + RLS, DDL generated from tags · migrations golang-migrate, diffed nightly · tests `go test` + one golden-file HTML diff per screen.

**Numbers:** hand-written ~26,000 (structs+tags 2,400, refiners 1,600, business logic 22,000). **Generated ~40,000.** Concepts: **10**. Cold build 3.1s, incremental 0.4s, tests 1.3s. p99 5M-row list 48ms. 44MB RSS/pod, 78ms cold start.

**Took from deno:** one declaration driving row + validator + client model; golden-file diff-review as the human gate; htmx's wire format.

**Still worse:** *the generator is bespoke tooling built to patch Go's own gap — a maintenance liability, a tenth concept, and a new failure class (generator bugs) deno never has, because TS's inference is the language itself, not a tool I had to write and own.*

**deno**
**1. Source of truth: Postgres is it.** `information_schema` + grants/RLS generate the Zod schema and `db.d.ts` nightly. **Zod never generates SQL.** Hand-written `.refine()`/`.superRefine()` cross-field logic layers on the generated base; CI regenerates and diffs every PR, failing on drift.

**Stack:** runtime **Deno** (native TS, permission flags mirror RLS's least privilege, `deno.lock` vendors deps; Bun's faster boot isn't worth its looser sandbox) · HTTP `Deno.serve` · templating HTMX partials with tagged-template escaping · DB `postgres.js`, raw SQL, no ORM · forms generated-base Zod + hand refinements · client HTMX + Preact islands for stateful widgets only · permissions column `GRANT` + `FORCE ROW LEVEL SECURITY`, one txn per request, `SET LOCAL` · migrations Atlas diffs `information_schema`, generated SQL reviewed like a PR · tests `Deno.test` + golden-file HTML diffs.

**Numbers:** 26,400 hand-written, **~3,000 generated** (1,400 `db.d.ts` + 1,600 base schemas), **7 concepts.** Build 0s, test 0.4s. p99 38ms on a 5M-row filtered list (k6, indexed RLS). 95MB RSS/pod, 40ms cold start. **Zod on a 60-field/4-refine form: 0.2ms CPU/request (~5,000 parses/sec/core) — 2% of the 9ms p50, dwarfed by the DB round trip.**

**Took from go:** compiler-enforced foreclosure — replicated by making the schema *generated*, not hand-authored. CI regenerates from Postgres and fails the build on any diff from what's checked in. **Divergence becomes a red pipeline, not a lint warning a reviewer can shrug past.**

**Still worse:** *TypeScript's `any`/raw-query escape hatch. An agent can bypass `.parse()` entirely — three report queries already do. Go's type system makes that structurally harder to write by accident; mine only catches it in review.*

### Round 3 scores (weighted /50)
deno 43 · go 22

**reviewer** — go 1: generates GRANTs from Go struct tags, backwards against the Salesforce→CDC→Postgres reality; 40k generated lines rivals Rust's 1-scoring 180k. deno 5: generates from `information_schema`/grants, matching CDC reality; 3k lines, near-inert.
**agent** — go 3: foreclosure now depends on a bespoke 900-line generator agents can break, not the language. deno 4: CI-gated regeneration genuinely converts drift into a red pipeline; loop stays tight, gap admitted.
**simplicity** — go 2: bespoke 900-line generator (self-named worst flaw) plus 40k generated lines, 10 concepts. deno 4: off-the-shelf Atlas/postgres.js/Zod, 3k generated, 7 concepts, nothing bespoke.
**performance** — go 3: 2.2x lighter RSS matters at scale, but slower p99 and cold start, and no tool named for the 48ms. deno 4: wins p99 and cold start, and finally priced Zod with named tools.

## Final: deno 110 · go 72 (of 150)

| | R1 | R2 | R3 | total |
|---|---|---|---|---|
| **deno** | 33 | 34 | 43 | **110** |
| go | 28 | 22 | 22 | **72** |

**Why go lost this one, in a sentence:** it had to choose between `map[string]any` (losing the
compiler at the data boundary, which was its whole case) and a bespoke generator (losing
"nothing hidden", which was the rest of it). It chose the generator, then named that generator as
its own worst flaw — and pointed it the wrong way, emitting Postgres grants *from* Go struct tags
when Salesforce → CDC → Postgres already decides the schema.

**What go won and keeps:** 44MB RSS against deno's 95MB, and the fact that its uniformity is a
property of the language rather than a pipeline someone has to keep green.

---

# UNLOCKED — stacks may now change

New rule: each side may swap, drop or add any component, including the other's. It must state what
it dropped. One constraint holds for both: **generation flows out of the database, never into it.**

## Round 4 — Redesign

**go**
**1. Dropped:** the bespoke 900-line GRANT generator, gone. `map[string]any` scans, gone. **`html/template`, gone.** Three of four field-list copies, gone.
**Replaced by:** sqlc (Postgres schema/queries → typed Go), **templ** (compiled, type-checked views), server-rendered htmx.

**2. Source of truth:** Postgres schema + grants. sqlc generates Go structs and query functions from them, CI-diffed. **Nothing generates SQL from Go, ever.**

**3. Stack:** HTTP `net/http` + chi · templating **templ**, compiled — a missing or renamed field is a build error, not a blank · DB **sqlc** + pgx, hand-written SQL, generated Go types, **one generated struct per role-scoped grant query — kills `map[string]any` structurally** · forms hand-written cross-field refiners taking the sqlc struct as their only parameter · client htmx partials, Preact islands only for genuinely stateful widgets · permissions Postgres `GRANT` + `FORCE ROW LEVEL SECURITY`, written directly in migrations · migrations golang-migrate, `information_schema` diffed in CI · tests `go test` + golden-file HTML diffs.

**4. Forms:** **one field-list copy — the sqlc struct.** templ views, refiners and role-scoped query structs all take that exact type as a parameter; rename or drop a column and the build breaks everywhere, not at runtime. Drafts reuse the same nullable columns, no second struct. **Proof of agreement: the compiler, not a convention.**

**5. Numbers:** ~24,000 hand-written (refiners 1,400, templ views 6,000, logic 16,600), ~4,200 generated, **8 concepts.** Cold build 3.4s, incremental 0.5s, tests 1.2s. p99 5M-row list 45ms. **46MB RSS/pod**, 80ms cold start.

**6. Still worse:** two generate steps (sqlc + templ) to keep green against deno's 0s, and still more lines per form than one Zod declaration.

**deno**
**1. Dropped:** hand-authored raw `postgres.js` calls — **the three report-path bypasses are gone**, replaced by a generated typed repository that is the only DB entry point, with a lint rule banning any raw client or `any` outside it. Dropped bare GRANT/RLS with no history — replaced by a **Postgres event trigger writing every DDL/GRANT change into `audit.grant_history`** (actor, object, column, timestamp). Kept Deno over Bun: Bun's RSS edge doesn't survive once the repo layer and a compiled binary land.

**2. Source of truth: Postgres domains, CHECK constraints and generated columns hold validation itself.** `information_schema` + `pg_constraint` + grants generate the base Zod schema, `db.d.ts` and the repo client. **Zod mirrors, never authors.**

**3. Stack:** runtime Deno, **`deno compile` to a static binary per pod** · HTTP `Deno.serve` · DB generated repository over `postgres.js`, no raw client reachable · forms generated-base Zod + hand `.refine()`/`.superRefine()` for cross-field, conditional and draft logic · client HTMX + Preact islands, client-bundled only · permissions column GRANT + FORCE RLS + `audit.grant_history` event trigger · migrations Atlas diffed against `information_schema` · tests `Deno.test` + golden HTML diffs + a CI schema-drift gate.

**4. Forms: two places, not three.** Existence, type, required and format live in Postgres constraints and generate Zod's base. Cross-field rules, conditional visibility and drafts stay hand-written `.refine()` on that base, CI-diffed nightly against the DB. **The raw-query bypass is gone — the repository is the only door.**

**5. Numbers:** 24,800 hand-written, 4,200 generated, **8 concepts.** Build 2.4s, incremental 0.3s, tests 0.4s. **p99 5M-row list 34ms** via repo-layer prepared statements. **RSS/pod 95MB → 61MB**: static compile drops module-resolution overhead (~14MB), a capped V8 heap cuts ~12MB, islands out of the server process cut ~8MB. Cold start 40ms → 25ms.

**6. Still worse:** 61MB RSS/pod still loses to go's 44MB — the binary narrowed the gap, it didn't close it.

### Round 4 scores (weighted /50) — **38 · 38, a tie**

**reviewer** — go 4: templ makes a missing or renamed column a build error; compiler proof beats CI-diffed drift. deno 3: bypass and audit gaps closed, but it admits two validation sites — DB constraints plus hand refines.
**agent** — go 4: the compiler threads one struct everywhere, a true self-correcting loop, but two generate steps and 1.2s tests lag. deno 5: faster loop end to end (2.4/0.3/0.4s) and the bypass an agent could exploit is now structurally closed.
**simplicity** — go 4: one struct, compiler-proven everywhere; the cost is two codegen tools and a new template language to learn. deno 3: elegant to unify in Postgres, but constraints + `.refine()` is still two validation spots, plus a new audit concept.
**performance** — go 3: lighter RSS (46 v 61) holds, but no measurement tool named in four rounds and cold start crept up. deno 4: p99 and cold-start wins are real; the RSS breakdown sums too perfectly to the gap to trust as measured.

Running: deno 148, go 110. **Go's redesign was the strongest single round it has had — sqlc and templ fixed all four things that beat it.**

---

## Round 5 — Cross-exam on the new designs, then one last iteration

### Questions

**go asks deno**
- Q1: Zod runs after a CHECK fails, but which layer produces the field-level error a 60-field form actually shows the user?
- Q2: Conditional visibility and draft logic live only in `.refine()` — so which "source of truth" governs when your DB and Zod disagree?
- Q3: Your lint rule bans raw clients today — what stops an agent adding a new repository function that skips RLS checks tomorrow?

**deno asks go**
- Q1: 40 objects × N roles × M queries — how many generated structs total, and does sqlc regenerate every one on each weekly schema drift?
- Q2: templ is a compiled DSL with its own syntax and LSP — is that in a 2030 agent's training corpus, and who maintains it if abandoned?
- Q3: Your cross-field refiners are hand-written Go, not database constraints — isn't validation-in-app-code the exact flaw you're criticising?

### Answers and final iteration

**go answers**
- A1: 40 × ~6 roles × ~5 queries ≈ **1,200 generated structs. sqlc has no incremental mode — yes, all regenerate on every weekly drift.**
- A2: templ (single maintainer, since 2023) risks thin 2030 corpus coverage; **but it compiles to plain Go, so abandoned tooling still leaves working, compiling code.**
- A3: **Fair hit — I'm moving type/required/format into Postgres domains/CHECK constraints, sqlc-read**; cross-field and draft logic stay hand-written Go, the same split deno keeps.

**go — final iteration:** sqlc now reads Postgres domains and CHECK constraints via `pg_constraint`, generating field-level validation alongside the structs, closing the app-code-validation gap; refiners shrink to true cross-row and conditional logic only. **23,200 hand-written** (refiners 700, templ views 6,000, logic 16,500), 4,400 generated, 8 concepts. Build 3.4s cold / 0.5s incremental, tests 1.2s. p99 45ms, 46MB RSS, 80ms cold start.

**deno answers**
- A1: The generator parses each CHECK expression into a Zod `.refine()` keyed to that column via `pg_constraint` — **the rule is duplicated, but generated from one source and CI-diffed.**
- A2: **Zod.** Drafts bypass DB constraints by design — partial rows must save — so visibility and draft logic is app-owned there, not mirrored from Postgres.
- A3: True — same least-privileged role, FORCE RLS applies regardless of function code. **What escapes it: `SECURITY DEFINER`, which runs as the table owner.**

**deno — final iteration:** a CI rule banning `SECURITY DEFINER` in the repo layer, closing the one real RLS-bypass path, and the CHECK→Zod mapping becomes an explicit generated catalog reviewed like the schema. **24,850 hand-written**, 4,300 generated, 8 concepts, build 2.4s / 0.3s incremental / 0.4s tests, p99 34ms, 61MB RSS, 25ms cold start.

**Note what just happened: go adopted deno's constraint-as-source design, and deno kept it. Both stacks now have the same architecture in two languages.**

### Round 5 scores (weighted /50)
deno 38 · go 25

**reviewer** — go 3: 1,200 generated structs, no incremental mode, a full weekly regen a reviewer must re-trust every time. deno 4: caught the real `SECURITY DEFINER` RLS hole nobody else found; honestly conceded the duplication and draft gap.
**agent** — go 2: 1,200 structs full-regen weekly and a thin templ corpus risks loop churn. deno 4: fastest loop plus a named, CI-enforced fix closing the one real RLS bypass.
**simplicity** — go 3: the templ DSL plus 1,200 regenerating structs still cost a head, but refiners shrank to 700 lines honestly. deno 4: one language still wins, but the `SECURITY DEFINER` ban and CHECK→Zod catalog aren't the free additions "8 concepts" implies.
**performance** — go 2: never named a measurement tool in five rounds, and left the 1,200-struct CI regen cost unpriced despite the memory win. deno 3: wins p99 and cold start but restated stale numbers and ignored the flagged 14+12+8 breakdown.

## Final: deno 186 · go 135 (of 250)

| | R1 | R2 | R3 | R4 unlocked | R5 | total |
|---|---|---|---|---|---|---|
| **deno** | 33 | 34 | 43 | 38 | 38 | **186** |
| go | 28 | 22 | 22 | **38** | 25 | **135** |

**What unlocking proved:** go's real fix existed the whole time and it never reached for it. sqlc
and templ closed all four faults in one round and tied it 38-38 — its best round of the series.
Then round 5 priced the fix: 1,200 generated structs regenerated in full every week with no
incremental mode, and a template language with a single maintainer.

**What both proved together:** after two free iterations they converged on the same architecture —
Postgres owns schema, grants, RLS and validation constraints; types and base validators are
generated out of the database; only cross-field and draft logic is hand-written; HTMX on the wire;
golden-file HTML diffs as the human gate. **The architecture was the answer. The language is the
smaller decision, and it comes down to 15MB of RAM against 11ms of p99 and a tighter loop.**
