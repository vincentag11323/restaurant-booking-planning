# End-to-End Test Specification
**ResortOS AI — Restaurant Booking App**

**Status:** APPROVED 2026-11-03 (owner request). Repo: `resortos-booking-e2e` (fourth repo).

## 1. Rule: no mocking
Every test drives the **real stack**: a real browser on the PWA's production build → the real booking API → a real database, and for the assistant, real `POST /agents/users/chats` → the real agent → the **real AI model** (Cent AI, `deepseek-v4.1-flash`) → the real API. Nothing is faked. The per-repo mocked tests stay as the fast checks; this suite is the proof.

Scenarios come from [TEST_SCENARIOS_APP.md](TEST_SCENARIOS_APP.md) and [TEST_SCENARIOS_ASSISTANT.md](TEST_SCENARIOS_ASSISTANT.md). **A test exists only for a scenario listed there** (no spec, no work). Test titles start with the scenario ID, e.g. `[B1] …`.

## 2. How a run works (`npm test` in the e2e repo)
Playwright starts the stack from each repo's own scripts, on ports that don't clash with dev servers:

| Service | Started with | Port |
|---|---|---|
| Test database | `resortos-booking-api`: `npm run db-test` | 5433 |
| Booking API | `resortos-booking-api`: `npm run start:test` | 4400 |
| Web (production build) | `resortos-booking-web`: `npm run preview:test` | 5190 |
| Booking assistant | `resortos-booking-agent`: `npm run start:test` | 4500 |

* **`npm run db-test`** starts an in-memory database, applies the migrations and loads the **test data set**: the normal tables and menu plus clearly fake guests (`*@guests.test`), sample bookings in every status, the admin and the assistant account. The banner says `TEST DATABASE (fake data)`. It's rebuilt on every start.
* The web build's `/api` proxy target and the agent's rate limits are configurable through env for this run; production defaults don't change.

## 3. Recording and speed
* Every browser test records a **video** (`videos/<scenario-id>.webm`) with clicks slowed by 250 ms so they're visible.
* Tests are short: one scenario each, fresh guest per test, no waits beyond what the UI needs.
* Assistant tests are HTTP conversations (no screen); their transcript is written to the results instead of a video.

## 4. Results document
Each run writes **`RESULTS.md`** in the e2e repo: one row per test with scenario ID, title, **expected result**, **actual result** (pass, or the failure message), duration, and the video file. Expected results are written in each test; failures are reported to the owner.

## 5. AI usage
The assistant tests use the real model with **no fixed call cap**: they run until done or until Cent AI refuses (rate limit or budget). The run records the number of AI calls and the cost. If Cent AI refuses, remaining AI tests are marked "skipped: AI unavailable" rather than failed.

## 6. Additional checks (added 2026-11-03)
| Check | How | Notes |
|---|---|---|
| **Accessibility (T4)** | `@axe-core/playwright` on the main screens; no serious or critical violations | Automated audit only; a screen-reader pass stays manual |
| **Other browsers (T3)** | Key browser scenarios also run on WebKit (iPhone 14) and Firefox | |
| **Visual regression (T5)** | `toHaveScreenshot()` of sign-in, booking, menu and admin log; the first run records baselines | Kept in `tests/visual/__screenshots__` |
| **Session expiry (T7)** | API started with `SESSION_TTL=8s`; the page waits past it, keeps showing what it had, and the next server call sends the guest to sign in | Production default stays 1 hour |
| **Chat screen (C-UI)** | The PWA's `/chat` against the real agent and model | Uses AI calls |
| **Load (T6)** | `npm run load` (autocannon): availability reads, sign-ins and bookings against the test API, reporting requests/s, p95 latency and errors | Test database is PGlite (one query at a time), so numbers are a floor, not production capacity |

## 7. Out of scope for this suite (stay unit-tested)
| Scenario | Why it can't be real here |
|---|---|
| AI busy / budget used up (F9, F10) | Would require draining the owner's key |
| Booking API down mid-chat (F8) | Stopping the shared API would break the other tests |
| Rate limits (A13, X8) | The run raises limits so the suite isn't throttled |
| True parallel races (B9, M6, F6) | Need time or real PostgreSQL; tracked as to-be-tested items |
