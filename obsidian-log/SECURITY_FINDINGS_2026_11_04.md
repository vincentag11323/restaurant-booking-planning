---
tags: security, findings, 2026-11-04
---

# Security findings: 2026-11-04 test round

What the security and review agents found after the sessions, API-key, route-group, logs and Swagger change, explained in plain words. Every finding was checked against the code by the orchestrator (file and line given).

**Overall:** no Critical or High problems. The new sessions, API key and middleware work as designed (the full real-stack e2e run passed 67 of 67). What's below are smaller holes.

**Severity:** Medium = should be fixed (or accepted by the owner in writing). Low = nice to fix. Decision = needs the owner to choose.

---

## 1. Web app: a crafted link can make your browser call a different API address (Medium)

**Where:** `resortos-booking-web/src/backendApi/reservations.ts:9` and `src/pages/ConfirmationPage.tsx:15`.

**How it works today:** after booking, the app shows a page like
`/bookings/abc123/confirmed`. It takes `abc123` out of the address bar and asks the API for `GET /api/users/reservations/abc123`. It copies the text straight in without "escaping" it.

**The problem:** someone can send a signed-in guest a link where the "id" is not an id but a path, for example:
```
/bookings/..%2F..%2Fadmins%2Freservations%2Fexport/confirmed
```
The browser turns `..%2F` into `../`, so the app ends up calling
`GET /api/admins/reservations/export` **with the guest's own sign-in cookie**, instead of a booking.

**Why it's only Medium today:**
* It can only make **reads** (GET), and nothing in the API changes data on a GET.
* The page only shows an error message, so the attacker sees nothing.
* An admin route still refuses a guest (403); it would only "work" against a signed-in **admin** who clicks the link.

**Why fix it anyway:** if a GET that changes something is ever added, or an admin clicks such a link, it becomes real damage.

**Fix:** encode the id before putting it in the path (`encodeURIComponent(id)`), so `../` stays harmless text. Add a test with the crafted link.

---

## 2. Assistant: one confirmation can book the same table twice (Medium)

**Where:** `resortos-booking-agent/src/booking/draftToken.ts:26` and `src/chat.ts:211`.

**How it works today:** when the assistant reads a booking back ("Table for 2, Friday 19:30, book it?"), it gives the browser a **signed ticket** (a token like `1762250000000.Ab3x...`). Pressing **Book it** sends the ticket back. The assistant checks the signature, then books, and uses the ticket text as the booking API's **"book only once" key** (`Idempotency-Key`), so pressing Book it twice gives the same booking, not two.

**The problem:** the signature check only looks at the first two parts of the ticket and ignores anything after. So these all pass the check:
```
1762250000000.Ab3x...
1762250000000.Ab3x....junk
01762250000000.Ab3x...
```
But each one is a **different** "book only once" key. Someone sending the ticket again with `.junk`, `.junk2`, ... gets a **new booking each time** from one read-back, for 15 minutes.

**What still limits it:** 3 confirmations per hour per connection, and the booking API's cap of 1 to 3 upcoming assistant bookings per email.

**Fix:** accept the ticket only in its exact format (13 digits, a dot, 43 characters), and use the clean ticket as the key. Add a test that `.junk` is refused.

---

## 3. Assistant: when the booking system is down mid-chat, the guest isn't told clearly (Medium)

**Where:** `resortos-booking-agent/src/chat.ts` (the tool wrappers around line 145) and the Mastra library.

**How it should work (spec F6):** if the booking API is down, the guest gets a clear fixed reply: **"Nothing was booked"** with an error status, so nobody thinks they have a table.

**What happens instead:** while the AI is checking availability, the booking API fails. The AI library (Mastra) quietly turns that failure into a message *to the AI*, and the AI writes something itself, so the guest gets a normal-looking AI answer (status 200) that may be vague or wrong. The booking API's error text is also passed to the AI, which it shouldn't see.

**Why the tests missed it:** the test uses a scripted AI that passes errors through; the real library swallows them.

**Fix:** catch booking-API failures inside the assistant's own tools, stop the turn, and send the fixed "Nothing was booked" reply. Test it through the real AI library.

---

## 4. API: an error log line can contain a guest's email (Medium)

**Where:** `resortos-booking-api/src/lib/logs.ts` (how 5xx errors are written).

**The problem:** when the database library (Prisma) fails, its error message includes the values of the query, e.g. a search for `ayu@guests.test`. The error log stores the full error, so the **email lands in `logs/errors.log`**, which the spec forbids (API_SPEC §8.3: no names, emails, phones in logs).

**Fix:** for database errors, log only the error type and code, not the message with values. (Separate from the ignored "500 on bad input" item below; this is about the log contents.)

---

## 5. API: one guest account can book every table (Decision)

**Where:** `resortos-booking-api/src/routes/reservations.ts` (guest bookings) and `src/app.ts` (no rate limit on `/api/users/*`).

**The problem:** a signed-up guest can keep booking with no limit, enough to fill every sitting for 60 days. Assistant bookings have a cap (1 or 3 per email); app bookings don't.

**Options:** (a) build the planned rate limits now (30 changes per hour per user, API_SPEC §7b); (b) cap upcoming bookings per guest (e.g. 5); (c) both.

---

## Ignored by the owner (2026-11-04)

* **API returns 500 instead of 400/413 for some bad input** (impossible dates like `2026-13-01`, oversized bodies, garbled URLs). Nothing breaks and no data is exposed; only the status code is wrong. The same kind of item in the assistant (oversized or garbled body → 500) is treated the same way.

---

## Low (nice to fix)

**API**
* The admin search treats `%` and `_` as "match anything" (`q=%` returns every booking). Still safe from SQL injection; just a wrong search result.
* The notification stub prints the guest's email to the console (`src/lib/notify.ts:5`).
* Rename `RATE_LIMIT_ERROR` to `AUTH_RATE_LIMIT_ERROR` (it's the sign-in-only message).

**Web app**
* Build every query string with `URLSearchParams` instead of gluing text together.
* The dev proxy also forwards look-alike paths such as `/apiX`; match only `/api/` and `/agents/`.
* If sign-out fails (e.g. offline), the app shows you signed out while the server session stays alive up to 1 hour. Show "Couldn't sign out, try again" instead.
* Two missing tests: a 401 is not retried; sign-out is sent to the server before the screen clears.
* Document `VITE_AGENT_URL` (now a full address with no path); make the profile-update fields optional to match the spec.

**Assistant**
* No security headers (`helmet`) on the assistant server, including its docs page.
* When the AI provider is unreachable, the guest gets a generic error instead of "busy, try again", and the AI library retries silently twice (each retry may be billed).
* The AI library's own logger could one day print guest messages; set it to quiet.
* Move six reply texts out of the Swagger file into their own small file; document the most common 400 ("Invalid request: …") in Swagger.

## Owner decisions still open
* **Assistant docs** turn on whenever `NODE_ENV` isn't `production`; a production deploy that forgets `NODE_ENV` would show them. Keep, or require `API_DOCS=true` explicitly?
* **`TRUST_PROXY=true`** on the assistant is only safe if it can be reached solely through that proxy; otherwise its per-connection limits can be bypassed. Deployment note.
* **Assistant limit replies** say whether an email has signed up ("sign in to manage them"), so anyone typing an email learns that. The wording is spec-approved; keep it?
* **Web app security headers** (CSP, anti-framing) are set by the hosting, at deployment.

## Cross-stack test of sessions + API key + middleware (done)
Ran the real web app, API and agent together on the dev database, attacking them from a real browser and with curl. **Everything passed:**
* Sign-in cookie is HttpOnly, never visible to page scripts; only a SHA-256 of the token is in the database; sign-out kills a copied cookie at once (directly and through the web proxy); expiry works; a new token on every sign-in (no session fixation); tampered cookies all refused.
* Every route group answered every credential combination exactly as API_SPEC says (7 routes × 12 credentials); a role change or a deleted user takes effect on the very next request.
* The agent key went **only** to `/api/agents/*` (checked with a logging proxy between agent and API); a key owned by a non-agent account is refused; the key appears nowhere (logs, Swagger, replies, database apart from its hash); a prompt asking the assistant to reveal it was refused; a real chat booking was recorded as made by the assistant.
* Cross-site requests (54 attempts) all blocked; SQL injection (132 requests) found nothing; logs hold ids only.

**New items from it:**
* **Decision: the "dev database" on 5432 is not real PostgreSQL.** It's PGlite (an in-browser-style Postgres) run by `npm run db:local`, where every connection shares one session. The RULES say tests use the real local dev DB; today that DB is PGlite. Real PostgreSQL 17 is installed on this Mac (Homebrew). Options: switch the dev DB to real PostgreSQL, or record that PGlite counts. (SQL injection S18 *was* also run on real PostgreSQL 17 by the API security agent and passed; race tests T2 need real Postgres.)
* Low: guest emails printed by the notify stub (same as the API Low above).
* Low: a cookie written as `session=j:{...}` gets "Authentication required" instead of "Invalid or expired token". Still refused, no bypass.
* Fixed now: AGENT_SPEC §5.5 said the agent key gets 403 elsewhere and is sent on every call; it's 401 and only sent to `/api/agents/*`.
* Rows it left in the dev DB: two assistant bookings (RSV-B6LTJ8, RSV-YPS4PV) for two `sec-…@example.com` guest accounts, their audit rows, and one expired admin session (cleared at the next admin sign-in). Throwaway users and keys were deleted.

Related: [[OPEN_QUESTIONS]], [[TEST_RUNBOOK]], [[DECISIONS]].
