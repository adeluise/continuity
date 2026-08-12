# Example audit run

What the audit skill printed on 2026-06-12, run against the `decisions.md` in this directory and the relay codebase. Reproduced verbatim, including the exchange at the end.

---

## Decision Audit

Audited 7 active decisions (1 superseded entry skipped).

### Holding

- **2026-02-18 — SQLite (better-sqlite3) as the event store** — `src/db/client.ts:9` still opens a single file handle and sets `PRAGMA journal_mode = WAL`. `package.json` has `better-sqlite3` and no `pg`, `postgres`, or `ioredis` dependency. `git log -S "postgres"` returns nothing.
- **2026-02-24 — Hand-written SQL, no ORM** — 14 prepared statements across `src/db/events.ts` and `src/db/deliveries.ts`, all literal SQL. No `drizzle`, `prisma`, `typeorm`, or `knex` in `package.json`, and `git log -S "drizzle"` is empty — this was never tried and reverted, it simply hasn't been reached for.
- **2026-05-02 — Payloads on disk, pointer in SQLite** — `src/store/payloads.ts` implements it, migration `006_payload_pointer.sql` adds `payload_path` and `payload_digest`, and `src/routes/ingest.ts:71` writes through it. Noted, not counted against the decision: `src/queue/replay.ts:58` still reads `events.payload` directly. That's the in-flight migration `state.md` describes, not drift away from the decision.

### Violated

- **2026-03-12 — Webhook push only, no polling reconciliation** — commit `a4f9c2e` ("Backfill missed Shopify orders", 2026-05-19) added `src/sync.ts`, which runs `setInterval(pollShopifyOrders, 300_000)` at line 88 and is started from `src/index.ts:57`. This is the nightly reconciliation poll the decision explicitly rejected, on a five-minute period instead of nightly. No decision entry accompanies it. `state.md` lists it as unsettled, so the drift is known — it just isn't recorded where the next person would look.
- **2026-03-27 — Retry: 6 attempts, then dead-letter** — `src/queue/retry.ts:23` reads `const MAX_ATTEMPTS = 12`. Commit `7d1e4b0` ("Give flaky endpoints more room", 2026-04-22) changed it from 6, with no entry in `decisions.md`. Backoff and full jitter are intact at lines 31-38; only the attempt ceiling moved. The dead-letter table is reached after roughly 4 hours now rather than 45 minutes.

### Superseded candidates

- **2026-04-30 — Run the worker in-process, not as a separate service** — commit `0b8c31d` ("Split worker out of the web process", 2026-06-02) added `src/worker/main.ts` and a `Procfile` with separate `web:` and `worker:` entries; `src/index.ts` no longer starts the loop. The split is deliberate and complete, so this reads as the decision being replaced rather than violated — but there's no entry recording what replaced it. Proposed: flip this entry to `superseded`, and write the new decision explaining why the split happened.

### Cannot verify

- **2026-04-18 — Ingest timeout enforced at the edge, not in app code** — half of this is confirmed: there is no timeout wrapper in `src/routes/ingest.ts` and no `timeout` option on the Fastify instance in `src/index.ts`, so the app side of the decision holds. The other half — that a 30-second budget is actually enforced by nginx — lives in the ops repo and isn't visible from here. What would settle it: the `proxy_read_timeout` value in the ops repo's nginx config for the `/hooks/` location, or one slow request against staging.

---

Two of your decisions have drifted and neither drift is recorded. The `src/sync.ts` poll is the consequential one — it's the exact thing the 2026-03-12 entry rejected, and it's been running for three weeks.

Flip **2026-04-30 — Run the worker in-process** to `superseded`?

> yes

Done — `**Status:** superseded` on that entry, nothing else in the file touched. The worker-split decision that replaced it still needs writing; preserve will pick it up if you decide it this session.
