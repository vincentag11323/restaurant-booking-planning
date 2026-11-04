---
tags: testing, runbook
---

# Test Runbook: how to rerun every test

**Why this note exists (owner, 2026-11-04):** every check must be a saved, rerunnable script, so each iteration just reruns the same commands instead of rebuilding tests from scratch. Rule: `RULES.md` §4, "Every check is a saved script".

Run from each repo's folder. Ports listed so runs don't clash.

## Booking API (`resortos-booking-api`)
| What | Command | Ports | Covers |
|---|---|---|---|
| Typecheck + build | `npm run typecheck && npm run build` | — | TypeScript strict |
| Jest (100% coverage) | `npm run test:jest` | in-process PGlite | unit + integration: sessions (A14–A16), route groups (S4, S11–S15), logs (S9, S10, S16), Swagger drift guard (S17), **SQL injection S18** (`tests/jest/sqlInjection.test.ts`) |
| Playwright API suite | `npm run test:e2e` | 4100, 5498, 5499 | every endpoint over HTTP |
| Both | `npm test` | as above | — |
| On real PostgreSQL | `TEST_DATABASE_URL=<empty Postgres DB> npm run test:pg` | your Postgres | same suites on Postgres 17 |
| Dependency audit | `npm audit --omit=dev` (must be 0 High/Critical), `npm audit` | — | A06 |
| Throwaway test DB | `DB_TEST_PORT=5444 npm run db-test` | 5444 | for manual / migrate-diff checks |

## Web app (`resortos-booking-web`)
| What | Command | Ports |
|---|---|---|
| Typecheck + build | `npm run typecheck && npm run build` | — |
| Playwright (mocked today, to convert: "Never mock") | `npm test` | 5180 |
| Audit | `npm audit --omit=dev` | — |

## AI agent (`resortos-booking-agent`)
| What | Command | Ports |
|---|---|---|
| Typecheck | `npm run typecheck` | — |
| Tests (75; fakes today, to convert: "Never mock") | `npm test` | ephemeral |
| Real model smoke test (costs a few cents) | `npm run smoke` | — |
| Audit | `npm audit --omit=dev` | — |

## Real end-to-end stack (`resortos-booking-e2e`)
| What | Command | Ports |
|---|---|---|
| Typecheck | `npm run typecheck` | — |
| Everything (writes `RESULTS.md` + videos) | `npm test` | 4400 API, 4500 agent, 5190 web, 5433 DB |
| App only / assistant only (real model) | `npm run test:app` / `npm run test:assistant` | same |
| Safari + Firefox subset | `npm run test:browsers` (overwrites `RESULTS.md` with its rows; run `npm test` last) | same |
| Load | `npm run load` | same |
| Check a subset without touching `RESULTS.md` | `npx playwright test --project=app -g "A14\|S4" --reporter=line` | same |

## Security probes: still scratch scripts, to be saved as tests
Written by the security agents on 2026-11-04 in the job's temp folder (`~/.claude/jobs/6b67a2b1/tmp/`), **not yet in any repo**, so they disappear with that job. Next build round turns each into a saved test (see `OPEN_QUESTIONS.md`):
| Scratch script | What it does | Becomes |
|---|---|---|
| `sec-api/attack.mjs` | 168 live attacks on the API: every route group × every wrong credential, IDOR, mass assignment, oversized/malformed bodies, impossible dates, CSRF origins, fixation, secrets in logs | `resortos-booking-e2e/tests/security/api.spec.ts`, real stack |
| `sec-web/cspt.mjs` | proves the booking-id path traversal (web Medium) | a web regression test with the fix |
| `sec-agent/*.ts` | agent probes: malformed/oversized chats, tampered confirm tokens, rate limits, docs on/off, prompt injection | `resortos-booking-e2e/tests/security/agent.spec.ts` |
| `sec-stack/flow1.mjs`, `logProxy.mjs` | sessions + API key + middleware across web → API → agent on the real dev DB | `resortos-booking-e2e/tests/security/stack.spec.ts` |

Planned: `npm run test:security` in `resortos-booking-e2e` runs all of them. Not installed yet: OWASP ZAP baseline, Semgrep OWASP rules (`requirements/TESTING_PLAN.md` §2).
