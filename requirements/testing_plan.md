# Testing Plan
**ResortOS AI — Restaurant Booking App**

Both repositories use **Playwright** (`@playwright/test`) as the single test runner. Each repo has one tool and one command: `npm test`.

## 1. Functional Testing
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
* **Authorization Testing (IDOR):** Ensure Insecure Direct Object Reference is impossible. Example: Guest A changing `/api/reservations/<A's id>` to `<B's id>` must get `404`. Covered by an automated API test.
* **Vulnerability Scanning:** Run `npm audit` in CI on both repos, and enable Dependabot alerts for known CVEs.
* **Penetration Testing:** A lightweight third-party or OWASP ZAP scan before the first public launch.

## 3. Performance Testing
* **Concurrency Testing:** Fire many simultaneous bookings at the same slot and assert that no table is ever double-booked. A DB unique constraint on (table, slot) guarantees this.
* **Load Testing:** Simulate peak dinner-time traffic (e.g. 500 concurrent users browsing slots and booking) with k6 or Artillery on staging. Target p95 API latency < 300 ms.
