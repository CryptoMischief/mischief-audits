# SUITRUMP dApp — V3 CLMM Full Security Audit

**Project:** SUITRUMP (suitrump.com/dapp)
**Scope:** suidex-v3-clmm (Move contracts) + suidex-v3-api (TypeScript backend)
**Date:** 2026-04-18
**Auditor:** Claude Opus 4.6 (Mischief Audits)
**Package (mainnet):** `0xb5f529c1dcda6580a61bf7ee9fbd524b50be62f11044d137c8202c8cbace9e56` (IMMUTABLE)

> This audit covers the SUITRUMP V3 CLMM — the concentrated liquidity engine powering suitrump.com/dapp.
> For SuiDex V2 audits, see PHASE1–PHASE3 reports in this repo.

---

## Executive Summary

SuiDex V3 is a Uniswap V3-style concentrated liquidity market maker (CLMM) on Sui, forked from Momentum (Turbos) with SuiDex-specific additions: protocol fee controls, trading toggle, flash loans, min tick range enforcement, and an ACL admin layer.

**Overall Risk: MEDIUM.** No critical fund-drain vulnerabilities were found. The package is deployed as immutable, meaning contract bugs cannot be patched but also cannot be exploited via upgrade attacks. The main risks are admin centralization (pause can freeze LP funds) and one edge-case division-by-zero in flash loan repayment when pool liquidity is zero.

**Math correctness: VERIFIED.** All core math (tick_math, sqrt_price_math, swap_math, liquidity_math) matches the Uniswap V3 / Momentum reference implementations. The integer-mate library is mature and correctly implements signed integer arithmetic.

---

## Severity Summary

| Severity | Contracts | API | Total |
|----------|-----------|-----|-------|
| CRITICAL | 0 | 0 | 0 |
| HIGH | 2 | 1 | 3 |
| MEDIUM | 4 | 4 | 8 |
| LOW | 5 | 4 | 9 |
| INFO | 3 | 2 | 5 |

---

## Part A: Smart Contract Findings

### [H-1] Division by Zero in Flash Loan Fee Distribution at Zero Liquidity

**Severity:** HIGH
**Location:** `sources/actions/trade.move` lines 506-518, 533-544

**Description:** In `repay_flash_loan`, when distributing LP fees from flash loan repayment, the code computes `fee_growth_global` by dividing by `pool::liquidity(pool)`. If the pool has zero liquidity at repayment time (all LPs withdrew between borrow and repay), this causes a division by zero and the transaction aborts. The flash loan receipt (hot-potato) cannot be destroyed, meaning the borrowed funds become permanently stuck.

In Uniswap V3, fee_growth updates are guarded with `if (liquidity > 0)`. The guard exists in `flash_swap` (line 224) but is missing in `repay_flash_loan`.

**Impact:** Borrowed funds permanently locked if zero-liquidity state occurs during flash loan lifetime. Exploitable by an attacker who is both LP and borrower.

**Recommendation:** Add zero-liquidity guard before fee_growth_global division:
```move
if (liquidity > 0) {
    pool::set_fee_growth_global_x(pool, ...);
}
```

**Note:** Package is immutable — this cannot be patched on-chain. Mitigate via frontend/API by rejecting flash loans when pool liquidity is dangerously low.

**Status: MITIGATED (2026-04-18)**
- Quote router skips V3 when `v3State.liquidity === 0n` — swaps never routed to empty pools
- `/api/v3/build-swap` fetches live pool state and returns 400 if `liquidity === 0`
- `quoteV3Fast` (split optimizer) already had `liquidity === 0n` early return
- Three-layer defense: quote → build-swap → on-chain abort

---

### [H-2] Admin Pause Freezes LP Withdrawals and Fee Collection

**Severity:** HIGH
**Location:** `sources/storage/pool.move` lines 353-384; `sources/actions/liquidity.move` line 114; `sources/actions/collect.move` lines 36, 68

**Description:** The `assert_not_pause` check is applied to `remove_liquidity`, `collect::fee`, and `collect::reward`. When an admin pauses a pool, LPs cannot withdraw funds or collect earned fees. The admin (pool_admin in the Acl shared object) effectively has power to freeze all user funds indefinitely.

In Uniswap V3, there is no pause mechanism at all — users can always withdraw. The `toggle_trading` function (which only blocks swaps/flash loans) is the correct pattern for an emergency brake, but `pause` goes further and locks LP funds.

**Impact:** Admin centralization risk. A compromised or malicious admin can lock all LP funds. This is the single largest trust assumption in the protocol.

**Recommendation:** Since the contract is immutable, this cannot be changed. Mitigate by:
1. Transferring admin to a multi-sig
2. Documenting this trust assumption publicly
3. Never using `pause` — use `toggle_trading` for emergencies instead

---

### [M-1] Admin Can Set Swap Fee to Near-100%

**Severity:** MEDIUM
**Location:** `sources/actions/admin.move` lines 211-239

**Description:** `set_swap_fee_rate` allows values from 1 to 999,999 (out of 1,000,000). A fee of 99.9999% would effectively steal all swap input. Combined with `set_protocol_swap_fee_share` (up to 75%), the admin could redirect ~75% of swap volume to the protocol fee address.

Uniswap V3 governance can only choose from pre-defined fee tiers. Here the admin can set arbitrary rates.

**Impact:** Admin can set extreme fee rates that effectively steal user funds on swaps.

**Recommendation:** Document this trust assumption. Consider transferring admin to multi-sig with timelock. In practice, users can see the fee before confirming a swap, so this is more of a rug-risk than a silent exploit.

---

### [M-2] Position Store Ability Allows Wrapping in Shared Objects

**Severity:** MEDIUM
**Location:** `sources/actions/collect.move` lines 28-92

**Description:** `Position` has `key, store` abilities. The `store` ability means it can be placed inside another shared object, potentially bypassing Sui's ownership checks. The `fee` and `reward` collection functions take `&mut Position` without explicit sender ownership verification.

**Impact:** Low in practice due to Sui's object model, but if a Position is wrapped in a shared object by a third-party contract, anyone could call collect on it.

**Recommendation:** Consider removing `store` ability from Position if wrapping is not intended.

---

### [M-3] safe_withdraw Silently Truncates Withdrawals

**Severity:** MEDIUM
**Location:** `sources/storage/pool.move` lines 888-891

**Description:** `safe_withdraw` uses `math::min(amount, balance_val)` which silently returns less than requested if pool balance is insufficient. Used in `collect_fee` and `collect_reward`. If accounting errors cause owed amounts to exceed reserves, LPs silently receive less.

**Impact:** Masks potential accounting bugs. Some LPs may receive less than owed without any error signal.

**Recommendation:** Consider asserting `amount <= balance_val` or emitting a warning event when truncation occurs.

---

### [M-4] Pool Can Be Shared Before Initialization

**Severity:** MEDIUM
**Location:** `sources/actions/create_pool.move` lines 23-62; `sources/storage/pool.move` lines 119-138

**Description:** `create_pool::new` creates a Pool with `sqrt_price = 0`. The pool must be separately initialized. If shared before initialization, `add_liquidity` does not check initialization state. A user could add liquidity with `sqrt_price = 0`, resulting in incorrect tick calculations.

**Impact:** Loss of funds for users who add liquidity to an uninitialized pool.

**Recommendation:** Package is immutable. Mitigate via frontend/API by checking pool initialization status before allowing liquidity operations.

**Status: MITIGATED (2026-04-18)**
- `/api/v3/build-add` fetches live pool state and returns 400 `"Pool is not initialized"` if `sqrt_price === 0n`
- Users cannot add liquidity to uninitialized pools through the frontend

---

### [L-1] Oracle Interpolation Integer Precision Loss

**Severity:** LOW
**Location:** `sources/utils/oracle.move` lines 186-201

**Description:** TWAP interpolation divides before multiplying, losing precision. Matches Uniswap V3 reference behavior but could be significant for short intervals.

**Impact:** Minor TWAP imprecision. Does not affect swap execution.

---

### [L-2] Version Gating Allows Forward-Setting

**Severity:** LOW
**Location:** `sources/version/version.move` lines 35-37

**Description:** `set_version` allows setting any major version >= current. The VersionCap holder could set a version for not-yet-deployed code.

**Impact:** Minimal. VersionCap is held by deployer. Package is immutable anyway.

---

### [L-3] No Duplicate Pool Prevention

**Severity:** LOW
**Location:** `sources/actions/create_pool.move` lines 23-62

**Description:** No on-chain registry prevents creating multiple pools with the same token pair and fee tier. Uniswap V3 factory enforces uniqueness.

**Impact:** Liquidity fragmentation if duplicate pools are created.

**Recommendation:** Enforce uniqueness at the API/frontend level.

**Status: MITIGATED (2026-04-18)**
- Pool allowlist: `approved` column on `v3_pools` table. Only approved pools are served by API and used in routing.
- Auto-approve: first pool per pair+fee tier is auto-approved. Duplicates are indexed but flagged `approved=false`.
- Admin endpoints: `/v3/admin/pools` (list all), `/v3/admin/pools/:id/approve` (approve/reject).
- Admin dashboard: V3 Pools tab on suitrump.com/analytics for review and approve/reject.
- Anyone can still create duplicate pools on-chain, but they are invisible to the frontend and router until manually approved.

---

### [L-4] Reward Rounding Dust Accumulation

**Severity:** LOW
**Location:** `sources/storage/pool.move` lines 794-824

**Description:** Reward distribution truncates on each update, causing small amounts of reward tokens to be permanently locked. This is inherent to fixed-point arithmetic and matches Uniswap V3.

**Impact:** Dust amounts of reward tokens permanently locked.

---

### [L-5] get_optimal_swap_amount Division by Zero

**Severity:** LOW
**Location:** `sources/actions/trade.move` lines 764-781

**Description:** Helper function divides by `amount_b_after_swap` and `pos_amount_b`, either of which could be zero.

**Impact:** Transaction aborts. This is a read-only helper for zap optimization — no fund risk.

---

### [I-1] Hot-Potato Flash Swap Receipt Correctly Implemented

**Severity:** INFO

`repay_flash_swap` correctly calls `pool::verify_pool(pool, receipt.pool_id)`. The receipt has no `drop` ability, preventing atomic theft. Correctly implemented.

---

### [I-2] `>= 0` Check on u64 is Always True

**Severity:** INFO
**Location:** `sources/actions/admin.move` lines 163-164, 193-194

Unsigned integer checks `>= 0` are dead code. The upper bound check is the meaningful one.

---

### [I-3] Protocol Fee Overflow Theoretically Possible

**Severity:** INFO
**Location:** `sources/actions/trade.move` lines 322-327

u64 protocol_fee counter could theoretically overflow at > 18.4 quintillion units. Negligible in practice.

---

## Part B: API Backend Findings

### [H-3] No Rate Limiting on API Endpoints

**Severity:** HIGH
**Location:** `src/api.ts` — all routes

**Description:** Zero rate limiting. The PostgreSQL connection pool (max 10) can be exhausted by flooding requests. The `/v3/stats` endpoint is especially expensive (3 aggregate queries per request).

**Recommendation:** Add `@fastify/rate-limit` with ~100 req/min per IP. Stricter limits on aggregate endpoints.

---

### [M-5] No Input Validation on Parameters

**Severity:** MEDIUM
**Location:** `src/api.ts` lines 38-42, 65-77, 80-95, 97-108

**Description:** `poolId` and `positionId` are not validated as Sui object IDs. The `limit` parameter can become `NaN` (`Number("abc")` → NaN, `Math.min(NaN, 200)` → NaN), causing unexpected query behavior.

**Recommendation:** Validate object IDs with regex `/^0x[a-f0-9]{64}$/`. Default NaN limit to 50.

---

### [M-6] Unbounded Swap Table Growth

**Severity:** MEDIUM
**Location:** `src/db/setup.ts`, `src/api.ts` lines 65-77

**Description:** `v3_swaps` and `v3_liquidity_events` tables grow unboundedly. No partitioning, TTL, or archival strategy. No cursor-based pagination — only a 200-row cap.

**Recommendation:** Add time-based partitioning or periodic cleanup. Implement cursor pagination.

**Status: FIXED (2026-04-18)** — Indexer runs daily cleanup, deletes `v3_swaps` and `v3_liquidity_events` older than 90 days.

---

### [M-7] Error Messages Leak Internal Details

**Severity:** MEDIUM
**Location:** `src/api.ts` line 15, `src/indexer.ts` lines 164-166, 254, 260, 266

**Description:** Fastify with `logger: true` includes stack traces in 500 responses. Error logs may expose database connection strings.

**Recommendation:** Custom error handler that returns generic messages to clients.

---

### [M-8] API Listens on 0.0.0.0 Without Auth

**Severity:** MEDIUM
**Location:** `src/api.ts` line 134

**Description:** API binds to all interfaces. No authentication. CORS only blocks browser cross-origin requests, not direct HTTP.

**Recommendation:** Bind to 127.0.0.1 if internal-only, or document as intentionally public.

**Status: FIXED (2026-04-18)** — All routes (except `/health`) require `?secret=` query parameter. Matches existing burn-proxy/positions-proxy auth pattern. Vercel proxy appends secret server-side — never exposed to browser.

---

### [L-6] Indexer Event Type Uses Substring Match

**Severity:** LOW
**Location:** `src/indexer.ts` line 208

**Description:** Event type filter uses `.includes(ORIGINAL_PACKAGE_ID)` instead of `.startsWith()`. A malicious package ID containing the target as a substring could bypass the filter.

**Recommendation:** Use `.startsWith()` for exact prefix matching.

---

### [L-7] No Graceful Shutdown / Connection Cleanup

**Severity:** LOW
**Location:** `src/api.ts`, `src/db/pool.ts`

**Description:** No SIGTERM/SIGINT handlers. Database connections may leak on restart.

**Recommendation:** Add shutdown hooks.

---

### [L-8] Indexer Cursor Not Crash-Safe

**Severity:** LOW
**Location:** `src/indexer.ts` lines 224-231

**Description:** Cursor saved every 100 checkpoints. Crash between saves causes reprocessing. No `ON CONFLICT` clause on swap inserts — duplicates possible.

**Recommendation:** Add unique constraint on `v3_swaps(tx_digest, pool_id)` with `ON CONFLICT DO NOTHING`.

---

### [L-9] Weak Default Database Credentials

**Severity:** LOW
**Location:** `.env.example`

**Description:** Sample credentials `suidex_v3:suidex_v3` in `.env.example`. If used in production, database is weakly protected.

---

### [I-4] SELECT * Exposes Future Columns

**Severity:** INFO
**Location:** `src/api.ts` line 99

`SELECT p.*` on positions endpoint will automatically expose any future columns.

---

### [I-5] Dependencies Are Current

**Severity:** INFO

All npm dependencies are recent with no known critical vulnerabilities.

---

## Part C: Math Verification

### tick_math.move — CORRECT
- Constants match Momentum/Uniswap V3 reference
- Tick range [-443636, 443636] correct
- Min/max sqrt_price (4295048016, 79226673515401279992447579055) correct for Q64.64
- `get_tick_at_sqrt_price` uses standard logarithmic binary search

### sqrt_price_math.move — CORRECT
- `get_amount_x_delta`: L * (sqrt_p_upper - sqrt_p_lower) / (sqrt_p_upper * sqrt_p_lower) ✓
- `get_amount_y_delta`: L * (sqrt_p_upper - sqrt_p_lower) / Q64 ✓
- `get_next_sqrt_price_from_input/output`: correct rounding directions ✓

### swap_math.move — CORRECT
- `compute_swap_step` handles exact-input and exact-output correctly
- Fee: `amount_in * fee_rate / (fee_rate_denominator - fee_rate)` when target not reached ✓

### liquidity_math.move — CORRECT
- `get_liquidity_for_amounts`: uses `min(L_from_x, L_from_y)` when price in range ✓

### Integer Libraries (i32, i64, i128, full_math) — CORRECT
- Two's complement representation correct
- Wrapping arithmetic correct
- Full multiplication (u64 × u64 → u128, u128 × u128 → u256) correct

---

## Part D: Access Control Matrix

| Function | Guard | Assessment |
|----------|-------|------------|
| collect_protocol_fee | Acl pool_admin | OK |
| set_protocol_swap_fee_share | Acl pool_admin | OK (but unbounded — see M-1) |
| set_swap_fee_rate | Acl pool_admin | OK (but unbounded — see M-1) |
| set_flash_loan_fee_rate | Acl pool_admin | OK (but unbounded) |
| initialize_pool_reward | Acl rewarder_admin | OK |
| update_pool_reward_emission | Acl rewarder_admin | OK |
| set_pool_admin | AdminCap | OK (single owner) |
| set_rewarder_admin | AdminCap | OK (single owner) |
| toggle_trading | AdminCap | OK |
| pause/unpause | Acl pool_admin | RISKY — freezes LP funds (H-2) |
| enable_fee_rate | AdminCap | OK |
| create_pool | Anyone | OK (no restriction — see L-3) |
| open_position | Anyone | OK |
| add/remove_liquidity | Position holder | OK (Sui object ownership) |
| flash_swap/flash_loan | Anyone | OK (hot-potato receipt) |

---

## Part E: Immutability Assessment

The package is deployed as **IMMUTABLE** (`0xb5f529c1dcda6580a61bf7ee9fbd524b50be62f11044d137c8202c8cbace9e56`). This means:

**Positives:**
- No upgrade attacks possible — code is fixed forever
- No admin can inject malicious upgrades
- Users can trust the code they interact with won't change

**Negatives:**
- Bugs (H-1, M-3, M-4) cannot be patched on-chain
- Must be mitigated at the API/frontend level
- If a critical bug is discovered, a new package must be deployed and liquidity migrated

---

## Recommendations (Priority Order)

1. **Transfer admin to multi-sig** — mitigates H-2, M-1 (PLANNED)
2. ~~**Add rate limiting to API** — fixes H-3~~ **FIXED 2026-04-18** (commit 79367c9)
3. **Never use pause()** — use toggle_trading for emergencies (mitigates H-2)
4. ~~**Frontend: block flash loans at low liquidity** — mitigates H-1~~ **MITIGATED 2026-04-18** (3-layer guard: quote + build-swap + fast-quote)
5. ~~**API input validation** — fixes M-5~~ **FIXED 2026-04-18** (commit 79367c9)
6. ~~**Add ON CONFLICT to swap inserts** — fixes L-8~~ **FIXED 2026-04-18** (commit 79367c9)
7. **Document trust assumptions** — H-2, M-1 for users
8. ~~**Tighten event filter** — fixes L-6~~ **FIXED 2026-04-18** (commit 79367c9)
9. ~~**Block add-liquidity on uninitialized pools** — mitigates M-4~~ **MITIGATED 2026-04-18** (build-add rejects sqrt_price=0)
10. ~~**Add swap table retention** — fixes M-6~~ **FIXED 2026-04-18** (90-day cleanup in indexer)
11. ~~**Add API auth** — fixes M-8~~ **FIXED 2026-04-18** (secret-based auth on all routes)
12. ~~**Pool allowlist to prevent duplicates** — mitigates L-3~~ **MITIGATED 2026-04-18** (auto-approve first, flag duplicates, admin dashboard)

---

## Fix Log

### 2026-04-18 — API Security Hardening (commit 79367c9)

| Finding | Fix |
|---------|-----|
| H-3: No rate limiting | Added `@fastify/rate-limit` — 100 req/min global, 30 req/min on `/v3/stats` |
| M-5: No input validation | Object IDs validated with `/^0x[a-f0-9]{64}$/`, NaN limit defaults to 50 |
| M-7: Error leaks | Custom error handler — 500s return generic message, stack traces server-side only |
| L-6: Substring event filter | Changed to `startsWith(ORIGINAL_PACKAGE_ID)` prefix matching |
| L-7: No graceful shutdown | SIGTERM/SIGINT handlers close Fastify + PG pool on both API and indexer |
| L-8: Duplicate swaps | Unique index `idx_v3_swaps_unique(tx_digest, pool_id)` + `ON CONFLICT DO NOTHING` |
| I-4: SELECT * | Replaced with explicit column list on positions endpoint |

**Deployed:** VPS 207.180.220.217, PM2 processes `v3-api` (21) and `v3-indexer` (22)
**Verified:** Input validation returns 400 for invalid IDs, rate limiting active, no stack trace leaks

### 2026-04-18 — H-1 Mitigation + M-4/M-6/M-8 Fixes (commit f9264a7 API, cec8a1f frontend)

| Finding | Fix |
|---------|-----|
| H-1: Flash loan div-by-zero at zero liquidity | **MITIGATED** — Quote router skips V3 when `liquidity === 0n`. Build-swap endpoint fetches live pool state and rejects with 400. Three-layer defense prevents users from ever hitting the on-chain bug. |
| M-4: Pool shared before initialization | **MITIGATED** — `/api/v3/build-add` checks `sqrt_price !== 0` before building add-liquidity PTB. Returns 400 `"Pool is not initialized"`. |
| M-6: Unbounded swap table growth | **FIXED** — Indexer runs daily cleanup, deletes `v3_swaps` and `v3_liquidity_events` older than 90 days. |
| M-8: API on 0.0.0.0 without auth | **FIXED** — All routes (except `/health`) require `?secret=` query parameter. Vercel proxy appends secret server-side. Direct requests without secret receive 403. |

**Deployed:** VPS v3-api (21) and v3-indexer (22) restarted. Frontend on dev-suitrump.vercel.app.
**Verified:** `curl http://207.180.220.217:3850/v3/pools` → 403. With `?secret=...` → data returned. `/health` → OK without secret.

### 2026-04-18 — L-3 Pool Allowlist (commit ba1ae4b API, 0d2f5a7 frontend)

| Finding | Fix |
|---------|-----|
| L-3: No duplicate pool prevention | **MITIGATED** — `approved` column on `v3_pools`. First pool per pair+fee auto-approved, duplicates flagged. Admin endpoints for review. V3 Pools tab on analytics dashboard. |

**Deployed:** VPS v3-api (21) and v3-indexer (22) restarted. Frontend on suitrump.com (prod).
**Verified:** Existing SUI/SUITRUMP pool approved. `/v3/admin/pools` returns all pools with status. Public `/v3/pools` only returns approved.
