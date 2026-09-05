# Verdict

**Go wins, 103 of 120.** Never best on any axis — deno reads cleaner, rust fails louder,
incumbent already runs — but never below 3 with any judge. Under-one-second feedback beat
rust's 6.3s and incumbent's 110s, and that gap compounds every time an agent guesses. Deno
lost on thin corpus and a young runtime. Rust lost on compile time; its typed `Err(Lagged(n))`
was the debate's best single idea. Incumbent lost on arithmetic: 765 lines, 3 repos, two
languages holding the same 30-day number, and a Kafka hop that can swallow 500 updates silently.

Steal from the losers: rust's explicit-lag error, deno's 214-line discipline.

---

# Migration path: incumbent → go

Each step ships alone, is deletable in an hour, and reverses with one `oc rollout undo`.

**1. Write the tests first, against the incumbent.** A `quotes_test.go` table: list renders,
status updates, a quote exactly 30 days old is not expired, a reconnect with Last-Event-ID
backfills. Point them at the Spring endpoint. Green against the old stack, or the spec is wrong.

**2. Kill the Kafka hop.** Replace `QuoteCdcListener`'s Kafka publish with a Postgres trigger
doing `NOTIFY quotes_changed`. Same Java, one dependency less, and it exposes the silent
500-row drop. Revert = redeploy one pod.

**3. Add a Go binary that serves nothing.** One pod on ROSA, `LISTEN quotes_changed`, log
each row. No traffic, no route. Proves the data path in production before anyone sees it.

**4. Serve the list, keep Angular's live tick.** Go renders `list.html` server-side behind a
1% route split; the existing MFE still owns SSE. Add `updated_at` and the backfill query —
the thing nobody had. Ratchet 1% → 100% over a week, roll back per-percent.

**5. Move SSE to Go, delete the BFF and the MFE.** Go writes the `<tr>` fragment, HTMX swaps
it, one Preact island paints the status pill. Delete `bff/`, delete `quotes-mfe/`, delete the
Kafka topic. Three repos become one, 765 lines become ~700, three copies of "30 days" become one.
