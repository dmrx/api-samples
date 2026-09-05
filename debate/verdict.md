# Verdict — CRM build

**deno, 162 of 200 — and go, winner of both previous debates, finishes last.** The CRM changes the
answer because the hard problem stops being "render a screen" and becomes "one rule holds across
120 of them." go's hand-write-everything stance produces 66,000 lines and the permission rule at
~470 call sites, where a breach is a *missing* wrapper — and no diff shows an absence. deno wins by
taking the rule out of code entirely: a Postgres column grant plus RLS, declared once, audited by
one SQL query, with screens that hold no rule at all. One line to read, one `REVOKE` to undo.

---

# The honest caveat

**deno did not win on TypeScript.** It won on one idea: *permissions belong in the database, not in
the screens.* Postgres column privileges and row-level security are available to all four
contenders. Go with the same design would score close to deno's; deno without it scored 27 in round
one and was heading for last place.

So the decision is really two decisions, and the second matters more:
1. **Where does the permission rule live?** → Postgres. Not negotiable. It is the only answer that
   doesn't multiply by 120.
2. **What renders the screens?** → deno, on the numbers: 26,400 lines, 7 concepts, 0s build, 0.4s
   tests, one language from SQL type to form validator, p99 38ms, 40ms cold start.

All four stacks fail open on a bad grant, and none of them goes red. **Build the CI check first:**
one query diffing `information_schema.column_privileges` against a checked-in expected-grants file.
It is about forty lines, it covers all 120 screens at once, and every contender conceded it doesn't
exist. Add a DDL-history trigger next — deno's one real gap is that it cannot tell a regulator who
saw the field, and incumbent's Envers is the only thing in this debate that can.

---

# Migration path: incumbent → deno

Each step ships alone, is deletable in an hour, and reverses with one command.

**1. Move permissions into Postgres, before moving any code.** Translate the field rules in your 120
screen-JSONs into `GRANT SELECT (col…)` and RLS policies in `schema.sql`. Keep `@PreAuthorize` in
place — belt and braces. Nothing is deleted yet; you have just created the single source of truth.

**2. Ship the grants CI check and the DDL-history trigger.** The check diffs
`information_schema.column_privileges` against a checked-in expected-grants file and fails the
build. The trigger logs every grant and policy change with a timestamp. This is the step that makes
the breach in round 4 go red, and it works while you are still 100% on the incumbent.

**3. Delete the Node BFF.** It relays and translates; Postgres and Spring already do both. One
layer, one repo, one deploy gone before you introduce anything new.

**4. Port one object end to end — Account — behind a 1% split.** Deno serves list, detail and form
with `SELECT *` and no permission code, because the grants decide what comes back. Prove the same
Playwright screenshots pass. Ratchet to 100% over a week, then delete `@PreAuthorize` on Account —
it is now redundant, and you can prove it with the CI check from step 2.

**5. Port the remaining 39 objects, then delete the Angular MFE.** Each object is a day, each is
revertible, each removes JSON metadata and a Java service. At the end: 3 repos become 1, ~9,600
lines plus a 1,400-line builder plus 3 runtimes become ~26,400 lines and one, 95s builds become
none, and the discount rule lives in exactly one place instead of fourteen.
