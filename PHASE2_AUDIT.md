# Mischief Phase 2 Audit — SuiDex V2

**Auditor:** CryptoMischief
**Date:** April 15, 2026
**Scope:** 4 new commits on Dev branch (`04fd1d0`→`5bae058`) — 143 files changed, ~13,600 lines added. Covers Docker/K8s infrastructure, CI/CD pipeline, tx-helpers completion, TimescaleDB fixes, verification scripts, and the full DEX frontend build (Next.js 16).

---

## Overall Assessment

Massive progress. The project went from "backend scaffolding with no frontend" to a deployable DEX with containerized infrastructure, CI/CD, and a full swap/pool/portfolio UI in a single push. **Sui SDK 2.0 compliance is flawless** — zero violations across the entire codebase. The architecture is sound and the code is well-organized.

However, the infrastructure layer shipped with **production-blocking security gaps** (plaintext secrets in git, root containers, no connection limits on SSE) and the frontend has a **critical swap race condition** that could cause users to sign transactions that don't match the quoted price. These need to be fixed before any mainnet exposure.

**Phase 1 follow-up status:** 2 of the 5 deferred findings are now addressed — TimescaleDB hypertable PK (`1e911f6`) and `volume_24h_usd` (new volume-aggregate worker in `04fd1d0`). The remaining 3 deferred items (Cetus single-tick approximation, split price impact linear scaling, fee analytics timestamp filter) are unchanged.

---

## What's New in Phase 2

| Commit | What It Adds |
|--------|-------------|
| `04fd1d0` | Dockerfiles for all 5 services, `docker-compose.yml`, GitHub Actions CI/CD, K8s manifests (prod + staging), SSE streaming endpoint, route pre-compute service, volume aggregation worker, farm/locker enrichment upgrades |
| `ca58e94` | Complete PTB builders in tx-helpers — swap, cetus-swap, split-swap, liquidity, locker, farm, zap, verify, sign (all on-chain verified) |
| `1e911f6` | TimescaleDB hypertable PK fix, `subscribeCheckpoints` live verification script, TimescaleDB verification script |
| `5bae058` | Full DEX frontend — Next.js 16 App Router, SwapWidget, TokenSelector, pool pages (list/detail/add/remove), portfolio, providers, global styles. Cleaned build artifacts from frontend packages |

---

## Sui SDK 2.0 Compliance — FULL PASS

Every file reviewed. Zero violations.

- All imports use deep subpath patterns (`@mysten/sui/transactions`, `@mysten/sui/grpc`, `@mysten/sui/bcs`, `@mysten/sui/graphql`)
- `Transaction` class used everywhere — no deprecated `TransactionBlock`
- `SuiGrpcClient` for all reads — no `SuiClient` / JSON-RPC anywhere
- `SuiGraphQLClient` used appropriately for event/transaction queries
- Object references wrapped with `tx.object()`
- Coins properly paginated (`getAllCoins` pages 50 at a time), merged, split, remainders transferred
- BCS encoding via `@mysten/sui/bcs` module (`bcs.Address.serialize()`, `bcs.u64()`, etc.)
- Type arguments use `typeArguments` field name (not deprecated `typeArgs`)
- Pure values use typed helpers (`tx.pure.u256()`, `tx.pure.u64()`, `tx.pure.u128()`, `tx.pure.bool()`, `tx.pure.string()`, `tx.pure.vector()`)
- dapp-kit setup uses `createDAppKit()` + `DAppKitProvider` (SDK 2.0 pattern)

---

## Critical Issues (8)

### 1. Plaintext K8s Secret manifests committed to git
**Severity:** CRITICAL — Security
**Files:** `infra/k8s/namespace.yaml:25-29`, `infra/k8s/postgres/deployment.yaml:94-97`, `infra/k8s/staging/namespace.yaml:22-30`, `infra/k8s/staging/deployments.yaml:89-95`

Kubernetes Secret objects with `stringData` containing passwords (`CHANGE_ME_IN_PRODUCTION`, `CHANGE_ME_STAGING`) are in version control. The `DATABASE_URL` in `namespace.yaml:28` includes the password inline: `postgresql://suidex:CHANGE_ME@suidex-postgres:5432/suidex_v2`. Even as placeholders, this pattern trains operators to put real credentials in manifests, and they will inevitably end up in git history.

**Fix:** Remove all Secret manifests from the repo. Use Sealed Secrets, External Secrets Operator, or SOPS. Create secrets at deploy time via `kubectl create secret` from CI environment variables. Add `*-secret*.yaml` to `.gitignore`.

---

### 2. Swap route quote/build race condition
**Severity:** CRITICAL — Correctness
**File:** `frontend/apps/dex/app/hooks/useSwapRoute.ts:80-83`

`fetchRouteQuote` and `fetchRouteBuild` are called in `Promise.all`. If the routing engine's state changes between these two calls (pool reserves update, new checkpoint), the quote shown to the user (price, impact, minimum received) could describe a different route than the one built into the transaction. The user sees one price but signs a different transaction.

**Fix:** Fetch sequentially — quote first, then build. If the API supports it, pass the quote's route ID to the build endpoint to guarantee atomicity. At minimum, compare the build's output amount against the quoted amount and reject if they diverge beyond slippage tolerance.

---

### 3. AbortController signal never passed to fetch calls
**Severity:** CRITICAL — Correctness
**File:** `frontend/apps/dex/app/hooks/useSwapRoute.ts:73-83`

An `AbortController` is created and its signal is checked after the fetch, but the signal is never passed to `fetchRouteQuote` or `fetchRouteBuild`. The HTTP requests are never actually cancelled — only the state update is skipped. Under rapid input changes, zombie requests pile up and can return stale data after the component has moved on.

**Fix:** Pass `{ signal: controller.signal }` to both fetch functions. The `@suidex/api-client` functions need to accept an options/signal parameter.

---

### 4. Docker builds ship devDependencies — images ~1-2GB
**Severity:** CRITICAL — Infrastructure
**Files:** `backend/apps/api/Dockerfile:45`, `backend/apps/workers/Dockerfile:40`, `backend/apps/indexer/Dockerfile:40`, `backend/apps/admin-api/Dockerfile:40`

The "production" stage copies the entire `/app/` from the builder stage, including all `node_modules` (dev+prod) and all source code. There is no `pnpm prune --prod`, no pruning of dev dependencies, and no actual TypeScript build step. Images are bloated and contain dev tooling that increases the attack surface.

**Fix:** Add `RUN pnpm --filter @suidex/<app>... deploy --prod /app/deploy` in the builder stage, then `COPY --from=builder /app/deploy /app` in the production stage. This gives you only production dependencies and the app code.

---

### 5. 61+ compiled build artifacts committed to git
**Severity:** CRITICAL — Repository hygiene
**Files:** `backend/packages/database/src/` (41+ files), `scripts/` (20+ files)

`.gitignore` does not exclude `*.js`, `*.d.ts`, `*.js.map`, `*.d.mts`, `*.mjs`, or `*.mjs.map` in source directories. As a result, compiled output is tracked alongside TypeScript source. This causes merge conflicts, bloats the repo, and creates confusion about source of truth when compiled output diverges from source.

Note: The Phase 2 frontend commit (`5bae058`) correctly cleaned build artifacts from `frontend/packages/` — the same treatment needs to be applied to `backend/packages/` and `scripts/`.

**Fix:** Add to `.gitignore`:
```
# Build artifacts in src dirs
backend/packages/**/src/**/*.js
backend/packages/**/src/**/*.js.map
backend/packages/**/src/**/*.d.ts
backend/packages/**/src/**/*.d.ts.map
scripts/**/*.mjs
scripts/**/*.mjs.map
scripts/**/*.d.mts
scripts/**/*.d.mts.map
!next-env.d.ts
```
Then `git rm -r --cached` the affected files.

---

### 6. CI/CD deploy failures silently suppressed
**Severity:** CRITICAL — Infrastructure
**File:** `.github/workflows/ci-cd.yml:217-219,224-227,254-258,262-265`

Both staging and production deploy steps use `|| true` on `kubectl set image` and `kubectl rollout status`. A failed deployment (wrong image tag, CrashLoopBackoff, quota exceeded) reports the pipeline as successful. Production could be down with no CI signal whatsoever.

**Fix:** Remove `|| true` from all `kubectl set image` commands. For `rollout status`, fail the step and add a notification mechanism (Slack webhook, GitHub deployment status). If partial failures are expected during early development, handle that with conditional logic rather than blanket suppression.

---

### 7. SSE endpoint has no connection limit — DoS vector
**Severity:** CRITICAL — Security
**File:** `backend/apps/api/src/routes/stream.ts:28,126-162`

The `clients` Map grows without bound. Each SSE connection holds a `Response` object in memory indefinitely. There is no max-connections cap, no per-IP limiting (the global rate limiter only limits the initial HTTP handshake, not persistent connections), and no max-age disconnect. An attacker can open thousands of connections and cause OOM.

**Fix:** Add `MAX_SSE_CLIENTS` cap (e.g., 500), reject new connections with 503 when exceeded. Add per-IP connection limiting (max 5). Add `maxAge` to auto-disconnect clients after 1 hour. Add authentication if the data is not intended to be fully public.

---

### 8. All Docker containers run as root
**Severity:** CRITICAL — Security
**Files:** All 5 Dockerfiles (`backend/apps/api/Dockerfile`, `workers/Dockerfile`, `indexer/Dockerfile`, `admin-api/Dockerfile`, `frontend/Dockerfile`)

No Dockerfile contains a `USER` directive. All containers run as root. Combined with no `securityContext` in any K8s deployment, a container escape or RCE gives root on the host.

**Fix:** Add to each Dockerfile:
```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```
Add to all K8s deployments:
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

---

## High Issues (18)

### 9. `ADMIN_ALLOWED_IPS` empty in all environments
**Severity:** HIGH — Security
**Files:** `docker-compose.yml:115`, `infra/k8s/admin-api/deployment.yaml:32`, `infra/k8s/staging/deployments.yaml:266`

The admin API is deployed with `ADMIN_ALLOWED_IPS: ""` everywhere. If the middleware treats empty as "allow all," the admin API is open. The K8s Ingress doesn't route to admin-api (good), but the Service is accessible cluster-internally and via port-forwarding.

**Fix:** Set actual VPN/bastion IPs. Add a NetworkPolicy restricting admin-api ingress to specific source pods. Never default to empty.

---

### 10. No K8s security context on any deployment
**Severity:** HIGH — Security
**Files:** All manifests in `infra/k8s/`

No deployment specifies `securityContext`. Containers can escalate privileges, run as root, and write to the filesystem freely.

**Fix:** Add pod-level and container-level security contexts to every deployment.

---

### 11. Docker-compose exposes DB/Redis on all interfaces with default creds
**Severity:** HIGH — Security
**File:** `docker-compose.yml:17-21,33-38`

PostgreSQL (`suidex`/`suidex`) on port 5432 and Redis (no password) on port 6379 are mapped to all host interfaces. On a cloud VM with a public IP, both are internet-accessible.

**Fix:** Bind to localhost: `"127.0.0.1:5432:5432"`. Add `--requirepass` to Redis. Or remove port mappings entirely and only communicate via Docker network.

---

### 12. Redis deployed without authentication everywhere
**Severity:** HIGH — Security
**Files:** `docker-compose.yml:33-45`, K8s redis manifests

Redis URL is `redis://redis:6379` with no auth in any environment. Any pod or container can read/write/flush.

**Fix:** Add `--requirepass` to Redis command and update all `REDIS_URL` values to include the password.

---

### 13. Docker base images not pinned to digest
**Severity:** HIGH — Security
**Files:** All Dockerfiles

Base images use tags (`node:22-alpine`, `nginx:1.27-alpine`, `redis:7-alpine`) rather than SHA256 digests. A compromised or updated tag can silently introduce malicious code.

**Fix:** Pin to digest: `FROM node:22-alpine@sha256:<hash>`. Use Renovate or Dependabot to track updates.

---

### 14. `verifySwapPTB` does not inspect transaction commands
**Severity:** HIGH — Security
**File:** `frontend/packages/tx-helpers/src/verify.ts:58-86`

`verifySwapPTB` validates sender, tokens, and amounts, but never inspects the actual PTB commands. The `ALLOWED_PACKAGES` and `ALLOWED_SHARED_OBJECTS` sets are defined but **never checked** against moveCall targets. A compromised API could return route metadata pointing to a malicious package, and the verify layer would not catch it.

**Fix:** Iterate `tx.getData().commands`, verify each moveCall target package is in `ALLOWED_PACKAGES`, and verify shared object arguments are in `ALLOWED_SHARED_OBJECTS`. The infrastructure is there — it just needs to be wired up.

---

### 15. No security scanning in CI/CD
**Severity:** HIGH — Infrastructure
**File:** `.github/workflows/ci-cd.yml`

Pipeline has typecheck, lint, and tests, but no container image scanning (Trivy/Grype), no dependency audit (`pnpm audit`), no SAST, no secret scanning.

**Fix:** Add `pnpm audit --audit-level=high` after install. Add Trivy scanning on built images before push. Add `actions/dependency-review-action` on PRs.

---

### 16. Workers don't clear setInterval on shutdown
**Severity:** HIGH — Infrastructure
**File:** `backend/apps/workers/src/main.ts:129-140,153-158`

The `shutdown()` function closes BullMQ workers and DB, but doesn't clear the two `setInterval` handles (route pre-computation at line 129, volume aggregation at line 136). Active timers keep the event loop alive. In K8s, pods hang until `terminationGracePeriodSeconds` expires and get SIGKILL'd.

**Fix:** Store interval IDs and call `clearInterval()` in shutdown before closing workers.

---

### 17. Workers and indexer K8s deployments have no health probes
**Severity:** HIGH — Infrastructure
**Files:** `infra/k8s/workers/deployment.yaml`, `infra/k8s/indexer/deployment.yaml`

No `readinessProbe` or `livenessProbe`. If the process deadlocks or the event loop starves, Kubernetes won't restart it. Especially dangerous for the indexer — it's a singleton and must be restarted promptly to avoid checkpoint gaps.

**Fix:** Add an HTTP health endpoint and configure both probes. Indexer health should report current checkpoint lag.

---

### 18. No PodDisruptionBudget for any deployment
**Severity:** HIGH — Infrastructure
**Files:** `infra/k8s/` (all deployments)

No PDB defined. During node drains or cluster upgrades, all replicas of API (2 replicas) or frontend (2 replicas) can be evicted simultaneously.

**Fix:** Add PDBs: `minAvailable: 1` for api and frontend.

---

### 19. Volume aggregation N+1 query pattern
**Severity:** HIGH — Efficiency
**File:** `backend/apps/workers/src/volume-aggregate.ts:126-134`

Runs a separate `UPDATE` per pair inside a `for...of` loop. With 50+ pairs, that's 50+ individual round-trips every 5 minutes.

**Fix:** Use a single batch update: `UPDATE ... FROM (VALUES ...)` or Kysely's `$call()` for raw SQL batch.

---

### 20. Cetus state cache unbounded growth
**Severity:** HIGH — Efficiency
**File:** `backend/apps/api/src/services/cetus.service.ts:46`

`stateCache` is a `Map` with 5-second TTL entries but no eviction. Old entries are only replaced on re-read, never proactively cleaned. Over time this map grows without bound.

**Fix:** Use an LRU cache with `maxSize` and TTL (e.g., `lru-cache` package), or add periodic sweep.

---

### 21. `tokensWithBalance` recomputed every render
**Severity:** HIGH — Frontend performance
**File:** `frontend/apps/dex/app/hooks/useTokens.ts:68-74`

Plain `.map()` in the render path, not wrapped in `useMemo`. Every consumer gets a new array reference on every render, causing cascading re-renders.

**Fix:** `useMemo(() => tokens.map(...), [tokens, balances])`.

---

### 22. Portfolio fires N parallel RPC calls with no concurrency limit
**Severity:** HIGH — Frontend correctness
**File:** `frontend/apps/dex/app/portfolio/page.tsx:54-81`

With 200 pools, this fires 200 simultaneous `getTokenBalance` calls. Will hit RPC rate limits and cause mass failures.

**Fix:** Use `p-limit` with concurrency of 5-10, or batch using multi-get coins API.

---

### 23. Balance refresh fires 1 call per token every 15s
**Severity:** HIGH — Frontend correctness
**File:** `frontend/apps/dex/app/hooks/useTokens.ts:47-51`

`refreshBalances` fires one `getTokenBalance` per token via `Promise.allSettled`, all simultaneously. Same rate-limit concern, and it runs every 15 seconds.

**Fix:** Use `multiGetBalance` or batch with concurrency limiting.

---

### 24. All fetch errors silently swallowed
**Severity:** HIGH — Frontend UX
**Files:** `frontend/apps/dex/app/pools/page.tsx:44`, pool detail, add liquidity, remove liquidity, useTokens

`.catch(() => {})` throughout. Failed API calls show empty UI with no error indication and no retry option.

**Fix:** Add error state, display error message with retry button. At minimum: `.catch((e) => setError(e.message))`.

---

### 25. Pool state array mutated in-place
**Severity:** HIGH — Frontend correctness
**File:** `frontend/apps/dex/app/pools/page.tsx:60`

`list.sort(...)` mutates the array. When search is empty, `list` is the same reference as `pools` (the state variable). Mutating React state directly causes subtle rendering bugs.

**Fix:** Always clone before sorting: `[...list].sort(...)`.

---

### 26. Hardcoded 9 decimals for pool reserve/LP display
**Severity:** HIGH — Frontend correctness
**Files:** `frontend/apps/dex/app/pools/[id]/page.tsx:236-238`, `pools/[id]/add/page.tsx:228-229`, `pools/[id]/remove/page.tsx:159`

`formatTokenAmount` always uses 9 decimals. USDC has 6 decimals. Displays are off by 1000x for non-9-decimal tokens. The add-liquidity price display divides raw reserves without adjusting for different decimal counts — also wrong for mixed-decimal pairs.

**Fix:** Fetch token metadata and use actual decimals. Or add decimals to the `PoolData` type so it's always available.

---

## Medium Issues (28)

### 27. Default `DATABASE_URL` in Zod config
**File:** `backend/packages/suidex-config/src/env.ts:25`

Zod default value `"postgresql://suidex:suidex@localhost:5432/suidex_v2"` means the app silently connects with `suidex/suidex` credentials if the env var is unset in production.

**Fix:** Remove the default. Use `z.string().min(1)` so it fails loudly.

---

### 28. No `Content-Security-Policy` header
**File:** `infra/k8s/frontend/nginx.conf:41-46`

Other security headers are set (X-Frame-Options, X-Content-Type-Options) but no CSP. Without it, injected scripts and third-party resources have no restrictions.

**Fix:** Add restrictive CSP: `default-src 'self'; script-src 'self'; connect-src 'self' https://fullnode.mainnet.sui.io; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;`

---

### 29. No NetworkPolicy — all pods communicate freely
**Files:** `infra/k8s/` (missing)

Zero NetworkPolicy resources. Frontend nginx can reach postgres directly. Any compromised pod has full cluster network access.

**Fix:** Create NetworkPolicies: postgres allows ingress only from api, admin-api, indexer, workers. Redis same. Admin-api allows ingress only from VPN/bastion.

---

### 30. SSE creates new DB/Repository instances every 15s
**File:** `backend/apps/api/src/routes/stream.ts:74-76`

`broadcastPriceUpdate()` instantiates `getDb()`, `new TokensRepository(db)`, and `new PairsRepository(db)` on every tick.

**Fix:** Create repositories once at module level or in `startBroadcastLoop()`.

---

### 31. Route pre-compute worker is a no-op stub
**File:** `backend/apps/workers/src/route-precompute.ts:39-51,84-94`

`processRoutePrecompute` logs and returns without doing anything. `enqueueTopPairPrecomputation` checks cache keys but the inner block is empty. Runs every 15 seconds consuming Redis round-trips for nothing.

**Fix:** Implement or remove.

---

### 32. Duplicate route precompute logic in two files
**Files:** `backend/apps/api/src/services/route-precompute.ts`, `backend/apps/workers/src/route-precompute.ts`

Both define `ROUTE_CACHE_PREFIX`, `buildCacheKey`, `getCachedRoute`, and precompute amount constants. API version uses `bigint`, worker uses strings. Will inevitably diverge.

**Fix:** Extract into a shared package. Have both files import from there.

---

### 33. `swap_events` not a TimescaleDB hypertable
**File:** `backend/packages/database/src/migrations/001_initial_schema.ts:91-126`

`swap_events` is queried by time range in volume aggregation but is a regular table. As swap volume grows, time-range queries degrade. `reserve_snapshots` and `token_price_history` are hypertables already.

**Fix:** Convert `swap_events` to a hypertable partitioned by `timestamp`.

---

### 34. `Number(rawVolume)` precision loss
**File:** `backend/apps/workers/src/volume-aggregate.ts:120`

Converts BigInt to Number. For 18-decimal tokens, values above `Number.MAX_SAFE_INTEGER` (~9×10^15) lose precision. A 10M BLUB swap at 18 decimals = 10^25, losing ~10 digits.

**Fix:** Use `decimal.js` or keep computation in BigInt until final division.

---

### 35. No rolling update strategy defined
**Files:** `infra/k8s/api/deployment.yaml`, `infra/k8s/admin-api/deployment.yaml`

Default rolling update with 2 replicas means 1 pod down during deploys. 1-replica admin-api has downtime.

**Fix:** Set `maxUnavailable: 0`, `maxSurge: 1` for API. Bump admin-api to 2 replicas or use `Recreate` strategy.

---

### 36. BFS path discovery allocates heavily
**File:** `backend/apps/api/src/services/routing-engine.ts:250-320`

Creates new `Set` copies and new arrays on every queue expansion. `MAX_QUEUE_SIZE = 1000` caps it but doesn't prevent burst allocations.

**Fix:** Consider stack-based DFS with backtracking, or limit BFS to 4-5 hops (explicit 2-hop and 3-hop logic already covers those cases).

---

### 37. No test coverage gates in CI
**File:** `.github/workflows/ci-cd.yml:88`

`pnpm test` runs but no coverage threshold enforcement.

**Fix:** Add `--coverage` and a check step that fails below threshold (e.g., 70%).

---

### 38. Docker `.dockerignore` excludes `*.d.ts` needed by build
**File:** `.dockerignore:8`

Excludes `*.d.ts` but committed `.d.ts` files in `backend/packages/database/src/` may be needed for type resolution in shared packages. `tsx` typically strips types so this may be benign, but it's a fragile assumption.

**Fix:** Either ensure `tsx` doesn't need `.d.ts` files, or remove `*.d.ts` from `.dockerignore`.

---

### 39. Stale closure in SwapWidget balance sync
**File:** `frontend/apps/dex/app/components/SwapWidget.tsx:49`

`eslint-disable-line react-hooks/exhaustive-deps` suppresses a warning for missing `tokenIn`/`tokenOut` in the dependency array. If `tokens` changes but token references haven't been captured, `.find()` uses stale refs.

**Fix:** Include `tokenIn?.coinType` and `tokenOut?.coinType` in deps (not full objects, to avoid infinite loops).

---

### 40. Every token shows verified badge unconditionally
**File:** `frontend/apps/dex/app/components/TokenSelector.tsx:179`

A `<CheckCircle>` badge appears on every token, including unverified or potential scam tokens.

**Fix:** Only show badge for tokens with `verified: true` in their data.

---

### 41. Token image error — no fallback avatar
**File:** `frontend/apps/dex/app/components/TokenSelector.tsx:240-242`

On image load error, the `<img>` is set to `display: none`. No fallback avatar appears.

**Fix:** Track error state and render color-hash fallback avatar.

---

### 42. SwapInfoBar division by zero
**File:** `frontend/apps/dex/app/components/SwapInfoBar.tsx:36`

`Number(inputFormatted)` can be 0, causing `rate` to be `Infinity`.

**Fix:** Guard: `const rate = Number(inputFormatted) > 0 ? ... : 0;`

---

### 43. SwapInfoBar fee calculation incorrect for multi-hop
**File:** `frontend/apps/dex/app/components/SwapInfoBar.tsx:44`

`hops.length * 0.3` assumes 0.3% per hop, but fees compound and different venues have different tiers.

**Fix:** Use actual fee data from route response, or label as "~estimated".

---

### 44. Remove liquidity LP decimals hardcoded to 9
**File:** `frontend/apps/dex/app/pools/[id]/remove/page.tsx:159`

`formatAmount(lpBalance.toString(), 9)` assumes LP tokens always have 9 decimals.

**Fix:** Fetch LP token metadata or store LP decimals in PoolData.

---

### 45. `tsconfig.json` alias points to wrong directory
**File:** `frontend/apps/dex/tsconfig.json:21-23`

`@/*` maps to `./src/*` but the app uses `./app/` (App Router). Any `@/` import would fail.

**Fix:** Change to `"@/*": ["./app/*"]`.

---

### 46. Wallet dropdown doesn't close on scroll or route change
**File:** `frontend/apps/dex/app/components/Header.tsx:89-122`

Full-screen invisible div handles click-outside, but dropdown stays open on scroll/route changes.

**Fix:** Add scroll listener to close. Close on route change via `usePathname` effect.

---

### 47. ConnectButton silent failure with no wallets
**File:** `frontend/apps/dex/app/components/ConnectButton.tsx:16-19`

If no wallet extension is installed, `preferred` is undefined and `handleConnect` does nothing. No feedback to user.

**Fix:** Show "No wallet detected" message or link to wallet installation.

---

### 48. Duplicated utility functions across 6+ files
**Files:** SwapWidget.tsx, SwapInfoBar.tsx, pools/page.tsx, pools/[id]/page.tsx, add/page.tsx, remove/page.tsx, portfolio/page.tsx

`formatAmount`, `parseAmountRaw`, `shortType`, `formatCompact`, `TokenDot`/`TokenAvatar` copy-pasted everywhere.

**Fix:** Extract into shared `@suidex/ui` or local `utils.ts`.

---

### 49. TimescaleDB image uses `latest` tag
**Files:** `docker-compose.yml:13`, `infra/k8s/postgres/deployment.yaml:22`, `infra/k8s/staging/deployments.yaml:28`

`timescale/timescaledb:latest-pg16` is a floating tag. Breaking changes can land without warning.

**Fix:** Pin to specific version, e.g., `timescale/timescaledb:2.17.2-pg16`.

---

### 50. SortHeader ignores sort direction visually
**File:** `frontend/apps/dex/app/pools/page.tsx:235`

`dir` param destructured as `_dir` and never used. Arrow icon never changes to indicate asc vs desc.

**Fix:** Use `dir` prop to show directional arrow.

---

### 51. `useSwapRoute`: `rawAmountIn` should be `useMemo` not `useCallback`
**File:** `frontend/apps/dex/app/hooks/useSwapRoute.ts:53-63`

Wrapped in `useCallback` but called as `rawAmountIn()` — it's a value computation, not a callback.

**Fix:** Convert to `useMemo`.

---

### 52. Add liquidity pool price uses raw reserves without decimal adjustment
**File:** `frontend/apps/dex/app/pools/[id]/add/page.tsx:228-229`

`Number(pool.reserve1) / Number(pool.reserve0)` doesn't adjust for different decimals. If token0 has 9 decimals and token1 has 6, price is off by 1000x.

**Fix:** `(Number(pool.reserve1) / 10**dec1) / (Number(pool.reserve0) / 10**dec0)`.

---

### 53. Unused first query in volume aggregation
**File:** `backend/apps/workers/src/volume-aggregate.ts:30-48`

First `volumeRows` query fetches `total_raw_volume` and `swap_count` per pair but results are never used. The actual USD computation uses the second `volumeByDirection` query.

**Fix:** Remove unused query or use for logging/metrics.

---

### 54. Locker enrichment computes but never persists claimable SUI
**File:** `backend/apps/workers/src/locker-enrich.ts:156-213`

Calculates `netClaimable` SUI but only logs it — no DB update to persist.

**Fix:** Add DB update or document why it's intentionally read-only.

---

## Low Issues (20)

### 55. SSE catch block swallows errors silently
**File:** `backend/apps/api/src/routes/stream.ts:105-107`

Catches all errors with no logging. Database connection drops go unnoticed.

**Fix:** Add `logger.warn({ err }, "SSE broadcast failed")`.

---

### 56. Farm enrichment silent fallback to hardcoded emission rate
**File:** `backend/apps/workers/src/farm-enrich.ts:66-69`

Tries three field names, then falls back to `6600000000` with no alert.

**Fix:** Log warning on field name fallback, log error when hardcoded fallback activates.

---

### 57. Frontend nginx serves all domains from one server block
**File:** `infra/k8s/frontend/nginx.conf:20-23`

`server_name` lists 4 domains but uses single root with generic `try_files`. Wrong `index.html` served for domain-specific apps.

**Fix:** Use separate `server` blocks per domain or `map` to set root by `$host`.

---

### 58. Missing `aria-label` on icon-only buttons
**Files:** SwapWidget.tsx (flip), Header.tsx (hamburger), SettingsPanel.tsx (close), SwapInfoBar.tsx (expand)

Screen readers cannot identify these buttons.

**Fix:** Add descriptive `aria-label` to each.

---

### 59. TokenSelector modal lacks focus trap and dialog role
**File:** `frontend/apps/dex/app/components/TokenSelector.tsx:77-217`

Tab key can move focus behind the modal. Missing `role="dialog"` and `aria-modal="true"`.

**Fix:** Implement focus trapping and add semantic dialog attributes.

---

### 60. `txDigest` never clears after successful swap
**File:** `frontend/apps/dex/app/components/SwapWidget.tsx:281`

Success banner with tx link persists indefinitely.

**Fix:** Clear after timeout or on new swap initiation.

---

### 61. No loading skeleton for pools page
**File:** `frontend/apps/dex/app/pools/page.tsx:164-168`

Single spinner instead of skeleton rows.

---

### 62. Missing `<head>` content in global-error.tsx
**File:** `frontend/apps/dex/app/global-error.tsx:11`

No `lang` attribute on `<html>`, no viewport meta tag. Error page may not render correctly on mobile.

---

### 63. `lightweight-charts` in package.json but unused
**File:** `frontend/apps/dex/package.json:28`

Listed as dependency but never imported. Unnecessary bundle size.

---

### 64. Settings panel custom input UX confusion
**File:** `frontend/apps/dex/app/components/SettingsPanel.tsx:99`

When a preset is active, custom input shows empty string. Confusing two-state logic.

---

### 65. Remove liquidity: no balance refresh after success
**File:** `frontend/apps/dex/app/pools/[id]/remove/page.tsx:105`

LP balance not refreshed after removal. Old balance shows until page reload.

---

### 66. `force-dynamic` on root layout disables static optimization
**File:** `frontend/apps/dex/app/layout.tsx:38`

Intentional per comment ("this is a dApp"), but pools list could benefit from ISR.

---

### 67. Framer Motion adds ~30KB gzip for simple animations
Used only for fade/slide in TokenSelector, SettingsPanel, SwapInfoBar. CSS animations could handle most.

---

### 68. `@tanstack/react-query` listed as direct dependency but may be transitive
**File:** `frontend/apps/dex/package.json:25`

Likely pulled in by `@mysten/dapp-kit-react`. Verify if needed as a direct dep.

---

### 69. No per-page error boundaries
Only `global-error.tsx` exists. A failed pools fetch takes down the entire page.

**Fix:** Add `error.tsx` per route segment.

---

### 70. Cetus single-tick approximation still in place (Phase 1 deferred)
**File:** `backend/apps/api/src/services/cetus.service.ts`

Still uses single-tick price approximation. Optimistic for large swaps. Deferred from Phase 1 — noting for tracking.

---

### 71. Split price impact still uses linear scaling (Phase 1 deferred)
Still not using per-leg recalculation. Deferred from Phase 1 — noting for tracking.

---

### 72. Fee analytics still filters by `created_at` not `timestamp` (Phase 1 deferred)
Deferred from Phase 1 — noting for tracking.

---

## Summary

| Domain | CRIT | HIGH | MED | LOW | Total |
|--------|------|------|-----|-----|-------|
| Security | 3 | 5 | 3 | 1 | **12** |
| Infrastructure/Backend | 3 | 7 | 10 | 4 | **24** |
| Frontend | 2 | 6 | 13 | 12 | **33** |
| SDK 2.0 | 0 | 0 | 0 | 0 | **0** |
| Phase 1 Deferred | 0 | 0 | 0 | 3 | **3** |
| **Total** | **8** | **18** | **26** | **20** | **72** |

---

## Top 10 Priority Actions

1. **Remove K8s Secret manifests from repo** — use External Secrets Operator or Sealed Secrets
2. **Fix swap race condition** — fetch quote then build sequentially; wire abort signals
3. **Fix Docker builds** — `pnpm prune --prod`, add `USER` directive, pin images to digest
4. **Add SSE connection limits + per-IP cap** — prevent DoS
5. **Remove `|| true` from CI deploys** — failures must fail the pipeline
6. **Clean build artifacts + update `.gitignore`** — `git rm --cached` the 61+ committed files
7. **Fix hardcoded decimals** — pool detail, add/remove liquidity, LP display all assume 9 decimals
8. **Implement PTB command verification** — `ALLOWED_PACKAGES` is defined but never checked
9. **Add health probes + PDB + security contexts** to all K8s deployments
10. **Batch RPC calls** — concurrency limiter for portfolio/balance, batch volume UPDATEs

---

## Positive Observations

- **Sui SDK 2.0 compliance is flawless** — best seen in any Sui project audit
- No hardcoded private keys, mnemonics, or API secrets anywhere in code
- No XSS vectors (no `dangerouslySetInnerHTML`)
- Zod validation on all API routes
- CORS properly configured per-origin (not wildcard)
- Error handler never leaks stack traces
- Rate limiting in place on API
- tx-helpers validate addresses and amounts before PTB construction
- Build artifacts cleaned from frontend packages (Phase 2 commit)
- Clean monorepo separation of concerns
- Good use of BullMQ for async work distribution
- Phase 1 audit findings largely addressed — 11/13 fixed, 2 more addressed in Phase 2

---

*Report generated by CryptoMischief. Audit methodology: full source review of all changed files across 4 commits, covering security, Sui SDK 2.0 compliance, infrastructure best practices, frontend code quality, and general correctness.*
