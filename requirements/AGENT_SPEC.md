# Booking Assistant (AI Agent) Specification
**ResortOS AI — Restaurant Booking App**

**Status:** BUILT 2026-11-03. Capabilities summary: `resortos-booking-agent/AGENT_CAPABILITIES.md`.
**Repo:** `resortos-booking-agent` (third app repo, decoupled from `resortos-booking-api` and `resortos-booking-web`).
**Stack:** 100% TypeScript · Mastra (`@mastra/core`, Apache-2.0) · Zod · Express · own `src/types/` folder.

---

## 1. Summary
A chat assistant that answers **booking questions only** and can **book a table on the guest's behalf**. Guests don't sign in. They give an **email (required)** and optionally a **phone number**, chat in plain language ("table for 6 at Branch 2 on Friday evening?"), confirm, and receive a reservation code.

## 2. Scope

### 2.1 ✅ The assistant DOES
| Capability | Notes |
|---|---|
| Answer booking questions | Sittings (11:00, 13:30, 17:00, 19:30, 2.5 h each), branches, Main Hall vs VIP rooms, party size 1–50, the 60-day booking window, how tables are joined |
| Check live availability | Calls the API's public `GET /api/public/availabilities`. It never guesses or caches |
| Book on the guest's behalf | Only after it reads the full booking back and the guest explicitly says yes |
| Take special requests / dietary notes | Passed through to the booking (max 500 chars) |
| Reply with the reservation code | e.g. `RSV-7K3Q9X`, plus the sitting, room and tables |

### 2.2 ❌ The assistant DOES NOT
| Request | Behaviour |
|---|---|
| Anything not about booking a table (weather, coding, jokes, general chat, other businesses) | Fixed refusal: "I can only help with table bookings at ResortOS Dining." |
| Show, change or cancel existing bookings | Directs the guest to sign in to the web app. Looking up bookings by email alone would let anyone read anyone's reservations |
| Admin data, other guests' data, prices/payments | Refused |
| Book without explicit confirmation | Never. The confirm step is enforced in code, not just in the prompt (§5.3) |
| Menu Q&A, allergy advice | Refused in v1 (see open decision D3) |

## 3. Architecture

```
 Guest (chat UI)                 resortos-booking-agent                 resortos-booking-api
 ────   POST /agents/users/chats ┌───────────────────────────┐   HTTP   ┌──────────────────────────┐
 messages +       ─────────────▶ │ 1. validate input (Zod)   │ ───────▶ │ GET /api/public/          │
 { email, phone? }               │ 2. topic guard            │          │ POST /api/agents/         │
                  ◀───────────── │ 3. agent + 2 tools        │ ◀─────── │      reservations (NEW)   │
 reply + booking?                │ 4. confirm-gate in code   │          └──────────────────────────┘
                                 └─────────────┬─────────────┘
                                               │ OpenAI-compatible HTTP
                                               ▼
                                        LLM endpoint (supplied by owner later)
```

* **Decoupled:** the agent talks to the backend **only over HTTP**, never to the database. It has its own types in `src/types/` (no imports from other repos).
* **Stateless server:** the client sends the conversation each turn. No chat database in v1.
* **Model-agnostic:** the endpoint URL, model name and key come from env (`LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY`). Any OpenAI-compatible endpoint works.

## 4. Interfaces

### 4.1 Agent API (this repo)
**Route groups (owner rule, 2026-11-04):** every agent server path starts with `/agents`, then a group, like the booking API. Names are plural.

| Group | Middleware | Routes |
|---|---|---|
| `/agents/public/*` | **none** | `GET /agents/public/health`, `GET /agents/public/docs`, `GET /agents/public/docs/openapi.json` |
| `/agents/users/*` | the chat checks: per-IP and per-email rate limits, the confirmation limit, body validation. **No sign-in yet** (the guest's email travels in the body); a sign-in check can be added to this group later without moving routes | `POST /agents/users/chats` |

`/agents/users/chats` is a collection so it can grow later (e.g. `GET /agents/users/chats/:id`); the protocol for that isn't decided. Today `POST /agents/users/chats` runs one chat turn and stores nothing.

`POST /agents/users/chats`
```ts
// request
{
  contact: { email: string; phone?: string };            // email required, validated by Zod, never parsed by the LLM
  messages: { role: "user" | "assistant"; content: string }[];   // max 30 turns, 2,000 chars each
  confirm?: PendingBooking;                               // sent back only when the guest taps "Book it" (a read-back or one of the alternatives)
}
// response 200
{
  reply: string;
  pendingBooking?: PendingBooking;   // details read back (draft + signed token), waiting for the guest's "yes"
  alternatives?: PendingBooking[];   // up to 3 bookable options when the request can't be seated (§13.1)
  booking?: { code: string; slotStart: string; slotEnd: string; tableIds: string[] };
}
```
Errors keep the chat shape so the chat screen can always show them: `{ reply }` (a sentence for the guest) with 400 (bad input or an expired read-back), 404 (unknown path), 429 (rate limits), 500, 502 (the AI model or the booking API failed) and 503 (busy, with `Retry-After`).

`GET /agents/public/health` → `{ status: "ok" }`

**Interactive docs (owner request 2026-11-04):** the agent server is an API too, so it publishes the same kind of docs as the booking API (`API_SPEC.md` §7a): `GET /agents/public/docs` (Swagger UI via `swagger-ui-express`) and `GET /agents/public/docs/openapi.json` (OpenAPI 3.1 built with `@asteasolutions/zod-to-openapi` from the Zod schema that validates `POST /agents/users/chats`). Every route has a plain-language summary and description, examples of each reply kind (a normal reply, a read-back with `pendingBooking`, a confirmed `booking`, a rate-limit reply), and every error. Shown in development and tests; in production only when `API_DOCS=true`. Same drift-guard test.

### 4.2 Tools the model can call (and nothing else)
| Tool | Input (Zod) | Calls |
|---|---|---|
| `checkAvailability` | `branch`, `sector`, `date` | `GET /api/public/availabilities` |
| `proposeBooking` | `branch`, `sector`, `date`, `startTime`, `partySize`, `specialRequests?` | Nothing. It validates and returns a `BookingDraft` for read-back |

The actual booking is **not** a model tool (§5.3).

### 4.3 Backend change (`resortos-booking-api`), approved
The backend learns a third role, **`AGENT`** (§5.5), and two agent-only endpoints:

| Method | Path | Permission | Purpose |
|---|---|---|---|
| POST | `/api/agents/reservations` | `reservation:create_on_behalf` | Book for a guest by email. Body: `{ email, phone?, branch, sector, date, startTime, partySize, specialRequests?, clientIp }`, header `Idempotency-Key` |
| POST | `/api/agents/guests/lookup` | `guest:read_one` | Read **one** guest by **exact** email (body `{ email }`, so the address stays out of URLs and access logs). Returns only `{ exists, upcomingAssistantBookings }` |

* Booking uses the same allocation, capacity and double-booking code as `POST /api/users/reservations` (shared function, not a copy).
* Data model (D1): `User.passwordHash` becomes optional (password-less **guest account**, found or created by email). Sign-up with a guest account's email **claims** it (sets the password), so earlier assistant bookings appear in "My bookings".
* Each reservation records **who booked it**: `source` (`APP` | `AGENT`) and `createdById` (the guest for app bookings, the agent's service account for assistant bookings).
* Audit: `CREATE_RESERVATION` with `actorId` = agent account, `metadata: { source: "AGENT", onBehalfOf: <guest id>, clientIp }`.
* `notify()` sends the confirmation to the guest email (still a log stub in v1).

### 4.4 Admin visibility (`resortos-booking-web`)
* Booking log cards show a **"Booked by AI assistant"** label on agent bookings. The label marks a real `source` value, so it's state, not decoration.
* New filter **Booked by: All / Guest app / AI assistant** (`?source=APP|AGENT` on `GET /api/admins/reservations`).
* Detail sheet adds a **Booked by** row: "AI assistant, on behalf of <guest email>" or "Guest (app)".
* Day view adds "of which N via AI assistant" under active bookings.
* CSV/JSON export adds a `source` column.

## 5. Safety & Guardrails

### 5.1 Topic guard (booking-only)
Three layers, so one failure doesn't open the gate:
1. **Intent tool:** the agent must call `notSupportedYet({ topic })` for anything that isn't a table booking. The server then replies with the fixed refusal, not model text, and logs the topic so later iterations can see what guests ask for. This is the extension point for future intents. It runs in the same model call as the agent, so it adds no latency or cost (a separate classifier call would add about 3 s per turn).
2. **System prompt** restricting the agent to booking help, with the rules from `TECHNICAL_DECISIONS.md` §2.
3. **Tools:** the model can only check availability and draft bookings. It has no way to reach other data.

### 5.2 Prompt injection
User text is treated as data. Whatever the message says ("ignore your rules…", "book all VIP rooms"), the limits in §5.3–5.4 are enforced in TypeScript and the backend.

### 5.3 Confirm-gate (enforced in code)
1. The model calls `proposeBooking` → the server returns `pendingBooking` and the reply reads it back.
2. The guest confirms with an explicit action ("Yes, book it" button), and the client sends the same draft back as `confirmDraft`. The server re-validates it, then **the server**, not the model, calls the backend.
3. A booking can only happen through `confirmDraft`. The model cannot trigger one.
4. Because guests don't sign in, a crafted `confirmDraft` is effectively a public booking request. The gate stops the *model* from booking unasked; the §5.4 limits and the backend rules are what protect capacity from a misbehaving *client*.

### 5.4 Abuse limits (no sign-in means anyone can type any email)
| Limit | Where |
|---|---|
| 10 chat requests / minute / IP | agent |
| **30 chat requests / 30 minutes per chat session (the guest's email)** (owner rule, 2026-11-03). The IP limit stays because an email can be made up | agent |
| 3 assistant bookings / hour / IP | agent |
| **1** upcoming assistant booking for an email that hasn't signed up; **3** once signed up (owner rules, 2026-11-03; more looks like spam). Over the limit the guest is told to sign up, cancel the existing booking in the app, then ask again. The assistant never cancels (the email is unverified) | backend |
| Party size and window rules | backend (existing) |
| Unverified email risk | See decision D2 |

### 5.5 RBAC: the agent as a principal
Three roles. Every route checks a **permission**, and roles map to permissions in one table in code:

| Permission | USER | ADMIN | AGENT |
|---|---|---|---|
| Own profile and own reservations (read, create, reschedule, cancel) | ✅ | | |
| Admin stats, booking log, detail, export (read-only) | | ✅ | |
| `reservation:create_on_behalf` (book for a guest email) | | | ✅ |
| `guest:read_one` (one guest per request, exact email, minimal fields) | | | ✅ |
| List / search guests or bookings ("everyone booked tonight") | | ✅ | ❌ |
| Public: availability, menu | ✅ | ✅ | ✅ |

* **Everything else is `403` for `AGENT`**: no reading or changing reservations, no admin routes, no profile, no listing or searching users. A test calls every other route with an agent token and expects `403`.
* **API key (approved 2026-11-03):** the seed creates the service account (`assistant@resortos.internal`, role `AGENT`, no password) and stores the SHA-256 of `AGENT_API_KEY`. The agent sends `X-Agent-Key: <AGENT_API_KEY>` on every call; no sign-in, no expiry. Rotate by changing the key in both `.env` files and re-running the seed. The account can't sign in, can't be created by sign-up and has no "My bookings".
* Routes check permissions, not how the caller authenticated.
* **One user at a time, never more (owner rule, 2026-11-03):**
  * Exact match only. The email is validated as an email and compared exactly (case-insensitive): no wildcards, prefixes or partial matches, and one email per request.
  * Minimal answer: `{ exists, upcomingAssistantBookings }`. No name, phone, booking details or IDs, because the email in a chat is **unverified** and whatever the agent reads can end up in the reply to whoever typed it.
  * No list, search, date or "tonight" queries exist for `AGENT`. Requests like "list everyone booked tonight" hit `notSupportedYet` in the agent and `403` in the API. Listing and searching stay **admin-only**.
  * Every lookup is audited (`VIEW_GUEST`, actor = agent account) and rate-limited (60 / minute for the agent account) so enumeration attempts are visible and slow.

### 5.6 Privacy
The email and phone are used only for the booking and its confirmation. They're never sent to the LLM; the model sees `contact: provided`. The agent logs no message content.

## 6. Framework Evaluation: Mastra vs LangGraph

| | **Mastra** (`@mastra/core` 1.81) | **LangGraph.js** (`@langchain/langgraph` 1.6) |
|---|---|---|
| Model | Agent + tools + workflows framework, TypeScript-first | Graph / state-machine orchestration, Python-first with a JS port |
| Tools | `createTool` with **Zod** schemas (our validation library) | Zod tools via `@langchain/core` |
| Models | Vercel AI SDK providers: any OpenAI-compatible endpoint in one line | LangChain chat model wrappers (`ChatOpenAI` with `baseURL`) |
| Our flow (guard → agent → confirm-gate) | Guard step + agent + plain TypeScript gate. Small | Nodes + edges + state + `interrupt()` for the confirm step. More concepts, more code |
| Local testing | Built-in dev studio to chat with the agent and inspect tool calls | LangGraph Studio (separate setup) |
| Fits an HTTP service | Yes (own server, or call the agent from any handler) | Yes (call the compiled graph from a handler) |
| Dependencies | `@mastra/core` + AI SDK provider | `@langchain/langgraph` + `@langchain/core` + model package |
| Runtime | ESM-only, Node ≥ 22.13 (we run 25) | ESM/CJS |
| Where it wins | One agent, a few tools, quick to read | Complex multi-step, branching, durable human-in-the-loop flows |

**Recommendation: Mastra.** Our assistant is one agent with two tools and a gate that lives in plain code. Mastra does that with the least code, in our language and with our validation library (Zod). LangGraph's strength is explicit graphs and durable pause/resume, which we don't need at this size. If the assistant later grows into multi-step workflows (waitlists, rebooking, staff handoff), Mastra workflows cover that before LangGraph becomes worth its overhead.

*KISS note:* Mastra is built on the Vercel AI SDK. If you'd rather have no framework, `ai` + `@ai-sdk/openai-compatible` alone can run this flow in about the same amount of code, without the dev studio.

## 7. Repo Layout (planned)
```
resortos-booking-agent/
  src/
    types/          # llm.ts (built), BookingDraft, ChatRequest, Slot, Branch... (own copies, decoupled)
    llm/            # cent_ai_api.ts (built): the ONLY provider-specific file; returns LlmProvider
    config.ts       # env: LLM_*, BOOKING_API_URL, AGENT_API_KEY (service account key), AGENT_DRAFT_SECRET, rate limits
    guard.ts        # topic classifier
    agent.ts        # Mastra agent + system prompt
    tools/          # checkAvailability, proposeBooking
    bookingApi.ts   # HTTP client for resortos-booking-api
    server.ts       # POST /agents/users/chats, GET /agents/public/health
  tests/
  README.md
```

## 8. Testing
See §14.

## 9. Decisions (owner answers, 2026-11-03)
| # | Decision | Answer |
|---|---|---|
| D1 | Guest bookings | A **password-less guest account** is found or created per email, and the booking is attached to it. If the guest later signs up with that email, sign-up **claims** the account (sets the password), so their assistant bookings show in "My bookings" automatically |
| D2 | Email verification | None. **Confirm straight away**, with the §5.4 limits |
| D3 | Menu / allergy questions | Ignored this iteration |
| D4 | Chat surface | **API only** for now |
| D5 | Framework | **Mastra** (open source) + TypeScript. No other framework |
| D6 | Test runner | Node's built-in runner (`node --test`), no extra dependency |
| D7 | LLM | Cent AI, one model: `deepseek-v4.1-flash` |
| D8 | Cent AI format | Decided by testing (§11.2c): **Messages** (`LLM_API_FORMAT=messages`) |
| D9 | Test budget | 5 AI calls for capability tests (used, §11.2c), then 5 for end-to-end |
| D10 | Fallback model | None for now |
| D11 | Key safety | Done by owner (spend limit and allowlist) |
| D12 | Data notice | Not needed while it's API-only; add with the chat screen |
| D13 | Language | **English only** |

**Extensibility:** only booking topics are handled now, but the design has an explicit *intent* step (§5.1) so other topics can get their own handlers later without reworking the booking flow.

## 10. Out of Scope (assistant v1)
Viewing, changing or cancelling bookings · menu Q&A · payments · languages other than English · voice · saved chat history · real email/WhatsApp delivery · staff-facing assistant.

## 11. LLM Provider: Cent AI
Checked 2026-11-03 against cent-ai.dev/docs (`api/responses`, `api/messages`, `api/models`, `authentication`, `guides/chat-vs-responses`, `reference/rate-limits`) and the live API.

### 11.1 The two formats we can use
| | **Messages** (Anthropic format) | **Responses** (OpenAI format) |
|---|---|---|
| Endpoint | `POST https://api.cent-ai.dev/v1/messages` | `POST https://api.cent-ai.dev/v1/responses` |
| SDK base URL | Official Anthropic SDK: `https://api.cent-ai.dev`. **AI SDK (ours): `https://api.cent-ai.dev/v1`** (`@ai-sdk/anthropic` appends `/messages`) | `https://api.cent-ai.dev/v1` |
| Auth header | `x-api-key: cai_…` + `anthropic-version: 2023-06-01` | `Authorization: Bearer cai_…` |
| Body | `model`, `messages`, **`max_tokens` (required)**, `system`, `tools`, `thinking`, `stream`, `cache_control` | `model`, `input`, `instructions`, `tools`, `reasoning`, `stream`, `store`, `previous_response_id` (extra params "sent to the model as is") |
| Tool calls come back as | `content` blocks of type `tool_use`, `stop_reason: "tool_use"` | `output` items of type `function_call` |
| Cost | `usage.cost` (stream: final `message_delta`) | `usage.cost` (stream: `response.completed`) |
| Extras | `POST /v1/messages/count_tokens` (free) | `previous_response_id` works only if the model keeps state. **Cent AI stores no conversation state** |
| Errors | Anthropic error shape | OpenAI error shape |
| Docs example model | `claude-opus-5` | `gpt-6-sol` |

Cent AI's guide: "Pick the one your tool or code already speaks." Same host, same pricing.

**Our choice:** decided by the model.
* **Claude model → Messages** (`LLM_API_FORMAT=messages`): Anthropic's own format, so tool use is native. Mastra uses `@ai-sdk/anthropic` with `baseURL` set.
* **GPT-style model → Responses** (`LLM_API_FORMAT=responses`): Mastra uses `@ai-sdk/openai`'s Responses model with `baseURL` set.
* Either way the server stays **stateless**: full history each turn, `store: false`, no `previous_response_id`, since Cent AI doesn't keep state.

### 11.2 Specifying the model
* Set `LLM_MODEL` in `.env` to an ID **exactly** as listed by `GET https://api.cent-ai.dev/v1/models` (Bearer auth). Unknown IDs return 404 "The model '…' does not exist or is not available."
* Each model in that list has `capabilities`, `context_window`, `pricing` and `status`. **Choose one whose capabilities include tools**, with status `operational`.
* The agent sends it as the `model` field on every request. Switching models is an `.env` change, plus `LLM_API_FORMAT` if the vendor family changes.

### 11.2b Live ping results (model `deepseek-v4.1-flash`)
Two tiny requests ("Reply with exactly: pong", output capped at 16 tokens), sent with curl:

| | Messages (`/v1/messages`) | Responses (`/v1/responses`) |
|---|---|---|
| Result | HTTP 200, `stop_reason: end_turn`, text "pong" | HTTP 200, `status: completed`, `output_text` "pong" |
| Latency | 3.3 s | 2.4 s |
| Tokens (in / out) | 27 / 11 (includes a `thinking` block) | 27 / 3 |
| Cost | $0.000020 (Cent AI minimum) | $0.000020 (Cent AI minimum) |
| Rate limit headers | `x-ratelimit-limit: 600`, `remaining: 599` | same |

Auth, base URL and model ID are correct; both formats work for this non-Claude model.

### 11.2c Capability tests (5 calls, tool calling)
Same booking question with one `checkAvailability` tool; then the tool result sent back.

| Call | Format | Result | Time | Tokens in / out |
|---|---|---|---|---|
| 1 | chat/completions + tool | `finish_reason: tool_calls`, correct args | 3.3 s | 383 / 89 |
| 2 | responses + tool | `function_call`, correct args | 2.1 s | 383 / 83 |
| 3 | messages + tool | `tool_use`, correct args | 3.4 s | **127** / 89 |
| 4 | messages, tool result → answer | `end_turn`, correct summary (17:00 full, 19:30 has 12) | 3.3 s | **165** / 148 |
| 5 | responses, tool result → answer | `completed`, same correct summary | 3.1 s | 543 / 129 |

All cost Cent AI's minimum ($0.00002 each). **Messages chosen (D8):** same quality at ~3.3× fewer input tokens per call, a gap that grows with every turn. Watch item: Messages replies may include `thinking` blocks. Replaying the assistant content unchanged worked in call 4. Responses is the fallback if Mastra's Anthropic provider has trouble with them.

Style finding: the model answers with markdown tables and em dashes. The system prompt asks for short plain sentences (the text reaches guests).

### 11.3 Keys, limits, costs
* Keys look like `cai_…` and are shown once at creation. Each key belongs to one project and can carry **daily/monthly spend limits, an IP allowlist and a model allowlist**. New keys have no limits, so set them.
* Keep keys server-side only: the API allows cross-origin calls, so a browser-exposed key is public.
* Rate limit: 600 requests/minute per key (`x-ratelimit-*` headers, `Retry-After`). 429 codes: `rate_limited`, `key_limit_exceeded`, `project_budget_exceeded`. The agent maps them to "Assistant is busy / unavailable, please try again".
* Prepaid, no auto-renewal: an empty balance makes the assistant fail. The agent logs `usage.cost` per request.

### 11.4 Where the key goes
`resortos-booking-agent/.env` (git-ignored; template in `.env.example`): `LLM_API_FORMAT`, `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL`.

### 11.5 Smoke test before building (curl only, a few cents)
1. `GET /v1/models` → confirm `LLM_MODEL` exists, is `operational` and lists tools.
2. One request in the chosen format with a single tool → expect a `tool_use` / `function_call`.
3. One plain request → check latency and `usage.cost`.
4. Topic-guard output: JSON from the model validated by Zod, with no reliance on provider-specific structured-output flags.

**Considerations:** Cent AI is a third-party reseller in front of the model vendors, so prompts pass through it. The design keeps guest email/phone out of prompts (§5.6).

## 12. Provider Adapter (built 2026-11-03)
The agent calls a provider-neutral interface, so changing AI vendors means replacing one file:

```ts
interface LlmProvider {
  complete(request: { system; messages; tools?; maxOutputTokens? }): Promise<{ text; toolCalls; stopReason; usage: { inputTokens; outputTokens; costUsd? } }>;
  listModels(): Promise<{ id; capabilities; status?; contextWindow?; ownedBy? }[]>;
}
```
* `src/types/llm.ts`: the neutral types (`LlmMessage`, `LlmTool`, `LlmToolCall`, `LlmResult`, `LlmError` with a `retryable` flag).
* `src/llm/cent_ai_api.ts`: `createCentAiProvider(config)` and `centAiConfigFromEnv()`. Translates to the Cent AI **Messages** or **Responses** format (per `LLM_API_FORMAT`), maps tool calls and tool results both ways, reads `usage.cost`, and normalises both error shapes (`invalid_api_key`, `rate_limited` + `Retry-After`, `project_budget_exceeded`, `timeout`, `network`, `bad_response`).
* **Another provider later:** add `src/llm/<provider>_api.ts` exporting a factory that returns `LlmProvider`; the agent code doesn't change.
* Tests: `tests/cent_ai_api.test.ts` (16 cases, `fetch` mocked, Node's built-in test runner). Live check: `npm run smoke` once `.env` has the key and model.

## 13. Failure Handling & Alternatives

### 13.1 Suggesting other sittings, rooms or dates
Alternatives are found **by code, not by the model**, so suggestions are always real availability. They cost no extra AI calls.

`findAlternatives({ branch, sector, date, startTime, partySize })` reads `GET /api/public/availabilities` (at most 6 lookups) and returns up to **3** options that can seat the whole party, in this order:
1. **Same day, same room, other sittings**, closest time first ("19:30 is full, 17:00 has room").
2. **Same day, same branch, other room** (Main Hall ↔ VIP), same sitting first.
3. **Same day, other branch**, same room and sitting.
4. **Next 3 days**, same branch, room and sitting.

Each option is a ready-to-confirm `BookingDraft`. The reply lists them as short plain sentences, and the client can show them as buttons. If nothing fits: "There's no room for 50 guests at Branch 1 in the next few days. Try a smaller group per booking, or another date." Over 50 guests is a rule, not a capacity problem; the reply explains the 50-guest limit.

### 13.2 What can fail, and what the guest sees
| # | Failure | Detected by | Guest sees | Booking? |
|---|---|---|---|---|
| F1 | Missing details (no date, no party size) | `proposeBooking` schema | The assistant asks for the missing piece | No |
| F2 | Rule broken: party > 50, past date, > 60 days ahead, not a fixed sitting | `proposeBooking` (same rules as the API) | Plain explanation plus the valid options ("Sittings are 11:00, 13:30, 17:00, 19:30") | No |
| F3 | Requested sitting already full | `proposeBooking` checks availability first | "That sitting is full" plus **alternatives (§13.1)** | No |
| F4 | Sitting filled between read-back and "yes" (race) | API returns `Insufficient capacity` / `Table overlap` on confirm | "Someone just took the last tables" plus **alternatives** | No |
| F5 | Email already has 2 upcoming assistant bookings | API per-email cap | "This email already has 2 upcoming bookings. Sign in to manage them." | No |
| F6 | Booking API down / 5xx / timeout | HTTP client | "The booking system isn't responding. Nothing was booked; please try again in a minute." (`502`) | **Never double-books** (F6a) |
| F6a | Timeout on confirm: did it book? | Idempotency key | Retrying the same `confirmDraft` returns the **same** reservation instead of a second one | At most once |
| F7 | LLM rate limited / timeout (**25 s per turn**, owner e2e finding 2026-11-03) | `LlmError.retryable`; the turn is aborted at 25 s | "The assistant is busy, please try again shortly." (`503` + `Retry-After`) | No |
| F8 | LLM budget used up (`project_budget_exceeded`, `key_limit_exceeded`) | `LlmError.code` | "The assistant is unavailable right now. You can book in the app." (`503`), with a server warning | No |
| F9 | Model returns broken tool arguments or loops | Zod on tool input; max **5** tool steps per turn | "Sorry, I didn't catch that. Could you rephrase?" | No |
| F10 | Tampered or expired `confirmDraft` | Draft token (§13.3) | `400 Booking details changed or expired. Please ask again.` | No |
| F11 | Off-topic question | `notSupportedYet` tool | Fixed refusal | No |

**Replies decided by code (e2e finding, 2026-11-03):** when `proposeBooking` refuses (any F2 rule or an F5 per-email limit), the guest gets that refusal text exactly. The model loop stops after `proposeBooking` either way, so a model-written explanation would never arrive.

**Per-turn log line (no message content):** `turn tools=<names> steps=<n> ms=<duration> status=<code>`, so slow or looping turns can be diagnosed.

### 13.3 Draft token (read-back = what gets booked)
`proposeBooking` returns the draft plus `draftToken = HMAC-SHA256(draft fields + email + expiry, AGENT_DRAFT_SECRET)`, valid for **15 minutes**. `confirmDraft` must carry the same token, so the guest books exactly what was read back. The token doubles as the **idempotency key** sent to the API (`Idempotency-Key` header, stored unique on the reservation), which is what makes F6a safe.

## 14. Testing
Every scenario (tested and still to test), the suites that cover it, and the live AI runs are in [TEST_SCENARIOS_ASSISTANT.md](TEST_SCENARIOS_ASSISTANT.md). Guest and admin scenarios without the assistant: [TEST_SCENARIOS_APP.md](TEST_SCENARIOS_APP.md).

## 15. Build Plan (after D1–D13 are answered)
Each step is test-first and ends in a commit; nothing ships to guests until step 6.

| # | Step | Repo | Done when |
|---|---|---|---|
| 0 | ✅ Provider adapter + live pings | agent | `cent_ai_api.ts` with 16 tests; both formats return 200 |
| 1 ✅ | Verify tool calling (D9) | agent | `npm run smoke` shows a tool call from the chosen model |
| 2 ✅ | Backend: `AGENT` role + permission table, agent service account, password-less guest accounts + sign-up claim, `source`/`createdById`, the two agent endpoints (per-email cap, `Idempotency-Key`), admin `source` filter | api | Playwright + Jest green, coverage still 100% |
| 2b ✅ | Admin UI: "Booked by AI assistant" label, filter, detail row, day-view count | web | Playwright UI green |
| 3 ✅ | Agent core: booking rules prompt (today's date in UTC+7 injected), topic guard, `checkAvailability` + `proposeBooking` tools, tool loop | agent | Deterministic tests with a scripted fake `LlmProvider` |
| 4 ✅ | Confirm-gate + draft token + `findAlternatives` + `POST /agents/users/chats` + rate limits + failure mapping (§13) | agent | Every §13.2 row and §14.2 threat tested |
| 5 ✅ | End-to-end E1–E3 with the real model (§14.4, max 5 calls) | agent | All pass, cost recorded |
| 6 | Chat screen in the PWA (D4) | web | Playwright UI tests with the agent mocked |

Ports: API 4000, web 5173, **agent 4300** (4100 is the API test server; 5180 the web test server).

## 16. End-to-End Results
See [TEST_SCENARIOS_ASSISTANT.md](TEST_SCENARIOS_ASSISTANT.md) §6 (live runs) and §7 (still to test with the real model).
