# Backend Security Audit: resortos-booking-api

**Date:** 2026-11-04
**Audited commit:** `9e3720c` (re-audit; first audit was at `18bcfcc`)
**Auditors:** ecc:security-reviewer, ecc:database-reviewer, ecc:typescript-reviewer, ecc:silent-failure-hunter (read-only). Every finding was checked against the code by the orchestrator. Severities were adjusted where the agents over-rated; noted inline.

---

## 1. Previous findings: status

| # | Finding | Status | Evidence |
|---|---|---|---|
| 1 | Logout / role change didn't take effect (stateless JWT) | ✅ FIXED | `src/middleware/auth.ts:25-35`: session row + live role every request; logout deletes the row (`src/routes/auth.ts:130-141`); user delete cascades sessions and API key. Token = `randomBytes(32)`, new on every login/register, only SHA-256 stored. |
| 2 | Account takeover by signing up with a password-less guest account's email | ❌ STANDS (by design, AGENT_SPEC D2) | `src/routes/auth.ts:101-106`: sets a password and signs in with no proof of email ownership. |
| 3 | Default admin password `Admin12345!` in git; re-seed never rotates it | ❌ STANDS | `.env.example:12`; `prisma/seed.ts:59` `update: {}`. The agent key now rotates on re-seed; the admin password still doesn't. |
| 4 | No TLS enforcement on the DB connection in production | ❌ STANDS | `src/lib/db.ts` passes `DATABASE_URL` as is; `.env.example:3` has `sslmode=disable`. |
| 5 | Audit row written outside the change's transaction | ❌ STANDS | `src/routes/reservations.ts:213,246,267`; `src/routes/agent.ts:118` |
| 6 | Blank or non-numeric `AUTH_RATE_LIMIT` turns rate limiting off (NaN) | ❌ STANDS | `src/config.ts:30`; reproduced: 10/10 requests passed with `NaN` |
| 7 | No rate limit on `/api/agents/*` (before key check), public routes, booking changes, admin export | ❌ STANDS | `src/app.ts:52`; `POST /agents/reservations` unthrottled. A leaked key could book out every table. |
| 8 | Log write I/O unguarded: a disk error breaks error handling | ❌ STANDS (wider) | `src/lib/logs.ts:56-62`; now also on the success path (see A) |
| 9 | Failed cookie / agent-key auth not written to AuditLog or alerted | ❌ STANDS | `src/middleware/auth.ts:27,32,48` |
| 10 | Register reveals which emails have accounts ("Email is already registered") | ❌ STANDS | `src/routes/auth.ts:102` |
| 11 | `trust proxy 1` only correct behind exactly one proxy | ❌ STANDS | `src/app.ts:25`; verify against the real deployment |
| 12 | Notify stub prints guest email + booking to console | ❌ STANDS | `src/lib/notify.ts:5` |
| L | Feb 30 rolls over to Mar 2; bad date gives 500; login email has no `.max(254)` | ❌ STANDS | `src/domain/slots.ts:9-12`, `src/lib/validate.ts:16`, `src/routes/auth.ts:27-30` |
| L | `PATCH /me` with a deleted user's token gave 500 | ✅ FIXED | Session cascade gives 401 |
| L | Unbounded guest booking list; admin `q` search unindexed; ReservationTable cascade | ❌ STANDS | Low; the list is spec-defined (API_SPEC.md:48) |

## 2. New findings

| ID | Severity | Finding | Where |
|---|---|---|---|
| A | MEDIUM (agent said CRITICAL) | `logChange` runs after the DB write and throws on disk errors. On login/register the cookie is already set, so the user gets a 500 but is actually signed in. Reliability issue, not access to anyone else's account. | `src/lib/logs.ts:56-62`, `src/routes/auth.ts:92-94` |
| B | MEDIUM | Log rotation keeps 1 old copy (was 3). Combined with #7/#9, an attacker can flood failures to push evidence out. | `src/lib/logs.ts:50-54` |
| C | MEDIUM | A missing or misspelled `NODE_ENV` means public Swagger + full OpenAPI (admin/agent routes) and a session cookie without `Secure`. Nothing checks or logs this at startup. | `src/config.ts:37-39` |
| D | MEDIUM | Non-numeric `*_LOG_MAX_BYTES` (NaN) means logs never rotate and grow until the disk is full, which then triggers A. | `src/config.ts:15`, `src/lib/logs.ts:51` |
| E | MEDIUM | 5xx Prisma error stacks can write guest data (e.g. emails) into errors.log, against API_SPEC §8.3 (already in OPEN_QUESTIONS). | `src/lib/logs.ts:64-68` |
| F | LOW (agent said HIGH) | Expired sessions are only purged on that user's next login; `Session.expiresAt` has no index. Documented as deliberate; only table growth. | `src/routes/auth.ts:86`, `prisma/schema.prisma` |
| G | LOW | `metadata as object` cast before a Prisma Json write; safe today, risky if a future caller passes `req.body`. | `src/lib/audit.ts:29` |

## 3. Verified sound

- **Route-group guards:**
  - `/api/public`: no auth.
  - `/api/auth`: rate-limited.
  - `/api/users`: session.
  - `/api/admins`: session + ADMIN.
  - `/api/agents`: agent key.
  - The cookie has no effect on `/api/agents`, and the key has no effect on `/api/users` and `/api/admins`.
- **No IDOR:** `findOwned` scopes every lookup to the caller and returns 404 on a miss.
- **CSRF:** Origin check plus `SameSite=Lax`.
- **CORS:** a single configured origin.
- **Cookie flags:** HttpOnly, SameSite=Lax, Secure in production.
- **SQL:** all queries go through Prisma; the one raw query is parameterized (`src/routes/agent.ts:112`); the S18 injection tests pass.
- **Double-booking:** blocked by the `@@unique([tableId, slotStart])` constraint, the `claimConfirmed` lock and the agent `FOR UPDATE` lock.
- **AuditLog:** a database trigger makes it immutable.
- **No mass assignment:** zod strips unknown keys.
- **Express defaults:** the query parser is `simple` and JSON bodies are capped at 100kb.
- **Swagger:** served from the app's own origin; helmet's Content-Security-Policy is unchanged.
- **Health route:** returns only `{status:"ok"}`.
- **No secrets in logs:** `src/lib/logs.ts` never writes tokens, hashes, passwords or emails.
- **Checks:** `tsc --noEmit` is clean and `npm audit --omit=dev` reports 0 vulnerabilities.

## 4. Suggested fixes (priority order)

Each is a few lines unless noted. All go through RULES.md: Plan, Build, Review, Test.

1. **#3 Admin seed:** replace `ADMIN_PASSWORD=Admin12345!` in `.env.example` with a placeholder (e.g. `CHANGE_ME`). Make `seed.ts` refuse the placeholder or a short password, and rotate the hash on re-seed (`update: { passwordHash }`).
2. **#6 + D Config validation:** add a `positiveInt()` helper in `config.ts` for `AUTH_RATE_LIMIT`, `PORT`, `*_LOG_MAX_BYTES`, and refuse to boot on bad values.
3. **#8 + A Log writes:** wrap `writeLine` in try/catch and fall back to `console.error`. A logging failure must never change the response.
4. **C Docs opt-in:** `apiDocs: process.env.API_DOCS === "true"` only. Optionally log the resolved `NODE_ENV`/`isProduction`/cookie `secure` at startup.
5. **#7 Agent rate limit:** IP-keyed `jsonRateLimit` in front of `requireAgentKey` on `/api/agents`, plus a per-key cap on `POST /agents/reservations`.
6. **#4 DB TLS:** in production, refuse to boot when `DATABASE_URL` has `sslmode=disable` or no SSL setting.
7. **E Error-log PII:** for Prisma/DB errors, log only `{ name, code }`, not `message` or `stack`.
8. **#9 + B Failed-auth evidence:** audit failed cookie/key checks (e.g. `AUTH_FAILED`) with IP thresholding like `checkFailedLogins`. Consider keeping more than one rotated log copy.
9. **#5 Audit atomicity:** write the AuditLog row inside the same `$transaction` as the change.
10. **Low items:**
    - Validate calendar dates (round-trip check) and return 400 instead of 500.
    - Add `.max(254)` to the login email.
    - Redact or remove the console output in `notify.ts`.
    - Paginate the user booking list (needs a spec change).
    - Add a pg_trgm index for admin search.
    - Add an `expiresAt` index for a future session purge.

## 5. Decisions needed from the owner

- **#2 Guest-account claim:** require email verification (link/code) before sign-up can claim a password-less account, or accept the takeover risk explicitly in SECURITY_SPEC.
- **#10 Register enumeration:** accept it, or switch register to a uniform response (needs an email flow).
- **#11 trust proxy:** confirm the production proxy topology (number of hops, whether it overwrites `X-Forwarded-For`).
- **F Session purge:** keep the deliberate no-cleanup-job design, or add a periodic sweep.
