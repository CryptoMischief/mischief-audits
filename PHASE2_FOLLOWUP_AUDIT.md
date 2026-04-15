# Mischief Phase 2 Follow-up Audit — SuiDex V2

**Auditor:** CryptoMischief
**Date:** April 15, 2026
**Scope:** 3 new commits on Dev branch (`a55b736`, `349428a`, `da28bb2`) — 167 files changed, ~8,600 lines added. Covers DEX frontend polish/rewrite, new zap page, Firebase hosting + GCP deploy workflows, database seed script, and production readiness audit.

---

## Overall Assessment

Significant improvements. The frontend got a proper rewrite with shared utilities, verified token badges, fallback avatars, route visualization, and a complete zap page. The infra shifted from K8s to Firebase Hosting + Docker-based GCP deployment. Backend services now have health endpoints and graceful shutdown handlers.

However, **several critical Phase 2 findings remain unfixed** (quote/build race condition, abort signals, in-place state mutations), and new issues were introduced (dead RouteDialog wiring, division-by-zero in split routes, build artifacts re-committed). The workers `setInterval` leak has been flagged across three consecutive audits and still isn't fixed.

> **UPDATE April 15, 2026:** All 30 frontend findings (2 critical, 6 high, 12 medium, 10 low) have been **fixed and deployed** to the Dev branch in commit `c2f49e1`, live on test-dex.suidex.org. Consolidated audit with fix status pushed to their repo as `MISCHIEF_AUDIT_PHASE2.md`. 29 backend/infra/security findings remain open for their team.

---

## Phase 2 Finding Resolution Status

### Fully Fixed (5 of 72)

| # | Finding | How Fixed |
|---|---------|-----------|
| F8/F9 | Hardcoded 9 decimals for pool display | Pool detail now looks up actual decimals from token metadata |
| F10 | Stale closure in SwapWidget | Hook dependencies corrected |
| F11 | Every token shows verified badge | TokenSelector now distinguishes verified (API) vs imported (unverified) tokens |
| F12 | Token image error no fallback | New `TokenAvatar.tsx` component with `onError` → colored initials fallback |
| F19 | Duplicated utility functions | New shared `format.ts` library (though some local duplicates remain) |

### Partially Fixed (5 of 72)

| # | Finding | Status |
|---|---------|--------|
| S4/S5 | Docker containers run as root | Dockerfiles now use `USER node`, but K8s manifests still lack `securityContext` |
| S8 | Docker base images not pinned | Dockerfiles pin `node:22-alpine`, but K8s/compose still use `:latest` for Redis/TimescaleDB |
| S12 | CI/CD `|| true` suppresses failures | New deploy workflows use `set -e`, but old `ci-cd.yml` unchanged |
| F6 | Fetch errors silently swallowed | `useSwapRoute` now surfaces errors; pools, zap, portfolio still use `.catch(() => {})` |
| F13 | SwapInfoBar division by zero | Route `null` guard added, but `inputFormatted === "0"` still produces `Infinity` |

### Not Fixed (7 critical/high items)

| # | Finding | Status |
|---|---------|--------|
| **F1 CRIT** | Quote/build race condition | Still uses `Promise.all` — client-side abort check added but signal not forwarded to fetch |
| **F2 CRIT** | AbortController signal not passed to fetch | `fetchRouteQuote`/`fetchRouteBuild` still don't receive the abort signal |
| **F3 HIGH** | `tokensWithBalance` no `useMemo` | Still recomputed every render |
| **F4/F5 HIGH** | N parallel RPC calls, no concurrency limit | `useTokens` and `portfolio` still fire all calls simultaneously |
| **F7 HIGH** | Pool state array mutated in-place | `pools/page.tsx:74` still does `list.sort()` on state reference |
| **F14 MED** | Fee calc incorrect for multi-hop/split | Still `hops.length * 0.3` |
| **I4 HIGH** | Workers don't clear setInterval on shutdown | **3rd consecutive audit flagging this** |

### Worse (1)

| # | Finding | Status |
|---|---------|--------|
| I2 | Build artifacts committed | **32 new** `.js`/`.d.ts`/`.js.map` files added in `frontend/packages/tx-helpers/src/` |

---

## New Critical Issues (1)

### N1. Quote/build race condition persists — plus RouteDialog not wired
**Severity:** CRITICAL
**Files:** `frontend/apps/dex/app/hooks/useSwapRoute.ts:80-83`, `frontend/apps/dex/app/components/SwapWidget.tsx:375-383`

The core race condition from F1/F2 remains: `fetchRouteQuote` and `fetchRouteBuild` run in `Promise.all`. A client-side abort check was added but the signal is never passed to the actual fetch calls, so zombie HTTP requests still complete server-side.

Additionally, the new `RouteDialog` component (which visualizes alternative routes) renders at `SwapWidget.tsx:375` but `onSelectRoute` is never passed. The route comparison dialog is completely dead — clicking a route does nothing.

**Fix:**
1. Fetch quote first, then build. Pass the quote's route selection to the build endpoint.
2. Pass `{ signal: controller.signal }` to both API client methods.
3. Wire `onSelectRoute` in SwapWidget to update the active route.

---

## New High Issues (6)

### N2. Division by zero in split route percentage display — will crash
**Severity:** HIGH
**Files:** `frontend/apps/dex/app/components/RouteDialog.tsx:232`, `RouteDialog.tsx:328`, `SwapInfoBar.tsx:166`

```typescript
const pct = (BigInt(hop.amountIn) * 100n) / BigInt(route.totalAmountIn);
```

If `route.totalAmountIn` is `"0"`, BigInt division by zero throws a `RangeError` that crashes the component. Same pattern in 3 locations.

**Fix:** Guard with `BigInt(route.totalAmountIn) > 0n` before division.

---

### N3. Route selection comparison is fragile — multiple routes appear selected
**Severity:** HIGH
**File:** `frontend/apps/dex/app/components/RouteDialog.tsx:119-121`

```typescript
const isSelected =
  route.totalAmountOut === selectedRoute.totalAmountOut &&
  route.type === selectedRoute.type;
```

Two routes with identical output and type show as both selected. Should compare by unique route ID or index.

---

### N4. Zap page pool array mutated in-place
**Severity:** HIGH
**File:** `frontend/apps/dex/app/zap/page.tsx:82`

```typescript
const sortedPools = poolRes.data.sort((a, b) => b.tvlUsd - a.tvlUsd);
```

`.sort()` mutates the original API response array.

**Fix:** `[...poolRes.data].sort(...)`.

---

### N5. Portfolio USD calculation has precision loss
**Severity:** HIGH
**File:** `frontend/apps/dex/app/portfolio/page.tsx:72-73`

```typescript
const usd0 = (Number(value0) / 10 ** dec0) * price0;
```

`Number(value0)` loses precision above `2^53`. For 9-decimal tokens, amounts above ~9,007 tokens overflow JavaScript number precision.

**Fix:** Use `parseFloat(formatAmount(value0.toString(), dec0)) * price0`.

---

### N6. Workers `setInterval` handles still not cleared on shutdown
**Severity:** HIGH — **3rd consecutive audit flagging this**
**File:** `backend/apps/workers/src/main.ts:147-157,169-183`

Two `setInterval` calls are started but interval IDs are not saved. `shutdown()` closes BullMQ workers and DB but intervals keep firing, potentially throwing after connections are torn down. In K8s, pods hang until `terminationGracePeriodSeconds` and get SIGKILL'd.

**Fix:**
```typescript
const intervals: NodeJS.Timeout[] = [];
intervals.push(setInterval(() => { ... }, 15_000));
intervals.push(setInterval(() => { ... }, 5 * 60_000));
// In shutdown():
intervals.forEach(clearInterval);
```

---

### N7. 32 new build artifacts committed in tx-helpers
**Severity:** HIGH — **Regression from Phase 2 cleanup**
**Files:** `frontend/packages/tx-helpers/src/*.js`, `*.js.map`, `*.d.ts`, `*.d.ts.map` (32 files)

The Phase 2 commit `5bae058` specifically cleaned these artifacts from tx-helpers. Commit `349428a` re-added all of them. The `.gitignore` still doesn't exclude them.

**Fix:** Add to `.gitignore`:
```
frontend/packages/**/src/**/*.js
frontend/packages/**/src/**/*.js.map
frontend/packages/**/src/**/*.d.ts
frontend/packages/**/src/**/*.d.ts.map
```
Then `git rm -r --cached frontend/packages/tx-helpers/src/*.js frontend/packages/tx-helpers/src/*.d.ts frontend/packages/tx-helpers/src/*.js.map frontend/packages/tx-helpers/src/*.d.ts.map`.

---

## New Medium Issues (10)

### N8. Seed script has no transaction wrapping — partial seed state possible
**Severity:** MEDIUM
**File:** `scripts/seed-database.mts:175-251`

Each pair and token is inserted individually. If the script crashes mid-run, the database has partial data. The script is idempotent (`ON CONFLICT ... DO UPDATE`) so re-running is safe, but a partially seeded DB could confuse the indexer if it starts before the seed completes.

**Fix:** Wrap the entire seed in a SQL transaction (`BEGIN`/`COMMIT`/`ROLLBACK`).

---

### N9. Seed script `getCoinMetadata` via `as any` — silent wrong decimals
**Severity:** MEDIUM
**File:** `scripts/seed-database.mts:224,231`

`(client as any).getCoinMetadata(...)` casts to `any`. If this method doesn't exist on `SuiGrpcClient`, it throws, the silent `catch {}` swallows it, and every token gets fallback metadata with 9 decimals. USDC (6 decimals) and other non-9-decimal tokens would be seeded wrong, causing display errors across the entire DEX.

**Fix:** Verify `getCoinMetadata` exists on `SuiGrpcClient` in SDK 2.0. Add logging in the catch block.

---

### N10. SwapInfoBar still divides by zero when input is "0"
**Severity:** MEDIUM
**File:** `frontend/apps/dex/app/components/SwapInfoBar.tsx:38`

`Number(outputFormatted) / Number(inputFormatted)` produces `Infinity` when input is `"0"`.

**Fix:** Guard `if (Number(inputFormatted) === 0) return null`.

---

### N11. Fee calculation still incorrect for split routes
**Severity:** MEDIUM
**File:** `frontend/apps/dex/app/components/SwapInfoBar.tsx:46`

`hops.length * 0.3` treats all hops as sequential. For split routes (parallel legs), the fee is 0.3% per leg independently, not additive.

**Fix:** Use actual fee data from route response, or distinguish sequential vs parallel hops.

---

### N12. TokenSelector on-chain lookup has no AbortController
**Severity:** MEDIUM
**File:** `frontend/apps/dex/app/components/TokenSelector.tsx:159-191`

When user pastes an address then quickly changes the search, the previous RPC lookup continues and may set state for a stale query.

**Fix:** Add AbortController or cancellation flag in the effect cleanup.

---

### N13. Zap page fetch errors silently swallowed
**Severity:** MEDIUM
**File:** `frontend/apps/dex/app/zap/page.tsx:105`

`.catch(() => {})` means API failures show empty page with no indication.

**Fix:** Set error state and display retry UI.

---

### N14. Pools page still uses `.catch(() => {})`
**Severity:** MEDIUM
**Files:** `frontend/apps/dex/app/pools/page.tsx:51`, `pools/[id]/page.tsx:52`, `pools/[id]/add/page.tsx:40`, `pools/[id]/remove/page.tsx:42`, `portfolio/page.tsx:91`

All pool-related pages silently swallow fetch errors.

---

### N15. SwapInfoBar has local `formatAmt()` duplicating `format.ts`
**Severity:** MEDIUM
**File:** `frontend/apps/dex/app/components/SwapInfoBar.tsx:231-239`

Despite creating `format.ts`, a local duplicate remains.

**Fix:** Import from `../lib/format`.

---

### N16. TokenSelector has local `TokenIcon` duplicating `TokenAvatar`
**Severity:** MEDIUM
**File:** `frontend/apps/dex/app/components/TokenSelector.tsx:666-695`

`TokenIcon` is functionally identical to the shared `TokenAvatar` component.

**Fix:** Replace with `TokenAvatar`.

---

### N17. `VENUE_LABELS` / `VENUE_COLORS` duplicated across files
**Severity:** MEDIUM
**Files:** `RouteDialog.tsx:10-20`, `SwapInfoBar.tsx:19-29`

Identical maps defined in two files.

**Fix:** Export from shared module.

---

## New Low Issues (12)

### N18. Seed script hardcoded username in fallback DATABASE_URL
**File:** `scripts/seed-database.mts:33` — `cryptomichaael@localhost` in fallback. Should match backend config default.

### N19. Seed script `process.exit(1)` skips pool cleanup
**File:** `scripts/seed-database.mts:321` — Empty pairs check exits before `finally` block closes pool.

### N20. API/Admin-API HTTP server not closed on shutdown
**Files:** `backend/apps/api/src/main.ts:112-121`, `backend/apps/admin-api/src/main.ts:138-147` — `app.listen()` return value not stored, so `server.close()` can't be called on SIGTERM.

### N21. Admin API no bounds validation on `limit`/`offset`
**File:** `backend/apps/admin-api/src/main.ts:105-106` — Attacker on VPN can pass `limit=999999999`.

### N22. SSE parse errors silently swallowed
**File:** `frontend/packages/api-client/src/index.ts:222-233` — Empty `catch {}` on JSON parse.

### N23. `suidex-config` default API URL disagrees with api-client default
**File:** `frontend/packages/suidex-config/src/index.ts:282` — `https://api.suidex.org/api/v1` vs api-client's `http://localhost:4000/api/v1`.

### N24. LP balance still displayed with hardcoded 9 decimals on remove page
**File:** `frontend/apps/dex/app/pools/[id]/remove/page.tsx:168`

### N25. Missing `aria-label` on all icon-only buttons and dialogs
**Files:** RouteDialog, SwapWidget flip button, Header hamburger, SettingsPanel, TokenSelector modal — no `role="dialog"`, `aria-modal`, or `aria-label` attributes.

### N26. `format.ts` doesn't handle negative BigInt input
**File:** `frontend/apps/dex/app/lib/format.ts:10` — Negative bigint produces unexpected modulo results.

### N27. No request timeout on API client `fetchJson()`
**File:** `frontend/packages/api-client/src/index.ts:54` — Hanging backend leaves UI in loading state forever.

### N28. Env validation test has no negative-path coverage
**File:** `backend/packages/suidex-config/src/env.test.ts` — Only tests valid defaults and numeric overrides, not invalid values.

### N29. `DEX_PRODUCTION_READINESS_AUDIT.md` has stale API client assessment
**File:** `DEX_PRODUCTION_READINESS_AUDIT.md:119-121` — Claims API client lacks analytics methods that already exist.

---

## Infrastructure Changes Summary

### Firebase Hosting (New)
- `.firebaserc` + `firebase.json` added — multi-site hosting (dex, farm, landing)
- Static HTML files in `frontend/hosting/` — SPA fallback with redirects
- **Missing:** No security headers configured in `firebase.json` (no CSP, no X-Frame-Options)

### New CI/CD Workflows (New)
- `deploy-backend.yml` — SSH into GCP VM, pull Docker images, restart with compose
- `deploy-dex.yml` / `deploy-farm.yml` — Firebase deploy via GitHub Actions
- Uses `set -e` (good, addresses old `|| true` issue)
- SSH key passed as secret (correct pattern)
- **Concern:** SSH key written to `~/.ssh/id_rsa` in CI runner — deleted after deploy but visible to other steps in same job

### Docker-Compose Updates
- Split into app services + infra services
- Default passwords now stronger (`suidex_db_password_change_me` vs old `suidex`)
- Redis still has no auth
- Ports still exposed to all interfaces

### GCP Dev Setup
- `infra/gcp/DEV_SETUP.md` + `setup-dev.sh` — development environment setup guide
- Overly broad IAM roles documented (`roles/compute.admin`, `roles/container.admin`)
- **Concern:** Setup script runs `gcloud` commands without confirming project/account

---

## Summary

| Category | CRIT | HIGH | MED | LOW | Total |
|----------|------|------|-----|-----|-------|
| Phase 2 still open | 2 | 5 | 2 | 0 | **9** |
| New findings | 1 | 6 | 10 | 12 | **29** |
| **Total active** | **3** | **11** | **12** | **12** | **38** |

---

## Top 10 Priority Actions

1. **Fix swap race condition (F1/F2/N1)** — Fetch quote→build sequentially, wire abort signals, connect `onSelectRoute`
2. **Fix BigInt division by zero (N2)** — Guard all `BigInt(x) / BigInt(y)` with `y > 0n` check — will crash in production
3. **Clear `setInterval` on worker shutdown (N6/I4)** — 3rd audit, still not done
4. **Remove re-committed build artifacts (N7)** — 32 files, update `.gitignore`
5. **Add `useMemo` to `tokensWithBalance` (F3)** — Causes cascading re-renders across entire app
6. **Fix in-place `.sort()` mutations (F7/N4)** — Pools page and zap page mutate React state
7. **Fix portfolio precision loss (N5)** — `Number()` on large BigInt values
8. **Add error states to pools/zap/portfolio pages (N13/N14)** — Users see blank pages on API failure
9. **Fix seed script `getCoinMetadata` cast (N9)** — Silent fallback to wrong decimals
10. **Add security headers to Firebase hosting (firebase.json)** — No CSP, no X-Frame-Options

---

## Positive Observations

- **Verified token badges** now properly distinguish API-listed vs imported tokens
- **`TokenAvatar` component** provides consistent fallback across the app
- **`format.ts` shared library** eliminates most utility duplication
- **`RouteDialog`** is a solid route visualization component (just needs wiring)
- **Zap page** is feature-complete with proper decimal handling
- **Health endpoints** on all 4 backend services
- **Graceful shutdown handlers** on all services (intervals aside)
- **Non-root Docker execution** via `USER node`
- **Firebase deploy workflows** use `set -e` and handle secrets properly
- **Seed script** uses parameterized SQL (no injection risk) and is idempotent
- **Pool pages now use actual token decimals** — major improvement from Phase 2

---

*Report generated by CryptoMischief. Audit methodology: full source review of all changed files across 3 commits, covering security, infrastructure, frontend functionality, SDK compliance, and Phase 2 finding regression tracking.*
