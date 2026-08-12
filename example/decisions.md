<!-- Append-only log of major decisions. Each entry: what was decided, when, why, and what was rejected. Versioned in git.

Entry format:

### YYYY-MM-DD — [Decision title]
**Status:** active
**Why:** [Rationale — what drove the decision]
**Rejected:** [What was considered and passed over, and why]

Status is `active` or `superseded`. An entry with no Status line is active. The Status line is the only part of an existing entry that may ever be edited. -->

# Decisions

### 2026-02-18 — SQLite (better-sqlite3) as the event store
**Status:** active
**Why:** relay runs on one box and will for the foreseeable future. Ingest is append-heavy and strictly ordered; SQLite in WAL mode handles the write volume we projected (peak ~400 events/min) with no network hop and no second thing to operate. `better-sqlite3` is synchronous, which makes the ingest handler a straight line with no connection pool to misconfigure.
**Rejected:** Postgres — real operational weight (backups, connections, a second deploy target) for a single-node service. Redis Streams — fast, but we need to query old events by provider and status months later, and Streams gives us no durable query surface.

### 2026-02-21 — Store raw provider payloads inline in `events.payload`
**Status:** superseded
**Why:** One write path. The whole event, verbatim, in the row it belongs to, so replay is a `SELECT` and nothing can drift between the payload and its metadata.
**Rejected:** Object storage from day one — a second dependency and a second failure mode before we had any idea what our payload volume would look like.

### 2026-02-24 — Hand-written SQL, no ORM
**Status:** active
**Why:** The whole service is about a dozen queries and their shape is stable — insert an event, claim a batch, mark delivered, count dead letters. An ORM buys nothing against that, and `better-sqlite3`'s prepared statements are already the fastest path. Every query stays greppable.
**Rejected:** Drizzle — closest fit, but the migration story and the type ceremony cost more than twelve queries are worth. Prisma — a query engine binary and a schema DSL to own an SQLite file we can read with the `sqlite3` CLI.

### 2026-03-12 — Webhook push only, no polling reconciliation
**Status:** active
**Why:** All three providers deliver at-least-once and expose a manual replay endpoint for anything we miss. Polling on top of that would double our API quota, and worse, it creates two paths that can each claim to be the truth about an event's existence. Missed events are recovered by asking the provider to replay, and the unique index on `(provider, provider_event_id)` makes the duplicate harmless.
**Rejected:** A nightly reconciliation poll against each provider's list endpoint. It's the obvious safety net, and it's exactly the second truth path we don't want. If we're dropping events, the fix is in ingest, not in a job that hides the drop.

### 2026-03-27 — Retry with exponential backoff and jitter, 6 attempts, then dead-letter
**Status:** active
**Why:** Delivery failures are overwhelmingly transient (provider 5xx, our own deploy restarts). Six attempts over roughly 45 minutes clears those. Past that, the failure is structural — a bad endpoint, a revoked token — and retrying forever just buries it. Full jitter, because our failures arrive in bursts and a synchronized retry wave is how one provider outage becomes our outage.
**Rejected:** Unbounded retry with a growing ceiling — nothing ever surfaces, and the dead-letter table stays empty and useless as a signal. Fixed 5-minute interval — simpler, but it thunders.

### 2026-04-18 — Ingest request timeout enforced at the edge, not in app code
**Status:** active
**Why:** A 30-second budget on `POST /hooks/:provider` belongs in one place, and that place is nginx in the ops repo. Putting a second timeout in the handler means two numbers that drift apart, and the app's version can't actually stop a hung socket the way the proxy can.
**Rejected:** A per-request timeout wrapper in `src/routes/ingest.ts`. Rejected specifically because it looks like defense in depth and is really just a second source of truth.

### 2026-04-30 — Run the worker in-process, not as a separate service
**Status:** active
**Why:** One process, one deploy, one log stream. The worker loop claims a batch every 2 seconds and dispatches it; at our volume it's idle most of the time. A separate worker service would mean a second deploy target and a shared queue we don't need yet.
**Rejected:** BullMQ on Redis — a queue is the right shape eventually, but it adds Redis to a service whose entire point is that it's one box and one file.

### 2026-05-02 — Move raw payloads to an on-disk blob store, keep a pointer in SQLite
**Status:** active
**Why:** The database hit 8.4 GB and the nightly backup window went past 20 minutes, almost all of it payload text we read maybe once. Payloads now land in `var/payloads/<event_id>.json` and the row keeps a path plus a SHA-256 digest, which also gives us a corruption check we never had. The database drops to roughly 300 MB.
**Rejected:** Compressing the inline column — buys one order of magnitude and makes the rows unreadable from the `sqlite3` CLI, which is how we debug. Truncating payloads past 30 days — replay is the one thing the event store exists for; we're not trading it away for disk.
