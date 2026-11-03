# Requirements Index
**ResortOS AI — Restaurant Booking App**

Read in this order. Every v1 code change must trace back to one of these docs (spec-driven development).

| # | Document | Purpose | Status |
|---|---|---|---|
| 1 | [PRD_BOOKING.md](PRD_BOOKING.md) | Original full product vision (author-owned, unchanged) | ✅ |
| 2 | [PRD_BOOKING_v1.md](PRD_BOOKING_v1.md) | **v1 build contract**: in-scope / out-of-scope, definition of done | ✅ |
| 3 | [technical_decisions.md](technical_decisions.md) | DB review, Prisma decision, booking rules & numbers | ✅ |
| 4 | [data_model.md](data_model.md) | Tables, constraints, seed data | ✅ |
| 5 | [api_spec.md](api_spec.md) | Endpoints, payloads, exact error messages | ✅ |
| 6 | [ui_spec.md](ui_spec.md) | Screens, routes, UX rules | ✅ |
| 7 | [workflow.md](workflow.md) | Guest / staff / admin flows | ✅ |
| 8 | [security_spec.md](security_spec.md) | Auth, RBAC, encryption, compliance | ✅ |
| 9 | [audit_trail.md](audit_trail.md) | Audit log format, retention, alerts | ✅ |
| 10 | [testing_plan.md](testing_plan.md) | Unit / API / UI / security / performance testing | ✅ |
| 11 | [agent_spec.md](agent_spec.md) | Booking assistant (AI agent): scope, guardrails, RBAC, Cent AI, decisions | ✅ Built |
| 12 | [test_scenarios_app.md](test_scenarios_app.md) | Every guest/admin scenario: tested, where, and still to test | ✅ |
| 13 | [test_scenarios_assistant.md](test_scenarios_assistant.md) | Every guest → AI → API scenario, live AI runs, still to test | ✅ |
| 14 | [e2e_spec.md](e2e_spec.md) | Real end-to-end suite (`resortos-booking-e2e`): no mocking, test database, videos, results | ✅ |
| — | [../AGENTS_TO_BE_USED.md](../AGENTS_TO_BE_USED.md) | Agent skills and when to use them | ✅ |

### Per-repo docs
* `resortos-booking-api/README.md`, `resortos-booking-web/README.md` (+ `DESIGN.md`), `resortos-booking-agent/README.md` (+ `AGENT_CAPABILITIES.md`): setup, run, test.
* `deployment.md`: post-v1 (hosting and CI/CD are out of scope for v1).
