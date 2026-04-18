# SUITRUMP dApp — V3 CLMM Developer Audit Guide

How to independently audit the SUITRUMP V3 concentrated liquidity contracts and API (suitrump.com/dapp).

> This guide is for the SUITRUMP V3 CLMM. For SuiDex V2 audits, see PHASE1–PHASE3 reports in this repo.

---

## What You're Auditing

SuiDex V3 is a Uniswap V3-style CLMM on Sui (Move language). Two repos:

| Repo | Contents | Language |
|------|----------|----------|
| [suidex-v3-clmm](https://github.com/CryptoMischief/suidex-v3-clmm) | On-chain Move contracts | Move |
| [suidex-v3-api](https://github.com/CryptoMischief/suidex-v3-api) | Indexer + REST API | TypeScript |

**Mainnet package (IMMUTABLE):**
```
0xb5f529c1dcda6580a61bf7ee9fbd524b50be62f11044d137c8202c8cbace9e56
```

---

## Setup

```bash
# Clone both repos
gh repo clone CryptoMischief/suidex-v3-clmm
gh repo clone CryptoMischief/suidex-v3-api

# Build contracts (requires Sui CLI)
cd suidex-v3-clmm
sui move build

# Run contract tests
sui move test

# API setup
cd ../suidex-v3-api
npm install
cp .env.example .env
# Edit .env with your database URL
```

---

## Contract Audit Checklist

### 1. Core Swap Logic
**Files:** `sources/actions/trade.move`, `sources/utils/swap_math.move`

- [ ] `flash_swap` correctly iterates through ticks
- [ ] `compute_swap_step` handles exact-input and exact-output
- [ ] Fee calculation: `amount_in * fee_rate / (fee_rate_denominator - fee_rate)`
- [ ] Slippage protection: `sqrt_price_limit` enforced
- [ ] Hot-potato receipt pattern: `FlashSwapReceipt` has no `drop` ability
- [ ] `repay_flash_swap` verifies pool ID matches receipt
- [ ] Protocol fee accumulation doesn't overflow

**Compare against:** [Uniswap V3 SwapMath.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/SwapMath.sol)

### 2. Liquidity Operations
**Files:** `sources/actions/liquidity.move`, `sources/actions/collect.move`

- [ ] `add_liquidity` correctly updates tick state
- [ ] `remove_liquidity` returns correct amounts
- [ ] Fee growth accounting per-position is correct
- [ ] Tick crossing updates fee_growth_outside correctly
- [ ] `collect::fee` and `collect::reward` transfer correct amounts

**Compare against:** [Uniswap V3 Position.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Position.sol)

### 3. Math Libraries
**Files:** `sources/utils/tick_math.move`, `sources/utils/sqrt_price_math.move`, `sources/utils/liquidity_math.move`

- [ ] `get_sqrt_price_at_tick` constants match Uniswap V3
- [ ] Tick range: [-443636, 443636]
- [ ] Min sqrt_price: 4295048016, Max: 79226673515401279992447579055
- [ ] `get_amount_x_delta` and `get_amount_y_delta` formulas correct
- [ ] Rounding directions: round up when protocol receives, round down when protocol pays
- [ ] `get_liquidity_for_amounts` uses `min(L_x, L_y)` when price in range

**Compare against:** [Uniswap V3 TickMath.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol), [SqrtPriceMath.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/SqrtPriceMath.sol)

### 4. Integer Arithmetic
**Files:** `sources/integer-mate/*.move`

- [ ] i32, i64, i128 use correct two's complement representation
- [ ] Overflow/underflow handled correctly
- [ ] full_math_u64 and full_math_u128 multiplication doesn't lose precision
- [ ] Division rounding matches expected behavior

### 5. Access Control
**Files:** `sources/app.move`, `sources/actions/admin.move`, `sources/storage/global_config.move`

- [ ] AdminCap is a single-owner object
- [ ] ACL correctly separates pool_admin and rewarder_admin roles
- [ ] `toggle_trading` only blocks swaps/flash_loans (not withdrawals)
- [ ] `pause` blocks ALL operations including withdrawals (document this!)
- [ ] Fee rate changes have reasonable bounds (currently 0–999,999 out of 1,000,000)
- [ ] Protocol fee share capped at 750,000 (75%)

### 6. Flash Loans
**Files:** `sources/actions/trade.move` (flash_loan, repay_flash_loan)

- [ ] Hot-potato receipt (`FlashLoanReceipt`) has no `drop` ability
- [ ] Repayment amount >= borrowed + fee
- [ ] Fee distribution to fee_growth_global handles zero liquidity (KNOWN BUG — see audit, **MITIGATED** at API/frontend layer: quote router + build-swap reject zero-liquidity pools)
- [ ] Pool ID verified on repayment

### 7. Oracle / TWAP
**Files:** `sources/utils/oracle.move`

- [ ] Observation ring buffer correctly wraps
- [ ] Interpolation between observations is mathematically sound
- [ ] `observe` handles uninitialized observations gracefully

### 8. Version Gating
**Files:** `sources/version/*.move`

- [ ] Version check runs on all public entry functions
- [ ] Package is immutable — version gating is moot but verify pattern is correct

---

## API Audit Checklist

### 1. SQL Injection
**File:** `src/api.ts`

- [ ] All queries use parameterized statements (`$1`, `$2`, etc.)
- [ ] No string concatenation in SQL queries
- [ ] Input parameters validated before use

### 2. Input Validation — **FIXED 2026-04-18**
**File:** `src/api.ts`

- [x] `poolId` and `positionId` validated as Sui object IDs — regex `/^0x[a-f0-9]{64}$/`
- [x] `limit` parameter handles NaN/negative values — defaults to 50
- [x] `owner` address validated

### 3. Rate Limiting — **FIXED 2026-04-18**
- [x] Rate limiting middleware present — `@fastify/rate-limit` 100 req/min global
- [x] Expensive endpoints (stats, aggregates) have stricter limits — 30 req/min on `/v3/stats`

### 4. Indexer Integrity — **FIXED 2026-04-18**
**File:** `src/indexer.ts`

- [x] Event type filter uses exact match (not substring) — `startsWith()` prefix match
- [ ] Event data fields type-checked before DB insert
- [x] Duplicate events handled (ON CONFLICT or unique constraint) — unique index on `(tx_digest, pool_id)`
- [x] Cursor persistence is crash-safe — per-module cursor saved after each batch
- [x] Table retention — daily cleanup deletes rows older than 90 days

### 5. Error Handling — **FIXED 2026-04-18**
- [x] 500 errors don't leak stack traces to clients — custom error handler
- [x] Database connection errors don't expose credentials

### 6. Authentication — **FIXED 2026-04-18**
- [x] API requires `?secret=` query parameter on all routes (except `/health`)
- [x] Vercel frontend proxy appends secret server-side — never exposed to browser
- [x] Direct requests without secret receive 403 Forbidden

---

## On-Chain Verification

Verify the deployed package matches the source code:

```bash
# 1. Build from source
cd suidex-v3-clmm
sui move build

# 2. Check the deployed package on-chain
# Visit: https://suivision.xyz/package/0xb5f529c1dcda6580a61bf7ee9fbd524b50be62f11044d137c8202c8cbace9e56

# 3. Verify GlobalConfig object
# https://suivision.xyz/object/0x4a0467757ce0bd770dcf696d33f1b732fa37477cc19a084be2235f3999255e83

# 4. Check protocol fee settings
# protocol_fee_share should be 150,000 (15%)

# 5. Verify package is immutable (no UpgradeCap exists)
```

### Key On-Chain Objects

| Object | ID |
|--------|----|
| Package | `0xb5f529c1dcda6580a61bf7ee9fbd524b50be62f11044d137c8202c8cbace9e56` |
| GlobalConfig | `0x4a0467757ce0bd770dcf696d33f1b732fa37477cc19a084be2235f3999255e83` |
| Version | `0x0999bbc9c063580eca62e888b8f0d8e6e9159cd9db1b8a8c88e448a2b5dd4d4d` |
| ACL | `0xbe43b74d23e11c8fa8ce69c2295f92bf1e6388e5118114cfc0e59ed2eeb91c6d` |
| Deployer wallet | `0x9600a1c55bce1419982183d9fad13a2b9d562d1223564dfb509c9088333241c2` |

### Live Pool

| Pool | ID |
|------|----|
| SUI/SUITRUMP (0.30%) | `0xdf8ccfcc10f7daf14e31101c8ca6ac05eaa953afad14195fd2db3a41bad4b284` |

---

## What to Focus On

**Highest priority areas** (most likely to have exploitable bugs):

1. **Swap loop** (`flash_swap` in trade.move) — this is where user funds are at risk
2. **Fee accounting** — fee_growth_global, per-position fee tracking, protocol fee split
3. **Tick crossing** — state transitions when price moves across initialized ticks
4. **Flash loan repayment** — must be atomic and verify correct amounts
5. **Admin powers** — what can admin do? Can they steal funds?

**Lower priority** (well-tested math libraries):
- Integer arithmetic (forked from battle-tested Momentum code)
- Tick math constants (copied from Uniswap V3)

---

## Reference Implementations

- [Uniswap V3 Core](https://github.com/Uniswap/v3-core) — original Solidity implementation
- [Momentum](https://github.com/turbos-finance/momentum-protocol) — Sui Move CLMM this is forked from
- [Cetus CLMM](https://github.com/CetusProtocol/cetus-clmm-sui) — another Sui CLMM for comparison

---

## Reporting

If you find vulnerabilities:
1. **CRITICAL/HIGH:** Contact the team immediately via DM before public disclosure
2. **MEDIUM/LOW:** Document in a structured report with: severity, location (file:line), description, impact, recommendation
3. Use the severity scale: CRITICAL (fund loss), HIGH (significant risk), MEDIUM (moderate risk), LOW (minor), INFO (informational)

Existing audit report: [SUITRUMP_V3_CLMM_AUDIT.md](./SUITRUMP_V3_CLMM_AUDIT.md)

---

## Current Fix Status (as of 2026-04-18)

| ID | Severity | Finding | Status |
|----|----------|---------|--------|
| H-1 | HIGH | Flash loan div-by-zero at zero liquidity | **MITIGATED** — 3-layer frontend/API guard (quote, build-swap, fast-quote). On-chain bug remains (immutable). |
| H-2 | HIGH | Admin pause freezes LP funds | **OPEN** — Multi-sig transfer planned. Use `toggle_trading` not `pause`. |
| H-3 | HIGH | No rate limiting | **FIXED** — `@fastify/rate-limit` |
| M-1 | MEDIUM | Admin can set fee to ~100% | **OPEN** — Mitigated by multi-sig (planned) |
| M-2 | MEDIUM | Position `store` ability | **ACCEPTED** — Low practical risk |
| M-3 | MEDIUM | safe_withdraw truncates | **ACCEPTED** — Immutable, inherent to design |
| M-4 | MEDIUM | Pool shared before init | **MITIGATED** — build-add rejects `sqrt_price=0` |
| M-5 | MEDIUM | No input validation | **FIXED** — Object ID regex + NaN defaults |
| M-6 | MEDIUM | Unbounded table growth | **FIXED** — 90-day retention cleanup in indexer |
| M-7 | MEDIUM | Error message leaks | **FIXED** — Custom error handler |
| M-8 | MEDIUM | API on 0.0.0.0 no auth | **FIXED** — Secret-based auth on all routes |
| L-6 | LOW | Substring event filter | **FIXED** — `startsWith()` prefix match |
| L-7 | LOW | No graceful shutdown | **FIXED** — SIGTERM/SIGINT handlers |
| L-8 | LOW | Indexer cursor not crash-safe | **FIXED** — Unique index + ON CONFLICT |
| I-4 | INFO | SELECT * | **FIXED** — Explicit column list |

**Remaining open items:**
1. Transfer admin to multi-sig (H-2, M-1)
2. Change default DB password (L-9)
3. Event data type-checking before DB insert
