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
