---
tags: learnings, feedback, working-style
date: 2026-11-03
---

# Learnings

Lessons about how to work with this user and project.

---

## 1. Plan and Validate Design Before Coding

**Rule:** Don't start coding until the user approves design/DB review.

**Why:**
- User interrupted mid-route-writing with "DONT START CODING YET, REVIEW THE DATABASE FIRST!"
- Prevents rework if design changes post-coding
- User wants visibility into decisions before implementation begins
- Wasted effort on wrong code damages trust

**How to apply:**
- When starting a feature: (1) create or present design docs, (2) wait for explicit approval, (3) *then* code
- Always ask "should I code now?" after design review if there's doubt
- Commit the *design* (requirements, schema, API contract) first, *then* start writing routes/components

---

## 2. Commit as You Go, Small Commits

**Rule:** Frequent, small, logical commits with clear messages. Don't accumulate 10 files in one commit.

**Why:**
- User explicitly said "commit as you go"
- Small commits are easier to review and revert
- Helps track progress and intent
- Atomic commits are safer for rollback

**How to apply:**
- After each logical step (e.g., "add database schema", "implement POST /reservations", "add zod validation"), commit
- Use conventional commit style (feat:, fix:, docs:, refactor:)
- Keep commit messages to 1–2 sentences, focused on *why* not *what*
- Don't batch scaffolding, schema, and routes in one commit

---

## 3. KEEP IT SIMPLE is a Hard Requirement

**Rule:** Simplicity is not optional; it's a constraint on all decisions.

**Why:**
- User explicitly stated "KEEP IT SIMPLE" as a requirement
- Affects database choice (PostgreSQL for simplicity, not MongoDB)
- Affects framework choice (Express, not Next.js or Fastify)
- Affects when to add features (v1 scope is tight)
- Violating this will trigger pushback

**How to apply:**
- When proposing a tool/pattern/library, justify why it doesn't add unnecessary complexity
- Lean toward fewer dependencies, not more
- Prefer built-in features (Express middleware) over third-party libraries
- Ask "is this necessary for v1?" before adding anything
- Challenge your own instinct to over-engineer

---

## 4. Ask and Justify Tool Choices

**Rule:** Don't assume a tool choice. If the user is unfamiliar, explain and let them challenge.

**Why:**
- User asked "why Prisma vs just Zod?" — they want to understand every dependency
- Shows you respect their architecture vision
- Catches over-engineering early
- Builds confidence in your decisions

**How to apply:**
- When choosing a library/framework, write 1–2 sentences on why it's the right fit
- Expect pushback on non-obvious choices (Prisma, JWT library, etc.)
- Be ready to explain the alternative and why it's worse
- If they challenge a choice, be open to switching (they did: decided to keep Prisma after explanation)

---

## 5. User Wants Original Docs Untouched; Create New Ones

**Rule:** When asked to adapt hospital docs to restaurant context, create *new* files in English, don't overwrite originals.

**Why:**
- Original PRD_BOOKING.md and Bahasa docs are reference material
- User wants to compare originals vs. adaptations
- Prevents accidental loss of source material
- Clearer git history (new files = new work, not edits)

**How to apply:**
- When adapting existing docs: `cp` to a new name, then edit
- Create new requirements/ files in English alongside originals
- Leave all original Bahasa files untouched
- Mention in commit message that these are new English adaptations

---

## 6. Validate Assumptions by Running Specific Checks

**Rule:** Before claiming something is impossible (e.g., "Postgres can't be installed locally"), verify the actual constraint.

**Why:**
- User said "local machine has no Docker and no Postgres installed" — but that doesn't mean it can't run Postgres via Homebrew
- An assumption ("no Docker") doesn't prevent other options
- Avoid creating blockers that don't exist

**How to apply:**
- If a task seems blocked, ask what the actual constraint is
- List alternatives (Homebrew, Neon, Supabase, Docker Desktop if it can be installed)
- Don't assume one blocker eliminates all paths forward
- Keep the project moving by identifying what *is* possible

---

## 7. Create AGENTS_TO_BE_USED.md When Requested

**Rule:** User wants a skill/agent checklist to stay aware of what tools are in the toolkit.

**Why:**
- User explicitly requested: "create AGENTS_TO_BE_USED.md listing skills"
- Helps them understand which agents to invoke for specific tasks
- Maintains visibility into the Claude Code harness capabilities
- Lists both installed and unavailable skills (with notes)

**How to apply:**
- Create a skills matrix: skill name, description, status (✓ installed or ✗ not found), mapped ecc equivalent
- Group by area (TDD, code review, simplify, spec-driven, etc.)
- Link to skill documentation or /help
- Update whenever a new skill is used or discovered

---

## 8. Maintain Obsidian Log via Haiku Agent

**Rule:** Don't maintain this log manually; create a Haiku agent task that keeps it updated.

**Why:**
- User requested: "maintain this Obsidian log via a Haiku agent"
- Reduces manual overhead for future sessions
- Haiku can parse session facts and auto-fill decisions/learnings
- Keeps log current without blocking the main agent

**How to apply:**
- Create a background agent task (loop or recurring schedule) that:
  - Parses git commits and pull requests
  - Extracts decisions from git messages and code reviews
  - Adds new session log entries at end of day
  - Flags open questions and closes resolved ones
- Agent should write directly to obsidian-log/ files (append to session log, update decisions/learnings)

---

## 9. Root Planning Repo vs. Backend Docs — Clarify Early

**Rule:** Confirm whether docs belong in root repo or inside backend repo.

**Why:**
- Created a third repo (root booking-restaurant-v1) for planning/docs
- Could have put docs inside backend repo instead
- Affects how future contributors find docs
- Affects git organization

**How to apply:**
- Ask user to clarify on next session (see [[Open Questions#Root Repo vs Backend Docs]])
- If docs should be inside backend: move obsidian-log/ and requirements/ into backend repo
- If root repo is correct: document why (e.g., "docs are shared between backend and frontend")

---

## 10. User Expects Honesty About Unknowns

**Rule:** Say "I don't know" and research rather than guessing.

**Why:**
- Skills check revealed antislop and grill-me-with-docs aren't installed
- Rather than guess where they come from, the log flags this as an open question
- User will respect "I'll look that up" more than a wrong answer
- Keeps the working relationship transparent

**How to apply:**
- When unsure about a skill/library/pattern, explicitly note it's an open question
- Offer to investigate next session
- Ask the user directly if needed: "Is antislop available in your harness?"
- Build credibility through honesty, not confidence in uncertain answers

---

## 11. Only Log Facts the User or Main Agent Provided; Don't Speculate

**Rule:** Don't invent open questions if the PRD or user already resolved them.

**Why:**
- Earlier log listed "Frontend Framework Choice" and "v1 Feature Scope" as open
- But the original PRD fixed React 19 + Vite + Tailwind, and user explicitly asked for v1 IN/OUT in PRD_BOOKING_v1.md
- Speculating creates false blockers, clogs the open questions list, and suggests those items need discussion when they don't
- Erodes credibility: "I know React was already decided, but I'll ask anyway?"

**How to apply:**
- Before listing something as open: cross-check the PRD, user messages, and coordinator updates
- If the PRD fixes it (React, v1 scope, Prisma), it's not open — record it as a decision
- If the user or coordinator decided it, it's done — update the decision, not the open list
- Only list items that genuinely need user input, not re-litigation of closed decisions
- When in doubt, ask: "Is this still open, or was it decided?"
