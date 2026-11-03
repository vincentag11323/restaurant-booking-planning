# Product Requirement Document (PRD) — v1 Scope
**ResortOS AI — Hospitality Enterprise Platform**  
**Version:** 1.1 (scoped from `PRD_BOOKING.md` v1.0)  
**Role Focus:** User (Diner App), Admin (Booking Logs Only)  
**Tech Stack:** 100% TypeScript | Express.js API | React Frontend (Decoupled 2 Repositories)

> This document is the **build contract for v1**. `PRD_BOOKING.md` remains the full product vision. If the two conflict, this document wins for v1. Concrete numbers and rules live in `technical_decisions.md`.

---

## 1. Product Summary
ResortOS AI lets restaurant guests (Users) browse the menu, reserve tables and manage their reservations at two locations (`BRANCH_1`, `BRANCH_2`). Restaurant administrators (Admin) get a **read-only** dashboard and booking log across both branches.

---

## 2. Objectives
* **Streamline Table Reservations:** Online booking in fixed 2.5-hour slots, with capacity enforced automatically.
* **Automate Table Allocation:** Main Hall joins two-seat tables (⌈party ÷ 2⌉). VIP Wing merges 1–4 private rooms.
* **Auditability:** Read-only admin visibility into all bookings, plus an immutable audit trail of critical actions.
* **KISS:** Two small decoupled repos, one test runner (Playwright), no infrastructure beyond Node + PostgreSQL.

---

## 3. Scope Boundaries

### 3.1 ✅ IN SCOPE — We ARE building in v1

| Area | Feature | Notes |
|---|---|---|
| **Auth** | Guest registration & login (email + password) | bcrypt hashing, 15-min JWT access token + 7-day refresh cookie |
| | Admin login | Admin accounts are created by seed script only |
| | Logout, token refresh | |
| **Profile** | View/edit name, phone, allergy list | |
| **Booking** | Select branch, sector, date, fixed 2.5-h slot, party size (1–50), special requests | Slots: 11:00, 13:30, 17:00, 19:30 local time |
| | Slot availability view (free tables/rooms per slot) | |
| | Automatic table allocation | Main Hall: ⌈party ÷ 2⌉ of 50 tables. VIP: ⌈party ÷ 13⌉ of 4 rooms |
| | Double-booking prevention | DB unique constraint on (table, slot) |
| | Short human-readable reservation code | e.g. `RSV-7K3Q9X` |
| **Manage** | List my bookings (upcoming + past) | |
| | Reschedule (date/slot/party size) with capacity re-check | Only `CONFIRMED`, before the slot starts |
| | Soft-cancel | Status → `CANCELLED`, tables released, record kept |
| **Menu** | Browse menu (Seasonal Fine Dining / À La Carte) | Seeded items: Chicken Steak, Beef Steak, Barramundi, Fish & Chips, Burgers… |
| | Allergy-safe recommendations | Rule-based: hide items with the guest's allergens, chef's picks first |
| **Notifications** | Booking confirmation | In-app confirmation screen + server-side `notify()` log stub |
| **Admin (read-only)** | Dashboard stats for a date: bookings, covers, cancellations, sector utilization | |
| | Booking log table with filters (branch, date range, status, sector) and search (name, email, phone, reservation code) | Paginated |
| | Detail drawer | |
| | Export filtered log to CSV / JSON | Audited. A >1,000-row export raises an alert |
| **Security & Audit** | RBAC (USER / ADMIN), IDOR protection (404 on others' records), rate-limited auth, Zod validation, helmet, CORS | See `security_spec.md` |
| | Immutable `AuditLog` table + failed-login & bulk-export alerts (logged) | See `audit_trail.md` |
| **App shell** | Mobile-first installable PWA: web manifest, home-screen icons, cached app shell, clear offline message | Single-column phone layout with bottom tab bar. API data is always fetched live, never cached (added 2026-11-03) |
| **Quality** | Playwright API suite (backend) + Playwright UI suite with mocked API (frontend) + unit tests of slot/allocation logic | See `testing_plan.md` |

### 3.2 ❌ OUT OF SCOPE — We are NOT building in v1

| Feature (from full PRD or common asks) | Why deferred | Earliest |
|---|---|---|
| Passcode / OTP / social login | Email + password is enough to launch | v2 |
| Admin MFA | Admin is read-only. Add MFA before any admin write features | v2 |
| **AI/ML recommendations** | No order history to learn from yet. Rule-based filter covers allergy safety | v2+ |
| Spending history overview | There is no order/POS data source | v2 (with POS) |
| Reminder notifications (scheduled), email/WhatsApp/push delivery | Needs a job scheduler + provider accounts. `notify()` stub is the hook point | v2 |
| Host-stand actions: check-in, `SEATED`, `COMPLETED`, `NO_SHOW` updates | Admin is read-only by PRD. Statuses exist in the schema and in the admin filters; nothing sets them yet | v2 |
| Any admin create/edit/delete of bookings, tables, menu or users | PRD: Admin = read-only | v2 |
| Flexible start times / custom durations | Fixed slots keep overlap logic trivial | v3 |
| Table map / manual table picking by guest | Auto-allocation only | v3 |
| Payments, deposits, no-show fees | No PCI scope in v1 | v3 |
| Waitlist, walk-ins, online ordering | | Later |
| Multi-language UI, native mobile apps, offline booking or offline data | English PWA only. Availability must be live, so nothing is booked or shown from cache | Later |
| More than 2 branches / branch management UI | Branches are a fixed enum | Later |
| Account self-deletion & data export UI (UU PDP) | Handled manually on request in v1 | v2 |
| Real-time push updates (WebSockets) on admin log | Manual refresh / refetch is enough | Later |
| CI/CD pipelines, hosting/infra setup | Repos are runnable locally. Deployment comes after v1 code is accepted | Post-v1 |

---

## 4. Non-Functional & Architecture Requirements

### 4.1 System Structure (2 Loose Repositories)
1. **Backend Repository (`resortos-booking-api`):**
   * Express.js 5, 100% TypeScript.
   * PostgreSQL 17 managed via Prisma ORM 7 (pinned 7.10.1).
   * Zod schema validation for all HTTP input.
2. **Frontend Repository (`resortos-booking-web`):**
   * React (Vite) SPA, 100% TypeScript.
   * Houses both the User App view and the Admin Log Viewer (role-based routes).
   * Tailwind CSS.

### 4.2 API Specifications & Error Handling
* Error body is always `{ "message": string }`.
  * `400 Bad Request`: invalid payload, capacity overflow (`Insufficient capacity for the selected slot`), table overlap race (`Table overlap conflict, please try again`), invalid state (e.g. cancelling a cancelled booking).
  * `401 Unauthorized` / `403 Forbidden`: missing/invalid token, wrong role.
  * `404 Not Found`: reservation does not exist **or** belongs to another guest.
  * `200 OK` / `201 Created`: success.
* Full endpoint list: `api_spec.md`.

### 4.3 Quality Assurance & Playwright Testing
* **API Suite:** full CRUD cycle, 2.5-hour slot computation, capacity/allocation, 400/401/403/404 triggers, IDOR.
* **Frontend UI Suite:** reservation flow, booking list (reschedule/cancel), form validations, read-only admin filters & drawer.

---

## 5. v1 Definition of Done
1. Every ✅ item in §3.1 is implemented and covered by at least one Playwright test.
2. `npm test` passes in both repos. `tsc --noEmit` has zero errors. No `.js` source files.
3. A fresh clone runs with: set `.env` → `npm install` → `npm run db:migrate && npm run db:seed` → `npm run dev`.
4. No ❌ item from §3.2 has been partially built.
