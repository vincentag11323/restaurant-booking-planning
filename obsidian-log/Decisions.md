---
tags: decisions, architecture
date: 2026-11-03
---

# Decisions

## PostgreSQL as Primary Database

**Decision:** Choose PostgreSQL over SQLite, MySQL, and MongoDB.

**Why:**
- **Double-booking prevention at DB level:** UNIQUE(tableId, slotStart) constraint enforced by Postgres prevents concurrent double-booking without application logic
- **Immutable audit log:** Database trigger can enforce immutable audit records (UPDATE/DELETE revoked on AuditLog table)
- **Transactions:** Full ACID guarantees for multi-step operations
- **Free hosting:** Neon and Supabase offer free PostgreSQL tiers with good DX
- **Simplicity:** Single source of truth for business rules; no need for application-level locking

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Database Review]]

**Related:**
- Schema uses `@db.Timestamptz` for all timestamps (portable across timezones)
- Short human reservation code (RSV-XXXXXX) added to AuditLog for customer-facing references
- Fixed reservation slots → unique constraint sufficient; flexible slots would need `tstzrange` exclusion constraint

---

## Keep Prisma (Don't Use Raw pg)

**Decision:** Continue using Prisma ORM rather than replacing it with Zod + raw pg.

**Why:**
- **Separation of concerns:** Zod handles runtime input validation; Prisma handles DB access and migrations
- **Schema migrations:** Prisma migrations provide version control and rollback for schema changes
- **Type safety:** Prisma generates TypeScript types from the Prisma schema
- **Maintainability:** Moving to raw pg would introduce SQL strings and manual type definitions, increasing bugs
- **KISS principle:** Prisma + Zod is the simpler, proven stack for TypeScript/Node

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Prisma vs. Zod Question]]

**User challenged this** (expected, healthy friction) but decided to keep Prisma after explanation.

---

## TypeScript 100% Across Both Repos

**Decision:** Entire backend and frontend in TypeScript (no JavaScript, no mixed lang).

**Why:**
- Type safety at compile time
- Shared types between frontend and backend (via shared package or code generation)
- Easier refactoring at scale
- User requirement

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Initial Request]]

---

## Express 5 (Not Next.js, Fastify, or Hono)

**Decision:** Use Express 5 for backend REST API.

**Why:**
- KISS — most developers know Express
- Mature ecosystem
- No opinionated framework lock-in (easy to add/remove middleware)
- Sufficient for v1 scope

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Backend Scaffolding Began]]

---

## Prisma 7.10.1 (Pinned, Not Latest)

**Decision:** Pin Prisma to 7.10.1 rather than using `latest` (8.0 RC).

**Why:**
- npm 'latest' tag currently points to Prisma 8.0 release candidate
- RC versions have breaking changes and instability
- 7.10.1 is stable and battle-tested

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Backend Scaffolding Began]]

---

## Don't Code Before Design Approval

**Decision:** Pause coding until user approves DB review and remaining design docs.

**Why:**
- User interrupt: "DONT START CODING YET, REVIEW THE DATABASE FIRST!"
- Prevents wasted effort on routes that may change post-review
- Maintains user trust (they want review → plan → code, not code → review → rework)
- Aligns with KISS: validate design before implementation

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Critical Interrupt Code Before Design Review]]

**Lesson:** [[Learnings#1. Plan and Validate Design Before Coding]]

---

## v1 Scope Boundaries Defined

**Decision:** v1 IN/OUT scope codified in requirements/PRD_BOOKING_v1.md §3.

**IN (built for v1):**
- Email/password authentication
- Booking with fixed time slots
- Reschedule reservation
- Soft-cancel (mark as no-show, don't delete)
- Rule-based allergy recommendations (simple heuristic, not ML)
- Read-only admin dashboard
- View/export reservation log
- Full audit trail

**OUT (v2+ or later):**
- AI/ML recommendations (personalized, model-based)
- Spending history / customer loyalty
- Reminders (email, SMS, WhatsApp)
- Passcode auth / MFA
- Host stand-by status updates
- Admin write access (modify/delete from UI)
- Flexible time slots (current: fixed 1-hour slots)
- Payments / billing integration
- CI/CD pipeline

**Why:** KISS principle; concentrate on core happy path (reserve, reschedule, cancel, audit). Extended features create scope creep and complexity.

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Planning Docs Completed]]

---

## Frontend Framework = React 19 + Vite + Tailwind

**Decision:** Frontend uses React 19 + Vite + Tailwind CSS.

**Why:**
- Stated in original PRD — was never open for negotiation
- React 19 stable, widely known, large ecosystem
- Vite for fast dev and build
- Tailwind for utility-first styling (aligns with KISS)

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Planning Docs Completed]]

**Note:** This was assumed open in earlier Open Questions but PRD fixed it; lesson recorded in [[Learnings#11]].

---

## API Unversioned (Proposed Default, Awaiting Confirmation)

**Proposed:** API routes live under `/api/` (no `/api/v1/`) in v1.

**Why:**
- KISS — no versioning overhead for v1
- Assumes v1 is short-lived or clients update quickly
- Can be split into `/api/v1/` later without breaking code (redirect middleware)

**Status:** Proposed by Claude; user has NOT confirmed. May be overridden if user expects API stability.

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Planning Docs Completed]]

**Related:** [[Open Questions#API Versioning and Type Sharing]]

---

## TypeScript Type Sharing = Hand-Written (Proposed Default, Awaiting Confirmation)

**Proposed:** Frontend hand-writes TypeScript types from api_spec.md (no shared npm package).

**Why:**
- KISS — no monorepo or package publishing setup
- api_spec.md is source of truth (structured Markdown)
- Frontend types auto-sync with spec when spec changes
- Keeps repos decoupled

**Status:** Proposed by Claude; user has NOT confirmed. Alternative: shared @resortos/types package if user prefers DRY.

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Planning Docs Completed]]

**Related:** [[Open Questions#API Versioning and Type Sharing]]

---

## PGlite for Local Dev + Tests; Real Postgres for Production

**Decision:** Use PGlite (embedded Postgres via @electric-sql/pglite-socket) for dev/test. Production uses real PostgreSQL 17.

**Why:**
- **Zero install:** npm test runs immediately without Docker or Postgres install
- **Fast local feedback:** Tests don't need external database setup
- **Verified:** P2002 unique violations, transaction rollback, AuditLog immutability trigger all tested against PGlite
- **Production safety:** Real Postgres 17 ensures no dev/prod parity surprises
- **Simplicity:** Single dependency; no Docker/Homebrew decisions needed

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]

**Related:** Resolves [[Open Questions#Local Postgres Method|Local Postgres Method]] — Docker/brew still optional; PGlite is the standard.

---

## Admin Booking Log: Reservation Code as Drawer Trigger

**Decision:** Reservation code in first column of admin booking log doubles as the drawer trigger; no separate action column.

**Why:**
- Keeps critical info (code) always visible, even on phones
- Reduces horizontal scrolling on mobile
- Reservation code is the unique identifier customers use; making it the affordance is intuitive
- Saves space (one column, not two)

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]

---

## Light Theme Only; Light Blue Brand Ground

**Decision:** Application is light theme only. Light blue (#E4EFF7) is the ground color and brand anchor.

**Why:**
- User specified warm fine dining on friendly light blue background — this is the brand
- v1 scope doesn't include dark mode
- Light blue sets the emotional tone; extending to dark would dilute brand
- antislop R-21 reason recorded in DESIGN.md

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]
