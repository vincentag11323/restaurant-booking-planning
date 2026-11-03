# Testing Plan
**ResortOS AI — Restaurant Booking App**

Both repositories use **Playwright** (`@playwright/test`) for behaviour tests. The backend also has a **Jest** suite (added 2026-11-03 at the owner's request) that enforces **100% statement, branch, function and line coverage** of `src/`. `npm test` runs everything in each repo.

**Scenario lists (what is tested, where, and what's left):** [test_scenarios_app.md](test_scenarios_app.md) for guests and admins using the app, [test_scenarios_assistant.md](test_scenarios_assistant.md) for guest → AI → API.

## 1. Functional Testing
* **Coverage (backend, Jest):** supertest against the Express app plus direct unit tests of helpers and middleware, on an in-memory PGlite. The 100% threshold fails the run if any line or branch goes untested. Unreachable defensive code is removed, not ignored.
* **Unit Testing:** Tests pure functions at the code level, such as 2.5-hour slot computation, table count calculation (⌈party ÷ 2⌉ for Main Hall, 1–4 rooms for VIP Wing), and table selection.
* **API Testing (backend repo):** Calls the real Express API against a test PostgreSQL database. Covers:
  * The full CRUD cycle: create → read → reschedule → soft-cancel.
  * `400` triggers: invalid payload, capacity overflow, invalid slot time, past date.
  * `404` triggers: unknown Reservation ID, another guest's Reservation ID.
  * Admin endpoints are read-only and reject `USER` tokens.
* **UI Testing (frontend repo):** Drives the React app in a real browser with the API mocked via `page.route`, so UI tests run without the backend. Covers:
  * The reservation flow (login → book → confirmation).
  * Booking list interactions (reschedule, cancel).
  * Form validations.
  * Admin log filters and the detail drawer.
* **User Acceptance Testing (UAT):** Restaurant staff and a few real guests try the full booking → arrival flow on a staging environment before public release.

## 2. Security Testing
* **Authorization Testing (IDOR):** Ensure Insecure Direct Object Reference is impossible. Example: Guest A changing `/api/users/reservations/<A's id>` to `<B's id>` must get `404`. Covered by an automated API test.
* **Vulnerability Scanning:** Run `npm audit` in CI on both repos, and enable Dependabot alerts for known CVEs.
* **Penetration Testing:** A lightweight third-party or OWASP ZAP scan before the first public launch.

## 3. Performance Testing
* **Concurrency Testing:** Fire many simultaneous bookings at the same slot and assert that no table is ever double-booked. A DB unique constraint on (table, slot) guarantees this.
* **Load Testing (local, now):** `npm run load` in `resortos-booking-e2e` (autocannon) hits the test API with availability reads, sign-ins and bookings, and reports requests/s, p95 latency and error count. Run it against real PostgreSQL for meaningful numbers.
* **Load Testing (staging):** Simulate peak dinner-time traffic (e.g. 500 concurrent users browsing slots and booking) with k6 or Artillery on staging. Target p95 API latency < 300 ms.
