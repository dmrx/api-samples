# Stack debate — quotes screen

Scenario: quotes list, live status via SSE, Salesforce → CDC → Postgres, deployed on ROSA.
Contenders: incumbent (Angular MFE + Node BFF + Java Spring), go, rust, deno.
Judges: everyman (1), reviewer (2), agent (2), operator (1).

---

## Round 1 — Opening (80 words each)

**incumbent**
Angular's federated remote loads live status for 500 quotes and re-renders the list on every SSE tick because zone.js patches the EventSource callback into change detection — CDR runs, diffs the whole tree, not just the changed row. Spring Boot core service starts in 3-4s cold, hot-reloads in under 2 via DevTools, and its Postgres CDC consumer is a proven Debezium pattern already running in prod. Module Federation's remoteEntry.json versioning gives one-command rollback per micro-frontend — smallest blast radius of any option here, and OpenShift already runs this exact topology today.

**go**
Go compiles this whole screen in under a second, ships a static binary, one process on ROSA. html/template renders the list server-side; HTMX swaps rows over SSE with zero client router. One Preact/htm island for live-updating badges — no bundler, no npm install, no build step at all: `<script type="module">` from a CDN or vendored file. Reviewer reads Go top to bottom: no macros, no generics gymnastics, no ORM magic — pgx queries are plain SQL you can paste into psql. Standard library `net/http` means fewer dependencies to patch by 2031.

**rust**
Rust compiles to one static binary — no JVM, no node_modules, no npm audit churn. Axum's type system makes handler bugs compile errors, not 2am pages. Askama templates compile against your Rust types: rename a field, template fails to build — the compiler is your reviewer's second pair of eyes. HTMX swaps HTML fragments over SSE; zero client state to debug. tokio-postgres streams CDC rows with no GC pause, no OOM-killed pod. One Cargo.lock, one deletable crate, ROSA-friendly minimal container image (~20MB).

**deno**
Deno ships TypeScript-native: no tsc, no webpack, no bundler config to review — `deno run server.ts` is the entire build. That's zero build-artifact diffs, ever (principle 8, 10). Standard library, no node_modules tree of 40,000 files (rust and go don't even have that problem, but incumbent's Angular MFE does). Deno.serve + HTMX fragments over SSE: one process, one language front-to-back, no BFF translation layer (principle 5, 6). `deno test` needs no config. Upgrading Deno is one binary swap — no lockfile hell across 200 npm packages.

### Round 1 scores (weighted /30)
go 26 · deno 23 · rust 20 · incumbent 12

Judge notes: everyman punished incumbent's full-tree re-render and marked deno + rust down for
ignoring bad-network behaviour. reviewer rewarded deno (nothing hidden) and dinged rust for
Askama macro expansion, incumbent for zone.js runtime wiring. agent rewarded go's sub-second
build, dinged rust for slow compiles. operator rewarded rust's 20MB binary, dinged go's
CDN-fetched Preact island and deno for giving no pod numbers.

---

## Round 2 — Scenario: layout, data path, numbers (150 words each)

**incumbent**
```
quotes-mfe/src/app/quotes/
  quotes.component.ts (118)
  quotes-list.component.ts (86, OnPush, trackBy)
  quote-row.component.ts (52)
  quotes.service.ts (71, EventSource)
  quote.model.ts (18)
  quotes.module.ts (24)
bff/src/routes/quotes.ts (95)
bff/src/sse/quotes-stream.ts (63)
core-svc/src/main/java/com/acme/quotes/
  QuoteController.java (110)
  QuoteCdcListener.java (88)
  Quote.java (40)
```
Total 765 lines, 3 repos.

Trace: Salesforce quote update fires Platform Event → Debezium CDC connector replicates row into Postgres `quotes` via WAL → QuoteCdcListener.java polls slot, publishes to internal Kafka topic → QuoteController.java streams SSE to BFF → quotes-stream.ts relays SSE to browser → quotes.service.ts EventSource wrapped in ngZone.runOutsideAngular → quote-row.component.ts (OnPush, trackBy id) patches one row.

Admission: Round 1's full-tree diff was real — 500-row CDR cost ~40ms per tick. Fixed here with OnPush + trackBy + runOutsideAngular: single-row patch, under 2ms.

Cold build (mvn+ng+tsc, parallel): 52s. Incremental (ng serve HMR + Spring DevTools): 1.9s.

**go**
```
quotes/
├── main.go
├── handlers.go
├── db.go
├── cdc.go
├── sse.go
├── templates/
│   ├── list.html
│   └── row.html
├── static/
│   ├── htmx.min.js
│   └── badge.js
└── quotes_test.go
```
All vendored into `//go:embed`, zero CDN calls — fixed since round 1.

Trace: Salesforce Platform Event fires on quote change → Debezium CDC connector writes the row into Postgres `quotes` inside one transaction → trigger does `NOTIFY quotes_channel` → `cdc.go` goroutine already `LISTEN`ing wakes, reads the changed row with pgx → `sse.go` renders `row.html` (html/template, ~12 lines) → writes `event: quote-update\ndata: <tr>...` onto the open ResponseWriter → browser's native EventSource, no client framework, hands the chunk to HTMX's SSE extension, which swaps `<tr id="q-123">` in place; badge.js Preact island re-paints just the status pill.

Numbers: 640 lines total for the screen, 1 repo, cold build 0.9s, incremental 0.2s.

**rust**
```
quotes/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── routes/quotes.rs
│   ├── sse.rs
│   ├── cdc.rs
│   └── db.rs
└── templates/
    ├── quotes_list.html
    └── quote_row.html
```
Salesforce Platform Event → Debezium connector → Postgres logical replication writes `quotes` row → trigger fires `NOTIFY quotes_changed` → `cdc.rs`'s `PgListener` task wakes, pushes id onto a `tokio::broadcast` channel → `sse.rs` handler re-queries that one row, renders `quote_row.html` (Askama, checked against the struct at compile time — that's the "hidden macro" reviewer flagged: 40 lines of generated `impl Display`, visible via `cargo expand`, not runtime reflection) → HTMX swaps the `<tr>` by `hx-swap-oob`, no full re-render, no client JS.

1 repo, 1 crate, 460 lines for this screen (routes 140, sse 60, cdc 90, db 70, templates 100). Cold build (clean target, 38 crates): 51s. Incremental (`cargo check` after a template edit, cache warm): 1.6s.

**deno**
```
quotes/
  server.ts        (61 lines — routes + SSE hub)
  db.ts            (37 lines — pg.Pool, LISTEN)
  sse.ts           (23 lines — client Map, broadcast)
  views/
    list.ts        (30 lines — html`` template)
    row.ts         (17 lines — html`` template)
  static/htmx.min.js (vendored, 14KB, no CDN)
  quotes_test.ts   (46 lines)
```
Total: 214 lines, one repo, zero package.json.

Data path: Salesforce quote update fires Platform Event → CDC connector writes row into Postgres `quotes` inside one transaction → trigger does `NOTIFY quotes_channel, id` → db.ts's single `LISTEN` connection wakes → server.ts re-selects that row, renders row.ts, pushes `event: quote-update` + `hx-swap-oob` fragment to every socket in sse.ts's Map → browser replaces `<tr id="q-123">` in place.

Numbers: 2 pods, 128Mi/50m each, no sidecar, no init container. Cold build (`deno cache` + `deno check`): 2.1s. Incremental: 140ms. `deno test`: 0.4s, no watcher config.

### Round 2 scores (weighted /30)
go 27 · deno 25 · rust 19 · incumbent 9 — running: go 53, deno 48, rust 39, incumbent 21

Judge notes: every judge asked the same unanswered question — **what happens when the connection
drops?** Nobody has answered it. reviewer rewarded deno's 214 lines / 1 repo, gutted incumbent
for 765 lines / 3 repos / hidden Kafka hop. agent called rust's `cargo check` incremental number
a cheat. operator counted moving parts and put go first.

---

## Round 3 — Cross-exam

### Pass 1 — questions

**incumbent asks**
- go: cdc.go's NOTIFY carries only an id — two updates before the goroutine reads: does the second silently vanish?
- rust: `cargo check` only type-checks — what's your real incremental time to a rebuilt, running binary?
- deno: Your SSE client Map is per-pod, in-memory — after a dropped connection reconnects, full resync or nothing?

**go asks**
- incumbent: You have Postgres NOTIFY already; why still hop through internal Kafka in QuoteCdcListener? What does it buy you?
- rust: tokio::broadcast drops messages under lag with no replay buffer — how does a reconnecting client recover missed row updates?
- deno: Your SSE hub relies on one LISTEN connection — when it drops, how do connected browsers recover the rows they missed?

**rust asks**
- incumbent: Kafka relays SSE to your BFF — when that pod restarts mid-stream, how does the browser recover updates it missed?
- go: Your single LISTEN connection reconnects after a drop — how does a client learn which quotes changed during the gap?
- deno: Each pod holds its own LISTEN connection with no shared broker — after a drop, how do you replay the missed rows?

**deno asks**
- incumbent: Your SSE relay crosses Kafka, BFF, and ngZone — when any hop drops, how does the browser know it missed an update?
- go: Your LISTEN is one Postgres connection — when it drops, how do you detect and backfill quotes changed during the gap?
- rust: Agent called `cargo check`'s 1.6s a cheat — what's your real edit-test loop time including tests, not just type-check?

### Pass 1 — answers, and pass 2 questions

**incumbent answers**
- to go: Kafka buys replay and fan-out to other consumers, not this screen — for one screen it's pure overhead we'd cut.
- to rust: Nothing today — BFF restart loses in-flight SSE and the browser reconnects blind, no gap-fill exists.
- to deno: It doesn't — no sequence numbers or Last-Event-ID anywhere in the chain, so any hop drop is silently lost.

**go answers**
- to incumbent: No: NOTIFY just triggers a fresh SELECT on that id; the second update is what gets read — nothing vanishes.
- to rust: Client resends Last-Event-ID (row's `updated_at`) on reconnect; server runs `SELECT WHERE updated_at > $1` to backfill before resuming.
- to deno: pgx connection error triggers reconnect + re-LISTEN; handler then runs `SELECT WHERE updated_at > last_watermark` to backfill missed rows.

**rust answers**
- to incumbent: Real incremental `cargo build` to a running binary: 4.8s, not 1.6s — restart near-instant; honest number, not a cheat.
- to go: tokio::broadcast has no replay buffer — on reconnect, client sends last-seen id, server re-queries Postgres for every row changed since.
- to deno: Full loop — edit, `cargo build`, `cargo test`, rerun binary — is 6.3s real: 4.8s build plus 1.5s for 40 tests.

**deno answers**
- to incumbent: Neither — EventSource auto-sends Last-Event-ID on reconnect; server replays `SELECT * FROM quotes WHERE updated_at > $1`, then resumes stream.
- to go: Map holds only sockets, not state; on LISTEN reconnect db.ts requeries `WHERE updated_at > watermark`, backfills the gap via SSE.
- to rust: No broker needed — Postgres is the source of truth; each pod's browsers resend Last-Event-ID, server catches up via one indexed range query.

### Pass 2 — questions

**incumbent asks**
- go: Fixing your NOTIFY drop needs a resume cursor and backfill query — where does that code live, and does 640 lines still hold?
- rust: Fixing broadcast's lag-drop needs a replay buffer keyed by row version — where's that code, and does 460 lines still hold?
- deno: Fixing per-pod LISTEN drops needs a shared broker or backfill query — where's that code, and does 214 lines still hold?

**go asks**
- incumbent: Does QuoteCdcListener persist a per-row offset, or does a Kafka rebalance mid-stream silently drop in-flight quote updates?
- rust: tokio::broadcast has no replay buffer — do you persist a version column for backfill, or is that still unimplemented?
- deno: With 2 pods and no shared broker, does a reconnecting browser's backfill query even hit the pod that missed the NOTIFY?

**rust asks**
- incumbent: Kafka pod restart mid-stream — give a row count: how many quote updates can vanish before your backfill query even fires?
- go: Your NOTIFY carries no ordering guarantee across two connections reconnecting simultaneously — do both replay the same rows twice, or silently drop one?
- deno: Two pods, two independent LISTEN connections, no shared broker — does a browser pinned to pod A ever see pod B's updates?

**deno asks**
- incumbent: Given three independent hops (Kafka, BFF, ngZone) each needing resume logic — which single component owns detecting a missed update, and how?
- go: Postgres NOTIFY fans out to every LISTENer — with 3 replica pods, does each independently requery and re-render on every single quote update?
- rust: tokio::broadcast has fixed capacity — when a slow task hits Lagged, do you detect it and backfill, or silently miss updates?

### Pass 2 — answers

**incumbent answers**
- to go: No per-row offset persisted — Kafka auto-commit means a mid-poll rebalance can silently drop in-flight updates today.
- to rust: Up to 500 — one uncommitted Kafka poll batch — could vanish before any backfill query, which doesn't exist yet.
- to deno: None owns it today; the browser's EventSource Last-Event-ID would be the natural single point, but it's unimplemented.

**go answers**
- to incumbent: No such file yet — add cursor table + backfill `SELECT WHERE updated_at > $1`, ~40-60 lines: 640 becomes ~700.
- to rust: Neither — re-SELECT is idempotent on primary key, so duplicate NOTIFYs just re-render the same row twice, no drop.
- to deno: Yes, each pod's LISTENer fires independently — 3x redundant re-SELECT/render per update; needs a shared listener or leader election.

**rust answers**
- to incumbent: Backfill lives in sse.rs's reconnect handler, ~15 lines for `SELECT WHERE updated_at > $1` — 460 becomes 475.
- to go: No new column — reuse existing `updated_at`; match `Err(Lagged(n))` on recv, then backfill via the same range query.
- to deno: Detected, not silent — `recv()` returns `Err(Lagged(n))` explicitly; that triggers the Postgres backfill, no updates lost.

**deno answers**
- to incumbent: Backfill lives in db.ts (+9) and server.ts (+6) reading Last-Event-ID — 214 becomes ~229 lines, still smallest.
- to go: Postgres NOTIFY fans to both pods' LISTEN connections equally, and backfill re-queries Postgres directly, not pod-local state.
- to rust: Yes, pod A sees pod B's updates too — NOTIFY broadcasts to every LISTENer, so both pods stay in sync automatically.

### Round 3 scores (weighted /30)
rust 29 · deno 23 · go 22 · incumbent 10 — running: go 75, deno 71, rust 68, incumbent 31

Judge notes: rust won the round by conceding its build number upward and by having the only
*typed* failure signal (`Err(Lagged(n))`). go lost ground admitting 3 replicas triple-render every
update and may need leader election. incumbent conceded it can silently lose 500 rows on a Kafka
rebalance. Three points separate the top three going into the stress round.

---

## Round 4 — Stress: an agent writes `>=` instead of `>` in "expires after 30 days"

**incumbent**
1. Where: `core-svc/.../QuoteController.java` (or `Quote.java`'s `isExpired()`) — the canonical check. Duplicated in `quotes-list.component.ts`'s status pipe and `quote-row.component.ts` badge logic, since Angular re-derives "Expired" client-side for the SSE tick. Three copies, two languages, one number, no shared constant.
2. Caught by: nothing structural. Java and TypeScript don't see `>=` vs `>` as wrong — both compile clean. Only a boundary-value JUnit test (`createdAt = now - 30d exactly`) catches it, and only if someone thought to write it. Realistically: a human reviewer eyeballing the diff, or a customer filing a ticket.
3. Wall-clock: targeted `mvn test` on core-svc (Spring context boot 3-4s) ≈ 15s isolated; full pipeline — cold build 52s plus JUnit/Karma across three repos — ~110s to a red CI signal.

**go**
1. Lands in `sse.go` (round-3 backfill query) and `handlers.go`'s list-render path — the `30` appears twice: SQL `WHERE created_at > NOW() - INTERVAL '30 days'` and the Go helper feeding `row.html`'s `{{if .Expired}}`. No third place — badge.js just paints the class Go already decided.
2. No compiler catches it — Go's type system doesn't know `>=` from `>`. Caught by `quotes_test.go`: a table test with a quote timestamped exactly 30 days ago, asserting not-expired. That's it — review might miss it; production ticket is the failure mode if the test wasn't written.
3. Save file → `go test ./...` — incremental 0.2s compile + ~0.3s run. Red in under 1 second.

**rust**
1. Where: `sse.rs`'s backfill query and `cdc.rs`'s freshness check — both would use `>=`. Also `db.rs` if it computes expiry directly. The 30 lives nowhere else — no SQL constant, no Askama logic; templates render a bool the struct already computed.
2. Caught by: not the compiler — `>=` type-checks fine, `Duration` or not. A `const EXPIRY: Duration = Duration::days(30)` centralizes the number but not the operator. What catches it: a unit test asserting a quote created exactly 30 days ago is NOT expired. That test is the spec (principle 9) — compile succeeds, `cargo test` fails, red line names the exact assertion.
3. Wall-clock: `cargo build` 4.8s, `cargo test` runs the boundary case — 6.3s total to a named failing assertion.

**deno**
1. `views/row.ts:17` — `expiresAt < now` computed here. The same 30 appears twice more: db.ts's `SELECT WHERE created_at > now() - interval '30 days'` and quotes_test.ts's fixture. Three places, one repo — no template layer hiding it.
2. No compiler — TS's type is `boolean`, not a range. Caught by quotes_test.ts: boundary case `createdAt: now - 30d exactly`, asserting `expired === false`. Off-by-one on `>=` flips that assertion red. Review might catch it; the test is what actually fails.
3. Agent saves file, `deno test` runs: 0.4s to red. Incremental typecheck (140ms) shows nothing — types pass, logic doesn't. Real number is 0.4s, not 140ms — the loop is the test suite, not the compiler.

### Round 4 scores (weighted /30)
go 28 · deno 26 · rust 21 · incumbent 8

## Final: go 103 · deno 97 · rust 89 · incumbent 39 (of 120)
