# ResortOS Dining (restaurant booking)

Table booking for two resort restaurants: a mobile PWA for guests and admins, a booking API, and an AI booking assistant.

**Start here:** [DIAGRAM.md](DIAGRAM.md) (how the repos connect, flows, and every deliberate simplification).

| Folder | What it is | Its README |
|---|---|---|
| `requirements/` | Specs: PRD, API, data model, UI, agent, tests (planning repo) | [requirements/README.md](requirements/README.md) |
| `resortos-booking-api/` | Booking API (Express, Prisma, PostgreSQL) | [README](resortos-booking-api/README.md) |
| `resortos-booking-web/` | PWA for guests and admins, chat screen | [README](resortos-booking-web/README.md) |
| `resortos-booking-agent/` | AI booking assistant (Mastra + Cent AI) | [README](resortos-booking-agent/README.md) |
| `resortos-booking-e2e/` | Real end-to-end tests, videos, results, load test | [README](resortos-booking-e2e/README.md) |
| `obsidian-log/` | Working log, decisions, learnings (open as an Obsidian vault) | [README](obsidian-log/README.md) |
| `RULES.md` | How every task is run: Plan → Build → Review → Test subagents, evidence only | |
| `AGENTS_TO_BE_USED.md` | How the AI coding agents and skills are used | |

Each `resortos-*` folder is its own git repository; this root folder is the planning repository.

## Run everything locally
```bash
cd resortos-booking-api   && npm run db:local      # terminal 1: database
cd resortos-booking-api   && npm run dev           # terminal 2: API :4000
cd resortos-booking-agent && npm run dev           # terminal 3: assistant :4300 (needs .env with the Cent AI key)
cd resortos-booking-web   && npm run dev           # terminal 4: http://localhost:5173
```
Test everything for real: `cd resortos-booking-e2e && npm test`.
