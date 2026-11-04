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
**Session:** [[SESSION_LOG/2026_11_03#Database Review]]

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
**Session:** [[SESSION_LOG/2026_11_03#Prisma vs. Zod Question]]

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
**Session:** [[SESSION_LOG/2026_11_03#Initial Request]]

---

## Express 5 (Not Next.js, Fastify, or Hono)

**Decision:** Use Express 5 for backend REST API.

**Why:**
- KISS — most developers know Express
- Mature ecosystem
- No opinionated framework lock-in (easy to add/remove middleware)
- Sufficient for v1 scope

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Backend Scaffolding Began]]

---

## Prisma 7.10.1 (Pinned, Not Latest)

**Decision:** Pin Prisma to 7.10.1 rather than using `latest` (8.0 RC).

**Why:**
- npm 'latest' tag currently points to Prisma 8.0 release candidate
- RC versions have breaking changes and instability
- 7.10.1 is stable and battle-tested

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Backend Scaffolding Began]]

---

## Don't Code Before Design Approval

**Decision:** Pause coding until user approves DB review and remaining design docs.

**Why:**
- User interrupt: "DONT START CODING YET, REVIEW THE DATABASE FIRST!"
- Prevents wasted effort on routes that may change post-review
- Maintains user trust (they want review → plan → code, not code → review → rework)
- Aligns with KISS: validate design before implementation

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Critical Interrupt Code Before Design Review]]

**Lesson:** [[LEARNINGS#1. Plan and Validate Design Before Coding]]

---

## v1 Scope Boundaries Defined

**Decision:** v1 IN/OUT scope codified in requirements/PRD_BOOKING_V1.md §3.

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
**Session:** [[SESSION_LOG/2026_11_03#Planning Docs Completed]]

---

## Frontend Framework = React 19 + Vite + Tailwind

**Decision:** Frontend uses React 19 + Vite + Tailwind CSS.

**Why:**
- Stated in original PRD — was never open for negotiation
- React 19 stable, widely known, large ecosystem
- Vite for fast dev and build
- Tailwind for utility-first styling (aligns with KISS)

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Planning Docs Completed]]

**Note:** This was assumed open in earlier Open Questions but PRD fixed it; lesson recorded in [[LEARNINGS#11]].

---

## API Unversioned (Proposed Default, Awaiting Confirmation)

**Proposed:** API routes live under `/api/` (no `/api/v1/`) in v1.

**Why:**
- KISS — no versioning overhead for v1
- Assumes v1 is short-lived or clients update quickly
- Can be split into `/api/v1/` later without breaking code (redirect middleware)

**Status:** Proposed by Claude; user has NOT confirmed. May be overridden if user expects API stability.

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Planning Docs Completed]]

**Related:** [[OPEN_QUESTIONS#API Versioning and Type Sharing]]

---

## TypeScript Type Sharing = Hand-Written (Proposed Default, Awaiting Confirmation)

**Proposed:** Frontend hand-writes TypeScript types from API_SPEC.md (no shared npm package).

**Why:**
- KISS — no monorepo or package publishing setup
- API_SPEC.md is source of truth (structured Markdown)
- Frontend types auto-sync with spec when spec changes
- Keeps repos decoupled

**Status:** Proposed by Claude; user has NOT confirmed. Alternative: shared @resortos/types package if user prefers DRY.

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Planning Docs Completed]]

**Related:** [[OPEN_QUESTIONS#API Versioning and Type Sharing]]

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
**Session:** [[SESSION_LOG/2026_11_03#Build Phase]]

**Related:** Resolves [[OPEN_QUESTIONS#Local Postgres Method|Local Postgres Method]] — Docker/brew still optional; PGlite is the standard.

---

## Admin Booking Log: Reservation Code as Drawer Trigger

**Decision:** Reservation code in first column of admin booking log doubles as the drawer trigger; no separate action column.

**Why:**
- Keeps critical info (code) always visible, even on phones
- Reduces horizontal scrolling on mobile
- Reservation code is the unique identifier customers use; making it the affordance is intuitive
- Saves space (one column, not two)

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Build Phase]]

---

## Light Theme Only; Light Blue Brand Ground

**Decision:** Application is light theme only. Light blue (#E4EFF7) is the ground color and brand anchor.

**Why:**
- User specified warm fine dining on friendly light blue background — this is the brand
- v1 scope doesn't include dark mode
- Light blue sets the emotional tone; extending to dark would dilute brand
- antislop R-21 reason recorded in DESIGN.md

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Build Phase]]

---

## Backend Jest for 100% Coverage (Owner Request)

**Decision:** Add Jest + @swc/jest + supertest alongside Playwright for 100% statement/branch/function/line coverage on backend source (excluding generated code).

**Why:**
- Owner requested comprehensive unit test coverage
- Playwright tests API contracts; Jest tests internal logic, error paths, edge cases
- 100% branch coverage (not just statement) surfaces dead defensive code
- Threshold enforced in CI; no coverage-ignore comments allowed (fix the code instead)

**Stack choice: @swc/jest over ts-jest:**
- ts-jest requires TypeScript < 7; codebase is TypeScript 7
- TypeScript 7 removed the JS compiler API that ts-jest relied on
- @swc/jest uses SWC (Rust transpiler) and works with TS 7

**Coverage philosophy:**
- Unreachable code is removed (refactored), not excluded
- Example: admin stats "zero capacity" fallbacks removed instead of covered by mock
- Result: 62 Jest tests, 100% coverage metrics, codebase more maintainable

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Test Gap Pass + Jest]]

**Related:** In-memory PGlite on port 5498 for test isolation; npm test runs Playwright then Jest.

---

## Mobile-Only Frontend + Installable PWA (Owner Request)

**Decision:** Simplify frontend to mobile-first (Pixel 7, 28rem max width), remove all desktop breakpoints, and ship as an installable PWA.

**Scope:** Installable PWA + mobile layout moved into v1. Out of scope: offline booking, offline data sync (availability must be live).

**Architecture:**
- Bottom tab bar for navigation (guests: Reserve, Bookings, Menu, Profile; admins: Today, Booking log, with Sign out in the header)
- Bottom-sheet dialogs (change time, release booking, admin booking detail)
- Admin log refactored from table to card list
- Sign out in Profile (guests) / header (admins)
- Removed all Tailwind sm/md/lg breakpoints

**PWA implementation:**
- vite-plugin-pwa 2.0 with generateSW + autoUpdate
- Manifest: ResortOS Dining, standalone, theme #1e4636, background #e4eff7
- App shell precached (15 files, 366 KiB after font trimming to Latin woff2)
- /api excluded from cache; offline API errors prompt "Check your connection"
- Icons auto-generated at build from public/favicon.svg (placeholder: place-card motif until owner supplies logo)

**Testing:** Tests run against production build (vite build + preview); service workers blocked except in pwa.spec.ts (tests manifest, icons, registration, offline startup).

**Why:**
- Owner request: mobile-first reflects real user base (restaurants, phone-primary customers)
- PWA allows home-screen install and offline-partial experience (read cached data, show "offline" for live APIs)
- Offline booking/data stays out: availability changes constantly; don't cache stale tables

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Mobile-Only PWA]]

---

## Assistant RBAC: AGENT Role with Exactly Two Permissions

**Decision:** Create AGENT role with exactly two API permissions:
1. POST /api/agent/reservations (book on guest's behalf, Idempotency-Key, max 2 upcoming per email)
2. POST /api/agent/guests/lookup (exact email only, returns exists + upcomingAssistantBookings)

AGENT gets 403 everywhere else. Listing/searching guests stays admin-only.

**Why:**
- Owner rule: listing/searching guests is admin-only (privacy, scalability)
- AGENT needs to book without reading all guests (supply exact email, not search)
- Limits surface area: two endpoints, no admin data exposure, no write access to other entities
- Idempotency-Key + max 2 upcoming prevent duplicate bookings and spam

**Implementation:**
- RBAC permission table with roles USER, ADMIN, AGENT
- Password-less guest accounts per email; sign-up claims account + retains bookings
- Reservations record source (APP/AGENT) and createdById for audit trail
- Admins no longer create bookings (read-only)

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Booking Assistant Built]]

---

## Keep Cent AI Messages Format for DeepSeek; Handle Unsigned Thinking Blocks in SDK

**Decision:** Use Cent AI Messages format for DeepSeek (not Responses API) and handle unsigned thinking blocks in cent_ai_api.ts rather than switching to a different format.

**Why:**
- Cent AI Messages format uses ~3.3x fewer input tokens than Responses API
- DeepSeek via Cent AI returns thinking blocks without SDK signature
- Anthropic SDK (strict provider) rejects unsigned blocks
- Fix: drop unsigned thinking blocks in cent_ai_api.ts; regression test replays captured response
- Switching to Responses API would negate the token savings

**Implementation:**
- cent_ai_api.ts: filters out unsigned thinking blocks before passing to SDK
- Regression test: replay captured DeepSeek response with fix verified without spending calls
- No hand-written request translation (earlier code removed)

**Tradeoff:**
- Token savings (Messages) > SDK validation strictness (one-time filter fix)
- Cost-effective for high-volume agent runs

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Booking Assistant Built]]

---

## Session-Check 401 Stays Logged (Owner Decision)

**Decision:** Anonymous page-load session checks that return 401 are logged to errors.log.

**Why:**
- Missing or expired session is a real failure (user needs to sign in)
- Not noise; indicates session expired or never established
- Production monitoring can track sign-in rates via 401 log

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Docs Round and Handoff]]

**Recorded:** API_SPEC.md §8

---

## Fixed 1-Hour Cookie Session, No Refresh Token, Agent API Key (Owner Approved, Pending Implementation)

**Decision:** Simpler session design (approved, to be built in next session):
- One HttpOnly session cookie fixed at 1 hour from sign-in (not extended by activity)
- Signed out after 1 hour; no refresh token
- No PWA idle timer (screens keep data, next 401 goes to sign-in)
- AI agent uses predefined API key created at seeding (hash stored, X-Agent-Key header, role AGENT)

**Why:**
- Simpler than refresh token + idle timer
- Clear expiry prevents stale sessions in production
- API key authenticates agent without user context
- Reduces session management complexity

**Status:** APPROVED, NOT YET IMPLEMENTED

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_03#Docs Round and Handoff]]

**Recorded:** SECURITY_SPEC.md §1, DIAGRAM.md §4

---

## Spec Narrowed: Deleted Account 401 Only on GET /me (Sessions Stay Stateless)

**Decision:** During Stage 3 Review the deleted-account 401 rule in `requirements/API_SPEC.md` was narrowed to `GET /me` only (root commit ecb583c), so sessions stay stateless.

**Date:** 2026-11-03  
**Session:** [[SESSION_LOG/2026_11_04#Review Stage Complete (Stage 3)]]

---

## Review Findings Not Fixed

**Decision (orchestrator, 2026-11-03):** not fixed: trailing slash in `CORS_ORIGIN`; deleted account on `PATCH /me` (covered by the spec narrowing above); the tracked `coverage/` folder in the api repo.

**Session:** [[SESSION_LOG/2026_11_04#Review Stage Complete (Stage 3)]]

---

## Dev-Only npm Audit Findings Left Unfixed

**Decision:** `npm audit --omit=dev` is 0 in every repo. The dev-only findings (api: 19 in the Jest tooling; web: 10 in vite-plugin-pwa) are left alone because clearing them needs breaking version changes.

**Date:** 2026-11-04  
**Session:** [[SESSION_LOG/2026_11_04#Test Stage Complete (Stage 4)]]

---

## Video Files Gitignored in All Repos; .gif Excluded From Rule

**Decision:** Video files (.mp4, .webm, .mov, .avi, .mkv, .m4v) are gitignored in all five repos. .gif is deliberately excluded (often used as doc images).

**Date:** 2026-11-04  
**Session:** [[SESSION_LOG/2026_11_04#Video Files Gitignored + History Audit]]

---

## API 500 on Bad Input: Ignored

**Decision (owner, 2026-11-04):** the API (and the assistant) answering `500` instead of `400`/`413` for impossible dates, oversized bodies or garbled URLs is accepted and not fixed. No data is exposed; only the status code is wrong. See [[SECURITY_FINDINGS_2026_11_04]].
