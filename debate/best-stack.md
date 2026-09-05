# The best stack for a 2030 CRM

**Deno + plain TypeScript + HTMX + Postgres, with Postgres as the source of truth for schema,
types and permissions alike.** Deno beat Go 110-72 across three rounds with the permission model
held constant, so this is not the permissions argument again. It won because a CRM's schema comes
from outside the application — Salesforce, through CDC — and everything should be generated in that
direction. Go's best design generated Postgres grants *from* Go struct tags: backwards, and it took
a 900-line bespoke generator to do it.

---

## The one rule everything else follows

**Generation flows out of the database, never into it.**

```
Salesforce  →  CDC  →  Postgres  →  information_schema + grants  →  types, schemas, forms
   (system of record)   (source of truth)          (generated, checked in, CI-diffed)
```

Postgres holds the columns, the types, the column grants and the RLS policies. Everything the
application knows about a record is generated from that and checked into the repo. CI regenerates
on every PR and **fails the build on any diff.** That is the whole design; the rest is detail.

Why it matters: Salesforce admins add fields weekly. A stack whose types are hand-authored fights
that every week. A stack whose types are derived from it absorbs it in one file.

---

## The stack

| layer | choice | why |
|---|---|---|
| Runtime | **Deno** | Native TypeScript, no build step, `deno.lock` vendors deps, permission flags mirror RLS's least-privilege posture. Bun is faster to boot; its looser sandbox isn't worth it here. |
| HTTP | `Deno.serve` | Stdlib. No framework, no router DSL, no middleware chain to trace. |
| Templating | HTMX partials, tagged-template escaping | Server renders HTML; the wire format is HTML. No client router. |
| DB access | `postgres.js`, raw SQL | No ORM. The query in the file is the query that runs. |
| Types | generated `db.d.ts` from `information_schema` | ~1,400 lines, imported not read, diffable in a PR. |
| Forms/validation | generated base Zod schema + hand-written `.refine()` | ~220 lines/form × 40 = 8,800. One declaration is the row type, the form model and the validator. |
| Client | HTMX + Preact islands for stateful widgets only | Islands are the exception, not the pattern. |
| **Permissions** | **Postgres column `GRANT` + `FORCE ROW LEVEL SECURITY`** | One declaration site. Screens hold no rule at all — they `SELECT *` and the grant decides. |
| Session context | one explicit transaction per request, `SET LOCAL app.user_id` | Survives pgbouncer transaction pooling. No query outside a transaction, enforced at the pool client. |
| Migrations | Atlas, diffing `information_schema` | Generated SQL, reviewed like any PR. |
| Tests | `Deno.test` + one golden-file HTML diff per screen | Diffs grouped by changed file, so one shared-partial change is one diff block, not 120. |

**Numbers:** 26,400 hand-written lines, ~3,000 generated, 7 concepts. 0s build, 0.4s tests.
p99 38ms on a 5M-row filtered list (k6, indexed RLS), 95MB RSS/pod, 40ms cold start. Live GROUP BY
640ms. Zod on a 60-field form: 0.2ms CPU/request, ~2% of p50.

---

## The five things to build before the first screen

Every contender in this series conceded these are missing, and every one of them fails **open**.
None of them is the CRM; all of them are why the CRM doesn't leak.

1. **The grants CI check.** One query diffing `information_schema.column_privileges` against a
   checked-in expected-grants file. ~40 lines. It covers all 120 screens at once, and it is the
   only thing that would have caught the round-4 breach — a `GRANT SELECT ON opportunity TO partner`
   missing its column list is syntactically normal SQL that no linter flags.
2. **The DDL-history trigger.** Log every grant and policy change with a timestamp. Without it you
   can revert a breach in seconds and still not tell a regulator who saw the field or since when.
   This is the incumbent's one genuine win in the whole series — Envers could answer that question
   and nothing else could.
3. **A read-audit table for sensitive columns.** Narrow: a handful of fields, not everything.
   Enough to answer "which partners read `discount`, and when".
4. **The repository layer plus a lint rule banning raw `db.query` outside it.** ~12 files. This
   closes Deno's own worst flaw, which it named itself: `any` and raw queries let an agent bypass
   `.parse()`, and three report paths already do.
5. **The schema-drift gate.** CI regenerates `db.d.ts` and the base Zod schemas from
   `information_schema` and fails on any diff from what's checked in. This is what replaces Go's
   compiler-enforced uniformity: divergence becomes a red pipeline instead of a lint warning a
   reviewer can wave through.

---

## What was taken from Go, and what was left

**Taken:** the insistence that uniformity be *enforced*, not conventional. Deno's answer is
generation plus a CI diff gate rather than a compiler, and that is a fair substitute — a red
pipeline is not a suggestion. Also taken: golden-file HTML diffs as the human review gate, and
`html/template`'s discipline of rendering on the server and shipping HTML on the wire.

**Left:** Go's 44MB RSS against Deno's 95MB — a real 2.2x, and the one number where Go simply wins.
At 120 screens across many pods that is a genuine hosting cost, and it is the honest price of this
choice. Left too: Go's uniformity being a property of the language rather than a pipeline someone
has to keep green. If your organisation cannot keep a CI gate green for a decade, Go's argument
gets stronger and this recommendation gets weaker.

---

## What this stack is still worse at

- **The escape hatch.** An agent can write `any` and a raw query and skip validation. The
  repository layer and lint rule catch it at review, not at compile. Go makes it structurally
  harder to do by accident.
- **Memory.** 95MB/pod against Go's 44MB.
- **Runtime age.** Deno is younger than Go's toolchain, and "will it build in 2031" is a fair
  worry. `deno.lock` plus vendored dependencies is the mitigation, not a guarantee.
- **Reports.** Three read paths skip Zod for speed. That is a deliberate, bounded exception and it
  should stay bounded — write it down, and check the boundary in CI.
