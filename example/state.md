<!-- Context bridge between sessions. Replace the contents of this file before ending a session so the next one can pick up without re-reading the entire codebase. -->

# State

## Where we ended
Mid-way through cutting the read path over to the blob payload store (the 2026-05-02 decision). `src/store/payloads.ts` is written and tested — `writePayload(eventId, raw)` and `readPayload(eventId)`, both resolving against `PAYLOAD_DIR` from `src/config.ts`, both verifying the SHA-256 digest on read. Ingest already writes through it: `src/routes/ingest.ts:71` calls `writePayload` and stores `payload_path` + `payload_digest` instead of the raw text.

The read side is half-converted. `src/queue/worker.ts` goes through `readPayload`. `src/queue/replay.ts` still does `SELECT payload FROM events` directly at line 58 and hands the result straight to `JSON.parse` at line 64. That's the failing test.

Migration `006_payload_pointer.sql` is applied on the dev box (adds `payload_path`, `payload_digest`, makes `events.payload` nullable). Not applied in production yet — the backfill has to finish first.

## What's working
- Ingest for all three providers: `POST /hooks/:provider` with signature verification in `src/verify/{stripe,github,shopify}.ts`. 41 tests over the verify module, all green, including the replayed-signature and clock-skew cases.
- Worker claim/dispatch loop (`src/queue/worker.ts`) against the blob store — 12k events processed on the dev box since the cutover with no digest mismatches.
- Retry and dead-lettering (`src/queue/retry.ts`), email and Slack channels (`src/notify/channels/email.ts`, `slack.ts`).
- `npm test` — 231 of 238 passing.

## What's broken / in-progress
- **7 failing tests, all in `test/queue/replay.test.ts`.** `TypeError: Cannot read properties of null (reading 'length')` at `src/queue/replay.ts:64` — `events.payload` is NULL for anything written after migration 006, and replay never learned about the pointer columns.
- **Backfill is partial.** `scripts/backfill-payloads.ts` has moved 1,208,340 of 3,847,912 rows out of the `payload` column and onto disk. It's resumable: `npm run backfill -- --since-id 1208340`. Last run took 38 minutes for that first tranche.
- **SMS channel is a stub.** `src/notify/channels/sms.ts` throws `NotImplementedError`. `dispatch.ts:44` filters `sms` out of the channel list, so nothing reaches it in practice — but the filter is a temporary guard, not a design.
- `src/sync.ts` exists and polls Shopify every 5 minutes. It was added in a4f9c2e to chase missed orders and it contradicts the 2026-03-12 decision. Nobody has decided whether to keep it or delete it. Do not treat it as settled.

## Decided this session
Nothing new. The session was implementation of the 2026-05-02 blob-store decision, not new ground. The `src/sync.ts` question was raised and deliberately not resolved.

## Next session should start with
Fix `src/queue/replay.ts` — replace the direct `SELECT payload` at line 58 with `readPayload(event.id)`, matching what `worker.ts:96` already does, and let the digest check run. That clears all 7 failing tests. Then:
1. Resume the backfill (`npm run backfill -- --since-id 1208340`), on its own box — see landmines.
2. Settle `src/sync.ts`: either delete it and reopen the 2026-03-12 decision honestly, or write the decision entry that supersedes it. Leaving it undocumented is the worst of the three.
3. Only after the backfill completes: apply migration 006 in production.

## Landmines
- **Do not run the backfill and ingest in the same process.** `better-sqlite3` is synchronous; the backfill holds the event loop for seconds at a time and ingest requests time out at the proxy (30s, ops repo) with no error in our logs. Run it against a separate process on the replica path.
- `src/verify/shopify.ts` decodes the HMAC as base64 while `stripe.ts` uses hex. That looks like a copy-paste bug and it is not — Shopify sends base64. There's a comment at line 19; leave it.
- `events.payload` is nullable as of migration 006, so any query that reads it directly is now a latent null-deref. `replay.ts` is the one we know about. `grep -rn "SELECT payload" src/` before assuming it's the only one.
- The digest check in `readPayload` throws on mismatch rather than returning null. That's intentional — a corrupt payload should stop the worker, not get silently dropped into the dead-letter table where it looks like a delivery failure.
- Dead-letter alerting fires off the ops repo's Prometheus rules, not from anything in this codebase. If you're wondering why nothing here pages, that's why.
