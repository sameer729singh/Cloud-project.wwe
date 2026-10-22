# Stock Alerts: core engine

Core of the AI Stock Finder & Price Alert Agent for NSE/BSE. This package contains the parts
that decide *correctness*: the alert engine, the provider abstraction, notification delivery,
ownership checks, and tests. UI, auth, AI chat and real provider adapters are **not built yet**.

## Run the tests
```
npm install
npm test          # 37 tests, node:test via tsx, no database or network needed
npm run typecheck
```

## What is here
| Path | Purpose |
|---|---|
| `src/market/provider.ts` | `MarketDataProvider` interface, `ProviderError`, freshness classification |
| `src/market/mockProvider.ts` | Scriptable **DEMO** provider (every quote has `isDemo: true`) |
| `src/alerts/evaluate.ts` | Pure condition logic (integer-paise comparison) |
| `src/alerts/engine.ts` | One monitoring cycle: lease, recovery, market status, quotes, trigger, deliver |
| `src/alerts/notifier.ts` | Message format, per-channel retries, idempotent delivery |
| `src/alerts/service.ts` | Alert CRUD with validation, per-user limit, ownership checks |
| `src/watchlists.ts` | Watchlist CRUD with ownership checks |
| `src/alerts/store.ts` | `Store` interface + `InMemoryStore` |
| `prisma/schema.prisma` | Postgres schema (not yet validated, see below) |

## Condition semantics
- `ABOVE` / `BELOW`: strict comparison.
- `REACHES`: price equals target, or jumped across it since the last observation.
- `CROSSES_ABOVE`: previous < target and now >= target. `CROSSES_BELOW` is the mirror.
- Crossing needs a previous price. The first observation never counts as a crossing.
- An alert fires only while "armed". Firing disarms it; it re-arms when the condition turns false.
  Repeat-enabled alerts therefore notify again only after leaving and re-entering the zone.
  One-shot alerts disable themselves after firing.
- Editing target/condition re-arms. Resuming a paused alert also clears the stored previous price,
  so an old price can't create a fake crossing.

## Safety behaviour of the engine
- Evaluates only when the market status is `OPEN` (holiday, closed, unknown: skipped, no quote requests).
- Quotes older than `staleAfterMs` (plus any declared feed delay), future-dated, or non-positive: skipped.
- End-of-day data is never used to trigger alerts.
- Rate limit: cycle stops cleanly, engine backs off for `retryAfterMs` without calling the provider.
- One worker at a time via a lease; duplicate triggers blocked by a unique dedup key
  (`alertId:quoteTimestamp`). Event + alert state commit together (`commitTrigger`).
- Notifications use the quote's market timestamp. Undelivered events are retried on later cycles
  (including after a crash); channels already delivered are never resent.
- Simulated prices always carry "DEMO DATA" in the message.

## Known gaps (honest list)
1. **No Postgres `Store` yet.** `InMemoryStore` is atomic only because JS is single-threaded.
   The Prisma store must make `commitTrigger` one transaction (unique index on `deduplicationKey`)
   and `acquireLease` an atomic compare-and-set.
2. **`schema.prisma` is unvalidated.** Run `npx prisma validate` then `migrate dev`.
3. **No real market-data adapter.** Provider choice depends on current pricing, terms, and
   real-time entitlements; verify those with the vendor before picking. Implement `MarketDataProvider`
   and the engine needs no changes. Until then only the DEMO provider exists.
4. **Email and Web Push notifiers** are not implemented (`InAppNotifier` is a no-op by design;
   the event row is the in-app notification).
5. **Not built:** HTTP API, authentication, AI query layer, scanner/screener, charts, dashboard UI.
6. No market calendar yet; market status comes from the provider.
7. Tests ran under Node 22 with a minimal type shim for typechecking because `@types/node`
   could not be installed offline; run `npm run typecheck` after `npm install`.

## Next steps (suggested order)
Prisma store + integration tests on a real Postgres, then a real provider adapter, then API routes
and auth, then the dashboard and AI layer.
