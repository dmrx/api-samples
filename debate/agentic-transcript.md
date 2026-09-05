# Greenfield agent-native CRM, 2030 — find the ideal stack

Fixed: Postgres as system of record, MCP as the agent tool surface, intents not CRUD, three systems
kept apart (record / interaction / reasoning), a graph of durable entities, every state change an
event, an agentic SDLC over machine-readable artifacts. Judged on the owner's three principles:
simplicity, efficiency, readability.

Contenders: **maximal · postgres-first · event-log · graph-native**
Judges: reviewer (3), sdlc-agent (3), simplicity (2), efficiency (2). Max 50 per round.

---

## Round 1 — Show the stack

**maximal**
**Kept, 11:** Postgres · MCP · React/TS · S3 · OTel · K8s · GitHub · LLMs · pgvector · GraphQL+REST (two thin surfaces over one service layer, not two stacks) · Cedar/OPA (*"agents mutate autonomously; `Org.discount` visibility can't live in app-code ifs when five agent types and humans all call the same tool — needs one declarative, testable, auditable source"*).
**Deferred, 6, with triggers:** Kafka — outbox + LISTEN/NOTIFY until >5k events/s sustained. OpenSearch — pgvector + FTS until search p99 >500ms or facets exceed SQL. Temporal — *"MCP tool calls are short-lived; retries belong in the agent loop"* — until a workflow spans >1 human-day or needs durable compensation. Snowflake — read replica + nightly dbt until analytics degrade OLTP. LangGraph/ADK — a loop plus MCP calls is enough for six tools. Okta/Auth0 — OIDC covers it until the first enterprise buyer.
**Runtime probe:** `search_customers({risk:"churn"})` → RiskService → one query on a materialised view (Activity/Outcome recency + pgvector similarity to churned cohorts) → Postgres + pgvector only → p99 ~80ms → ranked list + score + contributing signals.
**SDLC probe:** migration, domain function, MCP tool def, GraphQL SDL field, REST endpoint, Cedar policy line, integration test, security test. **7 files, 4 languages** (SQL, TS, SDL, Cedar). ~3 hours.
**Authorization:** one Cedar policy file, evaluated at the MCP boundary; GraphQL/REST call the same check.
**Numbers:** 11 boxes, ~9 concepts, **3 pageable (Postgres, K8s, Cedar service)**, ~$800/mo.
**Weaknesses:** GraphQL + REST is two schemas kept in sync by hand. *"Cedar adds a DSL and a service most 6-tool CRMs could still do with column GRANTs + RLS at this scale."*

**postgres-first**
**Boxes, 7:** Postgres · TS backend · React/TS · MCP · K8s · GitHub · OTel. Postgres replaces: events → outbox + logical replication; search → FTS + pgvector; policy → GRANTs + RLS; audit → append-only tables; queues → SKIP LOCKED / pg-boss; analytics → read replica.
**Runtime probe:** `analyze_account`-style tool → `ChurnService.findAtRisk()` → one query on `mv_account_health` (activity recency, case volume, renewal window, FTS sentiment on notes) joined to pgvector similarity on churned accounts, refreshed by pg_cron every 15 min → **p99 45ms (pgbench, 5M rows, indexed)** → `{org_id, score, top_factors[], similar_churned[]}`.
**SDLC probe:** `health_score.sql`, `health.ts`, `mcp/tools.ts`, `mv_account_health.sql`, `health.test.ts`, `rls_policy.sql`. **6 files, 2 languages (SQL, TS).** Half a day, one PR, one reviewer.
**Authorization:** one place — an RLS policy + column GRANT on `orgs`, enforced at the DB role every service connects through.
**Triggers:** Kafka at outbox lag p99 >5s or >3 consumer services. OpenSearch at FTS p99 >300ms or fuzzy/multilingual. Cedar at >50 rules or cross-service. Snowflake at >30s analytics on the replica. **Temporal: "pg-boss handles cron + retries, but multi-week human-in-the-loop with compensating actions is the honest gap — Temporal earns its slot day one for renewals and escalations, not deferred."**
**Numbers:** 7 boxes (8 with Temporal), ~10 concepts, **3 pageable (Postgres, K8s, Temporal)**, ~$900/mo.
**Weaknesses:** pgvector ANN degrades past ~10M vectors without tuning; RLS policies scattered across tables get hard to audit past ~30 tables.

**event-log**
**Log lives in Postgres.** One `events` table (entity_type, entity_id, type, payload jsonb, occurred_at, actor), append-only, indexed on (entity_id, occurred_at). Projections are materialised views/tables refreshed by one TS projector consuming LISTEN/NOTIFY. *"Kafka buys nothing at 500 users / 5M entities and adds partition, rebalance and schema-registry pages for zero throughput need."* **Boxes: 4** (Postgres, MCP server, domain services, projector).
**Runtime probe:** projection query on `org_health_projection`, not a log scan → p99 ~45ms → **plus a `history` array (last 20 events by entity) from the same table — real history, cheaply, which the others can't surface as naturally.**
**SDLC probe:** both a new event type `HealthScoreComputed` and a projection column. Event Zod schema, migration, projector handler, service, MCP return type, RLS row, tests. **3 languages.** 5M entities: no full replay — a backfill job emits the event from current state, ~20 min at 5k/s. One day.
**Authorization:** *"Not in the log. RLS sits on projections. Raw events carry full payloads (`DiscountChanged{old,new}`) unmasked — so auth is enforced twice, once on projections and once on the events table. That duplication is a real, permanent tax."*
**Numbers:** 4 boxes; concepts event/projection/projector/replay; **4 pageable (projector lag, out-of-order apply, migration, Postgres)**; replay ~20 min; ~$1,100/mo.
**Weaknesses:** projection drift when the projector dies mid-batch — "which is truth" becomes a live debugging question. *"Routine additive fields still pay full event + projector ceremony for something postgres-first gets with one column and a trigger."*

**graph-native**
**Choice: Postgres + a typed edge table** (`relationships(from_id, to_id, type, valid_from, valid_to, props)`) + recursive CTEs. **Not AGE** (openCypher, a second catalog, unreplicated extension state, thin managed-cloud support — a ninth box for one query shape). **Not a graph DB** (dual-write, no ACID join, a new pager). **Boxes: 8, zero added over postgres-first. One query language: SQL.**
The real pain in plain SQL: *"all Orgs reachable via ownership/reseller chains, cycle-safe, ≤4 hops"* — a hand-unrolled 4-way self-join, or 12 lines of recursive CTE with a `visited` array.
**Runtime probe:** recursive CTE over `relationships` (Org → Contract → Case/Activity) aggregating open cases, activity decay, renewal proximity; cycle guard `NOT id = ANY(path)`. **p99 ~55ms at depth ≤3 with a `(from_id, type)` index; unbounded depth is the failure mode.**
**SDLC probe:** health score is a **node property** — *"it's account state, not a relationship."* migration, service, MCP schema, RLS policy, React badge, tests. 3 languages. ~1 day.
**Authorization — the traversal question:** *"RLS composes — Postgres re-checks the policy per row inside recursion, so Org B rows vanish and the walk dead-ends there. But the edge row itself needs its own RLS too, or the agent sees that an edge exists without its target — a leak. Concede: needs deliberate double-policy design, not free."*
**Weaknesses:** recursive CTEs past 3 hops or multiple edge types get unreadable — fails the reviewer. No native graph algorithms; pgRouting later.

### Round 1 scores (weighted /50)
postgres-first 45 · graph-native 31 · maximal 30 · event-log 20

Judge notes: postgres-first won three of four judges on two numbers — **2 languages** for the SDLC
probe and **one authorization site**. maximal cut 6 of its own 17 boxes with named triggers, then
kept Cedar and GraphQL+REST while listing both as weaknesses. event-log took 2s across the board:
authorization enforced twice, and the most ceremony of anyone for an additive field. graph-native
chose plain SQL and zero added boxes, but omitted cost and pageable counts and named its own p99
cliff — an unbounded traversal an agent can trigger.

---

## Round 2 — Cross-exam

### Questions

**maximal asks** — postgres-first: You conceded RLS across 30+ tables is hard to audit, and you have Temporal on day one with no Kafka — who feeds it, and isn't that just hand-rolled Kafka? · event-log: Raw events carry unmasked `DiscountChanged{old,new}` forever — how do you delete a person from an append-only log under GDPR erasure? · graph-native: Reviewer flagged unreadable CTEs past 3 hops — what's the deepest hop count any real CRM intent needs, and if it's 2, why keep the edge table at all?

**postgres-first asks** — maximal: You keep Cedar and GraphQL+REST while calling both weaknesses — pick one to cut today, and why is your churn p99 80ms when the identical Postgres + pgvector query runs at 45ms? · event-log: A field turns sensitive next year — you re-mask the projection, but the unmasked old/new payload is already permanent in the events table; how do you unleak it? · graph-native: At 7 entity types and N edge types, every edge needs its own RLS alongside its node's — how many policies is that, and who signs off on each?

**event-log asks** — maximal: You deferred Kafka to >5k events/s — until then, where do agent-triggered state changes live so "why did X drop" is answerable at all? · postgres-first: Your audit table is append-only rows, not events — can it answer "state of Org X on March 3rd" without hand-replaying every table's changes? · graph-native: `valid_from/valid_to` gives history on relationships — where's the history for a node property like health score itself changing?

**graph-native asks** — maximal: Is GraphQL's resolver graph just a second, hand-maintained copy of the relationship graph already in your schema — who keeps them in sync? · postgres-first: What does `get_customer_context()` look like as seven hand-written joins per entity type — who edits all seven consistently when entity eight arrives? · event-log: Is `Org→Contract` an event, a projection join, or both — and when they disagree, which is truth?

### Answers

**maximal**
- A1: **Cut GraphQL+REST, keep GraphQL only.** Cedar earns its keep for autonomous agent mutations. *The 80ms includes Cedar's out-of-process policy-eval hop, absent from postgres-first's 45ms.*
- A2: In Postgres — the same outbox / append-only audit table kept to seed Kafka later, queried by actor + timestamp until 5k events/s.
- A3: **Yes — a second, hand-maintained copy; the engineer on that PR keeps both in sync manually, no codegen enforcing it today.**

**postgres-first**
- A1: The domain service starts Temporal in the same transaction as the outbox row; **that holds to ~3 consumers on one slot, breaks past ~5 workflow types needing independent replay or backpressure.**
- A2: `PERIOD` columns + trigger-maintained history tables per audited entity, `AS OF` via `WHERE valid_from <= $d AND valid_to > $d`; **costs 2x storage and a trigger per table, not free.**
- A3: **One `get_customer_context(org_id)` recursive CTE over a `relationships` table** returns all entity types generically by `entity_type` — entity eight adds a row filter, not a new join. *(Note: this is graph-native's design, adopted.)*

**event-log**
- A1: **Crypto-shred**: per-subject encryption key in a separate table; erasure = delete the key, ciphertext stays but is permanently unreadable — one extra table, one KMS call per write.
- A2: **Crypto-shredding can't fix data already in plaintext — concede append-only breaks here:** UPDATE the payload to a tombstone, emit `RedactionApplied`, purge backups and WAL on the next retention cycle.
- A3: Both: Contract creation is the event; Org→Contract is a projection join for reads. On disagreement the log wins — drift means the projector crashed, replay fixes it.

**graph-native**
- A1: **2 hops covers Person→Org→Contract, the common case; reseller/parent-org chains hit 4 — the edge table earns its keep only there, not everywhere else.**
- A2: 7 node policies + ~10 edge-type policies = **17 policies; each edge policy needs joint sign-off from both endpoint entities' owners — the real audit tax.**
- A3: Nowhere in `relationships` — node property history needs its own append-only audit table on UPDATE, **borrowed from postgres-first; a real gap, not free.**

**The round's shape: each contender conceded its founding claim.** maximal cut REST and admitted Cedar
costs 35ms per call. event-log conceded the log is not append-only once erasure and retroactive
masking arrive. graph-native conceded the edge table is only earned for org hierarchies. And
postgres-first quietly adopted graph-native's generic `relationships` CTE for `get_customer_context`.

### Round 2 scores (weighted /50)
postgres-first 44 · graph-native 31 · maximal 23 · event-log 18 — running: postgres-first 89, graph-native 62, maximal 53, event-log 38

Judge notes: three contenders conceded their founding claim. event-log gave up append-only —
simplicity's 1: *"the log's whole reason to exist."* maximal cut REST and then priced Cedar at 35ms
per MCP call for a benefit RLS provides at zero boxes. graph-native shrank its own justification to
org hierarchies. postgres-first adopted graph-native's generic `relationships` CTE and took two 5s.

---

## Round 3 — Unlocked iteration

**maximal**
**Dropped: Cedar/OPA, cold** — its whole value is what RLS + column GRANTs give free, and it cost 35ms per call to duplicate. **Dropped GraphQL** — a hand-synced second copy of the domain graph; no fix earns that back. REST already cut. pgvector no longer counted as a box.
**Replacement:** React talks a **typed RPC layer (tRPC-style) generated from the same Zod-typed domain functions the MCP tools call.** One function signature → one MCP tool def + one RPC procedure. Zero schema duplication, no resolver graph.
**Boxes, 8:** Postgres · typed RPC layer (MCP + tRPC, one schema) · React/TS · S3 · OTel · K8s · GitHub · LLMs. **Deferred, 6:** Kafka (>5k ev/s) · OpenSearch (FTS p99 >300ms) · **Temporal (>5 workflow types needing independent replay — postgres-first's own number)** · Snowflake · LangGraph/ADK (>~10 tools) · Okta (first enterprise SSO buyer).
**SDLC:** migration, `health.ts` (fn + Zod), `mcp-register.ts`, `trpc-register.ts`, RLS policy, test, `HealthBadge.tsx`. **7 files, 2 languages, ~4 hours.**
**Runtime:** same query path as postgres-first, auth inside the plan not a hop: **p99 45ms, pgbench.** The 35ms is gone.
**Authorization:** one place — RLS + GRANT at the DB role both adapters use.
**Numbers:** 8 boxes, ~7 concepts, **2 pageable (Postgres, K8s), ~$600/mo.**
**Still worse:** two generated adapters (MCP + tRPC) is one codegen step postgres-first's single service never needs.

**postgres-first**
**Dropped** PERIOD + trigger-per-table history (concept bloat, 2x storage). **Replaced with one `entity_history` table** (entity_type, entity_id, event, actor, at, diff jsonb), trigger-populated, returned as `history[]` (last 20) on every MCP entity response — stolen from event-log. **Stole graph-native's `valid_from/valid_to`, on `relationships` rows only.** Added `HealthBadge.tsx` and `security.test.ts` — genuinely missing before.
**Boxes, 8:** Postgres, TS backend, React, MCP, K8s, GitHub, OTel, **Temporal — settled, day one:** renewals and escalations need durable compensation an MCP loop can't give. Deferred triggers unchanged.
**SDLC:** `health_score.sql`, `mv_account_health.sql`, `health.ts`, `mcp/tools.ts`, `rls_policy.sql`, `health.test.ts`, **`security.test.ts` (cross-org read attempt)**, `HealthBadge.tsx`. **8 files, 2 languages, ~1 day.**
**Runtime:** `mv_account_health` ⋈ pgvector ⋈ `entity_history` → **p99 52ms** → `{org_id, score, top_factors[], similar_churned[], history[]}`.
**The three systems:** record = `public` schema, role `app_write`, RLS + GRANT. interaction = services/MCP connecting only as `app_write`. reasoning = a separate **`reasoning` schema** (embeddings, agent_memory, eval_traces), role `reasoning_svc`, **zero foreign keys from `public` into `reasoning`** — it can be truncated with no migration touching record data; `reasoning_svc` has no write GRANT on `public`.
**Numbers:** 8 boxes, **11 concepts** (net +1), 3 pageable (Postgres, K8s, Temporal), ~$950/mo.
**Still worse:** RLS across 30+ tables is still harder to audit at a glance than one Cedar file — conceded, not fixed.

**event-log**
**Dropped: the log as system of record**, the projector, replay, "which is truth", double authorization. **Postgres tables are truth. The event log is `events`, written by the same domain function in the same transaction, as derived append-only history — not a rebuild source.**
**Boxes, 5** (postgres-first's baseline adopted whole; event emission is a write path inside the domain function, not a box).
**SDLC:** column + migration, domain function emits `HealthScoreComputed` in the same transaction, MCP return type, RLS policy, test. **5 files, 2 languages, half a day.** Health score is a **column**; the event is the audit trail of it changing.
**Runtime:** p99 45ms unchanged; `history` = last 20 rows from `events WHERE entity_id = $1`, indexed, **+5-8ms, on demand not default.**
**Authorization:** one place — RLS on entity tables; events inherit their entity's policy.
**Numbers:** 5 boxes, ~7 concepts, 3 pageable, ~$900/mo.
*"Still worse: postgres-first needed no rename to get here — I only match it now. Steal from me: cheap per-entity `history` as a queryable side-effect of writes, for free auditability agents can cite."*

**graph-native**
**Dropped "graph-native" as an architecture.** *"What survives: one modelling rule living inside postgres-first. Not a separate stack — say it plainly."*
**The rule:** `relationships(from_id, to_id, type, valid_from, valid_to, props jsonb, PK(from_id, to_id, type, valid_from))`, index `(from_id, type) WHERE valid_to IS NULL`. `get_customer_context(org_id)`: one recursive CTE, **hard depth cap `WHERE depth <= 4`**, cycle guard `NOT to_id = ANY(path)`. **RLS: one node policy per entity table, one *generated* edge policy per edge type** — `USING (from_id IN visible_nodes(from_type) AND to_id IN visible_nodes(to_type))`, templated from the endpoints' own policies at migration time. *"Sign-off becomes one macro review, ever."*
**SDLC:** same 6 files as postgres-first; `relationships` untouched. **Runtime:** churn via depth-4 CTE ⋈ `mv_account_health` **p99 62ms**; `get_customer_context` alone, depth ≤3, **p99 38ms**.
**Numbers:** 0 boxes over postgres-first, ~10 concepts, 3 pageable, ~$900/mo.
**Still worse:** *"The depth cap is discipline, not structure — postgres-first's per-type joins can't run unbounded; this can, if someone forgets the `WHERE`."*

**Round 3's shape: total convergence.** All four now run Postgres as truth, RLS as the single
authorization site, plain TS domain functions, MCP as the agent surface, 2 languages for the SDLC
probe, p99 45-62ms, 5-8 boxes, ~$600-950/mo. What differs is one idea each brought and kept.

### Round 3 scores (weighted /50)
maximal 47 · postgres-first 37 · event-log 37 · graph-native 31

## Final: postgres-first 126 · maximal 100 · graph-native 93 · event-log 75 (of 150)

The series winner and the final-round winner are different, and both are right. postgres-first
was correct from round one about the *shape* — one box until it hurts, RLS as the one auth site,
two languages. maximal, having started with seventeen boxes, ended with the leanest stack in the
room by deleting Cedar, GraphQL and REST and replacing all three with one idea nobody else had:
**the human client's API is generated from the same typed domain function the MCP tool is.**
