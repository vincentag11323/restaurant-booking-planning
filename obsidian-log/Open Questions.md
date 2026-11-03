---
tags: open-questions, blockers, unresolved
date: 2026-11-03
---

# Open Questions

Unresolved items that need user input or research before proceeding.

---

## Local Postgres Method

**Question:** How should Postgres be installed and run locally?

**Context:**
- Local machine has Node 25, npm 11, Homebrew
- No Docker installed
- No Postgres installed
- Need local dev environment to run API and tests

**Options:**
1. **Homebrew:** `brew install postgresql` — native, no Docker overhead
2. **Neon:** Free cloud Postgres, connect via env var (no local infrastructure)
3. **Supabase:** Free cloud Postgres with UI dashboard
4. **Docker Desktop:** Install Docker first (requires macOS/system changes), then run Postgres in container

**Blocker for:** Writing integration tests, running API locally, seeding database

**Decision needed:** User preference on simplicity/preference tradeoff

**Session:** [[Session Log/2026-11-03#Local Database Setup Question]]

---

## Skill Availability: antislop & grill-me-with-docs

**Question:** Where do `antislop` and `grill-me-with-docs` skills come from?

**Context:**
- User requested: "create AGENTS_TO_BE_USED.md listing skills: antislop, grill-me-with-docs, ..."
- Skills audit showed both are not installed in ecc:/
- Other skills map cleanly to ecc equivalents (tdd-workflow, simplify, etc.)
- These two may be from a custom harness, external package, or different skill namespace

**Known:**
- ✗ ecc:antislop — not found
- ✗ ecc:grill-me-with-docs — not found

**Possibilities:**
- Custom skills in user's ~/.claude/
- Community skills repo (not ecc:)
- Deprecated skills (renamed in latest ecc)
- Internal tools specific to user's org

**Blocker for:** Accurate AGENTS_TO_BE_USED.md file

**Decision needed:** User clarification on where to find or substitute these skills

**Session:** [[Session Log/2026-11-03#Skills Audit]]

---

## Root Planning Repo vs. Backend Docs Location

**Question:** Should docs live in root repo (booking-restaurant-v1) or inside backend repo?

**Context:**
- Created 3 repos:
  1. booking-restaurant-v1 (planning/docs) — root, ignores app repos
  2. resortos-booking-api (backend)
  3. (frontend repo not yet created)
- requirements/ and obsidian-log/ currently in root repo
- Could have been inside backend repo instead

**Options:**
1. **Root repo (current):** Docs at top level, shared reference for both backend and frontend teams
2. **Backend repo:** Docs live alongside code, easier for API developers, frontend refs upstream
3. **Submodule:** Docs as shared git submodule (adds complexity)

**Tradeoffs:**
- Root repo: centralized, but adds a third repo to manage
- Backend repo: fewer repos, but frontend might feel less ownership of docs
- Current setup assumes docs are shared platform; if just for backend, feels redundant

**Blocker for:** Repository organization clarity, contributor onboarding instructions

**Decision needed:** User preference on repo structure

**Session:** [[Session Log/2026-11-03#Git State]]

---

## API Versioning and Type Sharing (Proposed Defaults, Awaiting Confirmation)

### API Versioning

**Proposed:** API routes live under `/api/` (no `/api/v1/`) in v1.

**Why:**
- KISS — no versioning overhead for v1
- Assumes v1 is short-lived or clients update quickly
- Can be split into `/api/v1/` later without breaking code (redirect middleware)

**Status:** Proposed by Claude; user has NOT confirmed. May be overridden if user expects API stability.

**Session:** [[Session Log/2026-11-03#Planning Docs Completed]]

**Related decision:** [[Decisions#API Unversioned (Proposed Default, Awaiting Confirmation)]]

### TypeScript Type Sharing

**Proposed:** Frontend hand-writes TypeScript types from api_spec.md (no shared npm package).

**Why:**
- KISS — no monorepo or package publishing setup
- api_spec.md is source of truth (structured Markdown)
- Frontend types auto-sync with spec when spec changes
- Keeps repos decoupled

**Status:** Proposed by Claude; user has NOT confirmed. Alternative: shared @resortos/types package if user prefers DRY.

**Session:** [[Session Log/2026-11-03#Planning Docs Completed]]

**Related decision:** [[Decisions#TypeScript Type Sharing = Hand-Written (Proposed Default, Awaiting Confirmation)]]
