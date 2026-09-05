# What is the domain layer made of?

Architecture settled: TypeScript, Postgres as system of record, domain layer as the single
interpretation of CRM behaviour, GraphQL for humans + MCP for agents, Temporal for workflows,
Kafka out. Three amendments carried in: thin the magic (A1), Postgres grants + RLS as the
enforcement floor under the domain layer (A2), Salesforce is a peer not a source (A3).

Contenders: **nestjs · thin · effect · codegen**
Judges: reviewer (3), agent (3), simplicity (2), performance (2). Max 50 per round.

---

## Round 1 — Show the domain layer

**nestjs**
```ts
@Injectable()
export class OpportunityService {
  constructor(private db: DbService, private ability: AbilityFactory) {}
  async reassignOwner(ctx: RequestCtx, id: string, newOwnerId: string) {
    const ability = this.ability.for(ctx.user);
    const opp = await this.db.opportunity.findOrThrow(id);
    ForbiddenError.from(ability).throwUnlessCan('update', subject('Opportunity', opp), 'ownerId');
    return this.db.opportunity.update(id, { ownerId: newOwnerId });
  }
}
```
`OpportunityModule` provides it once. The GraphQL resolver and the `@Tool` MCP handler both take it by constructor injection — the container hands both **the same singleton**, so there is no second copy of the CASL check. `@UseGuards(GqlAuthGuard)` sits on the resolver only for session extraction; the authorization is CASL *inside* the service, so MCP — which has no GraphQL guard — still gets it.
**New MCP tool:** one service method, one `@Tool` method that destructures args and calls it, one zod schema. *"The agent cannot invent a second interpretation because the tool method has nowhere to put logic."* Backed by a lint rule banning `this.db` in `*.mcp.ts`.
**A2:** `DbService` is a request-scoped provider wrapping a pooled client; on inject it runs `SET LOCAL app.user_id` inside the transaction, tied to Nest's request lifecycle — not global pool state, no leakage across requests.
**Numbers:** ~1,120 hand-written for 8 objects, ~480 schematic-generated. **6 concepts.** Cold build 38s, incremental 2.1s (SWC), tests 45s. p99 41ms. **RSS 210MB. Cold start 1.6s** (DI graph resolution). MCP call 9ms server-side.
**Weaknesses:** request-scoped providers cost real cold-start and per-request DI resolution. *"And yes — the reviewer judge is right that `@UseGuards` alone would fail open silently; that's why A2, not Nest, is what actually saves us."*

**thin**
```ts
export async function reassignOpportunityOwner(
  db: Pool | PoolClient, actor: Actor, input: { oppId: string; newOwnerId: string }
): Promise<Opportunity> {
  const opp = await getOpportunity(db, input.oppId);
  if (!canReassign(actor, opp)) throw new ForbiddenError("reassign:opportunity", actor.id);
  return updateOpportunityOwner(db, input.oppId, input.newOwnerId);
}
```
*"Auth check is line 3, a plain `if`, visible in the diff — no guard decorator, no interceptor."* The GraphQL resolver and the MCP tool handler each import the same function; two three-line callers.
**New MCP tool:** one tool file (imports the domain fn, a Zod schema, no logic) and one line in `tools/registry.ts`. **`register()`'s signature requires a domain function reference, not inline logic — you cannot register a tool with a hand-rolled body.** Plus a lint rule banning DB imports outside `domain/`, and a contract test asserting every registered tool maps 1:1 to a `domain/*.ts` export.
**A2:** `FORCE ROW LEVEL SECURITY` on opportunities, column grants on `discount_pct`. Composition root opens the pool once; a per-request wrapper (Fastify `onRequest` and the MCP tool wrapper) does `BEGIN; SET LOCAL app.actor_id`, commit, release. *"No container — it's 15 lines in `context.ts`, called by both entrypoints."*
**Numbers:** 8 services ≈ **440 hand-written lines, 0 generated. 4 concepts.** Cold build 2.1s (esbuild), incremental 180ms, tests 3.4s. p99 **38ms**. **RSS 85MB. Cold start 210ms.** MCP call 12ms p50 / 40ms p99.
**Weaknesses:** *(a)* uniformity isn't enforced by the language, only by lint plus one contract test — trusting CI, not the compiler. *(b)* no typed error channel: `ForbiddenError` vs `NotFoundError` is a runtime `instanceof` at the boundary, invisible in the signature.

**effect**
```ts
const reassignOpportunityOwner = (in: ReassignInput) =>
  Effect.Effect<Opportunity, NotFoundError | ForbiddenError | StageLockedError,
                OpportunityRepo | AuthZ | Clock>
```
R is the exact set of services the operation touches. Authorization is `AuthZ.check(...)` piped into the effect — and **omit providing the `AuthZ` Layer at the composition root and `Effect.runPromise` fails to typecheck**. GraphQL and MCP both run the identical effect with `Effect.provide(AppLayer)`.
**New MCP tool:** a Schema for input/output and a handler calling the existing effect. *"If they invent a second path — a raw query bypassing the effect — it still compiles. Effect can't stop bypass, only stop unhandled failure inside the effect it wraps. A2 catches what the type system can't."*
**A2:** per-request `PgSession` is `Layer.scoped` — acquire runs `BEGIN; SET LOCAL app.user_id`, release commits or rolls back; built fresh per call, never pooled across identities.
**Numbers:** 8 objects ≈ **2,300 hand-written**, ~350 generated. 6 core concepts. **Cold build 95s, incremental 5-9s. Tests 45s run + 35s typecheck — "tsc on Effect's conditional types is genuinely slow."** p99 **180ms** ("same SQL, no runtime tax"). RSS 210MB. Cold start 900ms. **MCP call +25ms dispatch/auth on top of the query.**
**Weaknesses:** thin corpus — agents drift to `async/await` and lose the error union; cryptic `R` mismatch errors slow fixes even when they name the right line.

**codegen**
**Spec:** the `opportunities` table plus a `schema.yaml` entry (`reassignOpportunityOwner(id, newOwnerId): Opportunity`, `requires: role:sales_manager`).
**Generated:** the TS interface, the GraphQL field and resolver stub, the MCP tool JSON schema from the same types, a zod validator from the column constraints, and the SDK method. **Hand-written: one file — the verb body.** *"Generated is every type-facing surface; hand-written is the verb body, one function per mutation, nothing else."*
**New MCP tool:** add the mutation to `schema.yaml`, run codegen, write the body against the emitted interface — three files. *"When the rule doesn't fit the spec's shape — streaming partials, a propose-then-confirm two-step — you write a hand resolver and tag it `custom: true` so codegen skips it. That tool now has its own interpretation; nothing keeps it aligned. Escape hatch, not free."*
**A2:** codegen introspects `pg_policy` at build time, emits an MCP grant annotation **and a CI fixture matrix: every generated mutation run as every role against a shadow DB, asserting Postgres itself allows or denies.** Enforcement is FORCE RLS plus the connection role — *"codegen can't fake it, only surface drift as a failing generated test."*
**Numbers:** ~1,400 hand-written; **~14,000 generated — 10:1. ~140 concepts.** Cold build + generate 95s, incremental 12s, tests 70s. p99 41ms. RSS 165MB. Cold start 900ms. MCP call 28ms p50 / 60ms p99.
**Weaknesses:** *"Generated code is 10x the surface a reviewer must trust unread — exactly the reviewer's penalty target. And the escape hatch for inexpressible rules creates a second interpretation right where uniformity matters most."*

### Round 1 scores (weighted /50)
thin 45 · nestjs 25 · effect 21 · codegen 18

Judge notes: **thin took every judge but reviewer** — 440 lines against effect's 2,300 for the same
eight objects, a 2.1s build against 95s, and its `register()` signature structurally blocking inline
logic in an MCP tool. **nestjs conceded the decisive point itself**: `@UseGuards` alone fails open
silently, and it is amendment A2, not Nest, that saves it. **effect's 180ms p99 contradicted its own
"no runtime tax" claim** in the same sentence and performance gave it a 1. **codegen declared ~140
concepts and a 10:1 generated-to-hand ratio**, taking a 1 from both reviewer and simplicity.

---

## Round 2 — Cross-exam

### Questions

**nestjs asks** — thin: When someone passes a raw `Pool` instead of `PoolClient` to `reassignOpportunityOwner`, where's the `SET LOCAL`, and who catches the missing RLS scope? · effect: You claimed 180ms p99 is "no runtime tax" against thin's 38ms on identical SQL — where did the other 142ms go? · codegen: Once a mutation is tagged `custom: true`, what stops its hand-rolled resolver silently diverging from `schema.yaml`?

**thin asks** — nestjs: If A2, not the DI container, saves you from fail-open, what does 1.6s of cold start per ROSA scale event actually buy? · effect: Where is the 139ms hiding, and what does a 35s typecheck cost an agent's loop over ten years? · codegen: Your CI fixture matrix already proves RLS holds per role against a shadow DB — what do 14,000 generated lines add that the fixture doesn't already cover?

**effect asks** — nestjs: If A2, not Nest, saves you, what does the DI container buy that 15 lines of composition-root wiring doesn't? · **thin: Nothing stops a caller passing a raw `Pool` that skipped `BEGIN; SET LOCAL app.actor_id` — how does your type signature catch that before RLS does?** · codegen: When hand-written `custom: true` code and the generated interface disagree on a rule, which one does the MCP tool actually run?

**codegen asks** — nestjs: When a field type changes, what regenerates your zod schema, GraphQL SDL and SDK together — or are all three hand-edited separately? · **thin: Your MCP tool hand-writes its own Zod schema separate from the domain function's TS types — what test catches the two silently drifting apart?** · effect: Your Schema, GraphQL type and MCP JSON schema are three hand-authored encodings of the same shape — what forces them back in sync after a field change?

---
*Interrupted after Round 2 questions and two answers (thin, nestjs). Superseded by the greenfield
agent-native rerun in `agentic-*.md`. Standing at interruption: thin 45, nestjs 25, effect 21,
codegen 18 after Round 1. thin conceded the `Pool | PoolClient` RLS-bypass hole and proposed a branded
`TxClient`; nestjs conceded its zod/SDL/SDK are hand-edited separately.*
