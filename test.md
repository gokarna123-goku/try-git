
# 01 — Phase 1b: automated tooling results

Audit date: 2026-11-17. Architecture and the full route/env inventory are in `00-architecture.md`.

## How the commands were run

Dev servers were live on ports 3000–3002 and 4000 when the audit started, so install, build, lint, typecheck and tests were **not** run inside the working tree. Both repos were copied (working tree including uncommitted changes; without `node_modules`, `.next`, `dist`, `src/generated`, `.git`, `.env`, `.env.local`) to a scratch folder and everything ran there. Consequences:

- `pnpm install --frozen-lockfile` was a true from-scratch install.
- No real secret was present: `RESEND_API_KEY` was unset, so no email could leave the machine.
- No source file in `backend/` or `frontend/` was modified.

Rule-4 check before the test run (recorded in `evidence/05-backend-tests.txt`): database host `localhost:5432`, database `workspace_by_crown_audit_test` (created by the test harness), `RESEND_API_KEY` not set. The dev database `workspace_by_crown_local` was not touched.

## Results

| # | Check | Command | Result | Evidence |
|---|---|---|---|---|
| 1 | Frozen install, backend | `pnpm install --frozen-lockfile` | **Pass.** Lockfile committed and in sync; 416 packages | `evidence/01-pnpm-install-backend.txt` |
| 2 | Frozen install, frontend | same | **Pass.** 495 packages, 11 workspace projects. pnpm printed "Agent skills awaiting approval: @tanstack/react-table" (informational) | `evidence/01-pnpm-install-frontend.txt` |
| 3 | Backend build | `pnpm build` (`prisma generate && tsc -p tsconfig.build.json`) | **Pass**, no warnings | `evidence/02-backend-build.txt` |
| 4 | Backend typecheck | `pnpm exec tsc --noEmit` | **Pass**, 0 errors | `evidence/03-backend-tsc.txt` |
| 5 | Backend lint | `pnpm exec eslint src` | **Cannot run**: no `eslint.config.*` exists (exit 2) | `evidence/04-backend-lint.txt` |
| 6 | Backend tests | `pnpm test` (vitest) | **Pass**: 15 files, 145 tests, 12 s. One `pg` DeprecationWarning: "Calling client.query() when the client is already executing a query" | `evidence/05-backend-tests.txt` |
| 7 | Frontend lint | `pnpm lint` (3 apps) | **Pass**, 0 errors, 0 warnings. Shared packages are linted only through the apps that import them | `evidence/06-frontend-lint.txt` |
| 8 | Frontend typecheck | `pnpm typecheck` (11 projects) | **Pass** | `evidence/07-frontend-typecheck.txt` |
| 9 | Frontend production build | `NODE_ENV=production pnpm build` | **Pass** for admin, marketing, tenant; no warnings | `evidence/08-frontend-build.txt` |
| 10 | Frontend tests | — | **None exist** (0 test/spec files) | — |
| 11 | `pnpm audit`, backend | `pnpm audit`, `pnpm audit --prod` | All deps: 1 high + 1 moderate (lodash 4.17.21), 1 moderate (minimatch 10.2.6). **Prod deps: none** | `evidence/09-pnpm-audit-backend.txt` |
| 12 | `pnpm audit`, frontend | same | 1 moderate (minimatch 10.2.6), also listed under `--prod` | `evidence/09-pnpm-audit-frontend.txt` |
| 13 | OSV lookup of every locked version | `node evidence/scripts/osv-lockfile.mjs backend frontend` (499 + 582 package versions) | Same three advisories as `pnpm audit`, nothing else. Control query (lodash 4.17.20) returned 4 advisories, so the lookup works | `evidence/13-osv-lockfile.txt` |
| 14 | Framework advisories | OSV query for the installed versions | `next@16.4.5`, `express@5.3.0`, `pg@8.23.1`, `react@19.3.0`, `@prisma/client@7.10.1`, `axios@1.21.0`, `nodemailer@10.5.0`, `zod@4.6.7`, `helmet@8.3.0`, `express-rate-limit@8.7.3`, `@tanstack/react-query@5.107.1`: **no advisories returned** by api.osv.dev on 2026-11-17 | `evidence/13-osv-lockfile.txt` |
| 15 | Dependency authenticity | `node evidence/scripts/dep-authenticity.mjs backend frontend` | All **62** direct dependencies exist on registry.npmjs.org, registry integrity hash equals the lockfile hash for every one, all are long-established packages (oldest 2010, newest first-published 2026-07 = `@tailwindcss/turbopack`, 2.8 M weekly downloads). No typosquat or invented name found | `evidence/11-dependency-authenticity.json` |
| 16 | Install scripts / pnpm settings | lockfile + `pnpm-workspace.yaml` | Build scripts are opt-in. Backend allows `esbuild`, `prisma`, `@prisma/engines`, denies `workerd` (`backend/pnpm-workspace.yaml`). Frontend denies `sharp`, `unrs-resolver`. Only direct dep with an install script: `prisma` (preinstall). No `.npmrc`, no registry token, no `overrides` | `evidence/14-repo-hygiene.txt`, `evidence/01-*.txt` |
| 17 | Secrets in git history | regex fallback over `git log --all -p` (4 commits per repo) | **No real secret found.** Only placeholders (`.env.example`, test pepper, sample seed password). No `.env`, key or dump was ever committed | `evidence/12-secret-scan-git-history.txt` |
| 18 | Committed env files, keys, dumps, binaries | `git ls-files`, blob sizes | None. Largest tracked blob is `pnpm-lock.yaml` (269 kB). `.env` / `.env.local` exist on disk and are git-ignored | `evidence/14-repo-hygiene.txt` |
| 19 | Unused code and deps | `pnpm dlx knip` | Backend: 2 unused deps, 3 unused devDeps, 38 unused exports, 2 empty placeholder files. Frontend: 21 unused dependency entries, 15 unused exports | `evidence/15-knip-*.txt` |
| 20 | Node engines | manual | Backend `engines.node >=22.12`; frontend none; no `.nvmrc`; Docker `node:22-alpine`; audit ran on Node 24.19.0 | `evidence/14-repo-hygiene.txt` |

## Findings from Phase 1

Status: CONFIRMED = the code or tool output shows it unambiguously. None of these has been exploited yet (that is Phase 5).

| ID | Severity | Status | Finding | Location | Evidence |
|---|---|---|---|---|---|
| P1-01 | Medium | CONFIRMED | Known default super-admin password in code. `SEED_SUPER_ADMIN_PASSWORD` defaults to `ChangeMe@12345`; the seed creates a `SUPER_ADMIN` with it whenever no staff exist. docker-compose refuses to run without the variable, but a manual `pnpm db:seed` does not. The same value is in the committed `.env.example`. | `backend/src/config/env.ts:67`, `backend/prisma/seeds/super-admin.seed.ts:11-23`, `backend/.env.example` | code |
| P1-02 | Low | CONFIRMED | `TOKEN_PEPPER` placeholder passes validation. The only rule is length ≥ 16; the committed placeholder `change-me-to-a-long-random-string` is accepted, so a copied `.env.example` boots in production with a public pepper. | `backend/src/config/env.ts:44-46`, `backend/.env.example:26` | code |
| P1-03 | Low | CONFIRMED | `NODE_ENV` defaults to `development`. Unset in production means auth rate limit 1000 per 15 min instead of 30, lookup limit 2000 instead of 150, and OTP emails printed to the log when no mail provider is set. | `backend/src/config/env.ts:21-23`, `backend/src/middleware/rate-limiter.ts:41,53,70`, `backend/src/core/mail/mailer.ts:130-134` | code |
| P1-04 | Low | CONFIRMED | Backend has no lint. `eslint` is installed but there is no config and no `lint` script. | `backend/package.json` | `evidence/04-backend-lint.txt` |
| P1-05 | Low | CONFIRMED | No CI and no git hooks. `frontend/.github/workflows/` and `backend/.husky/` are empty; nothing enforces build, tests or audit before a push. | — | `evidence/14-repo-hygiene.txt` |
| P1-06 | Low | CONFIRMED | Vulnerable dev-only packages. lodash 4.17.21 (GHSA-r5fr-rjxr-66jc high, GHSA-xxjr-mmjv-4gpg moderate) arrives through `prisma` → `@prisma/studio-core`; minimatch 10.2.6 (GHSA-g3hv-75wg-m249) through `eslint`. Neither is imported by application code, and backend `--prod` audit is clean. In the frontend, minimatch appears in the `--prod` graph only because `packages/config` lists `eslint` and `eslint-config-next` under `dependencies` instead of `devDependencies`. | `frontend/packages/config/package.json:10-13` | `evidence/09-*.txt`, `evidence/13-osv-lockfile.txt` |
| P1-07 | Low | CONFIRMED | Tests can load the real Resend key. `src/config/env.ts:9` imports `dotenv/config` and `vitest.config.ts` does not override `RESEND_API_KEY`, so `pnpm test` in `backend/` runs with the key from `.env`. Whether any test triggers a send is not verified yet (Phase 4). | `backend/src/config/env.ts:9`, `backend/vitest.config.ts:20-25` | code |
| P1-08 | Low | CONFIRMED | Runtime Node version is not pinned. Docker uses the floating tag `node:22-alpine`; local build and test ran on Node 24; no `.nvmrc`; frontend has no `engines`. | `backend/infra/docker/api.Dockerfile:11`, `frontend/Dockerfile:14,46` | `evidence/14-repo-hygiene.txt` |
| P1-09 | Info | CONFIRMED | Uncommitted work at audit time: 8 modified files in backend, 19 modified + 3 untracked in frontend. The audit covers the working tree; a release cut from `main` would not match it. | — | `evidence/14-repo-hygiene.txt` |
| P1-10 | Info | CONFIRMED | Frontend has zero automated tests; backend tests cover accounting, leads and password reset only (no tests for login, registration, tenant isolation, billing or any admin route). | `backend/tests/` | `evidence/05-backend-tests.txt` |
| P1-11 | Info | CONFIRMED | Dead and unused pieces: `cookie-parser` (+ types) unused; `ROOT_DOMAIN` declared but never read by the backend; `src/config/redis.ts` and `swagger.ts` are empty placeholders; 38 unused backend exports. knip's "`@prisma/client` unused" is a false positive (the generated client imports it). | — | `evidence/15-knip-backend.txt` |
| P1-12 | Info | SUSPECTED | `pnpm test` with defaults would fail on this machine: the default `TEST_DATABASE_URL` points at port 5433 and nothing listens there. Not run with defaults. | `backend/vitest.config.ts:13` | — |

## Leads for Phase 2 and 3 (all SUSPECTED — seen while mapping, not proven)

These came up while reading code for the inventory. They are listed so nothing is lost; each will be confirmed or dropped with proof in the phase noted.

| Lead | What was seen | Where | Phase |
|---|---|---|---|
| L-1 | Rate-limit key may be attacker-controlled. The BFF forwards the **first** `X-Forwarded-For` entry; nginx appends the real client address at the **end** (`$proxy_add_x_forwarded_for`), so the first entry is whatever the client sent. The API then trusts it (`trust proxy 1`). A client could rotate the header to dodge `authLimiter`. | `frontend/packages/api/src/proxy.ts:101-105`, `frontend/packages/api/src/server.ts:53-54`, `backend/infra/nginx/crown.conf:22`, `backend/src/app.ts:28` | 2 + 5 |
| L-2 | Document permission taken from client input. `requireDocumentAccess` uses `?type=` or `body.type` before it looks at the stored document, so a user allowed only purchases might open or edit a sales document by passing a purchase type. | `backend/src/products/accounting/middleware/document-access.ts:26-30` | 2 + 5 |
| L-3 | Three `$queryRawUnsafe` / `$executeRawUnsafe` calls in the demo sandbox copier. | `backend/src/modules/demo/sandbox.service.ts:58,81,148` | 2 |
| L-4 | BFF prefix allow-list may be bypassable with encoded `../` segments in the catch-all path (e.g. reaching `/admin/*` through the tenant or marketing host). The API still enforces its own auth, so impact would be limited. | `frontend/packages/api/src/proxy.ts:123-129,148-150` | 3 + 5 |
| L-5 | BFF CSRF check only runs when an `Origin` header is present, and `new URL(origin)` throws on `Origin: null`. The upstream `Content-Type` is forced to JSON regardless of what the browser sent. | `frontend/packages/api/src/proxy.ts:133-144` | 3 + 5 |
| L-6 | No CSP and no frame protection on any Next.js app. `next.config.ts` has no `headers()`; nginx adds only HSTS, nosniff and Referrer-Policy; helmet runs on the API, which browsers never reach. | `frontend/apps/*/next.config.ts`, `backend/infra/nginx/crown.conf:34-36` | 3 + 5 |
| L-7 | Demo tenant is unauthenticated by design and can create per-browser sandbox tenants; cost and abuse limits need review. Accounting `ensureBooks` runs on every request. | `backend/src/middleware/authenticate.ts:27-36`, `backend/src/middleware/resolve-tenant.ts`, `backend/src/modules/demo/sandbox.service.ts` | 2 |
| L-8 | Rate limiters are in-memory per process; limits multiply with every API replica. | `backend/src/middleware/rate-limiter.ts` | 2 |
| L-9 | `GET /public/invoices/:token` reads the token without a schema; the share token is the only secret. | `backend/src/routes/v1/platform.routes.ts:35-37` | 2 |
| L-10 | Session cookie `Secure` flag depends on `NODE_ENV`; `SameSite=Lax`; one cookie per app host. Sibling-subdomain behaviour to check since every tenant shares the parent domain. | `frontend/packages/api/src/proxy.ts:74-80` | 3 |

## NOT CHECKED (and why)

| Item | Reason | What was done instead |
|---|---|---|
| gitleaks / trufflehog | Not installed; `brew install gitleaks semgrep trivy osv-scanner` was blocked by the session's permission policy | Regex scan of the full history of both repos (`evidence/12-secret-scan-git-history.txt`). Weaker than gitleaks: no entropy detection |
| semgrep (`p/owasp-top-ten`, `p/javascript`, `p/typescript`, `p/nodejs`, `p/react`, nextjs) | Same block | Nothing equivalent. Manual review in Phases 2–4 covers the same ground by hand |
| trivy fs / trivy image | Same block; Docker daemon is also not running, so no image could be built or scanned | `pnpm audit` + OSV cover npm packages only. OS packages in `node:22-alpine`, `postgres:18-alpine`, `nginx:1.29-alpine` are unscanned |
| osv-scanner binary | Same block | Equivalent API lookup of all 1,081 locked package versions (`evidence/13-osv-lockfile.txt`) |
| Docker image build | Docker daemon not running | Dockerfiles read by hand only |
| Backend lint | No ESLint config in the repo (finding P1-04) | `tsc --noEmit` passes with `strict` + `noUncheckedIndexedAccess` |
| Reachability of every transitive advisory | Only 3 advisories exist; checked by import grep and `--prod` graph | — |
| `pnpm install` inside the real working tree | Deliberately avoided: dev servers were running | Clean-copy install |

## Left on the machine by this phase

- `audit/` (this folder).
- Local PostgreSQL database `workspace_by_crown_audit_test` on `localhost:5432` (test data only). Can be reused in Phase 5 or dropped.
- A build copy in the session scratch folder (outside the project).
