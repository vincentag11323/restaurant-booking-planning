# Rules: how every task is run

The main agent (orchestrator) reads this file **before starting each task** and before launching each stage.
Owner rule, 2026-11-03. Details on skills and reviewers: `AGENTS_TO_BE_USED.md`.

## 1. Four stages, one after another

Every feature or change goes through four subagents **in order**. A stage starts only after the orchestrator has checked the previous stage's report (§3).

Each stage follows its ECC step: **plan → tdd → code review → verify**.

| # | ECC step | Skill / agent the stage loads |
|---|---|---|
| 1 | plan | `ecc:plan` workflow (restate requirements, risks, step-by-step plan), run by the `ecc:planner` agent |
| 2 | tdd | `ecc:tdd-workflow` (red → green → refactor, coverage kept), with `mattpocock-skills:tdd` for vertical slices |
| 3 | code review | `ecc:code-review` checklist, plus the reviewer agents below |
| 4 | verify | `ecc:verification-loop` (build, typecheck, lint, tests, security scan, diff review), plus `ecc:security-review` for auth changes |

| # | Stage | Agent / model | Does | Must not |
|---|---|---|---|---|
| 1 | **Plan** | `ecc:planner`, Opus | Reads the approved spec and the real code; returns a file-by-file plan: every file to change, every test to add or change (with the exact status code and message from the spec), the order of the work and the commit messages | Write or edit any file |
| 2 | **Build** | `general-purpose`, Opus | Follows the plan test-first (red, then green), one repo at a time; runs that repo's test suites; small commits | Touch files not in the plan, weaken or delete tests, lower coverage |
| 3 | **Review** | Reviewers from `AGENTS_TO_BE_USED.md` §5 that apply, run side by side: `ecc:security-reviewer` (Opus) for auth or personal data, `ecc:database-reviewer` (Sonnet) for schema or queries, `ecc:typescript-reviewer` (Sonnet), `ecc:react-reviewer` (Sonnet) for `.tsx`, plus a Spec check against `requirements/` (`mattpocock-skills:code-review` Standards + Spec) | Report findings with `file:line` and the spec section each one breaks | Edit files. Fixes go back to stage 2 (Build), then are reviewed again |
| 4 | **Test** | `general-purpose`, Sonnet | Runs every suite (API Jest + Playwright, web, agent, the full e2e stack), then probes security (auth bypass, CSRF, expired or forged sessions, key leaks, injection), vulnerabilities (`npm audit`) and usability (what the user sees when a session ends, error messages). Adds a test only where a spec rule has none | Fix app code (failures go back to stage 2) |

### Models and security checks for stages 3 and 4 (owner rule, 2026-11-04)
* **Sonnet** runs every review and test agent **except security and vulnerability work**, which always runs on **Opus**: one security review agent and one security test agent per backend repo (`resortos-booking-api`, `resortos-booking-agent`), and one Opus security pass for `resortos-booking-web`. They run in parallel with the Sonnet agents.
* **OWASP checklists, item by item.** The security agents go through every item and give each a verdict (pass / fail / not applicable) **with evidence** (`file:line`, or the command and its real output). An item with no evidence counts as not checked, which fails the stage.
  * **OWASP Top 10 (2021)**, A01–A10: all repos.
  * **OWASP API Security Top 10 (2023)**, API1–API10 (object-level and function-level authorization, authentication, property-level exposure, resource consumption and rate limits, mass assignment, SSRF, misconfiguration, inventory, unsafe consumption of other APIs): `resortos-booking-api` and `resortos-booking-agent`.
  * **OWASP Top 10 for LLM Applications (2025)**, LLM01–LLM10 (prompt injection, sensitive data disclosure, excessive agency, improper output handling, unbounded consumption, ...): `resortos-booking-agent`.
* **Stricter for the backend:** the security test agent attacks the running API and agent (not only reads code): IDOR on every `:id` route, every route group with every wrong credential, mass assignment on every write body, injection in every input, oversized and malformed bodies, rate limits, CSRF, session fixation and reuse after sign-out, secrets in logs, responses and errors, and the dependency audit.
* **Pass bar:** any Critical or High finding blocks the change. A Medium is fixed, or the owner accepts it in writing in `obsidian-log/DECISIONS.md`. `npm audit --omit=dev` must show 0 High / Critical in every repo; dev-only findings are reported.
* **Tools:** the agents use what is installed. An OWASP ZAP baseline scan and a Semgrep OWASP rule scan are listed in `requirements/TESTING_PLAN.md` §2; until those tools are installed they are reported as "not run", never as passed.

### Stages 3 and 4 run per repo, in parallel (owner rule, 2026-11-03)
Stage 1 stays sequential. **Stage 2 (Build) may run one agent per repo in parallel** (owner rule, 2026-11-04) when the plan gives each repo its own changes; an agent edits only its own repo, and cross-repo checks (the e2e full stack) wait until every Build agent has finished. For Review and Test, the orchestrator launches **one agent per changed repo, all at the same time**, then waits for all of them before moving on:

| Repo | Review agent (stage 3) covers | Test agent (stage 4) runs |
|---|---|---|
| `resortos-booking-api` | Opus: security, database (schema, migration, queries), TypeScript, spec | Jest + Playwright API suites, security probes, `npm audit` |
| `resortos-booking-agent` | Sonnet: security (key handling, logging), TypeScript, spec | `npm test`, typecheck, `npm audit` |
| `resortos-booking-web` | Sonnet: React, TypeScript, security (no tokens in JS), usability, spec | mocked Playwright suite, build, `npm audit` |
| `resortos-booking-e2e` | Sonnet: test quality (does each test prove its spec row), TypeScript | the full real-stack run (all repos), cross-browser, usability |

Each per-repo review agent loads `ecc:code-review` and `mattpocock-skills:code-review` (Standards + Spec) and applies the checklists of the `AGENTS_TO_BE_USED.md` §5 reviewers that fit its repo. Each agent stays inside its own repo (reading other repos' code is fine; editing nothing in Review, only tests in its own repo in Test). Test agents use the ports their repo's config already uses, so they don't clash. Fixes from any repo go back to stage 2, then that repo's stage 3 and 4 agents run again.

The Haiku Obsidian logger runs after stage 4 (see `AGENTS_TO_BE_USED.md` §4). The orchestrator commits.

## 2. No hallucination, necessary files only

* Every claim in a report needs evidence: a command and its real output, or `file:line`. "Should work" is not evidence.
* Test results are reported as the exact command plus the pass/fail/skip counts it printed. Expected vs actual, never rounded up.
* **Create a file only if the plan names it** and the spec needs it. No new docs, notes, summaries, helper scripts or "examples" beyond the plan. Prefer editing existing files.
* Don't invent APIs, env vars, options or library behaviour: read the code or `node_modules` first. If unsure, say so in the report.
* Stay inside the task's scope; anything extra found goes in the report as a suggestion, not into the code.
* Never run destructive git (`reset --hard`, `push`, force anything). Commits are local.

## 3. The orchestrator checks each report before the next stage

1. Re-read this file.
2. `git status` and `git diff --stat` in every repo the stage touched: only planned files changed? Any unexpected new file gets removed or justified.
3. Re-run at least one test command the stage claims passed, and compare the counts.
4. Confirm each review finding against the code before sending it to Build.
5. Then launch the next stage, passing it the previous stage's verified output.

## 4. Always true (from `AGENTS_TO_BE_USED.md` §6)
* **API routes must be plural** (`/api/users`, `/api/admins`, `/api/agents`, `/public/menus`, `/users/reservations`). Only words with no sensible plural stay singular: `public`, `auth`, actions (`register`, `login`, `logout`, `export`, `lookup`), `health`, `me`, `stats`. Owner rule, 2026-11-04.
* **camelCase, never snake_case** (owner rule, 2026-11-04): file names, folders, functions, variables, keys. The only exception is UPPER_SNAKE_CASE (fully upper case) for constants, env vars, database enum values and audit/log action names. Details in the coding standards.
* **Every `.md` file name is UPPER_SNAKE_CASE** (owner rule, 2026-11-04), e.g. `API_SPEC.md`, `docs/DATABASE.md`.
* **Coding standards (owner rule, 2026-11-04):** backend repos (`resortos-booking-api`, `resortos-booking-agent`, and `resortos-booking-e2e` for TypeScript and tests) follow `requirements/CODING_STANDARDS_BACKEND.md`; the React app (`resortos-booking-web`) follows `requirements/CODING_STANDARDS_REACT.md`. Build follows them; Review checks against them. Everything is TypeScript (`strict`); no `.js` source files.
* No code before the spec is approved by the owner. KISS. TypeScript only.
* Backend Jest keeps `src/` at 100% coverage; no coverage-ignore comments.
* Original owner docs (`PRD_BOOKING.md`, Bahasa files) are never edited.
