# SuiDex V2 — Phase 3 Security Audit

**Scope:** Commits a55b736, 349428a, da28bb2 (since 5bae058 on Dev branch)
**Date:** 2026-04-15
**Auditor:** Claude Opus 4.6 (Mischief Audits)

---

## Part A: Phase 2 Finding Resolution Status

### S1: Plaintext K8s secrets in git — NOT FIXED

K8s Secret manifests with plaintext `stringData` are still committed:

- `/tmp/suidex-v2/infra/k8s/namespace.yaml:27-29` — `suidex-secrets` with `DATABASE_URL: postgresql://suidex:CHANGE_ME@...` and `REDIS_URL`
- `/tmp/suidex-v2/infra/k8s/postgres/deployment.yaml:88-97` — `suidex-db-secret` with `POSTGRES_USER`, `POSTGRES_PASSWORD: CHANGE_ME_IN_PRODUCTION`, and full `DATABASE_URL`

These files were not touched by any of the 3 commits. The values are placeholder passwords but the pattern remains: K8s Secrets should be managed via Sealed Secrets, SOPS, or an external secret store — never committed to git.

### S4/S5: Docker containers run as root / no securityContext — PARTIALLY FIXED

**Dockerfiles (FIXED):** All 4 backend Dockerfiles now include `USER node` in the production stage, running the Node.js process as a non-root user. This is a significant improvement.

**K8s manifests (NOT FIXED):** No `securityContext` (runAsNonRoot, readOnlyRootFilesystem, allowPrivilegeEscalation: false) was added to any K8s deployment YAML. The K8s files were not modified in these commits.

**Frontend Dockerfile:** The nginx-based `frontend/Dockerfile` still runs as root (nginx default). This was not changed.

### S8: Docker base images not pinned — PARTIALLY FIXED

Backend Dockerfiles now pin `node:22-alpine` (minor-version pinned) and `pnpm@9.15.0` (exact version). This is improved but `node:22-alpine` still floats on patch versions. The frontend pins `nginx:1.27-alpine`.

K8s manifests still use `timescale/timescaledb:latest-pg16` and `redis:7-alpine` — the `:latest` tag is unpinned. `docker-compose.yml` also uses `timescale/timescaledb:latest-pg16`.

### S12: CI/CD `|| true` suppresses failures — PARTIALLY FIXED

The **new** deploy workflows (`deploy-backend.yml`) use `set -e` inside the SSH script, which is correct. However:

- `deploy-dex.yml:60-61` and `deploy-farm.yml:60-61` use `|| true` to suppress `docker stop/rm` errors. These are acceptable (container may not exist).
- `ci-cd.yml:219,226,257,264` still uses `|| true` on `kubectl set image` and `kubectl rollout status`. This was a Phase 2 finding and **remains unfixed** — a failed rollout is silently ignored.

### I1: Docker multi-stage builds ship devDeps — NOT FIXED

All 4 backend Dockerfiles copy the entire `/app/` from the builder stage to production:
```
COPY --from=builder --chown=node:node /app/ ./
```
This includes ALL `node_modules` (dev + prod), all source files, and the full workspace. A production image should run `pnpm install --prod` or `pnpm deploy` in the final stage, or use `pnpm prune --prod`.

### I2: Build artifacts committed (.js/.d.ts in src/) — WORSE

This finding is **worse than before**. These commits added 32 new `.js`/`.d.ts`/`.js.map`/`.d.ts.map` files to `frontend/packages/tx-helpers/src/` (cetus-swap, farm, liquidity, locker, split-swap, swap, verify, zap). The backend `src/` directories continue to have ~125+ committed build artifacts. The `scripts/` directory also has committed `.mjs`, `.d.mts`, and `.mjs.map` files.

Total committed build artifacts: ~155+ files across backend, frontend, and scripts.

### I4: Workers don't clear setInterval on shutdown — NOT FIXED

`/tmp/suidex-v2/backend/apps/workers/src/main.ts:147-157` creates two `setInterval` timers (route precompute at 15s, volume aggregate at 5min) but the `shutdown()` function at line 169 does not clear them. The intervals are not stored in variables that could be cleared.

### I11: Route pre-compute worker was no-op stub — PARTIALLY FIXED

The **API-side** precompute service (`backend/apps/api/src/services/route-precompute.js`) is now fully implemented — it calls `computeRoute()` and caches results in Redis with 15s TTL.

However, the **worker-side** stub (`backend/apps/workers/src/route-precompute.js:31-39`) `processRoutePrecompute()` is still effectively a no-op — it logs a message and returns without doing any work. The `enqueueTopPairPrecomputation()` function queries pairs and iterates but the inner loop body is empty (lines 71-74 are comments). This creates the illusion of work without actually precomputing routes.

---

## Part B: New Findings

### CRITICAL Findings

**(None)**

### HIGH Findings

#### H1: SSH private key written to disk in CI without cleanup
**File:** `.github/workflows/deploy-backend.yml:29-32`, `deploy-dex.yml:29-32`, `deploy-farm.yml:29-32`
**Description:** All 3 new deploy workflows write `${{ secrets.SSH_PRIVATE_KEY }}` to `~/.ssh/id_rsa` but never clean it up. While GitHub Actions runners are ephemeral, this pattern risks key exposure if: (a) the job fails mid-execution and artifacts are collected, (b) a subsequent step in a future workflow change logs or uploads workspace contents. Best practice is to use `ssh-agent` with key added to the agent (never touches disk), or at minimum add a cleanup step in a `post` or `always` block.
**Fix:** Use `webfactory/ssh-agent@v0.9.0` action, or add a cleanup `finally` step: `rm -f ~/.ssh/id_rsa`.

#### H2: Full source code synced to remote VM then deleted — race condition
**File:** `deploy-backend.yml:35-41`, `deploy-dex.yml:35-44`, `deploy-farm.yml:35-44`
**Description:** The workflows rsync the entire repository to `/tmp/suidex-*-build/` on the VM, build Docker images, then `rm -rf` the build directory. If the SSH session is interrupted after rsync but before cleanup, source code persists in `/tmp/` on the VM accessible to other users. Additionally, `pnpm-lock.yaml` and all backend source are sent to the VM even for frontend-only builds.
**Fix:** Use Docker build context streaming (`docker build -` with stdin tar) or ensure cleanup runs in a trap. Restrict synced files to only what's needed.

#### H3: Docker-compose uses weak default passwords
**File:** `docker-compose.yml:17-18`
**Description:** `POSTGRES_USER: ${POSTGRES_USER:-suidex}` and `POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-suidex}` use trivially guessable defaults. PostgreSQL port 5432 is exposed to the host. If someone runs `docker compose up` without setting env vars (likely in dev), the database is accessible with username=password=`suidex`.
**Fix:** Generate a random default password in a `.env` file created by a setup script, or remove the port mapping for postgres (only expose within the Docker network).

#### H4: Seed script hardcodes local username in fallback DATABASE_URL
**File:** `scripts/seed-database.mts:33`
**Description:** `const DATABASE_URL = process.env.DATABASE_URL ?? "postgresql://cryptomichaael@localhost:5432/suidex_v2"` — The fallback contains a developer's local username (`cryptomichaael`). This is a minor information leak but more importantly, if someone runs the seed script without `DATABASE_URL` set, it will attempt to connect to a local database with that username, which either fails confusingly or succeeds against the wrong database.
**Fix:** Change fallback to `postgresql://suidex:suidex@localhost:5432/suidex_v2` (matching docker-compose defaults) or require DATABASE_URL with a clear error message.

### MEDIUM Findings

#### M1: Frontend hosting HTML missing security headers
**File:** `frontend/hosting/index.html`, `404.html`, `dex/index.html`, `farm/index.html`, `landing/index.html`
**Description:** The static HTML placeholder pages served via Firebase Hosting lack security-relevant meta tags. While `firebase.json` adds `Cache-Control` headers for static assets, it does not configure:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Content-Security-Policy`
- `Referrer-Policy`
- `Strict-Transport-Security` (Firebase adds this by default for `.web.app` domains, but custom domains need explicit config)

**Fix:** Add security headers in `firebase.json`:
```json
{
  "source": "**",
  "headers": [
    { "key": "X-Content-Type-Options", "value": "nosniff" },
    { "key": "X-Frame-Options", "value": "DENY" },
    { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
  ]
}
```

#### M2: Firebase hosting catch-all rewrite could mask routing errors
**File:** `firebase.json:36-38`
**Description:** The catch-all rewrite `"source": "**"` to `/index.html` means ANY path (including `/api/*`, `/.env`, `/admin`) returns the index HTML with 200 OK. This could confuse security scanners and masks 404s for paths that should not exist. If Firebase is ever misconfigured to proxy to the backend, API paths would be swallowed.
**Fix:** Make the catch-all rewrite more specific (e.g., only match paths that don't start with `/api/` or known static extensions), or add explicit 404 handling for known-bad paths.

#### M3: Admin API audit-log endpoint has no pagination limit cap
**File:** `backend/apps/admin-api/src/main.ts:105`
**Description:** `const limit = Number(req.query.limit ?? 100)` — The `limit` query parameter is taken directly from user input with no maximum cap. An authenticated admin could request `?limit=999999999` and dump the entire audit log table, potentially causing memory exhaustion or a slow query.
**Fix:** Add `Math.min(limit, 1000)` or similar cap.

#### M4: Setup script uses `|| true` to suppress infra creation failures
**File:** `infra/gcp/setup-dev.sh:25-47`
**Description:** Every `gcloud` command in the setup script is suffixed with `|| true`, including critical operations like creating the database instance (line 37-40), creating the database user (line 44-47), and creating the VPC connector (line 33-35). If any of these fail for a reason other than "already exists" (e.g., permissions error, quota exceeded), the failure is silently swallowed and subsequent steps may fail in confusing ways.
**Fix:** Check for specific "already exists" exit codes or use `--quiet` flags that handle idempotency, rather than blanket `|| true`.

#### M5: DEV_SETUP.md instructs overly broad IAM roles
**File:** `infra/gcp/DEV_SETUP.md:77-80`
**Description:** The deployer service account is granted `roles/secretmanager.admin` (line 77-78) and `roles/cloudsql.admin` (line 79-80). These are overly permissive. The deployer only needs to read secrets (not create/delete them) and deploy to Cloud Run (not manage SQL instances).
**Fix:** Use `roles/secretmanager.secretAccessor` and `roles/cloudsql.client` for the deployer SA.

#### M6: Workers shutdown doesn't clear setInterval timers (from I4)
**File:** `backend/apps/workers/src/main.ts:147-157,169-184`
**Description:** Two `setInterval` calls at lines 147 and 153 create recurring timers. The `shutdown()` function does not clear them. After `process.exit(0)` is called, this is moot in practice, but if `shutdown()` ever changes to drain gracefully (await worker.close() can take time), the intervals would continue firing during shutdown and potentially cause errors when database/redis connections are already closed.
**Fix:** Store interval IDs in variables and call `clearInterval()` in shutdown.

#### M7: ESLint disables `no-explicit-any` globally
**File:** `eslint.config.mjs:28`
**Description:** `"@typescript-eslint/no-explicit-any": "off"` disables the `any` type warning across the entire codebase. This reduces type safety and makes it easier to introduce runtime errors that TypeScript could otherwise catch.
**Fix:** Set to `"warn"` instead of `"off"` to encourage gradual fixing.

### LOW Findings

#### L1: build-frontend.mjs has path traversal via env var
**File:** `scripts/build-frontend.mjs:7`
**Description:** `const apiUrl = process.env.SUIDEX_PUBLIC_API_URL ?? "https://api-dev-placeholder.example.com"` is interpolated directly into HTML template strings (line 22). If `SUIDEX_PUBLIC_API_URL` contains HTML special characters (e.g., `"><script>alert(1)</script>`), it would be injected into the placeholder HTML pages. Risk is low since this is a build-time variable controlled by the deployer, but it's an XSS vector if the env var is ever sourced from untrusted input.
**Fix:** HTML-escape the `apiUrl` before interpolation.

#### L2: Seed script connects with max 5 pool connections
**File:** `scripts/seed-database.mts:40`
**Description:** `const pool = new PgPool({ connectionString: DATABASE_URL, max: 5 })` — A one-time seed script using a connection pool of 5 is unnecessary overhead. A single connection would suffice and reduce risk of connection leaks if the script errors.
**Fix:** Use `max: 1` for a seed script.

#### L3: .firebaserc exposes project ID
**File:** `.firebaserc:3`
**Description:** The Firebase project ID `suidex-test` is committed. While not a secret per se, it gives attackers a target for Firebase enumeration (checking Firestore rules, Storage rules, etc.). Low risk since it's a dev/test project.
**Fix:** Add `.firebaserc` to `.gitignore` and manage per-environment.

#### L4: Docker Compose exposes Redis without authentication
**File:** `docker-compose.yml:38`
**Description:** Redis is exposed on port 6379 to the host without any password. Combined with the default `redis://redis:6379` connection string, this means any local process can connect to Redis and read/write cached data including session data, precomputed routes, and BullMQ job payloads.
**Fix:** Add `--requirepass` to the Redis command or remove the host port mapping.

#### L5: No HEALTHCHECK in backend Dockerfiles
**File:** All 4 backend Dockerfiles
**Description:** While `docker-compose.yml` defines healthchecks, the Dockerfiles themselves lack `HEALTHCHECK` instructions. If these images are run outside of docker-compose (e.g., direct `docker run` as the deploy workflows do for dex/farm), there's no healthcheck.
**Fix:** Add `HEALTHCHECK` instruction to Dockerfiles.

#### L6: Seed script logs database URL pattern
**File:** `scripts/seed-database.mts:310`
**Description:** `console.log(\`Database: ${DATABASE_URL.replace(/\/\/.*@/, "//<hidden>@")}\`)` — The regex replacement hides the user:password portion but still reveals the host, port, and database name. Acceptable for dev tooling but should be noted.
**Fix:** No action needed, but consider logging only the database name.

---

## Summary Table

| ID | Severity | Status | Description |
|----|----------|--------|-------------|
| S1 | HIGH | NOT FIXED | K8s secrets in plaintext in git |
| S4/S5 | MEDIUM | PARTIAL | Dockerfiles add USER node; K8s still no securityContext; nginx still root |
| S8 | LOW | PARTIAL | node:22-alpine pinned; K8s/compose still use :latest |
| S12 | MEDIUM | PARTIAL | New deploys use set -e; old ci-cd.yml still has || true on rollouts |
| I1 | MEDIUM | NOT FIXED | Production images ship all devDeps |
| I2 | MEDIUM | WORSE | 32 new tx-helpers build artifacts committed; total ~155+ |
| I4 | LOW | NOT FIXED | setInterval not cleared on shutdown |
| I11 | LOW | PARTIAL | API-side precompute works; worker-side still no-op |
| H1 | HIGH | NEW | SSH key written to disk without cleanup |
| H2 | HIGH | NEW | Source code left in /tmp on VM if interrupted |
| H3 | HIGH | NEW | docker-compose default postgres password = username |
| H4 | HIGH | NEW | Seed script leaks developer username in fallback |
| M1 | MEDIUM | NEW | Hosting HTML lacks security headers |
| M2 | MEDIUM | NEW | Firebase catch-all rewrite masks errors |
| M3 | MEDIUM | NEW | Admin audit-log has no pagination limit cap |
| M4 | MEDIUM | NEW | setup-dev.sh || true on all gcloud commands |
| M5 | MEDIUM | NEW | Overly broad IAM roles in setup guide |
| M6 | MEDIUM | NEW | Workers setInterval not cleared (restatement of I4) |
| M7 | MEDIUM | NEW | ESLint disables no-explicit-any globally |
| L1 | LOW | NEW | build-frontend.mjs potential XSS via env var |
| L2 | LOW | NEW | Seed script over-provisions connection pool |
| L3 | LOW | NEW | .firebaserc leaks project ID |
| L4 | LOW | NEW | Redis exposed without auth |
| L5 | LOW | NEW | No HEALTHCHECK in Dockerfiles |
| L6 | LOW | NEW | Seed script reveals DB host in logs |

---

## What Was Done Well

1. **Dockerfiles now use `USER node`** — All 4 backend Dockerfiles run as non-root in production stage. This addresses the most critical container security concern.
2. **Graceful shutdown implemented** — All 4 backend services (api, admin-api, indexer, workers) have proper SIGTERM/SIGINT handlers that close DB connections and exit cleanly.
3. **Health endpoints added** — All services expose `/health` for platform probes (Cloud Run, K8s).
4. **API-side route precompute is real** — The API service now actually calls `computeRoute()` and caches results in Redis.
5. **Volume aggregation is implemented** — New `volume-aggregate.js` worker properly computes 24h USD volume per pair.
6. **Seed script uses parameterized queries** — No SQL injection risk; all database operations use `$1, $2...` parameters via `pg`.
7. **New deploy workflows use `set -e`** — Backend deploy script fails fast on errors.
8. **`.gitignore` properly excludes `.env` files** — No real secrets committed.
9. **`.env.example` contains only placeholder values** — No real credentials leaked.
10. **GCP setup guide recommends Workload Identity Federation** — Avoids long-lived service account keys.
