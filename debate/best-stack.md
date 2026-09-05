# The best stack for a 2030 CRM

**The architecture is the answer; the language is the smaller decision.** Go and Deno were unlocked
and allowed to redesign freely against each other for two rounds. They converged on the same
design. Deno wins the series 186-135, but by round 5 the two stacks differed only in language and
in 15MB of RAM against 11ms of p99.

**Recommendation: Deno + TypeScript.** Take the Go variant instead if memory is your binding
constraint or if you cannot keep a CI gate green for a decade — both conditions are spelled out at
the bottom, and neither is a fringe case.

---

## The architecture — this part is not optional, in either language

**Postgres owns the truth. Everything else is generated out of it.**

```
Salesforce  →  CDC  →  Postgres  →  information_schema · pg_constraint · grants
(system of                (source     ↓
 record)                  of truth)   types · base validators · repository · form fields
                                      (generated, checked in, CI-diffed every PR)
```

Postgres holds four things, and it is the only thing that holds them:

| what | how | why here |
|---|---|---|
| **Columns and types** | the schema, mirrored by CDC | Salesforce adds fields weekly; a hand-authored type fights that every week |
| **Permissions** | column `GRANT` + `FORCE ROW LEVEL SECURITY` | one declaration site instead of a rule repeated across 120 screens |
| **Validation — the easy half** | domains, `CHECK` constraints, generated columns | required/type/format cannot then drift from what the database will actually accept |
| **Grant history** | an event trigger writing DDL and grant changes to `audit.grant_history` | the only way to answer "who could see this field, and since when" |

What stays hand-written, in either language: **cross-field rules, conditional visibility, and
drafts.** Drafts have to bypass constraints by design — a partial row must save — so the
application genuinely owns that logic. Both sides tried to claim otherwise and both conceded.

**Never generate SQL schema or grants from application code.** Go lost round 3 with a 900-line
generator emitting `GRANT` DDL from struct tags: generation pointed against the direction the data
actually flows.

---

## The stack

| layer | choice |
|---|---|
| Runtime | **Deno**, `deno compile` to a static binary per pod |
| HTTP | `Deno.serve` — stdlib, no framework, no router DSL |
| DB access | a **generated typed repository** over `postgres.js` — the only door; a lint rule bans raw clients and `any` outside it |
| Types | `db.d.ts` generated from `information_schema` |
| Validation | base Zod schema generated from `pg_constraint`, as an explicit reviewed catalog; hand-written `.refine()` for cross-field, conditional and draft logic only |
| Templating | HTMX partials, tagged-template escaping |
| Client | HTMX; Preact islands only for genuinely stateful widgets, client-bundled, out of the server process |
| Permissions | column `GRANT` + `FORCE ROW LEVEL SECURITY`, one explicit transaction per request, `SET LOCAL app.user_id` |
| Migrations | Atlas, diffing `information_schema`, reviewed like any PR |
| Tests | `Deno.test` + one golden-file HTML diff per screen, diffs grouped by changed file |

**Numbers:** 24,850 hand-written, 4,300 generated, 8 concepts. Build 2.4s, incremental 0.3s, tests
0.4s. p99 34ms on a 5M-row filtered list, 61MB RSS/pod, 25ms cold start.

---

## The five gates — build these before the first screen

Every one of them fails **open**. None is the CRM; all are why the CRM doesn't leak.

1. **Grants CI check.** Diff `information_schema.column_privileges` against a checked-in
   expected-grants file. ~40 lines, covers all 120 screens at once. `GRANT SELECT ON opportunity TO
   partner` missing its column list is normal-looking SQL that no linter flags — this is the only
   thing that catches it.
2. **Schema-drift gate.** Regenerate types, base validators and the repository from Postgres on
   every PR; fail the build on any diff from what's checked in. This is what replaces a compiler:
   drift becomes a red pipeline, not a lint warning someone waves through.
3. **`audit.grant_history` event trigger.** Log every DDL and grant change with actor and
   timestamp. Without it you can revert a breach in seconds and still not say who saw the field.
4. **Repository layer + lint ban on raw access, and a CI ban on `SECURITY DEFINER`.** RLS holds
   against anything the app role does — the one genuine escape is a `SECURITY DEFINER` function
   running as the table owner. Ban it in CI; it is the only real bypass in the design.
5. **A narrow read-audit table for sensitive columns.** A handful of fields, not everything —
   enough to answer which partners read `discount`, and when.

---

## The Go variant — when to take it instead

Same architecture, different language, and it is genuinely close: **sqlc** generates Go types out of
the Postgres schema and constraints, **templ** compiles views to type-checked Go so a renamed column
is a build error, permissions stay in migrations, `net/http` + chi, golden-file diffs. 23,200
hand-written, 4,400 generated, 8 concepts, build 3.4s / 0.5s incremental, p99 45ms, **46MB RSS**.

**Take it if:**
- **Memory is your binding constraint.** 46MB against 61MB per pod, across many pods, is real money.
- **You cannot rely on a CI gate staying green for ten years.** Go's uniformity is a property of the
  compiler; Deno's is a pipeline a person has to maintain. If that pipeline rots, Go degrades
  gracefully and Deno does not.
- **You want the strongest correctness guarantee available.** templ's compile-time proof that a
  renamed column breaks every view is stronger than any CI diff.

**Its price, disclosed by Go itself:** 1,200 generated structs — 40 objects × ~6 roles × ~5 queries
— regenerated in full on every weekly schema change, because sqlc has no incremental mode. That is a
large, recurring, unreadable diff on the exact PR a human most needs to read. And templ has a single
maintainer and thin likely 2030 corpus coverage — mitigated, not solved, by the fact that it
compiles to plain Go, so an abandoned templ still leaves working code.

---

## What this design is still worse at

- **Two places validation can be wrong.** The CHECK rule exists in Postgres and again in the
  generated Zod mirror. Generated from one source and CI-diffed, but duplicated all the same.
- **Drafts are app-owned.** They must bypass DB constraints, so Zod is the source there, not Postgres.
- **Memory.** 61MB/pod against Go's 46MB.
- **Runtime age.** Deno is younger than Go's toolchain. `deno.lock` plus vendored dependencies is a
  mitigation, not a guarantee.
- **Nobody named a measurement tool for the final latency numbers.** Treat every figure here as an
  estimate to be re-measured on your own hardware before it decides anything.
