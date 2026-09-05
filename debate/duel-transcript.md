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
