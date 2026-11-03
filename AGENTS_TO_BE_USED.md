# Agents & Skills To Be Used
**ResortOS AI — Restaurant Booking App**

Which agent skills are used, **when**, and **what they may not do**. Applies to both repos (`resortos-booking-api`, `resortos-booking-web`).

## 1. Install Status (updated 2026-11-03)

| Requested skill | Installed as | Notes |
|---|---|---|
| **antislop** | `antislop` (core) + `antislop-code`, `antislop-ui`, `antislop-copywriting`, `antislop-layoutmobile` | Mode: **DURING** (session choice). Delivery Gate runs before every UI hand-off |
| **grill-me-with-docs** | `/mattpocock-skills:grill-with-docs` (user-invoked only) → calls `mattpocock-skills:grilling` + `mattpocock-skills:domain-modeling` | Agent calls `grilling` / `domain-modeling` directly |
| **test-driven-development** | `mattpocock-skills:tdd` | Vertical slices, test at agreed seams (see §3.3) |
| **self-improving-agent** | `ecc:continuous-learning-v2`, `ecc:learn-eval` + Haiku Obsidian logger (§4) | |
| **code-simplify** | `/simplify`, `ecc:code-simplifier` | |
| **spec-driven-development** | `ecc:contract-first`, `ecc:intent-driven-development`, `mattpocock-skills:code-review` (Spec axis) | |

## 2. Workflow (per feature)

```
 Phase 1 SPEC        Phase 2 BUILD          Phase 3 CLEAN                Phase 4 LEARN
 grill-me-with-docs → test-driven-dev     → code-simplify → antislop   → self-improving-agent
 spec-driven-dev      (red → green)          + reviewers                  + Haiku Obsidian log
        │                    │                        │                         │
   docs updated &      tests + code            review findings fixed      lesson logged
   committed           committed               committed                  committed
```

A feature = one ✅ row of `requirements/PRD_BOOKING_v1.md` §3.1. Every phase ends with a **git commit**.

## 3. Skills — Purpose, Trigger, Boundaries

### 3.1 grill-me-with-docs — *Phase 1*
* **Purpose:** Interrogate the plan against `requirements/*` **before** any code: find ambiguities, contradictions, missing edge cases.
* **Trigger:** Start of every feature, and whenever the PRD/specs change.
* **Output:** Questions to the user. Answers are written back into the relevant spec doc and committed.
* **Boundaries:** Asks; never decides scope on its own. Never writes code.

### 3.2 spec-driven-development — *Phase 1*
* **Purpose:** Specs are the source of truth. `data_model.md` → `schema.prisma`, `api_spec.md` → routes + Zod schemas + tests, `ui_spec.md` → screens.
* **Trigger:** Before writing a route, schema or screen.
* **Boundaries:** No endpoint, column or screen that isn't in a spec. If one is needed: update the spec → commit → then code. Never build ❌ items from PRD v1 §3.2.

### 3.3 test-driven-development — *Phase 2*
* **Purpose:** Red → green with **Playwright only** (`@playwright/test`), one vertical slice at a time. Agreed seams (from `testing_plan.md`): pure slot/allocation functions, the HTTP API, and the rendered UI with a mocked API.
  * Backend: `tests/unit/*.spec.ts` (pure slot/allocation logic) → `tests/api/*.spec.ts` (real API + test DB).
  * Frontend: `tests/e2e/*.spec.ts` with the API mocked via `page.route`.
* **Trigger:** Every behaviour change.
* **Boundaries:** Write the failing test first and commit with the implementation. Tests assert exact status codes + `message` strings from `api_spec.md`. Don't weaken or delete a test to make it pass. Don't add a second test framework.

### 3.4 code-simplify — *Phase 3*
* **Purpose:** Enforce KISS after green: remove dead code, needless abstractions and duplication without changing behaviour.
* **Trigger:** After each feature is green, before commit.
* **Boundaries:** Behaviour-preserving only, and tests must stay green. Don't add new layers (no repositories/services/DI containers unless a spec demands it). Don't touch generated code (`src/generated/`).

### 3.5 antislop — *Phase 3*
* **Purpose:** Strip AI "slop" from code **and** docs: filler comments that restate code, over-hedged prose, unused helpers, generic names, speculative "future-proofing", emoji/marketing tone in code.
* **Trigger:** After code-simplify, and on every doc edit.
* **Boundaries:** Wording/cleanup only. No logic changes, no scope changes.
* **UI:** direction lives in `resortos-booking-web/DESIGN.md` (warm fine dining, light blue background). No em dashes in UI copy (R-02).

### 3.6 self-improving-agent — *Phase 4*
* **Purpose:** Capture what went wrong/right so the next session does better.
* **Trigger:** End of each feature, after any user correction, after any retracted claim.
* **Output:** A lesson in `obsidian-log/Learnings.md` (via Haiku, §4). Durable user preferences also go to Claude's memory.
* **Boundaries:** Records facts and the user's own words only. Never rewrites history or invents sentiment.

## 4. Haiku Obsidian Logger
* **Agent:** a subagent running on **Claude Haiku** (`model: haiku`), spawned by the main agent.
* **Vault:** `obsidian-log/` (open this folder as an Obsidian vault).
  * `00 Index.md` (home) · `Session Log/YYYY-MM-DD.md` (chronological) · `Decisions.md` · `Learnings.md` · `Open Questions.md`
* **When:** After every commit-worthy milestone, and at the end of each session.
* **Boundaries:** Writes **only** inside `obsidian-log/`. **Never runs git** (the main agent commits the log). Records facts given by the main agent and does not speculate.

## 5. Review Agents (installed, used in Phase 3)

| Agent | Repo | When |
|---|---|---|
| `mattpocock-skills:code-review` | both | end of each repo milestone (Standards + Spec) |
| `ecc:typescript-reviewer` | both | every feature |
| `ecc:database-reviewer` | api | schema / migration / query changes |
| `ecc:security-reviewer` | api | auth, admin, export, anything touching personal data |
| `ecc:react-reviewer` | web | every `.tsx` change |
| `ecc:a11y-architect` | web | new screens (forms, drawer, modals) |

## 6. Global Boundaries (all agents)
1. **No code before the plan/spec is approved by the user.**
2. **KISS:** smallest thing that satisfies the spec.
3. **100% TypeScript.** No `.js` source files.
4. **Commit as you go:** small conventional commits (`feat:`, `test:`, `docs:`, `refactor:`). Local only, no pushing. The user connects GitHub.
5. Original user docs (`PRD_BOOKING.md`, Bahasa files) are never edited. New docs are created instead.
6. Never build anything listed as ❌ OUT OF SCOPE in `PRD_BOOKING_v1.md`.
