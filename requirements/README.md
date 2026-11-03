# Requirements Index
**ResortOS AI — Restaurant Booking App**

Read in this order. Every v1 code change must trace back to one of these docs (spec-driven development).

| # | Document | Purpose | Status |
|---|---|---|---|
| 1 | [PRD_BOOKING.md](PRD_BOOKING.md) | Original full product vision (author-owned, unchanged) | ✅ |
| 2 | [PRD_BOOKING_V1.md](PRD_BOOKING_V1.md) | **v1 build contract**: in-scope / out-of-scope, definition of done | ✅ |
| 3 | [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) | DB review, Prisma decision, booking rules & numbers | ✅ |
| 4 | [DATA_MODEL.md](DATA_MODEL.md) | Tables, constraints, seed data | ✅ |
| 5 | [API_SPEC.md](API_SPEC.md) | Endpoints, payloads, exact error messages | ✅ |
| 6 | [UI_SPEC.md](UI_SPEC.md) | Screens, routes, UX rules | ✅ |
| 7 | [WORKFLOW.md](WORKFLOW.md) | Guest / staff / admin flows | ✅ |
| 8 | [SECURITY_SPEC.md](SECURITY_SPEC.md) | Auth, RBAC, encryption, compliance | ✅ |
| 9 | [AUDIT_TRAIL.md](AUDIT_TRAIL.md) | Audit log format, retention, alerts | ✅ |
| 10 | [TESTING_PLAN.md](TESTING_PLAN.md) | Unit / API / UI / security / performance testing | ✅ |
| 11 | [AGENT_SPEC.md](AGENT_SPEC.md) | Booking assistant (AI agent): scope, guardrails, RBAC, Cent AI, decisions | ✅ Built |
| 12 | [TEST_SCENARIOS_APP.md](TEST_SCENARIOS_APP.md) | Every guest/admin scenario: tested, where, and still to test | ✅ |
| 13 | [TEST_SCENARIOS_ASSISTANT.md](TEST_SCENARIOS_ASSISTANT.md) | Every guest → AI → API scenario, live AI runs, still to test | ✅ |
| 14 | [E2E_SPEC.md](E2E_SPEC.md) | Real end-to-end suite (`resortos-booking-e2e`): no mocking, test database, videos, results | ✅ |
| 15 | [CODING_STANDARDS_BACKEND.md](CODING_STANDARDS_BACKEND.md) | How backend TypeScript is written (api, agent, e2e): layout, plural routes, validation, logging, tests | ✅ |
| 16 | [CODING_STANDARDS_REACT.md](CODING_STANDARDS_REACT.md) | How the React app is written: where types and API calls live, components, security, tests | ✅ |
| — | [../AGENTS_TO_BE_USED.md](../AGENTS_TO_BE_USED.md) | Agent skills and when to use them | ✅ |

### Per-repo docs
* `resortos-booking-api/README.md`, `resortos-booking-web/README.md` (+ `DESIGN.md`), `resortos-booking-agent/README.md` (+ `AGENT_CAPABILITIES.md`): setup, run, test.
