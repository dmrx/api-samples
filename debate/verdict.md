# Verdict

**Go, 164 of 200.** It wins the two heaviest judges — reviewer and agent — and performance too:
0 generated lines, one idiom the language enforces rather than a lint rule, 0.9s to red, 19MB a
pod. deno (147) is the better *small* stack and won the final round outright: 7 concepts, 195
lines, zero build step, 0.4s feedback. It loses because its uniformity is convention — an agent
that skips the golden file forks a pattern and only review catches it. rust (114) pays 64s a loop
for a compiler that cannot catch the bug that mattered. incumbent (56) conceded its own BFF is
unearned and that this bug never goes red at all.

**Choose go if ten agents must write the same code. Choose deno if one human must read it.**

---

# What the debate actually proved

No stack has one source of truth for a business rule. Every contender duplicates the 30-day rule
across a language boundary, and no compiler catches `>=` versus `>`. So the stack does not decide
whether the bug ships — the boundary test does. The stack decides two things only: **how fast you
learn** (deno 0.4s, go 0.9s, rust 64s, incumbent never) and **how much noise hides the line**
(deno 22 lines of 2,000; go 300-400; incumbent 350 lines of autoconfig).

Optimize for those two numbers. Everything else in this debate was decoration.

---

# Migration path: incumbent → go

Each step ships alone, is deletable in an hour, and reverses with one `oc rollout undo`.

**1. Write the boundary tests first, against the incumbent.** A table test per business rule, each
with the exact-boundary case — 30 days to the second. Point them at the Spring endpoint. This is
the only step that would have caught the bug, and it works before you move a line of code.

**2. Delete the Node BFF.** Incumbent conceded it adds nothing Kafka and Postgres don't already
provide. Angular talks to Spring directly. One layer, one repo, one deploy gone, nothing built yet.

**3. Stand up a Go binary that serves nothing.** One pod, `LISTEN quotes_changed`, log each row,
no route. Proves the data path in production before anyone sees it. Delete = scale to zero.

**4. Move one screen, server-rendered, behind a 1% split.** Go renders `html/template`; the SSE
tick stays with Angular. Port the round-1 tests to `go test` — they must pass unchanged, or the
port is wrong. Ratchet 1% → 100% over a week.

**5. Move SSE to Go, delete the MFE.** Go writes the `<tr>` fragment, HTMX swaps it, one Preact
island for the status badge and nothing more. Delete `angular-mfe/`, `node-bff/`, the Module
Federation config. 15 concepts become 9; 520 hand-written lines plus 350 unread become 340 with
none unread; 95s to build becomes 1.8s; 3.2s cold start becomes 45ms.

**Steal from deno on the way:** one golden-path file per pattern, and a CI grep that fails the
build when a business rule appears in more than one place.
