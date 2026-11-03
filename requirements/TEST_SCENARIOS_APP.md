# Test Scenarios: Real Users (Guest and Admin)
**ResortOS AI — Restaurant Booking App**

People using the web app (PWA) and the booking API directly. For the AI assistant (guest → AI → API) see [TEST_SCENARIOS_ASSISTANT.md](TEST_SCENARIOS_ASSISTANT.md). Strategy and tools: [TESTING_PLAN.md](TESTING_PLAN.md).

**Status:** ✅ automated and passing · 👁 checked by hand on the real stack (browser → web → API → database) · ⬜ to be tested

**Where the tests live:**
| Code | Suite |
|---|---|
| `API` | `resortos-booking-api/tests/api/*.spec.ts` (Playwright, real API + database) |
| `UNIT` | `resortos-booking-api/tests/unit/*.spec.ts` (Playwright) |
| `JEST` | `resortos-booking-api/tests/jest/*.test.ts` (Jest, 100% coverage) |
| `WEB` | `resortos-booking-web/tests/e2e/*.spec.ts` (Playwright, phone viewport, API mocked) |

## 1. Guest account
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| A1 | Register with name, email, password, phone | Signed in, lands on "Reserve a table" | API auth, JEST auth, WEB account-and-menu | ✅ 👁 |
| A2 | Register with an email that's taken | "Email is already registered" | API auth, WEB account-and-menu | ✅ |
| A3 | Password under 8 characters | Explained before any request | API auth, WEB account-and-menu | ✅ |
| A4 | Invalid phone number | "phone: Invalid phone number" | API platform, JEST agent | ✅ |
| A5 | Wrong password / unknown email | Same "Invalid email or password", same timing | API auth, JEST auth | ✅ |
| A6 | Email typed in capitals | Same account (stored lowercase) | JEST auth | ✅ |
| A7 | Reload the page while signed in | Still signed in (session cookie, checked with `GET /api/users/me` on load) | WEB session-and-states | ✅ 👁 |
| A8 | Session ends (fixed 1 hour after sign-in) | Screen keeps what it shows; the next server call sends the guest to sign in | WEB session-and-states (simulated) | ✅ |
| A9 | Page load without a valid session | Sent to sign in | WEB session-and-states | ✅ |
| A10 | Sign out | Back to sign in; pages protected again; the session cookie is cleared | API platform, JEST auth, WEB session-and-states | ✅ 👁 |
| A11 | Sign up with an email the assistant booked for | Claims the guest account; assistant bookings appear in "My bookings" | API agent, JEST agent | ✅ |
| A12 | 10 failed logins from one IP in 5 minutes | One security alert in the audit log | JEST auth | ✅ |
| A13 | Too many auth attempts | 429 with a JSON message | JEST environment | ✅ |
| A14 | Sign out, then send a saved copy of the old cookie | `401 Invalid or expired token` at once; the `Session` row is gone | API auth, JEST auth, E2E | ⬜ new |
| A15 | Session older than 1 hour (row still in the table) | `401 Invalid or expired token`; signing in again deletes the old row | JEST auth | ⬜ new |
| A16 | Signing out on one device | Other devices' sessions keep working | JEST auth | ⬜ new |

## 2. Booking a table
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| B1 | Book Main Hall: branch, room, date, sitting, guests, note | Reservation code, ⌈party ÷ 2⌉ tables, 2.5-hour slot | API reservations, WEB booking | ✅ 👁 |
| B2 | Book VIP for 14 | 2 rooms joined | WEB session-and-states, click-through | ✅ 👁 |
| B3 | Sitting already full | Can't be picked ("Fully booked") | WEB booking | ✅ |
| B4 | Sitting too small for the party | Disabled ("Not enough room for N") | WEB session-and-states | ✅ |
| B5 | Server says no capacity (race) | Message shown, no booking | API reservations, WEB booking | ✅ |
| B6 | Party over 50 / not a whole number | Explained before calling the API | API reservations, WEB booking | ✅ |
| B7 | Time that isn't a fixed sitting | 400 with the valid sittings | API reservations | ✅ |
| B8 | Past date / more than 60 days ahead | 400 "Selected slot is in the past or too far ahead" | API reservations, JEST reservations | ✅ |
| B9 | Two guests take the last capacity at once | Exactly one booking, never double-booked | API reservations | ✅ |
| B10 | Open a confirmation link directly | Booking loads; unknown ID says "Reservation not found" | WEB session-and-states | ✅ |
| B11 | Times shown in restaurant time (UTC+7) | "Thu 24 Dec, 19:30 – 22:00" | UNIT booking-rules, WEB | ✅ |

## 3. Managing bookings
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| M1 | Upcoming and past tabs | Each booking in the right tab | WEB my-bookings | ✅ |
| M2 | Reschedule to another sitting | New slot and tables; dialog starts at the current sitting | API reservations, WEB my-bookings | ✅ 👁 |
| M3 | Reschedule into a full sitting | Rejected; original booking and tables kept | API reservations, WEB session-and-states | ✅ |
| M4 | Cancel after confirming | Status Cancelled, tables freed, record kept | API reservations, JEST http-basics, WEB my-bookings | ✅ 👁 |
| M5 | Cancel or change twice / after the sitting started | 400 with the reason | API reservations, JEST reservations | ✅ |
| M6 | Reschedule and cancel at the same moment | Never a cancelled booking holding tables | API reservations | ✅ (real Postgres ⬜, see §8) |
| M7 | No bookings yet | "No upcoming bookings yet" with a link to book | WEB my-bookings | ✅ |
| M8 | Escape / Keep buttons | Dialog closes, nothing changes | WEB my-bookings, WEB session-and-states | ✅ |

## 4. Menu and profile
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| P1 | Browse menu by type, signed out | Dishes and prices; no "Safe for you" | API menu, WEB account-and-menu | ✅ 👁 |
| P2 | Save allergies | Stored lowercase; "Profile saved" | API auth, WEB account-and-menu | ✅ 👁 |
| P3 | "Safe for you" hides allergens, chef's picks first | Matches case-insensitively | API menu, JEST auth-profile-menu | ✅ 👁 |
| P4 | Clear phone number | Phone removed | API platform | ✅ |

## 5. Admin (read-only)
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| D1 | Day view: guests, active bookings, cancellations, utilisation | Real numbers for the date and branch | API admin, JEST admin, WEB admin | ✅ 👁 |
| D2 | Day view counts assistant bookings | "of which N via AI assistant" | API agent, WEB admin-agent | ✅ |
| D3 | Booking log search by name, email, phone or code | Matching bookings only | API admin, JEST admin, WEB admin | ✅ 👁 |
| D4 | Filters: branch, room, status, booked by, dates; paging | Correct subset; 20 per page | API admin, API agent, WEB admin, WEB admin-agent | ✅ |
| D5 | A slow old filter result arrives late | Never overwrites the newer result | WEB admin | ✅ |
| D6 | Detail sheet | Guest, sitting, tables, notes, "Booked by", cancelled time | API admin, WEB admin, WEB admin-agent | ✅ 👁 |
| D7 | Export CSV / JSON | Filtered rows, `source` column, formulas neutralised | API admin, JEST admin, WEB admin | ✅ 👁 |
| D8 | Export over 1,000 rows | Security alert in the audit log | JEST admin | ✅ |
| D9 | Admin tries to create or delete a booking | 403 / no such route | API admin, API agent | ✅ |

## 6. Security and RBAC (direct API calls)
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| S1 | Guest opens another guest's booking (IDOR) | 404 for read, reschedule, cancel | API reservations | ✅ |
| S2 | Guest opens admin pages or routes | Redirected / 403 | API admin, WEB admin | ✅ |
| S3 | Forged or missing token | 401 | API platform, JEST http-basics | ✅ |
| S4 | Assistant's key on `/api/users/*` or `/api/admins/*` | `401 Authentication required` (the key isn't read there) | API agent | ⬜ changed |
| S5 | Sign up as the assistant's or an admin's email | "Email is already registered" | API agent | ✅ |
| S6 | Audit log edited or deleted | Refused by the database | JEST domain | ✅ |
| S7 | Malformed JSON, unknown route, security headers | JSON 400/404, `nosniff`, no `x-powered-by` | API platform | ✅ |
| S8 | Production mode | Secure cookie, trusts one proxy | JEST environment | ✅ |
| S9 | Any failed request (4xx/5xx) | One `[METHOD]`-tagged JSON line in `logs/errors.log` + `console.error`; no bodies or secrets | JEST, E2E | ⬜ new |
| S16 | Every successful create / update / delete (table in api_spec §8.1) | One `[POST]`/`[PATCH]`/`[DELETE]`-tagged JSON line in `logs/app.log` + `console.log`, with the `action`; nothing for GETs; no personal data or secrets | JEST | ⬜ new |
| S11 | Guest or admin cookie on `/api/agents/*` | `401 Authentication required` (the cookie isn't read there) | API agent, JEST | ⬜ new |
| S12 | Anyone, including the assistant, on `/api/public/*` (menu, availability, health) | Works with no cookie or key; any cookie or key sent is not read | API agent, JEST | ⬜ new |
| S13 | Old or forged cookie on `/api/auth/login` or `/api/public/menus` | Ignored: sign-in and menu work | JEST | ⬜ new |
| S14 | A key in `ApiKey` whose owner isn't `AGENT` | `401 Invalid API key` | JEST | ⬜ new |
| S15 | User row removed while signed in | `401 Invalid or expired token` on `/api/users/*` and `/api/admins/*` (sessions and key deleted with the user) | JEST | ⬜ new |
| S10 | `app.log` or `errors.log` over 5 MB | Renamed to `.1`, the previous `.1` deleted: one live file + one old copy per log | JEST | ⬜ new |

## 7. Mobile and PWA
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| W1 | Every screen at phone width | No sideways scrolling | WEB layout | ✅ |
| W2 | Bottom tabs per role; sheets open at their title | Right tabs; focus on the title | WEB, WEB session-and-states | ✅ 👁 |
| W3 | Manifest and icons | Name, standalone, theme, 192/512/maskable icons reachable | WEB pwa | ✅ |
| W4 | Service worker; open with the network off | App opens; API errors say "You're offline" | WEB pwa | ✅ |
| W5 | Error states (bookings, sittings, menu, numbers, profile) | Each names the problem | WEB session-and-states | ✅ |

## 8. To be tested
| # | Scenario | Why it matters | How |
|---|---|---|---|
| T1 | **Install on a real iPhone (Safari) and Android (Chrome)**, open from the home screen, go offline | Only emulated so far | Manual, on devices, after HTTPS hosting |
| T2 | **Races on real PostgreSQL** (two cancels/reschedules, last-table bookings, per-email cap) | The test database (PGlite) runs queries one at a time, so true parallelism isn't exercised | API suite against a Postgres 17 instance |
| T3 | **Other browsers**: Safari/WebKit, Firefox | Suite runs Chromium only | Add Playwright WebKit/Firefox projects |
| T4 | **Accessibility audit** with axe and a screen reader (VoiceOver, TalkBack) | Roles and labels are tested, not the full experience | axe in Playwright + manual pass |
| T5 | **Visual regression** of the main screens | Behaviour is tested, pixels aren't | Playwright `toHaveScreenshot()` |
| T6 | **Load test**: 500 concurrent users at dinner time, p95 < 300 ms | TESTING_PLAN.md target, not yet measured | k6 or Artillery on staging |
| T7 | **Real session expiry** (1 hour) | Expiry is simulated in UI tests | e2e with `SESSION_TTL=8s` |
| T8 | **HTTPS deployment**: same-site session cookie, Origin rule, CORS, `TRUST_PROXY` | Only localhost so far | Staging smoke test |
| T9 | **CSV opened in Excel / Google Sheets** | Formula neutralising is unit-tested, not opened in apps | Manual |
| T10 | **Penetration test / OWASP ZAP scan** | SECURITY_SPEC.md requires one before launch | ZAP baseline on staging |
| T11 | **UAT** with restaurant staff and a few guests | Real-world fit | Staging, guided session |

## 9. Real end-to-end coverage (no mocking)
Suite: `resortos-booking-e2e` ([E2E_SPEC.md](E2E_SPEC.md)). Latest results with expected vs actual and videos: `resortos-booking-e2e/RESULTS.md`.

| Area | Scenario IDs run end to end | Latest run |
|---|---|---|
| Account | A1 A2 A5 A7 A10 A11 | all passed |
| Booking | B1 B2 B3 B6 | all passed |
| Managing | M1 M2 M4 M7 | all passed |
| Menu / profile | P1 P3 | all passed |
| Admin | D1 D2 D3 D4 D6 D7 | all passed (D4 failed once on a test selector, fixed) |
| Security / logging | S1 S2 S4 S9 | all passed |
| PWA | W1 W3 W4 | all passed |

### Run on 2026-11-03 (latest e2e: 66 passed, 0 failed)
| To-be-tested item | Result |
|---|---|
| T2 races on **real PostgreSQL 17** | ✅ `npm run test:pg`: 61 Playwright + 77 Jest passed. **Found and fixed a bug:** simultaneous confirmations for a new email raced into 500s (guest-account upsert); now `INSERT … ON CONFLICT DO NOTHING` |
| T3 Safari/WebKit and Firefox | ✅ 10 scenarios each passed. **Found and fixed:** Safari formatted prices as `Rp320.000` (now `Rp 320.000` everywhere) |
| T4 accessibility (axe, WCAG 2 A/AA, 7 screens) | ✅ no violations at all (screen-reader pass still manual) |
| T5 visual regression | ✅ baselines recorded for sign-in, register, chat |
| T6 load (autocannon, test API on PGlite) | ✅ 0 errors / 0 non-2xx. Availability ~1,000 req/s (p99 61 ms, 50 connections); menu ~8,000 req/s; **sign-in ~10 req/s, p50 ~0.9 s** (bcrypt cost 12 is CPU-bound) |
| T7 real session expiry (8 s sessions) | to be re-run: next server call after expiry should send the guest to sign in |
| Still needs staging or people | T1 real phones, T8 HTTPS deploy, T9 CSV in Excel, T10 ZAP scan, T11 UAT |

