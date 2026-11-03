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

---

## 12. execSync Inside Playwright globalSetup Deadlocks In-Process PGlite

**Rule:** Don't use execSync (blocking I/O) in Playwright's globalSetup when an in-process PGlite server is running.

**Why:**
- execSync blocks the Node event loop
- PGlite server (running in the same process) can't service requests while the event loop is blocked
- Hangs indefinitely or times out

**How to apply:**
- Replace execSync with async exec in Playwright globalSetup/globalTeardown
- Use `const { execFile } = require('child_process'); await execFile('...')`
- Or use direct async libraries (e.g., PGlite's own async API if available)
- Test with in-process servers: blocking calls = bad

---

## 13. Tailwind v4: Can't @apply Custom Component Classes

**Rule:** Tailwind v4 doesn't support `@apply` on custom component classes (`.btn-primary`, etc.).

**Why:**
- `@apply` in v4 is restricted to built-in utilities
- Custom component classes defined in the same layer don't re-export

**How to apply:**
- Share base styles via a grouped selector: `.btn-primary, .btn-secondary { @apply py-2 px-4 rounded; }`
- Or use `@utility` (v4 API) for custom utilities that compose
- Or define shared Tailwind config values and apply them individually per class
- Plan component classes early; refactor to grouped selectors if @apply fails

---

## 14. Playwright getByRole Name Matching: Substring by Default; Beware Collisions

**Rule:** Playwright's getByRole name matching is substring by default; a button with aria-label "Reserve VIP table" will match getByRole('button', { name: 'table' }).

**Why:**
- Substring matching is the default; exact match is not
- Aria-labels containing codes (reservation codes, table names) can collide with other role names
- Tests pass unexpectedly, then fail when data changes

**How to apply:**
- Use `exact: true` when name matching should be precise
- Prefer explicit labels or test ids for cells/codes
- Test with real data that includes edge cases (codes that are substrings of other text)
- If getByRole is slow to debug, target the button directly by testid

---

## 15. Pre-Existing WIP Code Breaks TDD: Acknowledge it in Reports

**Rule:** If implementation code exists before tests are written, say so explicitly. Don't claim tests are "red-first" when they go green immediately.

**Why:**
- TDD's power is test-driven design: red → green → refactor
- If code is already written (WIP), tests go green immediately, breaking the feedback loop
- This hides design problems the red phase would have surfaced
- Future readers assume tests were red-first and guided design (they didn't)

**How to apply:**
- When authoring tests, check if implementation is already there
- If WIP exists: note "tests written against existing implementation; not true TDD" in the commit message or report
- Next time, write tests first, even if it means throwing away WIP
- Use `--grep` to run only new tests initially, verify they'd fail on blank code

---

## 16. Serif Old-Style Numerals: Set Numbers in Sans Face for Clarity

**Rule:** When using a serif font with old-style numerals (Cormorant Garamond), set numeric content (codes, times, counts) in a sans-serif font.

**Why:**
- Old-style numerals (common in serif fonts) make "0" look like "O", "6" like "G"
- Reservation codes (RSV-7K3Q9X), counts, times must be unambiguous at a glance
- Serifs look elegant but sacrifice legibility on small screens or in table cells

**How to apply:**
- Use Cormorant Garamond (serif) for headings and body text
- Switch to Source Sans 3 (sans) for: reservation codes, counts, times, prices, any numeric label
- CSS: `.code, .count, .time { font-family: 'Source Sans 3'; }`
- Test on mobile and in low-contrast contexts

---

## 17. Mobile-Only Test Failure Exposes Real UX Bugs

**Rule:** Run UI tests on both desktop and phone viewports. Desktop-only testing misses mobile layout bugs.

**Why:**
- Wide tables on mobile cause overflow
- Action buttons outside the viewport can't be tapped
- CSS breakpoints may be incorrect or untested
- Desktop tests pass; mobile users hit the bug

**How to apply:**
- Run Playwright tests on at least 2 viewports: desktop (1280px) and phone (Pixel 7, ~412px)
- Include "scroll to element" before interactions on mobile
- Test forms, tables, drawers at phone width
- If a test fails on mobile only, it's a real bug; prioritize it

**Session:** [[Session Log/2026-11-03#Build Phase]]

---

## 18. Embedded Single-Connection DBs Cannot Reproduce Concurrency Races

**Rule:** PGlite (single in-process connection) serializes all queries. If a fix targets a concurrency race, reason about it plainly and note that the test cannot reproduce it.

**Why:**
- PGlite runs in the Node process, serves one connection at a time
- Real Postgres with multiple concurrent clients can hit races (e.g., reschedule vs. cancel)
- A regression test passes even before the fix because the race never executes
- Claiming "test passes" implies coverage; being honest preserves credibility

**How to apply:**
- When fixing a race condition (row locks, transaction isolation): explain the fix (claimConfirmed() conditional UPDATE inside transaction)
- Explicitly say "PGlite serializes; test cannot reproduce race; fix is reasoned from concurrency rules"
- Mark such fixes as unverified-by-concurrent-test in commit messages or code review
- If possible, note where the race *would* happen on real Postgres (reschedule UPDATE before acquiring lock on CONFIRMED row)

**Session:** [[Session Log/2026-11-03#Review Round and Finish]]

---

## 19. Review Agents Can Overstate Severity; Verify Against Actual Backend Behavior

**Rule:** Code review agents may flag findings as CRITICAL that don't match the actual system behavior. Verify each rating.

**Why:**
- Agent flagged "concurrent refresh logs user out" as CRITICAL
- Actual behavior: backend refresh tokens are stateless JWTs; both concurrent refreshes succeed
- Refresh was already de-duplicated (shared in-flight promise); no logout
- False CRITICAL causes unnecessary thrashing

**How to apply:**
- When an agent flags CRITICAL/HIGH: stop and verify the backend behavior
- Ask "does this actually break the user's session / cause data loss / allow injection?"
- Check token handling (JWT stateless vs. session-based), race conditions (deduplicated promises?), error paths
- Downgrade findings if the behavior doesn't match the severity claim
- Trust reviewer reasoning, but verify impact independently

**Session:** [[Session Log/2026-11-03#Review Round and Finish]]

---

## 20. Any Effect that Refetches on Changing Inputs Needs a Stale-Response Guard

**Rule:** React effects that refetch data when inputs change must guard against stale responses overwriting newer results.

**Why:**
- User changes filter; old request completes after new request
- Older response (slower, stale data) overwrites newer response in state
- Observed in admin log, admin day view, menu when filters/tabs changed quickly

**How to apply:**
- Add a "current" guard: track the current request ID / timestamp
- In the effect callback, compare the response ID to the current ID before setState
- Or use AbortController to cancel previous requests
- Pattern: `useEffect(() => { let current = true; fetch(...).then(data => { if (current) setState(data); }); return () => { current = false; }; }, [filter])`
- Test with slow network (throttle in DevTools) and rapid filter changes

**Session:** [[Session Log/2026-11-03#Review Round and Finish]]

---

## 21. A 100% Branch-Coverage Push Surfaces Dead Defensive Code

**Rule:** When enforcing 100% branch coverage (not just statement coverage), you'll find branches that can never execute. Remove the code; don't add ignore comments.

**Why:**
- Defensive code with early returns or fallbacks often never triggers in practice
- Example: admin stats "zero capacity" fallback when all sectors are checked
- Coverage highlight makes dead code visible; it's now or never to clean it up
- Code that doesn't execute is code that won't be maintained; it rots
- Ignore comments hide the problem; refactoring fixes it

**How to apply:**
- Run 100% coverage and read the uncovered branches carefully
- Ask: "could this line ever run given the preconditions?"
- If no: remove the branch and simplify the logic
- Example: instead of `if (!sectors.length) return 0; return sectors.reduce(...)`, build from sectors directly
- Result: code is shorter, simpler, and provably correct

**Session:** [[Session Log/2026-11-03#Test Gap Pass + Jest]]

---

## 22. Check Third-Party Middleware Default Responses Against the API's Error Contract

**Rule:** Middleware like express-rate-limit send their own error responses. Verify they match your API's response schema before relying on them.

**Why:**
- express-rate-limit (by default) sends a plain-text 429 response
- API spec requires JSON with {message} schema
- Tests fail because response.json() throws on text content
- Easy to miss: middleware is added, works, but violates the contract

**How to apply:**
- When adding rate-limiting, CORS, auth, or other middleware: check its docs for response format
- Write a test that expects 429 (or your threshold) and parse the response as JSON
- If middleware sends plain text, configure it to send JSON (express-rate-limit has a `message` option) or wrap it
- Example fix: `express-rate-limit({ message: JSON.stringify({ message: 'Rate limit exceeded' }) })`
- Automate this in tests: every error path should return valid JSON

**Session:** [[Session Log/2026-11-03#Test Gap Pass + Jest]]

---

## 23. Service Workers Only Exist in Production Builds; Test Against vite build + preview

**Rule:** Service workers (from vite-plugin-pwa) don't run in Vite dev mode. To test PWA behavior, build production (vite build), preview it (vite preview), and block service workers in non-PWA tests.

**Why:**
- Dev mode skips service worker registration and precaching
- PWA behavior (offline, manifest, icon generation, update checks) only happens in production
- If you test against dev mode, mocked API calls see requests that would be cached in production
- page.route mocks may not match real request patterns

**How to apply:**
- Create a separate pwa.spec.ts that tests against vite preview (manifest, icons, registration, offline startup)
- In all other tests, block service workers: `await page.context().clearCookies(); await page.unroute('**/*');` or use `--disable-sw` flag
- Verify precache size after each font/asset add (check .vite/manifest.json or Lighthouse audit)
- Don't assume dev-mode behavior matches production; always verify against build output

**Session:** [[Session Log/2026-11-03#Mobile-Only PWA]]

---

## 24. Native <dialog> Focuses First Focusable Element; Autofocus the Title Instead

**Rule:** HTML `<dialog>` elements automatically focus the first focusable element (often a close/submit button). In a scrollable dialog, this scrolls to the bottom. Autofocus the title or a neutral element instead.

**Why:**
- Bottom-sheet dialogs (the admin booking detail especially) are long and scroll to the bottom
- Browser focuses the Close button (at the bottom) on open
- User sees the bottom of the form, not the title or first input
- Breaks the UX expectation: dialog opens at the top

**How to apply:**
- Add `autoFocus` to the sheet title or a neutral container, not action buttons
- Pattern: `<dialog><div autoFocus tabIndex={-1}>Sheet Title</div> ... <button>Close</button></dialog>`
- Or wrap the focusable title in a `<h1>` with `autoFocus`
- Test on a real device or with Playwright `page.goto(...); expect(page.locator('h1')).toBeFocused()`
- Note: focusing an unfocusable element (tabIndex={-1}) still scrolls to it

**Session:** [[Session Log/2026-11-03#Mobile-Only PWA]]

---

## 25. Check Precache Size After Adding a PWA; Font Packages Ship Many Subsets

**Rule:** When implementing PWA with vite-plugin-pwa, measure precache size and trim unnecessary assets (especially font subsets).

**Why:**
- Font packages (Google Fonts, Cormorant Garamond, Source Sans 3) include many subsets: Latin, Cyrillic, Greek, Arabic, etc.
- Precaching all subsets bloats the service worker and app shell
- Example: fonts alone were 872 KiB; trimmed to Latin woff2 = 366 KiB total shell
- Users on slow connections wait for precache download before app is usable

**How to apply:**
- Build the PWA: `vite build`
- Check `.vite/manifest.json` and service worker output for asset sizes
- Use Lighthouse DevTools audit to see precache contents
- Import only necessary font subsets: `@import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;700&display=swap&subset=latin');`
- Use `woff2` (smaller than woff, supported in all modern browsers)
- Re-check after every significant asset add (icons, new font, large image)

**Session:** [[Session Log/2026-11-03#Mobile-Only PWA]]
