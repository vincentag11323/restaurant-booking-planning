# API Specification
**ResortOS AI — Restaurant Booking App**

**Route names are plural** (owner rule, 2026-11-04; exceptions: `public`, `auth`, action words, `health`, `me`, `stats`).  
Base URL: `/api`. Paths in the tables below are relative to it (e.g. `/public/menus` is `/api/public/menus`). JSON in/out. Times are ISO-8601 UTC strings in responses. Dates in requests are restaurant-local `YYYY-MM-DD`.

**Auth:** guests and admins: an `HttpOnly`, `SameSite=Lax` cookie `session` (path `/`, `Secure` in production). Its value is a long random token; the server stores only its SHA-256, in the `Session` table (`DATA_MODEL.md`). A session lasts **1 hour from sign-in** and is never extended (`SESSION_TTL`, default `1h`). AI assistant: header `X-Agent-Key: <AGENT_API_KEY>`, checked against the `ApiKey` table (SHA-256 only, `DATA_MODEL.md`). `Authorization: Bearer` is not accepted.  
**When a session ends:** (1) **sign-out** deletes its `Session` row, so that cookie, and any copy of it, is refused from the next request on; (2) **time**: once `expiresAt` (sign-in + 1 hour) has passed the row no longer counts, so the session ends by itself with no job or timer. Removing a user removes their sessions and API key (cascade). An ended session gets `401 Invalid or expired token` on every route that needs sign-in.  
**CSRF:** `SameSite=Lax` plus an origin rule: a request other than GET/HEAD/OPTIONS whose `Origin` header isn't `CORS_ORIGIN` gets `403 Cross-origin request blocked`.  
**Errors:** always `{ "message": string }`, using 400 / 401 / 403 / 404 (see `PRD_BOOKING_V1.md` §4.2).

Legend: 🌐 public · 👤 USER (or any logged-in user where noted) · 🛡️ ADMIN · 🤖 AGENT (the AI booking assistant's service account)

### Route groups and their middleware (owner rule, 2026-11-04)
Every path starts with one group, and each group has its own middleware. The app-wide basics (security headers, CORS, the CSRF origin rule, JSON parsing, the logs in §8, the 404 and error handlers) apply everywhere; they are not auth.

| Group | Who | Auth middleware | What it does |
|---|---|---|---|
| `/api/public/*` (§3 availability, §4 menu, §7 health) | anyone, including the assistant | **none** | No cookie or key is read. |
| `/api/auth/*` (§1) | anyone | rate limit only | Sign-up, sign-in, sign-out. Sign-out reads its own cookie to delete that session. |
| `/api/users/*` (§2 profile, §3 reservations, §4 recommendations) | signed-in guests and admins | **sign-in check** | Reads only the `session` cookie. No cookie → `401 Authentication required`. No matching `Session` row, or `expiresAt` passed → `401 Invalid or expired token`. Otherwise sets `req.auth = { userId, role, sessionId }` (role read from the user row, so a role change applies at once). |
| `/api/admins/*` (§5) | admins | sign-in check, then admin check | Same sign-in check; a role other than `ADMIN` → `403 Forbidden`. |
| `/api/agents/*` (§6) | the AI assistant | **agent key check** | Reads only `X-Agent-Key`. Missing or empty → `401 Authentication required`. No matching `ApiKey` row, or its owner isn't `AGENT` → `401 Invalid API key`. Otherwise sets `req.auth = { userId, role: "AGENT" }`. |

So a guest's cookie means nothing on `/api/agents/*`, and the assistant's key means nothing on `/api/users/*` or `/api/admins/*` (both get `401 Authentication required`). Routes take the caller's id and role **only** from `req.auth`, never from the body or query.

**RBAC:** two layers: the group middleware above, then a permission check per route (`src/lib/permissions.ts`), e.g. an admin on `/api/users/reservations` gets `403 Forbidden`. Admins can't create bookings.

## 1. Auth
| Method | Path | Access | Body | Success | Errors |
|---|---|---|---|---|---|
| POST | `/auth/register` | 🌐 | `{ email, password (8–72), name, phone? }` | `201 { user }` + `session` cookie (a new `Session` row; that user's expired rows are deleted) | `400` validation / `Email is already registered` |
| POST | `/auth/login` | 🌐 | `{ email, password }` | `200 { user }` + `session` cookie (a new `Session` row; that user's expired rows are deleted) | `400` validation, `401 Invalid email or password` |
| POST | `/auth/logout` | 🌐 | — | `200 { message }`: deletes this cookie's `Session` row (if any) and clears the cookie. Sessions on other devices stay. | — |

`user` = `{ id, email, name, phone, allergies, role }`. There is no refresh: after 1 hour every call returns `401` and the guest signs in again. The app checks for a session on page load with `GET /users/me`. Rate limit: 20 req / 15 min / IP on `/auth/*`; over the limit → `429 { message: "Too many attempts. Please wait a few minutes and try again." }`.

## 2. Profile
| Method | Path | Access | Body | Success |
|---|---|---|---|---|
| GET | `/users/me` | 👤 any | — | `200 user` |
| PATCH | `/users/me` | 👤 any | `{ name?, phone? \| null, allergies?: string[] }` | `200 user` |

## 3. Availability & Reservations
| Method | Path | Access | Input | Success | Errors |
|---|---|---|---|---|---|
| GET | `/public/availabilities` | 🌐 | `?branch&sector&date` | `200 Slot[]` | `400` |
| GET | `/users/reservations` | 👤 | — | `200 Reservation[]` (own, newest slot first) | `401` |
| GET | `/users/reservations/:id` | 👤 | — | `200 Reservation` | `404` (missing or not owned) |
| POST | `/users/reservations` | 👤 | `{ branch, sector, date, startTime, partySize, specialRequests? }` | `201 Reservation` | `400` (see below) |
| PATCH | `/users/reservations/:id` | 👤 | `{ date, startTime, partySize? }` | `200 Reservation` | `400`, `404` |
| DELETE | `/users/reservations/:id` | 👤 | — | `200 Reservation` (status `CANCELLED`) | `400`, `404` |

```ts
type Slot = {
  startTime: "11:00" | "13:30" | "17:00" | "19:30";
  slotStart: string; slotEnd: string;      // ISO
  totalUnits: number; freeUnits: number;   // tables (Main Hall) or rooms (VIP)
  bookable: boolean;                       // false if started / > 60 days ahead
};

type Reservation = {
  id: string; code: string;                // code = "RSV-7K3Q9X"
  branch: "BRANCH_1" | "BRANCH_2"; sector: "MAIN_HALL" | "VIP_WING";
  slotStart: string; slotEnd: string; partySize: number;
  specialRequests: string | null;
  status: "CONFIRMED" | "SEATED" | "COMPLETED" | "CANCELLED" | "NO_SHOW";
  source: "APP" | "AGENT";                // who booked: the guest in the app, or the AI assistant on their behalf
  tableIds: string[]; createdAt: string; updatedAt: string; cancelledAt: string | null;
};
```

**Reservation 400 messages (exact strings, asserted by tests).** Zod validation messages are prefixed with the field path, e.g. `partySize: Party size must be between 1 and 50`. Business-rule messages have no prefix.
* `startTime must be one of 11:00, 13:30, 17:00, 19:30`
* `Party size must be between 1 and 50`
* `Selected slot is in the past or too far ahead`
* `Insufficient capacity for the selected slot`
* `Table overlap conflict, please try again`
* `Only confirmed reservations can be changed`
* `Reservation slot has already started`

## 4. Menu
| Method | Path | Access | Input | Success |
|---|---|---|---|---|
| GET | `/public/menus` | 🌐 | `?type=SEASONAL_FINE_DINING\|A_LA_CARTE` | `200 MenuItem[]` |
| GET | `/users/menus/recommendations` | 👤 USER (admins get `403 Forbidden`) | — | `200 MenuItem[]` (allergen-free for this user, chef's picks first) |

`MenuItem = { id, name, description, type, priceCents, allergens: string[], chefPick }`

## 5. Admin (read-only)
All require 🛡️ ADMIN. A `USER` token gets `403`. There are **no** POST/PATCH/DELETE admin routes.

| Method | Path | Input | Success |
|---|---|---|---|
| GET | `/admins/stats` | `?date=YYYY-MM-DD` (default today local) `&branch?` | `200 Stats` |
| GET | `/admins/reservations` | filters + `?page=1&pageSize=20` (max 100) | `200 { items: AdminReservation[], total, page, pageSize }` |
| GET | `/admins/reservations/export` | filters + `?format=csv\|json` | `200` file download (`Content-Disposition: attachment`) |
| GET | `/admins/reservations/:id` | — | `200 AdminReservation` or `404` |

**Filters:** `branch?`, `sector?`, `status?`, `source?` (`APP` | `AGENT`), `from?` / `to?` (local dates, inclusive), `q?` (matches reservation `code` exactly, or name / email / phone case-insensitive contains).

```ts
type AdminReservation = Reservation & { customer: { name: string; email: string; phone: string | null } };

type Stats = {
  date: string;
  totalBookings: number;      // non-cancelled reservations with a slot on that date
  agentBookings: number;      // of those, booked by the AI assistant
  totalCovers: number;        // sum of partySize of those
  cancellations: number;      // CANCELLED reservations with a slot on that date
  utilization: { sector: "MAIN_HALL" | "VIP_WING"; usedUnits: number; capacityUnits: number; rate: number }[];
  // capacityUnits = tables × 4 slots (× branches if no branch filter)
};
```

Side effects: `GET /admins/reservations/:id` → audit `VIEW_RESERVATION_DETAIL`. Export → audit `EXPORT_RESERVATIONS` (`metadata.rows`), plus `SECURITY_ALERT` if rows > 1,000. Export is capped at 10,000 rows.

## 6. Agent (AI booking assistant) 🤖
| Method | Path | Input | Success | Errors |
|---|---|---|---|---|
| POST | `/agents/reservations` | `{ email, phone?, branch, sector, date, startTime, partySize, specialRequests?, clientIp? }`, header `Idempotency-Key?` (8–200 chars) | `201 Reservation` (`source: "AGENT"`); same key again → `200` with the same booking | Same `400` messages as `POST /users/reservations`, plus `This email already has 3 upcoming assistant bookings. Sign in to manage them.`, `This email can't be booked through the assistant` (staff accounts), `Idempotency-Key was already used for another booking` |
| POST | `/agents/guests/lookup` | `{ email }` (one exact email) | `200 { exists, upcomingAssistantBookings, assistantBookingLimit }` | `400` for anything but a single email; 60 requests/minute |

* **Assistant booking limit per email:** an email that **hasn't signed up** (password-less guest account, or new) may hold **1** upcoming assistant booking; a signed-up guest may hold **3** (more looks like spam). Over the limit → `400 "This email already has a booking made with the assistant. To book a different time, sign up with this email in the app, cancel that booking there, then ask me again."` (not signed up) or `400 "This email already has 3 upcoming assistant bookings. Sign in to manage them."` (signed up). The check runs inside the booking transaction.
* Booking finds or creates a **password-less guest account** for the email. `POST /auth/register` with that email **claims** it (sets the password) and keeps its bookings; a password-less account can't log in (`401 Invalid email or password`, same as a wrong password).
* Audit: `CREATE_RESERVATION` (actor = assistant, `metadata: { source, onBehalfOf, clientIp }`), `VIEW_GUEST` for every lookup.
* Export adds a `source` column.

## 7. Misc
| Method | Path | Success |
|---|---|---|
| GET | `/public/health` | `200 { status: "ok" }` |
| * | unknown route | `404 { message: "Route not found" }` |

## 7b. Rate limits for every route group
**Owner rule 2026-11-04: every API has a rate limit. Decided now, built later** (only `/api/auth/*` and the assistant lookup are limited today).

| Group | Limit | Counted per | Why this number |
|---|---|---|---|
| `/api/public/*` | **300 requests / minute** | IP address | Menus and free sittings are read a lot, and the assistant's server (one IP) checks availability for every guest it serves (up to 6 lookups per chat turn). Health and docs share this limit |
| `/api/auth/*` | **20 requests / 15 minutes** (unchanged) | IP address | Slows password guessing; bcrypt makes sign-in costly (~10 sign-ins/s) |
| `/api/users/*` | **120 requests / minute**, and **30 changes / hour** (POST, PATCH, DELETE on reservations and profile) | signed-in user | A guest browsing never comes close; the change limit stops scripted booking churn |
| `/api/admins/*` | **120 requests / minute**, and **10 exports / hour** | signed-in admin | Paging and filtering the log is fast; exports read up to 10,000 rows |
| `/api/agents/*` | `guests/lookup`: **60 / minute** (unchanged); `reservations`: **30 / minute** | API key | A backstop: the assistant's own limits (3 confirmations / hour per guest IP) are tighter |

* Over the limit: `429 { "message": "Too many requests. Please wait a minute and try again." }` (auth keeps its own message), with the standard `RateLimit` and `Retry-After` headers. Each 429 is a line in `logs/errors.log` (§8.2).
* Counters live in the API process's memory (`express-rate-limit`). Running more than one API instance later needs a shared store (e.g. Redis); not now.
* Test runs raise the limits through environment variables, the way `AUTH_RATE_LIMIT` works today; production uses the numbers above.

## 7a. Interactive API docs (Swagger / OpenAPI)
**Owner request 2026-11-04.** The API publishes an **OpenAPI 3.1** document and a **Swagger UI** page, written for people, not only machines.

| Method | Path | Success |
|---|---|---|
| GET | `/public/docs` | `301` to `/public/docs/` (Swagger UI loads its files with relative URLs), which returns `200` the Swagger UI page (try every endpoint from the browser) |
| GET | `/public/docs/openapi.json` | `200` the OpenAPI 3.1 document (JSON) |

* **Framework:** `swagger-ui-express` serves the page; `@asteasolutions/zod-to-openapi` builds the document **from the same Zod schemas that validate requests**, so the docs can't drift from the real rules. No hand-written YAML.
* **Human-readable:** every endpoint has a one-line `summary` and a plain-language `description` (who may call it, what it does, side effects such as audit rows or log lines), a tag per route group (`Public`, `Auth`, `Users`, `Admins`, `Agents`), an example request and response, and every error response with its exact `message` from this spec. The document's top `description` explains sign-in (session cookie), the assistant key, the route groups and the error shape in a few short paragraphs.
* **Security schemes:** `sessionCookie` (`apiKey` in cookie `session`) for `/users/*` and `/admins/*`; `agentKey` (`apiKey` in header `X-Agent-Key`) for `/agents/*`; none for `/public/*` and `/auth/*`.
* **Where it's shown:** under `/api/public` (no auth middleware), in development and tests always; in production only when `API_DOCS=true` (default off, so the route list isn't advertised).
* **Security headers:** the strict `helmet` content-security policy stays on every route except the docs page, which gets only what Swagger UI needs.
* **Drift guard (test):** every route the app mounts appears in the OpenAPI document with the same method and path, and every documented path is mounted.

## 8. Logging: one success log, one error log
**Owner request 2026-11-04.** Two files, both JSON lines, both rotated the same way.

**Tag:** every line, in the file and on the console, starts with the request's HTTP method in brackets: `[POST]`, `[PATCH]` (this API updates with PATCH, there is no PUT), `[DELETE]`, and `[GET]` for failed reads.

### 8.1 Success log: `logs/app.log` (`APP_LOG_PATH`)
Every request that **creates, updates or deletes database rows** and succeeds writes one line and prints it with `console.log`:
`[POST] {"time","status","method","path","action","count?","resourceId?","userId","ip"}`. `userId` is the signed-in caller, or on sign-up / sign-in the account just signed in. `count` appears only on `SESSIONS_EXPIRED_DELETED` (how many rows were removed).

| Request | `action` | `resourceId` |
|---|---|---|
| `POST /auth/register` | `USER_CREATED` (or `USER_CLAIMED` when sign-up claims a password-less guest account the assistant made) + `SESSION_CREATED`, one line each | user id / session id |
| `POST /auth/login` | `SESSION_CREATED` (and `SESSIONS_EXPIRED_DELETED` with a count when old rows were removed) | session id |
| `POST /auth/logout` (the cookie matches a `Session` row, live or expired) | `SESSION_DELETED` | session id |
| `PATCH /users/me` | `PROFILE_UPDATED` | user id |
| `POST /users/reservations` | `RESERVATION_CREATED` | reservation code |
| `PATCH /users/reservations/:id` | `RESERVATION_RESCHEDULED` | reservation code |
| `DELETE /users/reservations/:id` | `RESERVATION_CANCELLED` | reservation code |
| `POST /agents/reservations` | `RESERVATION_CREATED` (plus `GUEST_ACCOUNT_CREATED` when the email was new) | reservation code / user id |

Reads (GET) and requests that change nothing (e.g. logout without a session, an idempotent agent retry) write no success line. Audit-log rows written alongside a change are not logged separately.

### 8.2 Error log: `logs/errors.log` (`ERROR_LOG_PATH`)
Every failed request (any `4xx` or `5xx` response, including unknown routes and rate limits) writes one line and prints it with `console.error`:
`[POST] {"time","status","method","path","message","userId?","ip"}`. `5xx` lines also carry the error name and stack.

**Owner decision (2026-11-03):** the 401 from the session check on page load (`GET /users/me` with a missing or expired session cookie) stays logged: there is no valid session, so it counts as a failure.

### 8.3 What is never logged
Request bodies, passwords, session tokens or cookies, API keys, query strings, names, emails, phone numbers, allergies, guest messages.

### 8.4 Rotation (both files)
Before writing, if the file is over **5 MB** (`APP_LOG_MAX_BYTES` / `ERROR_LOG_MAX_BYTES`), it's renamed to `.1` (e.g. `app.log.1`) and the previous `.1` is **deleted**. So each log has **one** live file and at most **one** old copy, and disk use stays under about 10 MB per log. Tests point the paths at temp files and can silence the console (`APP_LOG_CONSOLE=false`, `ERROR_LOG_CONSOLE=false`).
