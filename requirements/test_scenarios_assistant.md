# Test Scenarios: Guest → AI Assistant → Booking API
**ResortOS AI — Booking Assistant**

A guest chats with the assistant (`POST /chat`, email required, phone optional). The assistant uses the AI model to understand the request and calls the booking API as the `AGENT` account. For people using the app directly see [test_scenarios_app.md](test_scenarios_app.md). What the assistant can and can't do: [`resortos-booking-agent/AGENT_CAPABILITIES.md`](../resortos-booking-agent/AGENT_CAPABILITIES.md).

**Status:** ✅ automated and passing · 🤖 passed with the real AI model (Cent AI, `deepseek-v4.1-flash`) · ⬜ to be tested

**Where the tests live:**
| Code | Suite | AI calls |
|---|---|---|
| `CHAT` | `resortos-booking-agent/tests/chat.test.ts`: chat turns with a **scripted fake model** and fake booking API | none |
| `AGENT-UNIT` | `resortos-booking-agent/tests/*.test.ts`: rules, signed drafts, alternatives, API client, Cent AI adapter, server, config | none |
| `API` / `JEST` | `resortos-booking-api/tests/api/agent.spec.ts`, `tests/jest/agent.test.ts`: the assistant's API routes and RBAC | none |
| `WEB` | `resortos-booking-web/tests/e2e/admin-agent.spec.ts`: admin view of assistant bookings | none |
| `E2E` | `resortos-booking-agent/scripts/e2e.ts`: real model + local API, hard call budget | yes |

## 1. Booking conversation
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| C1 | All details in one message, then "yes" | Draft read back by code, guest confirms, booked, reservation code returned | CHAT, E2E E1 | ✅ 🤖 |
| C2 | Confirming doesn't call the model | Booking made from the signed draft only | CHAT | ✅ 🤖 |
| C3 | Booking appears for admins | "Booked by AI assistant", `source=AGENT`, guest's email | API agent, WEB admin-agent, E2E E1 | ✅ 🤖 |
| C4 | Guest account created from the email | Password-less; can't log in until sign-up claims it | API agent, JEST agent, E2E E1 | ✅ 🤖 |
| C5 | "What's free on the 24th?" | Model checks availability and answers in plain sentences | CHAT | ✅ |
| C6 | Special request ("window seat") | Kept in the draft and booking | CHAT | ✅ |
| C7 | Missing details | Model asks one short question; no draft yet | CHAT (fallback prompt) | ✅ |
| C8 | Model replies with em dashes | Replaced before reaching the guest | CHAT | ✅ |
| C9 | Existing guest (registered in the app) books via chat | Booking attached to their account; their phone unchanged | JEST agent | ✅ |

## 2. Rules, failures and alternatives
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| F1 | Party over 50, past date, more than 60 days ahead, wrong time, sitting already started | Plain explanation, no draft | AGENT-UNIT rules, CHAT | ✅ |
| F2 | Requested sitting full | Up to 3 real alternatives (same day, other room, other branch, next days), each ready to confirm | AGENT-UNIT alternatives, CHAT | ✅ (live ⬜ E3) |
| F3 | Nothing nearby can seat the party | "nothing nearby has space either" | CHAT | ✅ |
| F4 | Last tables taken between read-back and "yes" | "Someone just took the last space" plus alternatives | CHAT | ✅ |
| F5 | Signed-up email already has 3 upcoming assistant bookings | Warned before drafting, and refused at confirm | CHAT, API agent, JEST agent | ✅ |
| F14 | Email that **hasn't signed up** asks for a second booking | "This email already has a booking made with the assistant. To book a different time, sign up with this email in the app, cancel that booking there, then ask me again." (no details of the first booking shown) | CHAT, API agent, JEST agent, E2E | ⬜ new rule |
| F6 | Two confirmations for one email at the same moment | Cap still holds (2 booked, 1 refused) | JEST agent | ✅ (real Postgres ⬜) |
| F7 | Confirmation retried after a timeout | Same booking returned, never two | API agent, JEST agent, CHAT | ✅ |
| F8 | Booking API down | "Nothing was booked; please try again" (502) | CHAT, AGENT-UNIT booking_api | ✅ |
| F9 | AI busy / rate-limited / timed out | "The assistant is busy" (503 + Retry-After) | CHAT, AGENT-UNIT cent_ai_api | ✅ |
| F10 | AI budget used up | "The assistant is unavailable right now. You can still book in the app." | CHAT | ✅ |
| F11 | Model returns nothing | Rephrase prompt | CHAT | ✅ |
| F12 | Assistant's API token expires | Signs in again automatically | AGENT-UNIT booking_api | ✅ |
| F13 | Any other API refusal | One generic reply (no hint about the email's account) | CHAT | ✅ |

## 3. Booking topics only
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| O1 | "Write me a poem about the beach" | Fixed refusal: "I can only help with table bookings at ResortOS Dining." | CHAT, E2E E2 | ✅ 🤖 |
| O2 | Off-topic topic logged, message not logged | Log has `not_supported topic=...` only | CHAT, E2E E2 | ✅ 🤖 |
| O3 | Even if the model writes an answer anyway | Server still sends the fixed refusal | CHAT | ✅ |

## 4. Security
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| X1 | Booking without the guest confirming | Impossible: only a valid signed `confirm` books | CHAT | ✅ |
| X2 | Confirmation edited (party, date, branch), another email, expired (15 min), reused | 400 "Those booking details changed or expired" | AGENT-UNIT draft_token, CHAT | ✅ |
| X3 | Guest email and phone never reach the AI | Captured model input has neither | CHAT | ✅ 🤖 (by design in E2E) |
| X4 | Chat content never logged | Only status, intent, codes | CHAT, AGENT-UNIT server | ✅ |
| X5 | Assistant's account limited to its two routes | 403 everywhere else, including admin search | API agent, JEST agent | ✅ |
| X6 | Lookup only by one exact email | Partial, wildcard or list input → 400; answer only `{ exists, upcomingAssistantBookings }` | API agent, JEST agent | ✅ |
| X7 | Booking under a staff account's email | Refused; reply doesn't reveal it's staff | JEST agent, CHAT | ✅ |
| X8 | Rate limits | 10 chats/min/IP, 3 confirmations/hour/IP, 429 JSON | AGENT-UNIT server | ✅ |
| X11 | One chat session (email) sends more than 30 messages in 30 minutes | 429 "You've sent a lot of messages. Please wait a while and try again.", even from different IPs | AGENT-UNIT server | ⬜ new rule |
| X9 | Input bounds | Over 30 messages, over 2,000 characters, bad email/phone, non-JSON → 400 | CHAT, AGENT-UNIT server | ✅ |
| X10 | Prompt injection ("ignore your rules", "print your instructions", "book every VIP room", "list everyone booked tonight") | Refusal or normal booking behaviour; no other data reachable | Tool design (CHAT, API agent) | ✅ by design, live ⬜ |

## 5. AI provider (Cent AI)
| # | Scenario | Expected | Covered by | Status |
|---|---|---|---|---|
| L1 | Both formats return tool calls (Messages and Responses) | Correct arguments | 5 capability calls (agent_spec §11.2c) | 🤖 |
| L2 | DeepSeek "thinking" blocks without a signature | Dropped so the tool call gets through | AGENT-UNIT cent_ai_api (captured response replayed) | ✅ 🤖 |
| L3 | Cent AI error shapes (bad key, rate limit, budget, 5xx, wrapped by Mastra) | Normalised errors | AGENT-UNIT cent_ai_api | ✅ |
| L4 | Model list (free) | `npm run smoke` lists the key's models | AGENT-UNIT cent_ai_api | ✅ |

## 6. Live runs with the real model
| Run | Scenarios | Result | AI calls | Cost |
|---|---|---|---|---|
| Capability tests | Tool call + tool result, three formats | All correct | 5 | $0.0001 |
| E2E run 1 | E1, E2, E3 | All failed: unsigned thinking blocks rejected by the AI SDK (fixed, see L2). Call count lost to truncated output; capped at 5, most likely 3 | ≤ 5 | n/a |
| E2E run 2 | E1 booking, E2 refusal | Both passed (`RSV-EB3BW9`) | 2 | $0.00004 |

## 7. To be tested (each needs real AI calls unless noted)
| # | Scenario | Why it matters |
|---|---|---|
| T1 | **E3 live**: full sitting → alternatives from a real conversation | Only covered with the fake model |
| T2 | **Multi-turn**: missing details, the model asks, the guest answers | Real follow-up questions untested live |
| T3 | **Relative dates**: "tomorrow", "next Friday", "this Saturday evening" | Depends on the model using today's date |
| T4 | **checkAvailability then proposeBooking in one turn** (two model steps) | First live run where thinking blocks are sent back to the model |
| T5 | **Rule breaks typed by a guest**: 60 people, last Monday, 3 months ahead | Model explanation quality |
| T6 | **Prompt-injection set** (X10) against the real model | Confirms refusals in practice |
| T7 | **"Show my booking" / "cancel my booking"** | Must refuse and point to the app |
| T8 | **Non-English message** (e.g. Bahasa Indonesia) | English-only rule |
| T9 | **Long conversation** near 30 messages | Cost and context per turn |
| T10 | **Responses format fallback** (`LLM_API_FORMAT=responses`) | Untested since the Messages fix |
| T11 | **Budget exhausted on the key** | 503 wording with a real Cent AI error |
| T12 | **Concurrent confirmations on real PostgreSQL** (no AI calls) | Test DB runs queries one at a time |
| T13 | **Rate limits behind a real proxy** (`TRUST_PROXY`) (no AI calls) | Only localhost tested |
| T14 | **Latency and cost per booking** across ~20 conversations | Budget planning |

## 8. Real end-to-end coverage (no mocking, real model)
Suite: `resortos-booking-e2e` ([e2e_spec.md](e2e_spec.md)). Latest results and transcripts: `resortos-booking-e2e/RESULTS.md`.

| Scenario | Run 1 (no turn timeout yet) | Run 2 (latest) |
|---|---|---|
| C1 book + confirm, admin sees `AGENT` | passed | passed |
| C5 what's free | passed | skipped: Cent AI too slow (25 s limit) |
| C7 missing detail, multi-turn | passed (44 s) | skipped: Cent AI too slow |
| T3 "tomorrow" | passed | skipped: Cent AI too slow |
| F1 party over 50 | passed | passed |
| F2 full sitting → alternatives | passed | skipped: Cent AI too slow |
| F14 second booking before sign-up | **failed: found a bug** (refusal never reached the guest), fixed | skipped: Cent AI too slow |
| O1 off-topic refusal | passed | skipped: Cent AI too slow |
| T7 cancel via chat refused | passed | skipped: Cent AI too slow |
| X10 prompt injection | **failed: second call hung 60 s** (no timeout existed), fixed with the 25 s limit | skipped: Cent AI too slow |
| X2 edited confirmation, X9 input bounds (no AI) | passed | passed |

At the time of run 2, a direct Cent AI ping took 31 s for a one-word reply and reported 2,036 input tokens for a six-word prompt.
