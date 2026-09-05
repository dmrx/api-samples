# The stack

One database. One service. One language. Two doors — one for people, one for agents.

---

## The six pieces

**1. PostgreSQL** — holds everything.
The data. Who may see which column (`GRANT`) and which rows (`ROW LEVEL SECURITY`). Search (built-in
full-text, `pgvector` for similarity). The change log (`entity_history`, one row per change).
Background jobs (`pg-boss`). Relationships between records (one `relationships` table). Reports (a
read replica). It is the only place a permission rule lives.

**2. One TypeScript service, on Deno 2.**
Plain functions: `reassignOwner(tx, actor, input)`. No framework, no dependency injection, no ORM.
Each function opens a transaction, sets the acting user, calls SQL, returns. Compiled to a single
binary. Run with permissions off by default: it may talk to the database and the model endpoint and
nothing else, and that fact is one line in the deploy file.

**3. HTML from the server, for people.**
The service renders the page. HTMX swaps the parts that change. Server-sent events for anything
live. A small Preact island only where a widget genuinely holds state — a chat pane, a board.
No React app, no client-side state, no second copy of anything.

**4. MCP, for agents.**
Six tools: `search_customers`, `get_customer_context`, `create_opportunity`,
`update_relationship`, `create_task`, `analyze_account`. Each is a one-line registration of a
function from piece 2. The registry only accepts a function reference, so a tool cannot carry logic
of its own. Agents and people call the same code.

**5. A separate schema for the AI parts.**
Embeddings, agent memory, evaluation traces live in `reasoning`, under their own database role, with
no foreign keys into the real records and no write permission on them. It can be wiped without
touching a customer.

**6. Kubernetes, GitHub, OpenTelemetry, S3, OIDC login, any LLM.**
Plumbing. Nothing to decide.

---

## How one request works

A rep opens an account page: browser → service function → `BEGIN; SET LOCAL app.user = rep`
→ SQL (Postgres strips the rows and columns the rep may not see) → HTML back.

An agent asks for account context: MCP tool → **the same function** → the same transaction, the
same SQL, the same stripping → JSON back, with the last 20 changes attached if it asks.

Nobody can forget the permission check. It isn't in the code.

---

## How one change works

"Add a health score to accounts." One migration, one function, one tool registration, one template
change, one test that tries to read another rep's account and must fail. Two languages, SQL and
TypeScript. About a day. One person reads the whole diff.

---

## What is deliberately not here

| left out | why | the day it comes back |
|---|---|---|
| React as the app | a second copy of every screen and every type | a screen that is more app than form, or a mobile client |
| GraphQL, REST | a second copy of the schema | a partner needs a public API |
| a policy engine (Cedar/OPA) | Postgres already does it, at zero extra hops | more than ~50 rules |
| Kafka | an outbox table already records every change | more than 3 consumers, or lag over 5s |
| OpenSearch | built-in search is enough at millions of rows | search over 300ms |
| a workflow engine (Temporal) | `pg-boss` does retries, timers and cron | a process that waits days with a human step in it |
| a data warehouse | the read replica | a report over 30s |
| a graph database | one table and one recursive query | a walk deeper than 4 hops |
| any framework with decorators or a DI container | the diff can't show what it wires | never |

---

## Why it holds up to 2030

- **Postgres and TypeScript are the two safest bets in the room to still build in 2031.**
- **Agents will invent call paths.** The permission rule sits underneath all of them.
- **Agents write cheaply and humans review expensively.** Every choice above shrinks the diff.
- **Every box that isn't here has a number that adds it.** Nothing is bolted on in advance.

---

## Build these five things before the first screen

1. A CI check that the database's column grants match a file in the repo.
2. A CI check that regenerates types from the database and fails on any drift — required, no bypass.
3. A CI ban on `SECURITY DEFINER` — the one way around row security.
4. A trigger that logs every change to a grant or policy, with a timestamp.
5. `pgaudit` on the sensitive columns, so you can say who *read* a field, not just fix it.

Every one of these covers a failure that is silent without it.
