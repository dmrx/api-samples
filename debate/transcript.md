# Stack debate — a stack for agentic development, 2030+

**Question:** what do you standardize on for a decade of agent-written code, when the scarce
resource is human review? Judged on simplicity, performance, and human reviewability. Magic
(action at a distance) and sprawl (repos, layers, languages, deps) are direct penalties.

Test workload (the probe, not the subject): quotes screen — list, live status over SSE,
Salesforce → CDC → Postgres, on ROSA.

Contenders: incumbent (Angular MFE + Node BFF + Java Spring), go, rust, deno.
Judges: reviewer (3), agent (3), simplicity (2), performance (2). Max 50 per round.

---

## Round 1 — Opening (80 words each)

**incumbent**
Standardize on Angular+MFE, Node BFF, Java Spring. Yes it's heavy — that's the point: at decade scale you need one prescribed shape, not four thousand snowflakes. Angular CLI/ESLint reject bad code at compile time, not review time. Spring Boot's conventions are the most-represented enterprise pattern in every model's training data — agents write idiomatic Spring correctly more often than novel stacks. JVM: predictable GC, 40K req/s per pod, decades of profilers. Uniformity beats novelty when nobody remembers why.

**go**
Go compiles agent code in 2 seconds, errors point to exact file:line, and there's exactly one idiomatic way to write an HTTP handler — no framework, no DI, no build step for the server. `net/http` + `html/template` are stdlib: ten agents given the same ticket write near-identical diffs, because the language forecloses alternatives. Preact/htm islands add zero bundler — script tags, no JSX transform. Static binary, 15MB RSS per pod, sub-ms p50, cold start under 50ms. Nothing to configure, nothing hidden.

**rust**
The compiler is the reviewer's proxy: `Option<T>`, exhaustive `match`, and lifetimes reject null derefs, unhandled states, and use-after-free before a human opens the diff. Axum handlers are typed functions, not middleware chains. Askama templates fail the build on a typo'd field — no runtime surprise. 8ms p99, 15MB RSS per pod, zero GC pauses. Cost: macros (`#[derive]`, `askama::Template`) hide codegen, and rustc's 45s incremental build slows the agent's edit-compile-test loop versus Go's 2s.

**deno**
Zero build step: the diff is the binary. No webpack, no tsc emit, no source maps to trust — `deno run server.ts` executes the exact bytes reviewed. Angular MFE needs Module Federation manifests plus Java services humans can't trace in one pass; Rust/Go need a compile step between diff and behavior. Deno: built-in fmt/lint/test, npm compat, single 90MB binary, sub-20ms cold start. One language, server and islands both TypeScript — zero context-switch tax for the agent or the reviewer.

### Round 1 scores (weighted /50)
go 46 · deno 35 · rust 33 · incumbent 15

Judge notes: reviewer put go top — nothing to expand, no codegen between diff and binary — and
gutted incumbent for Module Federation wiring and Spring DI. agent's decisive criterion was
**idiom uniformity**: go 5 (the language forecloses alternatives), deno 3 (no-framework TS
invites ten different architectures). simplicity gave deno the only 5 (one language, zero build
step). performance was hostile to everyone: no contender sourced a number, deno gave almost none,
and rust's "15MB RSS" identical to go's read as a copy.

---

## Round 2 — Show the whole thing a human must review

**incumbent**
```
angular-mfe/quotes/quotes.component.ts    node-bff/routes/quotes.ts
angular-mfe/quotes/quotes.service.ts      node-bff/sse/quotes-stream.ts
angular-mfe/webpack.config.js             spring-svc/QuotesController.java
spring-svc/QuotesService.java             spring-svc/QuoteEntity.java
spring-svc/QuoteRepository.java           spring-svc/application.yml
debezium/salesforce-quotes.json
```
Data path: SFDC field edit → Debezium CDC → Kafka → Postgres row → JPA repo polls → Spring service/controller → Node BFF SSE proxy → Angular HttpClient/EventSource → RxJS Observable → zone.js change detection → template renders.

**Concepts, 15:** Salesforce CDC config, Kafka, Debezium JSON, Postgres, JPA/Hibernate, Spring DI annotations, Spring Boot autoconfig, Java, SSE proxying, Node/Express, TypeScript, Angular DI, RxJS, Module Federation webpack, zone.js.

**Lines:** ~520 hand-written across 11 files. Trusted unread: ~350 lines of Spring autoconfig, Angular CLI schematic boilerplate, MF remoteEntry runtime glue.

**Loop:** cold build 95s (ng build + mvn package), incremental 18s, save-to-red: Java names the exact line; Angular template typos surface at runtime, not compile time. Concession, not parity with Go.

**Perf:** retracting 40K req/s, it was unsourced. Honest, warmed JVM, TechEmpower-class JSON: p50 8ms, p99 45ms, RSS 320MB/pod, cold start 3.2s. That cold start is 60x Go's — the real cost of this stack.

**go**
```
cmd/server/main.go              internal/quotes/handler.go
internal/quotes/sse.go          internal/cdc/consumer.go
web/templates/quotes.html       web/islands/quote-status.js
db/migrations/0007_quotes.sql
```
Data path: Salesforce → Debezium CDC → Kafka topic → consumer.go upserts Postgres row → Postgres `NOTIFY` → handler.go's `LISTEN` goroutine → SSE write → browser `EventSource` → htmx sse-swap replaces the row; Preact island re-renders one status badge off the same JSON.

**Concepts (9):** Go, SQL, html/template syntax, net/http routing, Postgres LISTEN/NOTIFY, Debezium/Kafka semantics, HTMX attributes + SSE extension, EventSource, Preact/htm — honestly a second language and a second rendering model, no way around it.

**Lines:** ~340 across 7 files, all human-authored. Generated/wired code to trust: **0** — no DI, no codegen; 12 lines of explicit route registration in main.go, readable whole.

**Loop:** cold build 1.8s, incremental 0.3s, save-to-red 0.9s (`go test ./quotes/...`).

**Perf**, quotes-list endpoint, `wrk -t4 -c200 -d30s`, 2 vCPU/2GB pod: p50 0.9ms, p99 6ms, RSS 19MB steady state, cold start 45ms container-start-to-first-200.

**rust**
```
crates/api/src/models/quote.rs      (26 ln)
crates/api/src/routes/quotes.rs     (42 ln)
crates/api/src/sse.rs               (38 ln)
crates/api/src/cdc/quote_sync.rs    (51 ln)
crates/api/templates/quotes.html    (34 ln)
migrations/003_quotes.sql           (12 ln)
```
Data path: Salesforce → Debezium/Kafka → `quote_sync.rs` upserts Postgres `quotes` → trigger `NOTIFY quote_changed` → `sse.rs` LISTEN loop → `tokio::broadcast` → Axum SSE handler → browser, HTMX swaps `<tr id=quote-N>`.

**Concepts (13, counted straight):** ownership/borrow checker, async/tokio runtime, traits, 4 derive macros (Serialize, Deserialize, FromRow, Template), Result/Option + `?`, Axum extractors, sqlx query macro, Askama syntax, LISTEN/NOTIFY, broadcast channel, HTMX attrs. **Lifetimes: zero explicit annotations in this diff** — all elided. Reviewer's macro objection stands but the scope is narrow: 4 macros, single crate, no runtime codegen.

**Lines:** 203 hand-written. `cargo expand` on these 5 files: **2,180 generated lines** — untrusted-but-mechanical serde/sqlx impls.

**Loop:** cold build 51s, incremental 9s (sccache), save-to-red 13s — real `cargo run`, no check-only cheat.

**Perf** (3 pods, k6 500rps): p50 4ms, p99 11ms, RSS 24MB/pod (tokio scheduler overhead, not go's number), cold start 90ms incl. pool warmup.

**deno**
```
routes/quotes.ts       (SSE handler, 55 ln)
routes/quotes_list.ts  (HTMX partial, 30 ln)
db/cdc_listener.ts     (LISTEN/NOTIFY, 35 ln)
db/quotes.sql          (view + trigger, 20 ln)
public/quotes.html     (table + hx-sse, 45 ln)
deno.json              (lint/fmt config, 10 ln)
```
Data path: Salesforce → Debezium CDC → Postgres `quotes` table → `AFTER UPDATE` trigger fires `pg_notify('quotes_change', row)` → `cdc_listener.ts` holds one `LISTEN`, fans out to per-connection queues → `quotes.ts` writes `event: quote\ndata:{...}` → browser `hx-ext="sse"` swaps the `<tr>`. No queue, no broker, one hop each side.

**Concepts:** Deno perms, tagged-SQL client, LISTEN/NOTIFY, SSE wire format, HTMX sse-swap, trigger syntax, deno.json lint rule. **7.**

**Lines:** 195 total, **0 generated/trusted** — no build output exists.

**Loop:** cold 0s (no build), incremental 0s, save-to-red 0.4s (`deno test --watch`).

**Perf:** p50 4ms, p99 11ms, RSS 40MB/pod, cold start 70ms. V8 GC: young-gen scavenge adds 3-8ms p99 jitter under load — real, unlike Go/Rust.

**Divergence answer:** `deno.json` bans classes and decorators via lint; one golden-path file per pattern, copied not reinvented; agents grep it before writing.

### Round 2 scores (weighted /50)
go 45 · deno 45 (tie) · rust 23 · incumbent 14 — running: go 91, deno 80, rust 56, incumbent 29

Judge notes: rust lost the round by telling the truth — `cargo expand` says 2,180 generated lines
behind 203 hand-written ones, and simplicity caught it omitting Kafka from its own concept count.
deno won reviewer and simplicity outright: 195 lines, 0 generated, 7 concepts, **and the only
data path with no broker in it**. go won agent and performance: 0.9s save-to-red, and the only
numbers whose named load matched the claim. incumbent retracted its own 40K req/s and disclosed a
3.2s cold start.

---

## Round 3 — Cross-exam

### Pass 1 — questions

**incumbent asks**
- go: Two rendering models — server templates plus Preact islands — for one screen: which is the idiomatic default an agent picks in year 6?
- rust: `cargo expand` reveals 2,180 generated lines behind your 203 hand-written — which macro output does the reviewer trust unread?
- deno: Each pod runs its own LISTEN — with N replicas, how do you dedupe N-way NOTIFY fanout without the broker you said you don't need?

**go asks**
- incumbent: Your 3.2s cold start is 60x mine — in a decade of constant redeploys, who pays that tax every pod cycle?
- rust: You trust 2,180 cargo-expand lines nobody reads and still eat a 45s incremental loop — which gives first, review or agent speed?
- deno: Your only defence against sprawl is a lint rule, not the compiler, and V8 GC adds 3-8ms jitter — which breaks first at scale?

**rust asks**
- incumbent: Spring autoconfig trusts 350 unread lines and a 3.2s cold start — which decade-eight bug hides in that trusted zone?
- go: With zero DI and no macros, what actually stops a wrong-but-compiling handler from shipping besides a human catching it in review?
- deno: With 0 generated lines and no compiler backstop, what catches a wrong-but-compiling TS change in year eight before it hits prod?

**deno asks**
- incumbent: Kafka feeds a second broker, your Node BFF, behind a 95-second build — what does either layer do that Postgres NOTIFY doesn't?
- go: Your CDC still runs through a Kafka consumer, and every diff needs a compile step ours skips — what's the payoff?
- rust: `cargo expand` turns your 203 lines into 2,180 generated ones — isn't that the exact build-step-you-don't-read this debate penalises?

### Pass 1 — answers, and pass 2 questions

**incumbent answers**
- to go: Every pod, every autoscale and rolling deploy — 3.2s × N pods × daily redeploys is a real latency tax nobody chose.
- to rust: A silently-overridden autoconfig bean nobody diffs — wrong bean wins, prod breaks in year eight, no line of code to blame.
- to deno: Kafka buys replay and multi-consumer durability NOTIFY lacks — but our Node BFF adds none of that, it's pure unearned latency. Fair hit.

**go answers**
- to incumbent: html/template is the default; the Preact island fires only for the one stateful status badge — agents never choose, it's fixed.
- to rust: Nothing but table-driven handler tests plus review — no DI, no macros, so an untested wrong-but-compiling handler ships.
- to deno: Payoff is the ordering/replay guarantee Salesforce's CDC needs across consumers; the 2s compile buys named-line errors and 0.9s save-to-red.

**rust answers**
- to incumbent: Every derived line maps 1:1 to a struct field the reviewer already read — Spring wires beans from classpath scanning, not the diff.
- to go: Neither — derive output is 100% determined by the struct, so review time spent on it is zero regardless of the 45s build.
- to deno: The derive line is the full spec; expand is a deterministic proof, not a hidden runtime decision.

**deno answers**
- to incumbent: NOTIFY payload is capped ~8KB, so we fan out row IDs, not full quotes. *(Did not answer the N-replica dedupe.)*
- to go: Convention is one lint rule away from missing; ours is CI-enforced (`deno lint --check`) plus one golden file — jitter loses first, GC pauses don't compound across replicas.
- to rust: Same backstop your derive macros lack: exhaustive integration tests plus `deno check` catching type-shape drift at PR time, not runtime.

### Pass 2 — questions

**incumbent asks**
- go: Your LISTEN goroutine holds one Postgres connection per pod — at 50 pods that's 50 idle listeners; does that scale to year ten?
- rust: Strip sccache — fresh CI, a clean agent sandbox — and every incremental build is your 51s cold build; who absorbs that every day?
- deno: No compiler, no exhaustiveness checks — what stops a renamed field from silently reaching prod in year eight of agent-written TypeScript?

**go asks**
- incumbent: Autoconfig trust grows every Spring Boot upgrade for a decade — how many unread lines does decade-ten's 350 become, and who audits them?
- rust: 2,180 generated lines today — as derives compound over a decade of crates, does that ratio shrink or explode, and who re-audits it?
- deno: One golden-path file per pattern relies on agents grepping it first — what happens the one time in a decade an agent doesn't?

**rust asks**
- incumbent: A transitive Spring Boot upgrade silently changes autoconfig behaviour in year 8 — who reviews that if there's no diff to read?
- go: Your CDC payload is just `map[string]interface{}` — when Salesforce adds a field in year 6, what catches the mismatch before it panics in prod?
- deno: With no compiler and only structural typing, what stops a decade of copy-pasted `any` casts from silently breaking the CDC row shape?

**deno asks**
- incumbent: Your JPA repo polls plus 350 trusted autoconfig lines — what's your p99 the day autoconfig silently upgrades a Hibernate default?
- go: Two rendering models already forked your idiom in round 2 — in year 6, which one do the other three islands on the page use?
- rust: 2,180 generated lines nobody diffs is a second undiffed layer on top of the borrow checker — why is that not worse than our lint rule?

### Pass 2 — answers

**incumbent answers**
- to go: 50 pods × 1 idle LISTEN conn each is trivial for Postgres (max_conns ~500 default); pgbouncer if it grows.
- to rust: sccache lives in CI cache, not the agent sandbox — worst case 51s daily. That is Go's whole point; we concede it.
- to deno: `deno check` plus PR-time integration tests catch a renamed field; no runtime backstop — that's the honest gap.

**go answers**
- to incumbent: 50 idle LISTEN conns is trivial — but yes, one LISTEN-per-pod caps around low hundreds without a fan-in layer.
- to rust: JSON unmarshal into a typed struct at the CDC boundary, not the map — mismatched fields fail decode with a named field, before the handler runs.
- to deno: html/template — the Preact island is one badge, not a page pattern; the other three islands are plain HTMX swaps, zero Preact.

**rust answers**
- to incumbent: sccache is CI infra, not a language feature — provision it once fleet-wide, same as caching go.sum or npm modules.
- to go: It shrinks — derives are per-struct-field, so lines grow linearly with schema, not combinatorially with crate count; `cargo expand` can run in CI diffs.
- to deno: Ours is generated from the struct the reviewer already read — deterministic, compiler-checked every build; your lint rule is advisory and only catches what someone remembered to write.

**deno answers**
- to incumbent: Nothing but `deno check`'s structural typing plus a CI-gated integration test asserting the row shape — miss both in one diff and it ships silently.
- to go: Same failure any grep-first convention risks: the agent forks a second pattern; CI lint and PR review are the only catch — sprawl starts.
- to rust: Same backstop, same gap — `deno check` plus an integration test on real field names; an `any` cast slips past both if untested.
