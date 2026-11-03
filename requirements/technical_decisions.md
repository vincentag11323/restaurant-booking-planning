# Technical Decisions
**ResortOS AI — Restaurant Booking App**

This doc records the choices that turn the PRD into concrete, buildable rules. Guiding principle: **KISS**.

## 1. Database Review

| Option | Fit for table booking | Verdict |
|---|---|---|
| **PostgreSQL** | Real transactions, unique constraints that prevent double-booking at the DB level, triggers for an immutable audit log, native arrays/JSON for allergies and metadata, and free managed hosting (Neon, Supabase, Railway). | ✅ **Chosen** |
| SQLite | Zero setup, but only one writer at a time, weak multi-instance hosting story, and no triggers-per-role nuance. Fine for prototypes, risky for a live booking system. | ❌ |
| MySQL | Works, but has no array type and a weaker JSON/constraint story than Postgres. No advantage here. | ❌ |
| MongoDB / NoSQL | Bookings are relational (user → reservation → tables) and need multi-row transactional guarantees. A document DB adds work to get that. | ❌ |

**Decision:** **PostgreSQL 17 + Prisma ORM 7** (as already stated in the PRD). The core reason: a single `UNIQUE (tableId, slotStart)` constraint makes double-booking impossible even under concurrent requests, with no locks or queues to write.

Local dev: `docker compose up -d` (file included in the backend repo), `brew install postgresql@17`, or any hosted Postgres URL (e.g. Neon free tier) in `.env`.

### 1.1 ORM: Prisma vs "just Zod" (decided 2026-11-03: **keep Prisma**)
Zod and Prisma are different layers, and **both** are used:
* **Zod** validates *incoming* data at runtime (request bodies, query strings). It cannot talk to a database.
* **Prisma** is the DB layer: typed queries, transactions, generated migrations.

The real alternative was **Zod + raw SQL (`pg`)**. Prisma was kept because:
* Migrations are generated from `schema.prisma`, which doubles as living DB documentation.
* Query results are typed automatically, so a schema change breaks the build, not production.
* The admin log's optional filters are plain objects. That avoids hand-building `WHERE $1…$n` strings, the most error-prone code in this app.
* Unique-violation error code `P2002` maps cleanly to the PRD's `400 Table overlap conflict`.
* The PRD already specifies it.

Accepted costs: a `prisma generate` step and the Prisma 7 driver-adapter setup (`@prisma/adapter-pg`). Version is **pinned to 7.10.1** because npm's `latest` tag currently points to an 8.0 release candidate. Postgres-specific features (the audit-log trigger) live in a hand-edited SQL migration.

### 1.2 Schema rules (see `data_model.md`)
* All timestamps use `@db.Timestamptz` (Prisma's default `timestamp` has no time zone).
* Reservations get a short unique human code (`RSV-` + 6 chars from an unambiguous alphabet), so guests can read it out to the host. The internal `cuid` stays the primary key.
* In production, the app's DB role gets `INSERT, SELECT` only on `AuditLog`, in addition to the immutability trigger.
* The `UNIQUE (tableId, slotStart)` guard is sufficient **only because slots are fixed**. Flexible slots (out of scope) would need a `tstzrange` exclusion constraint (`btree_gist`).

## 2. Booking Rules (resolving PRD ambiguities)
* **Fixed 2.5-hour slots** (restaurant local time, `UTC+07:00` by default):
  * `11:00–13:30`, `13:30–16:00`, `17:00–19:30`, `19:30–22:00`.
  * Fixed slots mean two bookings "overlap" only when they share the same slot start, so no interval math is needed.
* **Bookable window:** from now up to 60 days ahead. A slot can't be booked or rescheduled once it has started.
* **Capacity per branch:**
  * `MAIN_HALL`: 50 two-seat tables (100 seats total). A booking takes ⌈party ÷ 2⌉ tables.
  * `VIP_WING`: 4 private rooms of 13 seats each. A booking takes ⌈party ÷ 13⌉ rooms (1–4).
* **Party size per booking:** 1–50 for both sectors. "≤ 100 pax" in the PRD is read as the hall's *total* capacity.
* **Table selection:** the free tables/rooms with the lowest numbers. This keeps joined tables next to each other.

## 3. API Error Contract
* `400` + `{ "message": "..." }` for validation errors, `Insufficient capacity for the selected slot`, and `Table overlap conflict, please try again` (a concurrent booking won the race).
* `404` for unknown reservations **and** for reservations owned by another guest (prevents IDOR probing).
* `401` / `403` for missing/invalid tokens and wrong roles. These are unavoidable with auth, so they are added to the PRD's list.

## 4. v1 Simplifications (deliberately out of scope)
| PRD item | v1 behaviour |
|---|---|
| Login with "Passcode" | Email + password only. |
| AI-driven recommendations | Rule-based: hide dishes containing the guest's allergens, show chef's picks first. Needs order history before ML is worthwhile. |
| Spending history overview | Not shown. There is no order/POS data source yet. |
| Notifications & reminders | Confirmation is shown in-app and logged server-side through a `notify()` stub, which is the single place to plug in email/WhatsApp later. No reminder scheduler. |
| `SEATED` / `COMPLETED` / `NO_SHOW` statuses | Exist in the schema and in admin filters. The v1 app only writes `CONFIRMED` / `CANCELLED`. Seed data covers the others. |
| Admin MFA | Future enhancement. |

## 5. Repositories
* `resortos-booking-api/`: Express 5 + TypeScript + Prisma + Zod + Playwright (API & unit tests).
* `resortos-booking-web/`: React 19 + Vite + TypeScript + Tailwind CSS 4 + Playwright (UI tests, API mocked).
