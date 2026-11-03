# Coding Standards: React Frontend
**ResortOS AI — Restaurant Booking App**

Applies to `resortos-booking-web` (React + Vite PWA). Owner rule, 2026-11-04. Screens and behaviour: `ui_spec.md`; look and feel: the repo's `DESIGN.md`.

## 1. Language and tooling
* **TypeScript only** (`.ts` / `.tsx`), `"strict": true`. No `.js` / `.jsx` source files.
* No `any`. Use `unknown` and narrow it. No `@ts-ignore`.
* `npm run typecheck` and `npm run build` must pass before every commit.

## 2. Folder layout
| Folder | Holds | Must not |
|---|---|---|
| `src/types/` | **All shared types**: `api.ts` mirrors the booking API's request and response shapes (`api_spec.md`), `chat.ts` the assistant's | Hold code that runs |
| `src/backend-api/` | **All calls to the booking API.** `client.ts` is the one fetch wrapper (cookie, 401 handling, errors). One file per route group or resource (`auth.ts`, `users.ts`, `reservations.ts`, `menus.ts`, `availabilities.ts`, `admins.ts`) with one typed function per endpoint, e.g. `listMyReservations(): Promise<Reservation[]>` | Be skipped: pages never build API paths themselves |
| `src/agent-api/` | All calls to the booking assistant (`/agent/chat`) | Share code with `backend-api` (no auth there) |
| `src/auth/` | The session context (`GET /api/users/me` on load, sign-in, sign-out) | — |
| `src/pages/` | One component per screen; `pages/admin/` for admin screens | Call `fetch` or write API paths |
| `src/components/` | Reusable UI pieces (`Layout`, `Dialog`, buttons, banners) | Call the API |
| `src/lib/` | Small pure helpers (formatting dates, prices) | Hold React state |

## 3. Components and hooks
* Function components only, named `PascalCase`, one main component per file named like the file.
* Props get an explicit `interface` or `type`.
* Keep state as close as possible to where it's used; lift it only when two components need it. No global store for v1.
* Data loading: call a `backend-api` function inside `useEffect` (or an event handler), handle the three states every time: loading, error (show `ErrorBanner` with the API's message), and data.
* Lists need stable `key`s (ids, never array indexes for changing lists).
* Follow the rules of hooks; dependency arrays are complete (no lint suppression).

## 4. Security and sessions
* The session lives only in the HttpOnly cookie. Never read or store tokens in `localStorage`, `sessionStorage` or `document.cookie`.
* A `401` outside `/api/auth/*` signs the user out locally and sends them to sign in (done once, in `client.ts`). No refresh, no retry.
* Never use `dangerouslySetInnerHTML`. React's escaping is the defence against XSS.
* The service worker never caches `/api/*` or `/agent/*`.

## 5. Accessibility and layout
* Use real elements (`button`, `a`, `label`, `form`) with visible labels; every input has a `<label>`.
* Errors use `role="alert"`. Dialogs trap focus and close with Escape.
* Mobile-first; tap targets at least 44 px; no horizontal scrolling at 320 px.
* The axe check in the e2e suite must stay at zero violations.

## 6. Naming and style
* Components `PascalCase.tsx`, other files `camelCase.ts` or as the folder already uses.
* Functions and variables `camelCase`, types `PascalCase`, constants `UPPER_SNAKE_CASE`.
* Comments say **why**, not what. No commented-out code. KISS: no new library unless the spec needs it.

## 7. Testing
* Test first. Playwright tests in `tests/e2e/` run against the mocked API (`tests/e2e/mock-api.ts`), phone viewport.
* Find elements the way a user would: by role and visible text, not CSS classes.
* Each test names its scenario ID from `test_scenarios_app.md`.
* When an API path changes, change it in `src/backend-api/` and in `mock-api.ts`, nowhere else.

## 8. Git
* Small local commits, Conventional Commits, tests committed before the code that makes them pass.
