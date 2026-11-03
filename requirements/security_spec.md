# Security Specification
**ResortOS AI — Restaurant Booking App**

The app stores guest personal data (name, email, phone number, allergy information), so protecting it is a top priority. No payment data is processed in v1.

## 1. Authentication & Authorization

> **APPROVED and IMPLEMENTED 2026-11-03: simpler sessions.**
> * Guests and admins: **one HttpOnly session cookie, fixed 1 hour from sign-in** (not extended by activity), then signed out. `SameSite=Lax`, `Secure` in production. No access token in memory, no refresh token, no `/api/auth/refresh`, no `ACCESS_TOKEN_TTL`.
> * PWA: **no idle timer.** When the session ends, screens keep the data they already show; the next server call returns 401 and the app goes to sign-in.
> * AI assistant: a **predefined API key** (`AGENT_API_KEY`) created at seeding; only its hash is stored. Sent as `X-Agent-Key`; maps to role `AGENT` with the same two permissions. No agent password.
> * Keep CSRF safe with `SameSite=Lax` + the CORS origin rule, and add a test for it. Update api_spec.md, DIAGRAM.md, READMEs and every test helper that sends `Authorization: Bearer`.

> **Owner request 2026-11-04: server-side sessions and an API key table.**
> * Sign-out must end the session **at once**: a copied cookie stops working. Sessions live in a `Session` table; the cookie holds a random token and only its SHA-256 is stored. Expiry (1 hour from sign-in) ends a session by itself.
> * The assistant's key lives in an `ApiKey` table (SHA-256 only). For now one key, owned by the one `AGENT` account; other key owners (vendors, special users) are to be decided later.
> * One auth middleware on every request checks the caller and passes `{ userId, role }` to the route. `/agent/*` routes are for the assistant only; every other route is for guests and admins only (`api_spec.md`, "Every request").
> * `SESSION_SECRET` and JWTs are no longer used.

* **Role-Based Access Control (RBAC):** Three roles, `USER`, `ADMIN` and `AGENT` (the AI booking assistant, planned in `agent_spec.md` §5.5). Routes check permissions; `AGENT` may only book on a guest's behalf and look up one guest by exact email.
  * Guests can only see and change **their own** reservations and profile. Requesting another guest's reservation returns `404 Not Found`, so record IDs cannot be probed.
  * Admins have **read-only** access to all reservations, stats and exports. There is no admin endpoint that creates, edits or deletes bookings.
* **Session Management:** sign-in creates a `Session` row (`data_model.md`) and sets the `session` cookie to a random 32-byte token; only its SHA-256 is stored. Every request looks the row up: it counts only while `expiresAt` (sign-in + 1 hour, never extended) is in the future. Sign-out deletes the row, so the cookie and any copy are refused at once. The agent key is checked against `ApiKey.keyHash` (SHA-256) on each call, and only a key owned by the `AGENT` account is accepted.
* **CSRF:** `SameSite=Lax` keeps the cookie off cross-site POSTs; non-GET requests with a foreign `Origin` header get `403`. The web app and the API must be on the same site (e.g. `app.example.com` + `api.example.com`).
* **Brute-force protection:** Rate limiting on `/api/auth/*` (20 requests / 15 minutes / IP).
* **MFA:** *Future enhancement* for Admin logins. Not in v1, to keep scope small.

## 2. Data Encryption
* **Data in Transit:** TLS 1.2 or 1.3 (HTTPS) for all API and web traffic, terminated at the hosting provider.
* **Data at Rest:** Use a managed PostgreSQL provider with storage-level AES-256 encryption enabled by default. There is no app-level column encryption in v1.
* **Passwords:** Hashed with **bcrypt** (cost factor 12). Never stored or logged in plaintext.
* **Secrets:** `AGENT_API_KEY` and DB credentials are provided via environment variables and never committed to Git.

## 3. Application Hardening
* All request bodies and query strings are validated with **Zod** schemas.
* All DB access goes through **Prisma** (parameterized queries, no raw string SQL).
* `helmet` security headers. CORS restricted to the configured frontend origin.
* React escapes all rendered output by default, and the app never uses `dangerouslySetInnerHTML`.

## 4. Regulatory Compliance
* Comply with Indonesia's Personal Data Protection Law (**UU PDP No. 27/2022**): collect only necessary data, state the purpose (reservations and allergy-safe recommendations), and support account data export/deletion on request.
* Allergy information is health-related data. Use it only for menu recommendations and kitchen notes, never for marketing.
