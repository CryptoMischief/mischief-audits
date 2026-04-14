# Mischief Phase 1 Audit — SuiDex V2

**Auditor:** CryptoMischief
**Date:** April 14, 2026
**Scope:** Full codebase review of Phase 1 (Dev branch) — all 84 source files across 13 packages and 4 apps, plus whitepaper, blueprint, CLAUDE.md, and TASKS.md.

---

## Overall Assessment

The scaffolding is **well-structured and architecturally sound**. The code is clean, well-documented, follows the rules in CLAUDE.md, and correctly implements the Sui SDK 2.0 patterns that have been battle-tested in production. That said, there are real bugs, gaps, and a few issues that would bite you in production.

---

## What's Actually Done (vs TASKS.md)

TASKS.md shows all items unchecked. In reality, **Track A — Foundation** is substantially built:

| Component | Status | Notes |
|---|---|---|
| pnpm monorepo | Done | Clean workspace config |
| `packages/constants/` | Done | All IDs, types, events, fee math |
| `packages/suidex-config/` | Done | Zod env, shared types |
| `packages/sui-clients/` | Done | gRPC + GraphQL singletons |
| `packages/database/` | Done | Schema, migration, 7 repositories |
| `packages/queue/` | Done | 6 BullMQ queues |
| `packages/admin-types/` | Done | Zod schemas |
| Indexer | Done | Subscriber, dispatcher, 10 handlers |
| Workers | Done | 5 enrichment workers |
| API | Done | 4 route groups, middleware |
| Admin API | Done | IP allowlist, audit logging |
| Frontend packages | Done | tx-helpers, wallet-core, ui, api-client, config |

**Not started:** Track B (routing engine), Track C (CDN/nginx), Track D (route pre-computation), no tests, no frontend apps.

---

## Critical Issues (Must Fix)

### 1. Price worker uses `Number()` for reserves — violates CLAUDE.md Rule #4

**File:** `backend/apps/workers/src/price-enrich.ts` lines 85–104

```typescript
const suiAmount = Number(reserve0) / 10 ** decimals0;  // PRECISION LOSS
tvlUsd = suiAmount * suiUsd * 2;
```

Reserves are `numeric(78,0)` — they can exceed `2^53`. Any pool with large reserves will get wrong TVL/prices. This is exactly the bug CLAUDE.md warns about.

**Fix:** Use a decimal library (e.g. `decimal.js`) or scale down BigInt before converting to Number. For price display, precision loss from Number is acceptable only after dividing by `10^decimals` since the result is small enough. But the current code does `Number(reserve0)` first which overflows for large reserves.

---

### 2. Indexer backfill has a potential infinite loop / memory issue

**File:** `backend/apps/indexer/src/subscriber.ts` lines 184–210

The backfill loop fetches one checkpoint at a time from `fromSeq` to `latestSeq`. If the indexer is days behind, this could be millions of sequential gRPC calls with no batching, rate limiting, or progress logging within the loop. It will:
- Potentially hit rate limits on the fullnode
- Take hours with no feedback
- Not resume if it crashes mid-backfill (cursor saves after each, but no progress indicator)

**Fix:** Add batch processing (fetch N checkpoints concurrently), progress logging every 100 checkpoints, and a delay between calls to avoid rate limiting.

---

### 3. Subscriber `subscribeCheckpoints({})` API may not exist as written

**File:** `subscriber.ts` line 152

```typescript
const streamCall = client.subscriptionService.subscribeCheckpoints({});
```

The `@mysten/sui` SDK v2.x exposes `subscriptionService` but the actual method name and proto shape varies between versions. As of `@mysten/sui@2.15.0`, the streaming API may use a different path. The code imports `GrpcTransport` from `@protobuf-ts/grpc-transport` which suggests manual proto setup, but the `SuiGrpcClient` may not expose `subscriptionService` directly.

**Fix:** Verify this compiles and connects against a live mainnet node. The test script (`scripts/test-wallet-queries.mts`) doesn't test streaming.

---

### 4. Sync handler uses `new Date()` instead of checkpoint timestamp

**File:** `backend/apps/indexer/src/handlers/sync.ts` line 47

```typescript
timestamp: new Date(),  // Indexer processing time, NOT on-chain time
```

Reserve snapshots should use the checkpoint's timestamp, not the time the indexer processes it. During backfill, this means all historical snapshots get today's date — breaking time-series queries entirely.

**Fix:** Pass the checkpoint timestamp through the event data or resolve it from the checkpoint metadata.

---

### 5. Swap handler doesn't validate token type extraction

**File:** `backend/apps/indexer/src/handlers/swap.ts` lines 36–38

```typescript
const genericMatch = event.type.match(/<(.+),\s*(.+)>/);
const token0Type = genericMatch?.[1]?.trim() ?? "";
const token1Type = genericMatch?.[2]?.trim() ?? "";
```

If the regex fails (malformed event type), both token types become empty strings `""`, and a swap event is inserted with blank token types. The DB allows this because `token_in_type` and `token_out_type` are `text NOT NULL` with no CHECK constraint.

**Fix:** Return early if `genericMatch` is null. Log a warning.

---

## Important Issues (Should Fix)

### 6. No `volume_24h_usd` calculation anywhere

The `pairs` table has `volume_24h_usd` but no code ever updates it. The API returns it in pool data, but it's always `"0"`. The price enrichment worker only updates `tvl_usd`.

**Fix:** Add a periodic job (or compute on-read) that sums `swap_events.amount_in` converted to USD for the last 24h.

---

### 7. CoinGecko rate limit risk in price worker

`price-enrich.ts` calls CoinGecko on every price enrichment job. With 5 workers and high swap volume, this will quickly exceed the free tier (~10-30 req/min). Every Sync event triggers a price enrichment.

**Fix:** Cache the SUI/USD price for 60s (Redis or in-memory) and share across all worker instances.

---

### 8. Migration hypertable with `serial` PK is problematic

`reserve_snapshots` and `token_price_history` have `serial` PKs but are converted to TimescaleDB hypertables. TimescaleDB requires the partitioning column (`timestamp`) to be part of any unique constraint. A `serial` PK doesn't include timestamp.

**Fix:** Use a composite primary key `(id, timestamp)` or remove the PK and rely on the `pair_id + timestamp` index for lookup. Test the migration against actual TimescaleDB.

---

### 9. Missing `liquidity_events` history API endpoint

TASKS.md lists `GET /dex/history/liquidity` but `routes/dex.ts` only implements `/history/swaps`. LP add/remove events have no API endpoint.

---

### 10. Locker reward accumulator race condition

**File:** `backend/apps/indexer/src/handlers/locker-events.ts`

VictoryRewardsClaimed accumulates via BigInt, but the handler needs to read the current `victory_rewards_claimed` from DB, add the new amount, and write back. If two events arrive for the same lock, a race condition could lose one update without proper serialization.

**Fix:** Use SQL `UPDATE SET victory_rewards_claimed = victory_rewards_claimed + $1` instead of read-then-write.

---

### 11. Token ordering needs verification against live pairs

**File:** `backend/packages/constants/src/pools.ts` lines 69–78

`sortTokenTypes` compares only the package address portion. Two tokens from the same package would fall through to plain string comparison (line 77). Move's actual sorting compares the full BCS-encoded type, not just lexicographic string comparison.

**Fix:** Verify against known live pairs that this ordering matches what the contract produces. Wrong ordering = reserves stored in wrong columns. This has caused production bugs before.

---

## Security Review

### Positive
- No SQL injection vectors (Kysely parameterized queries throughout)
- Deterministic event IDs prevent duplicates
- Cursor saved AFTER processing (crash-safe)
- Admin API has IP allowlist + audit logging with SHA-256 payload hash
- Rate limiting on public API (100 req/15s)
- CORS per-origin (not wildcard)
- Request size limit (1MB)
- No stack trace leakage in error responses
- Zod validation on all API inputs

### Concerns

**Admin API audit hash only covers request body** — doesn't include operator identity or timestamp in the hash. An operator could replay the same payload.
- Fix: Include operator + timestamp in the hash input.

**IP allowlist dev bypass** — `ip-allowlist.ts` allows all requests when `ADMIN_ALLOWED_IPS` is not set. If the env var is accidentally missing in production, the admin API is wide open.
- Fix: Default to deny-all when env var is missing in production.

**No auth on public API** — rate limiting is the only protection. No API keys. Fine for Phase 1, but document the plan for Phase 2+.

---

## CLAUDE.md Compliance Check

| Rule | Status | Notes |
|---|---|---|
| No JSON-RPC | **Pass** | gRPC everywhere |
| No hardcoded contract IDs | **Pass** | All from `packages/constants/` |
| No hardcoded token prices | **Pass** | CoinGecko fallback to 0, never a fixed value |
| BigInt for token amounts | **Fail** | Price worker uses `Number(reserve)` |
| Contracts read-only | **Pass** | No Move code, no deployment plans |
| Cursor after processing | **Pass** | Correctly implemented |
| Token ordering (padded hex) | **Partial** | Implemented but needs verification against live pairs |
| 9970/10000 fee math | **Pass** | Constants correct, `amm-math.ts` uses them |

---

## Code Quality

### Strengths
- Consistent patterns across all packages
- Good separation of concerns (handlers / repositories / workers)
- TypeScript strict mode everywhere
- Well-documented JSDoc comments referencing CLAUDE.md rules
- Barrel exports with clean public APIs
- Frontend tx-helpers correctly handle all SDK 2.0 gotchas (BCS dynamic fields, response shape variants, 2s buffer)

### Weaknesses
- **Zero tests** — not a single `.test.ts` or `.spec.ts` file exists. The testing protocol in CLAUDE.md is comprehensive but nothing has been written yet.
- **No lint config** — `.eslintrc` or `eslint.config.js` doesn't exist despite `pnpm lint` in scripts.
- **No build step tested** — packages use `"main": "./src/index.ts"` (source-to-source imports). This works in dev but needs verification that the build pipeline actually works.
- Some `any` casts in the indexer (proto types) — documented but worth tracking.

---

## Architecture Alignment with Blueprint

The code matches the architecture blueprint well:
- Hybrid gRPC + GraphQL (gRPC for hot-path, GraphQL singleton ready for complex queries)
- pnpm monorepo with correct workspace structure
- Express + Zod + Kysely stack as specified
- BullMQ + Redis for async processing
- CDN cache middleware ready (headers implemented, CDN infra not deployed)
- Separate deployables (api, admin-api, indexer, workers)
- Frontend packages extracted correctly

**Gap:** No frontend apps yet (`frontend/apps/` is empty). TASKS.md correctly marks this as Phase 2.

---

## Recommendations — Priority Order

1. **Fix `Number(reserve)` in price worker** — data corruption risk
2. **Fix sync handler timestamp** — time-series data is useless with wrong timestamps
3. **Add token type validation in swap handler** — prevents garbage data
4. **Cache SUI/USD price** — will hit rate limits immediately under load
5. **Add basic integration tests** — at minimum: migration up/down, event handler with mock data, API endpoints
6. **Verify subscriber proto API** — `subscribeCheckpoints` must be tested against a live node
7. **Add volume_24h calculation** — the API returns this field but it's always 0
8. **Fix hypertable PK** — test the migration against real TimescaleDB
9. **Harden admin IP allowlist default** — deny-all when env var missing
10. **Add progress logging to backfill** — operational visibility

---

## Summary

The Phase 1 foundation is **solid scaffolding** with correct architectural decisions. The team clearly understands Sui SDK 2.0 gotchas and has built patterns to avoid common production incidents. The critical issues are fixable — the `Number()` precision bug and the sync timestamp bug are the most urgent. The biggest gap is zero tests and the need to verify the gRPC streaming API actually works against a live node before building on top of it.

---

*Audit conducted by CryptoMischief — April 14, 2026*
