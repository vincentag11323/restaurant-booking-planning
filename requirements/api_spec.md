# API Specification
**ResortOS AI — Restaurant Booking App**

Base URL: `/api`. JSON in/out. Times are ISO-8601 UTC strings in responses. Dates in requests are restaurant-local `YYYY-MM-DD`.

**Auth:** guests and admins: an `HttpOnly`, `SameSite=Lax` cookie `session` (path `/`, `Secure` in production). Its value is a long random token; the server stores only its SHA-256, in the `Session` table (`data_model.md`). A session lasts **1 hour from sign-in** and is never extended (`SESSION_TTL`, default `1h`). AI assistant: header `X-Agent-Key: <AGENT_API_KEY>`, checked against the `ApiKey` table (SHA-256 only, `data_model.md`). `Authorization: Bearer` is not accepted.  
**When a session ends:** (1) **sign-out** deletes its `Session` row, so that cookie, and any copy of it, is refused from the next request on; (2) **time**: once `expiresAt` (sign-in + 1 hour) has passed the row no longer counts, so the session ends by itself with no job or timer. Removing a user removes their sessions and API key (cascade). An ended session gets `401 Invalid or expired token` on every route that needs sign-in.  
**CSRF:** `SameSite=Lax` plus an origin rule: a request other than GET/HEAD/OPTIONS whose `Origin` header isn't `CORS_ORIGIN` gets `403 Cross-origin request blocked`.  
**Errors:** always `{ "message": string }`, using 400 / 401 / 403 / 404 (see `PRD_BOOKING_v1.md` §4.2).

Legend: 🌐 public · 👤 USER (or any logged-in user where noted) · 🛡️ ADMIN · 🤖 AGENT (the AI booking assistant's service account)

### Every request: the auth middleware
One middleware runs on **every** `/api` request, before any route. It works out who is calling and hands that to the route; routes never read the user id or role from the body or query.

1. **`X-Agent-Key` header present** (the cookie is then ignored): hash it, look it up in `ApiKey` together with its owner. Valid only if the row exists and the owner's role is `AGENT` → caller is `{ userId, role: "AGENT" }`. Otherwise the caller is signed out with reason `Invalid API key`. An empty header counts as no header.
2. **Else a `session` cookie present:** hash it, look it up in `Session` together with its user. Valid only if the row exists and `expiresAt` is in the future → caller is `{ userId, role, sessionId }` (role `USER` or `ADMIN`, read from the user row, so a role change applies at once). Otherwise the caller is signed out with reason `Invalid or expired token`.
3. **Neither:** the caller is signed out with reason `Authentication required`.

The result is attached to the request (`req.auth`). Then the **route rules** below decide who may continue:

| Routes | Who may call | Signed out (or bad credentials) | Wrong role |
|---|---|---|---|
| `/agent/*` (§6) | `AGENT` only | `401` with the reason above | `USER` / `ADMIN` → `403 Forbidden` |
| `/me`, `/reservations`, `/menu/recommendations` (§2, §3, §4) | `USER` / `ADMIN` as each row says | `401` with the reason above | `AGENT` → `403 Forbidden` |
| `/admin/*` (§5) | `ADMIN` only | `401` with the reason above | `USER` / `AGENT` → `403 Forbidden` |
| Public: `/auth/*`, `/menu`, `/availability` | anyone signed out, guests, admins | allowed: bad or expired credentials are **ignored** here, so a guest with an old cookie can still sign in | `AGENT` → `403 Forbidden` |
| `/health` | anyone, including `AGENT` | allowed | — |

**RBAC:** two layers. The route rules above (agent routes are for the assistant only; every other route is for guests and admins only), then a permission check per route (`src/lib/permissions.ts`). Admins can't create bookings.

## 1. Auth
| Method | Path | Access | Body | Success | Errors |
|---|---|---|---|---|---|
| POST | `/auth/register` | 🌐 | `{ email, password (8–72), name, phone? }` | `201 { user }` + `session` cookie (a new `Session` row) | `400` validation / `Email is already registered` |
| POST | `/auth/login` | 🌐 | `{ email, password }` | `200 { user }` + `session` cookie (a new `Session` row; that user's expired rows are deleted) | `400` validation, `401 Invalid email or password` |
| POST | `/auth/logout` | 🌐 | — | `200 { message }`: deletes this cookie's `Session` row (if any) and clears the cookie. Sessions on other devices stay. | — |

`user` = `{ id, email, name, phone, allergies, role }`. There is no refresh: after 1 hour every call returns `401` and the guest signs in again. The app checks for a session on page load with `GET /me`. Rate limit: 20 req / 15 min / IP on `/auth/*`; over the limit → `429 { message: "Too many attempts. Please wait a few minutes and try again." }`.

## 2. Profile
| Method | Path | Access | Body | Success |
|---|---|---|---|---|
| GET | `/me` | 👤 any | — | `200 user` |
| PATCH | `/me` | 👤 any | `{ name?, phone? \| null, allergies?: string[] }` | `200 user` |

## 3. Availability & Reservations
| Method | Path | Access | Input | Success | Errors |
|---|---|---|---|---|---|
| GET | `/availability` | 🌐 | `?branch&sector&date` | `200 Slot[]` | `400` |
| GET | `/reservations` | 👤 | — | `200 Reservation[]` (own, newest slot first) | `401` |
| GET | `/reservations/:id` | 👤 | — | `200 Reservation` | `404` (missing or not owned) |
| POST | `/reservations` | 👤 | `{ branch, sector, date, startTime, partySize, specialRequests? }` | `201 Reservation` | `400` (see below) |
| PATCH | `/reservations/:id` | 👤 | `{ date, startTime, partySize? }` | `200 Reservation` | `400`, `404` |
| DELETE | `/reservations/:id` | 👤 | — | `200 Reservation` (status `CANCELLED`) | `400`, `404` |

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
| GET | `/menu` | 🌐 | `?type=SEASONAL_FINE_DINING\|A_LA_CARTE` | `200 MenuItem[]` |
| GET | `/menu/recommendations` | 👤 any | — | `200 MenuItem[]` (allergen-free for this user, chef's picks first) |

`MenuItem = { id, name, description, type, priceCents, allergens: string[], chefPick }`

## 5. Admin (read-only)
All require 🛡️ ADMIN. A `USER` token gets `403`. There are **no** POST/PATCH/DELETE admin routes.

| Method | Path | Input | Success |
|---|---|---|---|
| GET | `/admin/stats` | `?date=YYYY-MM-DD` (default today local) `&branch?` | `200 Stats` |
| GET | `/admin/reservations` | filters + `?page=1&pageSize=20` (max 100) | `200 { items: AdminReservation[], total, page, pageSize }` |
| GET | `/admin/reservations/export` | filters + `?format=csv\|json` | `200` file download (`Content-Disposition: attachment`) |
| GET | `/admin/reservations/:id` | — | `200 AdminReservation` or `404` |

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

Side effects: `GET /admin/reservations/:id` → audit `VIEW_RESERVATION_DETAIL`. Export → audit `EXPORT_RESERVATIONS` (`metadata.rows`), plus `SECURITY_ALERT` if rows > 1,000. Export is capped at 10,000 rows.

## 6. Agent (AI booking assistant) 🤖
| Method | Path | Input | Success | Errors |
|---|---|---|---|---|
| POST | `/agent/reservations` | `{ email, phone?, branch, sector, date, startTime, partySize, specialRequests?, clientIp? }`, header `Idempotency-Key?` (8–200 chars) | `201 Reservation` (`source: "AGENT"`); same key again → `200` with the same booking | Same `400` messages as `POST /reservations`, plus `This email already has 2 upcoming assistant bookings. Sign in to manage them.`, `This email can't be booked through the assistant` (staff accounts), `Idempotency-Key was already used for another booking` |
| POST | `/agent/guests/lookup` | `{ email }` (one exact email) | `200 { exists, upcomingAssistantBookings, assistantBookingLimit }` | `400` for anything but a single email; 60 requests/minute |

* **Assistant booking limit per email:** an email that **hasn't signed up** (password-less guest account, or new) may hold **1** upcoming assistant booking; a signed-up guest may hold **3** (more looks like spam). Over the limit → `400 "This email already has a booking made with the assistant. To book a different time, sign up with this email in the app, cancel that booking there, then ask me again."` (not signed up) or `400 "This email already has 3 upcoming assistant bookings. Sign in to manage them."` (signed up). The check runs inside the booking transaction.
* Booking finds or creates a **password-less guest account** for the email. `POST /auth/register` with that email **claims** it (sets the password) and keeps its bookings; a password-less account can't log in (`401 Invalid email or password`, same as a wrong password).
* Audit: `CREATE_RESERVATION` (actor = assistant, `metadata: { source, onBehalfOf, clientIp }`), `VIEW_GUEST` for every lookup.
* Export adds a `source` column.

## 7. Misc
| Method | Path | Success |
|---|---|---|
| GET | `/health` | `200 { status: "ok" }` |
| * | unknown route | `404 { message: "Route not found" }` |

## 8. Failure logging
Every failed request (any `4xx` or `5xx` response, including unknown routes and rate limits) is:
* written as one JSON line to **`logs/errors.log`** (`ERROR_LOG_PATH`): `{ time, status, method, path, message, userId?, ip }`. No request bodies, passwords, tokens or guest messages. `5xx` lines also carry the error name and stack;
* printed with `console.error`.

**Owner decision (2026-11-03):** the 401 from the session check on page load (`GET /me` with a missing or expired session cookie) stays logged: there is no valid session, so it counts as a failure.

**Rotation:** before writing, if the file is over **5 MB** (`ERROR_LOG_MAX_BYTES`), it's renamed `errors.log.1` (older copies shift to `.2`, `.3`; the 4th is deleted), so at most **3** old files are kept. Tests point `ERROR_LOG_PATH` at a temp file and can silence the console (`ERROR_LOG_CONSOLE=false`).

