# Coding Standards: TypeScript Backend
**ResortOS AI — Restaurant Booking App**

Applies to `resortos-booking-api` (Express + Prisma) and `resortos-booking-agent` (Node HTTP server + AI SDK). The test repo `resortos-booking-e2e` follows the TypeScript and testing rules here too. Owner rule, 2026-11-04.

## 1. Language and tooling
* **TypeScript only**, `"strict": true`. No `.js` source files (generated output such as `dist/` and `coverage/` doesn't count).
* No `any`. Use `unknown` and narrow it. No `@ts-ignore`, no coverage-ignore comments.
* `npm run typecheck` must pass before every commit.
* ES modules; relative imports end in `.js` (the api's Node16 resolution).

## 2. Folder layout (api)
| Folder | Holds | Must not |
|---|---|---|
| `src/routes/` | One router per route group or resource. Parses input, calls domain code, sends the response | Hold business rules or raw SQL |
| `src/domain/` | Business rules: slots, table allocation, booking logic. Plain functions, easy to test | Read `req` / `res` |
| `src/lib/` | Shared helpers: database client, errors, validation, logs, permissions, hashing | Know about specific routes |
| `src/middleware/` | Express middleware: the auth checks per route group, the error handler | Contain business rules |
| `prisma/` | Schema, migrations, seed | — |
| `docs/` | Plain-language guides: `database.md`, `api.md` | — |

The agent repo keeps the same idea: `src/booking/` (calls to the booking API and booking rules), `src/agent/` (the AI model), `src/llm/` (the LLM provider), `src/types/` (shared types).

## 3. Routes (REST)
* **Route names are plural**: `/api/users`, `/api/admins`, `/api/agents`, `/public/menus`, `/public/availabilities`, `/users/reservations`. Exceptions only where a plural makes no sense: `public`, `auth`, action words (`register`, `login`, `logout`, `export`, `lookup`), `health`, `me`, `stats`.
* Every route lives in a **route group**, each with its own middleware (`api_spec.md`, "Route groups"): `/api/public/*` none, `/api/auth/*` rate limit, `/api/users/*` sign-in check, `/api/admins/*` sign-in + admin check, `/api/agents/*` agent key check.
* Routes take the caller's id and role **only from `req.auth`**, never from the body or query.
* HTTP methods: `GET` reads, `POST` creates, `PATCH` updates part of a record, `DELETE` removes or cancels.
* Errors are always `{ "message": string }`, thrown as `HttpError(status, message)` and sent by the error handler. Status codes: 400 bad input, 401 not signed in / bad credentials, 403 signed in but not allowed, 404 not found or not yours, 429 rate limited.

## 4. Input, database, security
* Validate every body and query with a **Zod** schema before use.
* All database access through **Prisma**. No raw SQL strings in app code (hand-written SQL only in migrations).
* Never pass a possibly-`undefined` value into a Prisma `where` on `delete` / `update` (it matches every row).
* Multi-step writes that must succeed or fail together go in one transaction.
* Secrets (passwords, session tokens, API keys) are stored only as hashes and never logged, returned or put in an LLM prompt.
* Config comes from environment variables, read and checked once in `src/config.ts`; missing required values stop the app at start.

## 5. Logging (api)
* Every successful create / update / delete writes one `[METHOD]`-tagged line to `logs/app.log` and `console.log`; every failed request one tagged line to `logs/errors.log` and `console.error` (`api_spec.md` §8). Use the shared logger in `src/lib/logs.ts`; no other `console.*` calls in app code.
* Never log bodies, passwords, tokens, keys, names, emails, phone numbers or allergies.

## 6. Naming and style
* **camelCase everywhere (owner rule, 2026-11-04), never snake_case or kebab-case:** file names (`errorLog.ts`, `bookingApi.ts`, `routeRules.test.ts`), folders inside a repo, functions, variables, object keys, JSON fields, query parameters, database columns. Types, classes and React components are PascalCase.
* **The only exception is UPPER_SNAKE_CASE** (always fully upper case), used for **constants** (`MAX_PARTY_SIZE`, `SLOT_START_TIMES`, `SESSION_COOKIE`) and for data that leaves the code, where renaming would break stored rows or the API contract: environment variable names (`DATABASE_URL`), database enum values sent over the API (`MAIN_HALL`, `BRANCH_1`, `NO_SHOW`), and audit / log action names (`CREATE_RESERVATION`, `USER_CREATED`). Repo names (`resortos-booking-api`) and npm script names stay as they are.
* One clear name per file (`tokens.ts`, `permissions.ts`).
* Small functions with one job. Prefer early returns over deep nesting.
* Comments say **why**, not what. No commented-out code.
* KISS: no new dependency, abstraction or option unless the spec needs it.

## 7. Testing
* Test first (red → green → refactor).
* api: Jest keeps `src/` at **100%** coverage (statements, branches, functions, lines); the Playwright API suite covers each endpoint. agent: `node --test` covers the rules, the API client and the server.
* Each test names its scenario ID from `test_scenarios_app.md` / `test_scenarios_assistant.md` and asserts the exact status code and message from the spec.
* Tests never weaken: a failing test is fixed in the code, not deleted.

## 8. Git
* Small local commits, Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`, `refactor:`, `chore:`), tests committed before the code that makes them pass.
