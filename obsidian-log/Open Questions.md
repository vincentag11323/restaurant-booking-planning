---
tags: open-questions, blockers, unresolved
date: 2026-11-03
---

# Open Questions

Unresolved items that need user input or research before proceeding.

---

## ✓ RESOLVED: Local Postgres Method

**Resolution:** PGlite (embedded Postgres via @electric-sql/pglite-socket) for dev/test; real PostgreSQL 17 for production.

**Why:**
- Zero install: npm test requires no external setup
- Fast feedback in local tests
- Verified against PGlite: P2002 unique violations, transaction rollback, AuditLog immutability trigger
- Production safety: real Postgres ensures dev/prod parity

**Date resolved:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]  
**Decision:** [[Decisions#PGlite for Local Dev + Tests; Real Postgres for Production]]

**Note:** Docker/Homebrew remain optional for users who prefer them; PGlite is the standard.

---

## ✓ RESOLVED: Skill Availability — antislop & grill-me-with-docs

**Resolution:** User installed antislop (mode DURING) and mattpocock-skills (grill-me, grill-with-docs are user-invoked only; tdd, grilling, domain-modeling, code-review usable by agent).

**Date resolved:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]

---

## ✓ RESOLVED: Root Planning Repo (3-Repo Layout)

**Resolution:** User approved 3-repo layout:
1. booking-restaurant-v1 (planning/docs) — root
2. resortos-booking-api (backend)
3. resortos-booking-web (frontend)

Docs live in root repo, shared reference for backend and frontend.

**Date resolved:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]

---

## ✓ RESOLVED: API Versioning and Type Sharing

**Resolution:** User confirmed:
- API unversioned: routes under `/api/` (no `/api/v1/`)
- Type sharing: each repo maintains its own types folder (no shared npm package)

**Date resolved:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Build Phase]]  
**Related decisions:**
- [[Decisions#API Unversioned (Proposed Default, Awaiting Confirmation)]]
- [[Decisions#TypeScript Type Sharing = Hand-Written (Proposed Default, Awaiting Confirmation)]]

---

## Simpler Sessions: Approved, Pending Implementation

**Status:** Owner approved. To be built in the next session.

**Approved design:**
- One HttpOnly session cookie fixed at 1 hour from sign-in (not extended by activity)
- Signed out after 1 hour; no refresh token
- No PWA idle timer (screens keep data, next 401 goes to sign-in)
- AI agent uses predefined API key from seeding (hash stored, X-Agent-Key header, role AGENT)

**Why approved:**
- Simpler than refresh token + idle timer
- Clear expiry prevents stale sessions
- API key authenticates agent without user context

**Recorded:** security_spec.md §1, DIAGRAM.md §4

**Date:** 2026-11-03  
**Session:** [[Session Log/2026-11-03#Docs Round and Handoff]]  
**Decision:** [[Decisions#Fixed 1-Hour Cookie Session, No Refresh Token, Agent API Key (Owner Approved, Pending Implementation)]]

---

## Show "Your Session Ended" Message on Forced Sign-Out?

**Question:** When a user's session expires or is revoked, should the app show a dismissible "Your session ended" message before redirecting to sign-in?

**Context:** Spec doesn't require this. Current behavior: next API call hits 401, user is silently redirected.

**Decision (owner, 2026-11-04):** No. The 401 already sends them to sign in; keep it simple.

**Date:** 2026-11-04  
**Session:** [[Session Log/2026-11-04]]

---

## Server-Side Session Revocation

**Question:** Currently, sessions don't support server-side revocation. If a user logs out, their session cookie remains valid until expiry (1 hour by design). If the cookie is copied/stolen, it grants access until expiry.

**Context:** Stateless session by design. Owner decision at implementation time: is silent expiry acceptable, or does logout need to revoke?

**Note:** Would require session storage (Redis, database), adding complexity and state.

**Decision (owner, 2026-11-04):** Critical; must be fixed. Plan first, explained to the owner before any code.

**Date:** 2026-11-04  
**Session:** [[Session Log/2026-11-04]]

---

## E2E Reporter Links T4's Video to Blank 1.8 KB Stub

**Question:** In RESULTS.md, T4 (the accessibility check across 7 pages) links to `videos/T4.webm`, a 1.8 KB blank stub instead of the real recording.

**Context:** T4 navigates several times, so Playwright attaches 8 videos. `reporter/results-reporter.ts` takes the first attachment named "video", which is one of the blank stubs. The real recording is in `test-results/`. T7 and S2 have a single video each and link correctly.

**Fix (if wanted):** have the reporter pick the largest or the main `video.webm` attachment.

**Date:** 2026-11-04  
**Session:** [[Session Log/2026-11-04]]

---

## Scenario ID T7 Used Twice (Collision)

**Question:** Test scenario ID "T7" is used for two different checks:
1. Session expiry in tests/app/checks.spec.ts
2. Chat-cancel refusal in tests/assistant/scope-security.spec.ts

**Context:** IDs should be unique for reporting and video linking.

**Fix (if wanted):** rename one of them to an ID not already used in the scenario docs.

**Date:** 2026-11-04  
**Session:** [[Session Log/2026-11-04]]

---

## Local .env Still Has Old JWT_*/AGENT_PASSWORD Variables

**Question:** The owner's local `.env` files still have the old `JWT_*` / `AGENT_PASSWORD` variables. Should they be updated?

**Context:** `.env.example` already describes `SESSION_SECRET` and `AGENT_API_KEY`. Test runs set their own values, but `npm run dev` in the api fails until `SESSION_SECRET` (≥ 32 characters) is added to the local `.env`.

**Decision (owner, 2026-11-04):** Done. api and agent `.env` now have a random `SESSION_SECRET` and a shared random `AGENT_API_KEY`; old `JWT_*`, `AGENT_EMAIL`, `AGENT_PASSWORD` removed; local DB migrated and seeded.

**Date:** 2026-11-04  
**Session:** [[Session Log/2026-11-04]]
