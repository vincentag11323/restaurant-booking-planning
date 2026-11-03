# UI Specification
**ResortOS AI — Restaurant Booking App**

React 19 + Vite + Tailwind CSS 4 SPA. Responsive (mobile-first), English only. One app with role-based routes: a `USER` lands in the Diner App and an `ADMIN` lands in the Admin Log Viewer.

## 1. Routes
| Path | Access | Screen | Key elements |
|---|---|---|---|
| `/login` | public | Login | email, password, link to register, error banner |
| `/register` | public | Register | name, email, password, phone (optional), inline validation |
| `/` | USER | **Book a Table** | branch toggle → sector toggle → date picker → slot cards (time window, free tables/rooms, disabled if full or not bookable) → party size (1–50) → special requests → **Confirm** |
| `/bookings/:id/confirmed` | USER | Confirmation | reservation **code** (large, copyable), branch, sector, slot window, tables, party size |
| `/bookings` | USER | **My Bookings** | Upcoming / Past tabs. Each card: code, slot, sector, tables, party size, status badge. Actions on upcoming `CONFIRMED` cards: **Reschedule** (modal: date + slot + party size) and **Cancel** (confirm dialog) |
| `/menu` | public (recommendations need login) | Menu | type tabs (Seasonal Fine Dining / À La Carte), price, allergen chips. Logged in: a "Safe for you" section first |
| `/profile` | USER | Profile | name, phone, allergy tag input, save |
| `/admin` | ADMIN | **Dashboard** | date picker, branch filter, 4 stat tiles (bookings, covers, cancellations, utilization per sector) |
| `/admin/logs` | ADMIN | **Booking Log** | filter bar (branch, sector, status, from/to, search box), paginated table (code, customer, branch, sector, slot, party, status), **Export CSV / JSON** buttons, row click → **Detail Drawer** (customer, tables/rooms, created at, slot start → end, special requests) |

Any route the role can't access redirects (USER → `/`, ADMIN → `/admin`, guest → `/login`). The admin UI has **no** edit, cancel or create controls.

## 2. State & Data
* Fetch wrapper `api.ts`: attaches the access token and, on a `401`, calls `/auth/refresh` once and retries. If the refresh fails, it logs out.
* Access token is kept in memory (React context). The refresh cookie restores the session on reload.
* No global state library. Use React context for auth and local component state for everything else.
* Dev: Vite proxies `/api` to `http://localhost:4000` (same-origin, so cookies just work). Prod: `VITE_API_URL`.

## 3. UX Rules
* Show API `message` strings as-is in error banners. They are written to be user-readable.
* Disable submit buttons while a request is in flight.
* Display times in restaurant local time (`UTC+07:00`), e.g. "Sat 24 Dec, 19:30 – 22:00".
* Accessible: labelled inputs, keyboard-reachable drawer/modals, sufficient contrast.

## 4. Test Hooks
* Use accessible roles/labels (`getByRole`, `getByLabel`) in Playwright, not CSS selectors.
* `data-testid` only where no accessible handle exists (e.g. stat tiles).
