---
tags: index, hub
date: 2026-11-03
---

# Booking Restaurant v1 — Claude's Log

Claude's running log, learnings, and decisions for the ResortOS restaurant booking project.

## Navigation
- [[SESSION_LOG/2026_11_04]] — Today's chronological notes
- [[SESSION_LOG/2026_11_03]] — Previous session
- [[LEARNINGS]] — Lessons about this project and user
- [[DECISIONS]] — Architecture and tool choices
- [[OPEN_QUESTIONS]] — Unresolved items blocking progress
- [[TEST_RUNBOOK]] — Every test as a rerunnable command, per repo, with ports

## Quick Facts
- **Project:** Restaurant reservation booking system (TypeScript full-stack)
- **Repos:** 2 app repos (backend, frontend) + 1 planning/docs repo
- **Stack:** Express 5 + Prisma 7.10.1 + Zod 4 + PostgreSQL + JWT
- **DB lock-in:** PostgreSQL UNIQUE(tableId, slotStart) prevents double-booking at DB level
- **Status:** Requirements docs created; backend scaffolded; DB design reviewed; coding paused awaiting next steps
