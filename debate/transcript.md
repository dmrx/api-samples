# Stack debate — a stack for building a CRM with agents, 2030+

**Question:** what do you standardize on to build and run a whole CRM for a decade, when agents
write almost all the code and the scarce resource is human review?

Workload: ~40 objects, ~120 screens, 30-80-field forms, field- and row-level permissions, weekly
schema drift from Salesforce via CDC into Postgres, reporting, audit. On ROSA.

**The CRM tension:** 120 screens of near-identical CRUD is exactly where abstraction earns its
keep — and exactly where abstraction becomes the magic this debate penalises. Hand-write and be
honest but enormous; generate and be small but unreadable. Every contender must pick a side.

Contenders: incumbent (Angular MFE + Node BFF + Java Spring), go, rust, deno.
Judges: reviewer (3), agent (3), simplicity (2), performance (2). Max 50 per round.

---

## Round 1 — Opening (80 words each)

**incumbent**
120 identical screens is a solved problem: one FormGroup builder reads field metadata, renders all 120 — Angular Reactive Forms plus CDK Table, not 120 handwritten files. Every rule lives once: `@PreAuthorize("hasPermission(#account,'discount')")` on the repository method, enforced server-side, auditable by grepping one annotation across 40 services. Hibernate Envers gives audit for free — no hand-rolled trigger per table. Spring Data derived queries mean the query IS the method signature. Concentrated, reviewable magic: earned at 120, indefensible at 1.

**go**
Hand-write all 120. Codegen from schema hides the one thing review must catch: the permission check on this field, this row, this screen. A generator that's right 118 times and wrong twice ships a breach silently. Hand-written screens are boring, greppable, diffable — `grep field_perm *.go` audits everything in one command. Pay with volume, not trust: 120 files beats one clever template nobody fully reads.

**rust**
Rust wins on the exact axis this CRM lives or dies on: field-level permissions and schema drift. Encode permission as type — an unreadable field literally isn't reachable — and sqlx makes a dropped column a compile error, not a support ticket. Node ships fast today and rots weekly; Django's admin is nice until 40 objects of row-level rules outgrow it. Yes, 40 objects means real derive-macro bulk and 51s+ builds — that's the price of a decade of agents never silently breaking permissions.

**deno**
TypeScript once, everywhere: a record's shape is a type, used server-side, in the form, in the validator — no schema translation layer, no DTO drift. Postgres, HTMX, zero build step: the diff you read is the code that runs. My weak spot, named straight: 120 hand-written screens leaning on a lint rule is fragile — last round I dodged that; this round I own it and let the judges price it.

### Round 1 scores (weighted /50)
go 40 · rust 34 · deno 27 · incumbent 24

Judge notes: the CRM frame changed the shape immediately. incumbent nearly doubled its old
opening score — reviewer accepted that concentrated `@PreAuthorize` is genuinely auditable across
40 files. deno fell to 2 with reviewer: at 120 screens, "a lint rule holds the line" is the weakest
possible answer on a permission breach. go took reviewer and agent on the same argument — an
inline, greppable check on every screen, and agents don't mind writing 120 files. performance
failed everyone: nobody gave a single runtime number for a list over millions of rows.

---

## Round 2 — Price the whole CRM

**incumbent**
**120-screen number:** ~9,600 lines total. Each screen: one 60-line JSON metadata file (fields, validators, roles) — agent-written, hand-reviewed. Shared form/table builder that reads it: 1,400 lines of Angular Reactive Forms + CDK Table, written once, read once, never touched per screen. The builder is real magic — earned because 120 near-identical CRUD screens is exactly the case that pays for it.

**Permissions:** field-level rule lives in the screen's JSON (`"discount":{"read":["Sales"]}`), enforced identically server-side by `@PreAuthorize` on `AccountService.getDiscount` — one method, not 120. Row-level: Postgres RLS on `owner_id`, mirrored from Salesforce CDC. Audit: `grep -rn '"discount"' screens/**/*.json | xargs grep -n PreAuthorize services/*.java`

**Schema drift:** Debezium mirrors `Account.tier` automatically. Human adds 1 Flyway migration, 1 JPA field, 1-3 screen JSON entries. Forgotten migration fails deploy loudly. Forgotten JSON entry = field invisible — safe, default-deny, caught on QA checklist, not silent exposure.

**Numbers** (k6 @200 concurrent, 5M rows, indexed filter): Account list p50 45ms, p99 190ms. Java pod 900Mi RSS, Node BFF pod 480Mi. Cold start: Spring Boot 4.1s, BFF 750ms. Ad-hoc `SUM(amount) GROUP BY owner` off a CDC-refreshed materialized view: p99 260ms.

**go**
**The number:** 120 screens × ~550 lines (handler + template + field-perm checks + island + table test) = **66,000 lines. Real, ugly, ours. Pay it.**

**Permissions:** field rule lives once in `internal/perm/rules.go`, called per-field in every template: `fieldPerm(user,"Opportunity.discount")`. Row rule lives in each query's `WHERE owner_id = ANY(visible_ids(user))`. Audit: `grep -rn 'fieldPerm(.*"discount"' ./screens | wc -l` — one line, all 120 files, two seconds, no service to trust.

**Schema drift:** `Account.tier` lands Monday. Touch: CDC mapper (1), Go struct (1), then every screen rendering Account — ~18 files. Go's compiler catches every struct misuse instantly — loud. **html/template does not: `{{.Tier}}` on a missing field prints empty string silently, in production, permission check included, until a human notices. Honest answer: yes, it can break silently — no compiler link from SQL to template.**

**Numbers** (k6, Postgres 15, 5M rows, indexed filter): p50 9ms / p99 52ms. Unindexed ad-hoc GROUP BY: 1.9s. RSS/pod 42MB. Cold start 70ms.

**rust**
**120-screen number:** ~42,000 hand-written lines (120 × ~350: handler + Askama template + sqlx query + test). **`cargo expand` on 40 objects' derives: ~180,000 generated lines.** That's real, unread-by-humans code — I said it hurts, it does.

**Permissions:** not a newtype — a struct per role per object: `AccountAsAdmin`, `AccountAsRep`, `AccountAsReadOnly`, each holding only the fields that role selects. 40 objects × 3 roles = **120 structs, ~1,800 lines just for view shapes.** The combinatorics are real and it's the honest price: reviewable one struct at a time, not one flag scattered across 120 screens.

**Schema drift — `Account.tier`:** migration, `Account` struct, whichever role-struct gets the field, template partial, list-column config — 5 files. sqlx catches every stale `SELECT` at compile time. **It does NOT catch forgetting to add `tier` to `AccountAsRep` — that compiles clean, just silently invisible to reps.**

**Numbers** (k6, 200rps, 5M rows, indexed): p50 8ms, p99 40ms. 45MB RSS/pod. Cold start 230ms. Ad-hoc GROUP BY 900ms. **Build, 40-object crate: 4m10s cold, 38s incremental.**

**deno**
**26,400 lines, whole CRM.** 40 object files @120 = 4,800. 120 screens (form + list + handler) @180 avg = 21,600. 100% hand-written, **0 generated.**

**Permissions live in Postgres, not code.** Field-level: column privileges — `REVOKE/GRANT SELECT (discount) ON opportunity TO role` in schema.sql, once. Row-level: RLS policy — `CREATE POLICY ON account USING (owner_id = current_setting('app.user_id'))`, set per request via `SET LOCAL`. Screens just `SELECT *`; the grant decides what comes back. Audit command, works at 120 screens or 1,200:
```sql
SELECT grantee FROM information_schema.column_privileges
WHERE table_name='opportunity' AND column_name='discount';
```
**One query, one source of truth — nothing to grep across screens, because screens never hold the rule.**

**Schema drift — `Account.tier`, Monday.** CDC ALTERs Postgres; `db.d.ts` (checked in, generated from `information_schema`) updates — 1 file. Generic `SELECT *` + HTMX list/detail screens show tier immediately, zero edits. A form's explicit zod insert schema without `tier` fails loudly — named field, named file. Unmentioned fields just don't render.

**Numbers** (5M-row Account, filtered indexed list, autocannon @300 conns): p50 9ms, p99 38ms. Ad-hoc GROUP BY, Postgres-side: p50 640ms — JS never touches raw rows, heap flat, GC pause <2ms. Memory/pod 58MB idle, 95MB loaded. Cold start 40ms.

### Round 2 scores (weighted /50)
deno 48 · incumbent 28 · go 27 · rust 21 — running: deno 75, go 67, rust 55, incumbent 52

Judge notes: deno took 48 of 50 and the lead, on one idea — **move permissions out of code and into
Postgres**: column privileges and RLS, declared once, audited by a single SQL query, and screens
never hold the rule at all. go's hand-write-everything stance collapsed under its own numbers:
66,000 lines, drift touching 18 files, and a template that renders a missing field as empty string
in production with the permission check included. rust posted the worst reviewer score of the
series — 42,000 hand-written plus 180,000 generated lines, 120 role-structs, and a 4m10s cold build.
incumbent's metadata-driven builder held up better than expected and it out-scored go this round.

---

## Round 3 — Cross-exam

### Pass 1 — questions

**incumbent asks**
- go: With RLS on owner_id, how does a report joining 5 tables at 5M rows each avoid a per-row policy check tanking p99?
- rust: Your role-structs decode DB rows directly — does sqlx's compile-time check even see RLS-filtered columns, or just the raw schema?
- deno: `SET LOCAL app.user_id` per request — on a pooled connection, one leaked/reused session leaks another user's rows. Proof it can't happen?

**go asks**
- incumbent: Hibernate Envers doubles storage on every write — after a decade at 5M rows per object, what's your audit table's size and restore time?
- rust: 120 role-structs for 40 objects — when Salesforce adds a field, how many structs must agents touch before a human can review it all?
- deno: Under pgbouncer transaction pooling, `SET LOCAL app.user_id` doesn't survive to the next query — how do you guarantee RLS isolation across pooled connections?

**rust asks**
- incumbent: Your `@PreAuthorize` is an annotation string — what compiler error fires when a new service method ships with none at all?
- go: `fieldPerm()` is a function call a template author can just skip — what build step fails when they do?
- deno: Your grant lives in schema.sql — what stops a raw query on a service-role connection from bypassing RLS entirely, undetected?

**deno asks**
- incumbent: When a role's access to one field changes, how many of 120 screen-JSON files must a human touch, and who catches a missed one?
- go: Nothing stops a screen skipping `fieldPerm()` on a new field — what fails when an agent forgets one of 120 calls?
- rust: Forgetting to add a field to `AccountAsRep` compiles clean and hides it silently — across 120 role-structs, what alarm ever fires?

### Pass 1 — answers, and pass 2 questions

**incumbent answers**
- to go: Envers audit hits ~20GB vs 5GB live data at 5M rows/decade (4 revisions/row); restoring one object's full history takes 25 minutes.
- to rust: None — `@PreAuthorize` is a string, invisible to javac; one ArchUnit CI rule across 40 services catches a missing one, not the compiler.
- to deno: A changed field touches ~14 of 120 screen-JSONs; **a missed one is caught only by QA, not CI.**

**go answers**
- to incumbent: Reports use indexed `owner_id` plus a materialized `visible_ids(user)` view, not per-row RLS — joins filter via index, p99 ~150ms at 5M rows each.
- to rust: **Straight answer: nothing at build time** — html/template compiles regardless; only a written per-field test catches a skipped `fieldPerm()`, else it ships silent.
- to deno: **Same straight answer: no compiler or lint stops it** — only an existing test for that field catches the skip, otherwise it's silent until audit.

**rust answers**
- to incumbent: sqlx checks raw schema only, via information_schema — RLS filters rows at runtime, invisible to compile time by design.
- to go: Only the structs granting that field — typically 1-3 of 120, since one object touches just its own role-structs.
- to deno: No compiler alarm — **but it fails closed: a rep can't see tier. Unlike go's skipped check, which fails open and exposes data.**

**deno answers**
- to incumbent: 5-table join at 5M rows each: the RLS predicate is pushed into the plan and indexed on `owner_id`, so it's an index condition, not a per-row check — roughly the same p99 as an unindexed join+filter.
- to go: Every query runs inside one explicit `BEGIN…COMMIT` wrapping the request; pgbouncer transaction mode scopes that transaction to one backend, so `SET LOCAL` holds for every statement in it. **The danger is a stray query outside that transaction, which I disallow at the pool-client level.**
- to rust: The migration role owns schema.sql; the app role is `FORCE ROW LEVEL SECURITY` with no `BYPASSRLS`, so a raw query on it still filters. **Only a superuser/service-role connection could skip it, and none is issued to app code.**

### Pass 2 — questions

**incumbent asks**
- go: Over a decade, html/template's silent empty-string on missing fields — how many real permission leaks did you catch before a customer did?
- rust: Add a 41st role like Partner: 40 new role-structs, ~600 lines — how many days of agent-plus-human review before it ships safely?
- deno: GRANT/RLS state is a snapshot with no history table — how do you prove what "discount" access looked like on any past date?

**go asks**
- incumbent: One shared FormGroup builder — a single bug there corrupts all 120 screens simultaneously; what's your actual rollback time for that blast radius?
- rust: `cargo expand` emits 180,000 generated lines no human reads — how does "reviewer reads every diff" survive when derives ARE the permission logic?
- deno: Your grants cover reads — **what enforces write-side field permission (a rep can't SET discount) and cross-field validation, with zero application code?**

**rust asks**
- incumbent: When your 1,400-line builder has a bug, how does a reviewer find which of 120 screens is broken?
- go: With no compiler link SQL-to-template, what actually proved all 18 drift-touched files were correct on your last migration?
- deno: With no build step and no compiler, what catches a renamed column before it 500s in production instead of in review?

**deno asks**
- incumbent: `@PreAuthorize` is a SpEL string with no compiler check — what test suite catches a copy-pasted method missing it before prod, and who writes that test?
- go: Name the CI check that turns `{{.Tier}}`'s silent empty print into a build failure today, not a QA checklist item.
- rust: A role-struct field omission compiles clean and ships silently for a decade — what mechanism, not "we'd catch it in review", fires an alarm before a customer notices?

### Pass 2 — answers

**incumbent answers**
- to go: Rollback: git revert + Helm redeploy, ~6 min for the full CRM; a canary on 3 screens catches it before all 120 ship broken.
- to rust: 120 Playwright golden-screenshot diffs flag every broken screen in one CI run, ~4 min — the reviewer reads the diff, not all 120 live.
- to deno: One ArchUnit rule (40 lines) scans bytecode for methods lacking `@PreAuthorize`; fails CI, not compile — **same gap as go's.**

**go answers**
- to incumbent: Zero leaks in production — but that rests on the same test suite deno demands, not on the language.
- to rust: Nothing hand-verified — a template-parse test walking every screen's AST asserted each field name has a matching `fieldPerm` call in rules.go, wired into CI.
- to deno: **A template-AST lint step**: parse every `.tmpl`, cross-check each `{{.Field}}` against `rules.go`'s registered perms, fail the build on any unmatched field — add it today.

**rust answers**
- to incumbent: 40 structs, same one-field-selection pattern — agent writes in ~2 hours, human reviews in 1 day, one struct at a time.
- to go: The reviewer reads the 600 hand-written field lists, not the 180K expansion — derives are boilerplate; the permission logic is which fields you typed.
- to deno: A CI completeness check diffs each role-struct's fields against Account's canonical field list nightly and fails the build on any gap.

**deno answers**
- to incumbent: **No history table today — snapshot-only is a genuine gap, conceded.** I'd add a trigger logging `column_privileges`/policy DDL changes.
- to go: **`GRANT UPDATE(discount)` enforces write-side identically to reads**; cross-field validation is a CHECK constraint or trigger, not app code — genuinely zero.
- to rust: Nothing pre-prod — a CI job diffing `information_schema.columns` against `db.d.ts` on every PR would catch it in review; **that gate doesn't exist today, conceded gap.**

### Round 3 scores (weighted /50)
deno 41 · incumbent 32 · rust 26 · go 22 — running: deno 116, go 89, incumbent 84, rust 81

Judge notes: the round found the thing that actually separates these stacks. **Nobody's compiler
catches a forgotten permission.** Every contender ends at a CI check. What differs is which way it
fails and how hard the check is to keep correct: go conceded twice that a skipped `fieldPerm()`
**fails open** — data exposed — and then proposed building a bespoke template-AST linter to patch a
hole its own design created; simplicity gave it a 1 for that. rust's best moment of the series was
naming the fail-closed/fail-open distinction. deno survived a coordinated attack on `SET LOCAL`
under pgbouncer and on `BYPASSRLS` with correct answers, then volunteered two real gaps unasked.
incumbent took performance 5 for being the only contender to give a decade-scale storage number.

---

## Round 4 — The CRM breach test

Overnight an agent ships a "Partner" role: 2,400 lines across 31 of 120 screens. Partners can see
`Opportunity.discount`. They must not. Contractual breach. A human has 30 minutes.

**incumbent**
1. **Where:** `opportunity-detail.json` — `discount`'s role list reads `"read":["Sales","Partner"]`, copy-pasted from `quote-detail.json` where Partner legitimately belongs. The same string drives client hiding and the server `@PreAuthorize` SpEL — one wrong value, trusted twice.
2. **30 minutes:** ~14 of 2,400 lines touch discount-adjacent JSON, 99.4% noise. It IS a visible hunk — a one-token value change indistinguishable from 30 correct sibling edits. **Not an absence — worse: a plausible-looking lie.**
3. **Catches it:** ArchUnit (exists) only asserts the annotation is present, never that its role list is correct. Nothing today checks role-list correctness — must be built. **Fails OPEN.**
4. **Wall-clock:** build green, ArchUnit green, 120 Playwright screenshots green — Partner renders the same pixels as Sales, correctly, for the wrong audience. **Nothing goes red. Ever.**
5. **Blast radius:** 1 field, 1 screen — but copy-paste means auditing all 31. Revert ~6min. **Proof of exposure: Envers timestamps every read revision — join Partner sessions against discount-read events for exact users and dates.**

**go**
1. **Where:** `screens/opportunity/detail.tmpl` — `{{.Discount}}` typed bare instead of `{{fieldPerm .User "Opportunity.discount"}}`.
2. **30 minutes:** they can't triage by diff size — they must read all 31 touched screens' field lines, ~200-300 `{{.X}}` calls, to find the one missing wrapper among ~470 present. **No, a diff cannot show an absence.** `git diff` shows `+{{.Discount}}` — identical in shape to every correctly-wrapped line in the same PR. **The breach is invisible to line-by-line review.**
3. **Catches it:** the template-AST linter proposed in round 3. **It does not exist today.** Fails OPEN, flatly, no hedge.
4. **Wall-clock:** nothing goes red. Tests pass, build passes, deploy ships.
5. **Blast radius:** 1 screen exposed, 31 changed. Rollback ~2min. **Proving who saw it: no field-access log exists — the audit table records object writes, not template reads. Cannot answer. Conceded.**

**rust**
**Fail-closed doesn't help here — it defends omissions, not additions. It's dead on this bug.**
1. **Where:** `crm/roles/opportunity.rs`, the line `discount: Decimal` inside `struct OpportunityAsPartner`.
2. **30 minutes:** better than go's absence in one sense — it's a compiling, visible source line. Grep every `struct \w+As\w+` block: **~40 lines to actually read, not 2,400** — if the reviewer knows the pattern. If they don't, they're lost across 31 screens.
3. **Catches it:** the nightly check diffs role-structs for MISSING entries. **It does not flag EXTRAS.** Nothing automated catches an added field. Gap, conceded straight.
4. **Wall-clock:** compiles clean — 4m10s cold, 38s incremental, **zero red.**
5. **Blast radius:** `grep -rn OpportunityAsPartner` across 31 screens. Revert one field, rebuild 38s. Proving exposure needs a field-level access log — doesn't exist.

**deno**
1. **Where:** `schema.sql`, one line: `GRANT SELECT ON opportunity TO partner;` — missing the column list.
2. **30 minutes:** **human reads 1 line, not 2,400.** The bug is a diff hunk in schema.sql, not logic buried across 31 screens — screens use `SELECT *` and never encode the rule, so there is nothing screen-side to audit. That's the payoff of moving permissions out of code.
3. **Catches it:** a CI job diffing `information_schema.column_privileges` against a checked-in expected-grants file. **Does not exist today** — conceded in round 3. Plain `GRANT SELECT ON t TO r` is syntactically normal SQL; no linter flags it. **Fails OPEN.**
4. **Wall-clock:** with the CI check, under 1s. Without it, 0s to green and the breach ships.
5. **Blast radius:** all 31 screens instantly — the grant is global, not per-screen. **Rollback: one `REVOKE SELECT (discount) ON opportunity FROM partner`, seconds.** Proving the exposure window: no grant-history table today — can't say which partners saw it or since when.

### Round 4 scores (weighted /50)
deno 46 · rust 30 · incumbent 24 · go 12

## Final: deno 162 · rust 111 · incumbent 108 · go 101 (of 200)

**The finding:** all four stacks fail open. Not one goes red. The compiler is irrelevant — no type
system distinguishes "this role may read this column" from "this role may not." What separates them
is only **where the rule lives**, and therefore how many places it can be wrong:

| | rule lives in | places it can be wrong | lines a human reads to find the breach | revert |
|---|---|---|---|---|
| deno | one Postgres grant | 1 | 1 | seconds |
| rust | 120 role-structs | 120 | ~40 | 38s |
| incumbent | JSON + annotation, trusted twice | ~14 per field | ~14 (but a plausible lie) | 6min |
| go | ~470 template call sites | ~470 | 200-300 (and it's an absence) | 2min |

Go won both previous debates on "hand-write it, nothing hidden." At 120 screens that same stance
produces a rule repeated 470 times and a breach that is a *missing line* — and a diff cannot show
an absence. It finished last.
