# The ideal stack — greenfield agent-native CRM, 2030

**Postgres does almost everything, one TypeScript service does the rest, and every surface — MCP
tool, React RPC, validator, test — is derived from one typed domain function.** postgres-first won
the series 126/150; maximal won the final round 47/50 by deleting six of the seventeen boxes it
started with and ending leaner than the winner. The stack below is what the four converged on,
plus the one idea each brought that survived.

---

## The stack — 8 boxes

| box | does | replaces |
|---|---|---|
| **PostgreSQL** | system of record; column `GRANT`s + `FORCE ROW LEVEL SECURITY` as the only authorization site; FTS + pgvector for search and similarity; transactional outbox + logical replication for events; `SKIP LOCKED`/pg-boss for queues; one `entity_history` table for audit; a read replica for analytics | Cedar/OPA, OpenSearch, Kafka, a vector DB, Snowflake — all deferred with triggers below |
| **One TypeScript service** | plain exported domain functions, `(tx, actor, input)`, explicit wiring at one composition root — no DI container, no decorators | NestJS |
| **MCP** | the agent surface: `crm.search_customers`, `get_customer_context`, `create_opportunity`, `update_relationship`, `create_task`, `analyze_account` — each a thin registration of a domain function; `register()` takes a function reference, never inline logic | REST-for-agents, GraphQL-for-agents |
| **Typed RPC to React** | generated from the same Zod-typed domain function the MCP tool is registered from — one signature yields one tool def and one procedure, zero second schema | GraphQL, REST |
| **Temporal** | durable multi-week workflows with human steps: renewals, onboarding, escalation. Started by the domain function in the same transaction as the outbox row. The one contested box — see below | pg-boss for cron and retries only |
| **React + TypeScript** | fixed | — |
| **Kubernetes · GitHub · OpenTelemetry** | fixed | — |
| **Model-agnostic LLMs, S3, OIDC** | fixed; Okta/Auth0 deferred to the first enterprise SSO buyer | — |

**Numbers, from the converged designs:** 7-8 files and **2 languages** (SQL, TypeScript) to ship
"add a customer health score" end to end including the React badge and a cross-org security test,
about a day. Churn probe **p99 45-52ms** (pgbench, 5M entities, indexed). **2-3 things can page at
2am.** **$600-950/month** at 500 users / 5M entities, the spread being Temporal.

---

## The rules that made it converge

**1. Authorization lives in Postgres and nowhere else.** Every architecture that put a rule
anywhere else — Cedar (35ms per call, a DSL, a service), a second copy in resolvers, a second
enforcement on raw events — lost that round. RLS at the DB role both MCP and the RPC connect
through means an agent-invented call path inherits the rule it forgot.

**2. One typed domain function is the source; every surface is derived.** This is the owner's own
goal — one language across models, contracts, tools, validation, frontend types, SDKs, tests — and
it is a generation goal. The MCP tool definition and the React RPC procedure are both generated from
the Zod signature. The domain graph is declared once, in the schema; nothing hand-maintains a copy.

**3. Three systems, three schemas, three roles.** `public` is the system of record, role `app_write`.
The service layer is the system of interaction and connects only as `app_write`. `reasoning` is a
separate schema — embeddings, agent memory, evaluation traces — role `reasoning_svc`, **zero foreign
keys from `public` into `reasoning`**, no write grant on `public`. `reasoning` can be truncated
without a migration touching record data. That is what keeps the CRM from becoming an LLM-shaped
database.

**4. Every state change is an event — as history, not as truth.** The domain function writes the
row and an `entity_history` row in the same transaction. Agents get `history[]` **on demand**
(+5-8ms when asked, not by default) and can answer "why did this customer's renewal probability
drop" with cited events. The log is never the thing tables are rebuilt from: event-sourcing lost
every round it was the system of record — projector drift, "which is truth", authorization twice —
and won the round it became derived history.

**5. Relationships are one typed, temporal edge table with a depth cap.**
`relationships(from_id, to_id, type, valid_from, valid_to, props)`; `get_customer_context` is one
recursive CTE, `depth <= 4`, cycle guard on the path. Entity eight is a row filter, not a new join.
Edge RLS policies are generated from the two endpoints' node policies by one macro, so seventeen
policies are one review. This is graph-native's whole contribution, and it is eight lines.

---

## The deferred boxes and the number that adds each one back

| box | add it when |
|---|---|
| Kafka / Redpanda | outbox lag p99 > 5s, or more than 3 consumers on the replication slot |
| OpenSearch | FTS p99 > 300ms at 5M rows, or fuzzy/multilingual ranking is a user-visible need |
| Cedar / OPA | more than ~50 policy rules, or a rule must span services |
| Snowflake / Databricks | an analytics query exceeds 30s on the read replica |
| LangGraph / ADK-style orchestration | more than ~10 MCP tools, or multi-agent handoff needs checkpointing |
| Okta / Auth0 | the first enterprise buyer requiring SSO lifecycle management |
| a graph database | a traversal genuinely needs depth > 4 or graph algorithms (centrality, shortest path) |

**Temporal is the one box the room split on.** postgres-first kept it day one — renewals and
escalations need durable compensation across weeks with human steps, and the owner listed those as
core workflows. maximal deferred it to "more than 5 workflow types needing independent replay" and
was $350/month and one pager cheaper for it. **Recommendation: day one**, because the owner's own
workflow list — onboarding, renewal, escalation — contains a multi-week human-in-the-loop process
in week one, and pg-boss's honest ceiling is retries and timers. If that list changes, the trigger
is the one above.

---

## What each architecture was right about, and where it lost

| | right about | lost on |
|---|---|---|
| **postgres-first** | the shape: one box until it hurts, a named trigger for every deferral, RLS as the single auth site, 2 languages | concept count crept to 11 and it defaulted history into every response for +7ms |
| **maximal** | one typed function generating both the MCP tool and the human RPC — the cleanest contract in the series | it took two rounds to delete Cedar (35ms/call for what RLS does free) and a hand-synced GraphQL graph |
| **graph-native** | the relationships rule and generated edge policies | it is a modelling rule, not an architecture, and said so; the depth cap is discipline |
| **event-log** | agents need cited history to explain a change | append-only cannot survive erasure or retroactive masking; as system of record it enforced auth twice and could not say which store was truth |

---

## The two things this run settled that the owner's list had open

**GraphQL is out, and so is REST.** When MCP is the agent surface, GraphQL's only remaining job was
the human client — and its resolver graph was a hand-maintained second copy of the schema's
relationships with nothing keeping it in sync. A typed RPC generated from the domain functions does
the same job with no second schema. If a partner needs a public API, that is the trigger to add one
surface, generated the same way.

**The policy engine is out.** Cedar/OPA's value — declarative, testable, auditable policy — is what
column grants and RLS already are, at zero boxes and zero hops. Its one honest advantage, one file
to read instead of policies across thirty tables, is real and is the trigger for adding it back.
