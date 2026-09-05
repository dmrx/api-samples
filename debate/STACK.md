# The stack

*Supersedes everything else in this folder. Greenfield agent-native CRM, 2030 horizon, judged on
simplicity, efficiency, readability. Agents write most of the code; one human reads every diff.*

---

## In one line

**Postgres does nearly everything. One TypeScript service of plain functions does the rest.
Humans get server-rendered HTML; agents get MCP. Both call the same function.**

---

## The six boxes

| box | job |
|---|---|
| **PostgreSQL** | System of record. The *only* authorization site: column `GRANT`s + `FORCE ROW LEVEL SECURITY`. Also search (FTS + pgvector), events (transactional outbox + logical replication), queues and scheduled jobs (`SKIP LOCKED` / pg-boss — retries, timers, cron), audit (`entity_history`), relationships (one typed edge table), analytics (read replica). |
| **One TypeScript service** | Plain exported functions, `(tx, actor, input)`, wired explicitly at one composition root. No DI container, no decorators, no ORM — the query in the file is the query that runs. Runtime: **Deno 2**, compiled to one binary, run with default-deny permissions (`--allow-net=db:5432,llm-endpoint`) so an agent-written service cannot reach anything the manifest doesn't name. |
| **Server-rendered HTML + HTMX** | The human UI. The service renders HTML; the wire format is HTML; SSE for anything live. **Preact islands only where a widget is genuinely stateful** — a chat pane, a pipeline board. No SPA, no client state store, no second schema. |
| **MCP** | The agent surface. `crm.search_customers`, `get_customer_context`, `create_opportunity`, `update_relationship`, `create_task`, `analyze_account`. Each tool is a registration of an existing domain function — `register()` takes a function reference, so a tool *cannot* carry its own logic. |
| **Temporal** | Durable multi-week workflows with human steps: onboarding, renewal, escalation. Started by the domain function in the same transaction as the outbox row. |
| **Kubernetes · GitHub · OpenTelemetry** | Fixed. |
| **Model-agnostic LLMs · S3 · OIDC** | Fixed. |

**What it costs:** 2 languages (SQL, TypeScript). "Add a customer health score" touches ~7 files
including the UI and a cross-org security test, about a day. Churn query p99 ~45ms at 5M entities.
**Two things can page at 2am: Postgres and Kubernetes.** ~$600/month at 500 users.

---

## The five rules — these matter more than the boxes

**1. Authorization lives in Postgres and nowhere else.** A rule in application code fails open,
and its failure mode is a *missing line*, which no diff can show. RLS at the DB role means an
agent-invented call path inherits the rule it forgot. This is the single finding every debate in
this folder agreed on.

**2. One typed domain function is the source of everything.** Its signature produces the MCP tool
definition, the validator, the test fixture, and — if an island ever needs one — a typed RPC. No
surface hand-maintains a copy of another. The domain graph is declared once, in the schema.

**3. Three systems, three schemas, three roles.** `public` (record, role `app_write`) · the service
(interaction, connects only as `app_write`) · `reasoning` (embeddings, agent memory, eval traces,
role `reasoning_svc`). Zero foreign keys from `public` into `reasoning`; no write grant back. The
reasoning schema can be truncated without touching a record. **This is what stops the CRM becoming
an LLM-shaped database.**

**4. Every state change is an event — as history, never as truth.** The domain function writes the
row and an `entity_history` row in the same transaction. Agents get `history[]` on demand and can
answer *"why did this renewal probability drop"* with cited events. Tables are never rebuilt from
the log: event-sourcing as system of record lost on projector drift, double authorization, and
erasure.

**5. Relationships are one typed, temporal edge table with a depth cap.**
`relationships(from_id, to_id, type, valid_from, valid_to, props)`. `get_customer_context` is one
recursive CTE, `depth <= 4`, cycle guard on the path. Edge RLS policies are generated from the two
endpoints' node policies by one macro. Entity eight is a row filter, not a new join. No graph
database.

---

## What's out, and the number that puts it back

| out | back when |
|---|---|
| React as the app shell | a screen is more app than form (visual workflow editor, drag-and-drop board), a mobile client shares the API, or the team won't retrain |
| GraphQL, REST | a partner needs a public API — generate one surface from the domain functions, same as MCP |
| Temporal (workflow engine) | a process must wait days and include a human step — a renewal waiting a week for a signature. Until then pg-boss's retries, timers and cron are enough, and the outbox already records the events |
| Cedar / OPA | >50 policy rules, or a rule spans services |
| Kafka / Redpanda | outbox lag p99 >5s, or >3 consumers on the replication slot |
| OpenSearch | FTS p99 >300ms at 5M rows, or fuzzy/multilingual ranking is user-visible |
| Snowflake / Databricks | an analytics query exceeds 30s on the read replica |
| LangGraph / ADK orchestration | >10 MCP tools, or multi-agent handoff needs checkpointing |
| Okta / Auth0 | the first enterprise buyer who needs SSO lifecycle management |
| NestJS, any DI framework | never — the diff can't show what a container wires |
| a graph database | a traversal genuinely needs depth >4 or graph algorithms |
| Salesforce sync | the day a customer already lives there; it's a peer through the outbox, never a source |

---

## Runtime: why Deno 2

Same engine as Node (V8), an LTS channel, a single binary, and the one thing that matters for
agent-written code: **a permission sandbox a reviewer can verify in one line.** Bun is faster to
start and install, but runs a different engine on a faster release cadence with no sandbox; at CRM
scale the speed is noise because Postgres does the work. Node with native TypeScript was the
fallback only if a Node-only SDK forced it; nothing on this list does.

---

## How the debates got here

- **Go, Deno, Rust, .NET, Spring, Phoenix** all had their round. The winner tracked the workload —
  Go for one screen, Deno for 120, .NET for compiler-generated permissions — and once the frame was
  "one language spanning model, tool, validator, test," TypeScript held.
- **SSR won every screen debate** on principle 5 and on the numbers. It was displaced in one run
  only because React had been listed as fixed; that assumption is withdrawn here.
- **Seventeen boxes became seven.** The list's own author, arguing as "maximal," deleted six of them
  in the final round and ended with the leanest stack in the room.
- **The database, not the language, decided everything.** p99 spread across five languages was 14ms.
  Concept spread was three. What differed was where the permission rule lived.

---

## Build these before the first screen — every one of them fails open

1. **Grants CI check** — diff `information_schema.column_privileges` against a checked-in file.
   Forty lines; covers every screen and every tool at once.
2. **Schema-drift gate** — regenerate types, validators and tool definitions from the catalog on
   every PR; fail on any diff. A required status check, no bypass.
3. **`SECURITY DEFINER` ban** in CI — the one real RLS escape.
4. **A DDL/grant history trigger** — so you can say what access looked like on a past date.
5. **pgaudit on sensitive columns** — so you can say who *read* a leaked field, not just revert it.
