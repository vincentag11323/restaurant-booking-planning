# Booking Assistant (AI Agent) Specification
**ResortOS AI — Restaurant Booking App**

**Status:** PLAN, awaiting owner approval. No code yet.
**Repo:** `resortos-booking-agent` (third app repo, decoupled from `resortos-booking-api` and `resortos-booking-web`).
**Stack:** 100% TypeScript · Mastra (recommended, see §6) · Zod · own `src/types/` folder.

---

## 1. Summary
A chat assistant that answers **booking questions only** and can **book a table on the guest's behalf**. Guests don't sign in. They give an **email (required)** and optionally a **phone number**, chat in plain language ("table for 6 at Branch 2 on Friday evening?"), confirm, and receive a reservation code.

## 2. Scope

### 2.1 ✅ The assistant DOES
| Capability | Notes |
|---|---|
| Answer booking questions | Sittings (11:00, 13:30, 17:00, 19:30, 2.5 h each), branches, Main Hall vs VIP rooms, party size 1–50, the 60-day booking window, how tables are joined |
| Check live availability | Calls the API's public `GET /api/availability`. It never guesses or caches |
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
 ───────────────   POST /chat    ┌───────────────────────────┐   HTTP   ┌──────────────────────────┐
 messages +       ─────────────▶ │ 1. validate input (Zod)   │ ───────▶ │ GET /api/availability     │
 { email, phone? }               │ 2. topic guard            │          │ POST /api/agent/          │
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
`POST /chat`
```ts
// request
{
  contact: { email: string; phone?: string };            // email required, validated by Zod, never parsed by the LLM
  messages: { role: "user" | "assistant"; content: string }[];   // max 30 turns, 2,000 chars each
  confirmDraft?: BookingDraft;                            // sent back only when the guest taps "Yes, book it"
}
// response 200
{
  reply: string;
  pendingBooking?: BookingDraft;     // details read back, waiting for the guest's "yes"
  booking?: { code: string; slotStart: string; slotEnd: string; tableIds: string[] };
}
```
Errors use the same contract as the API: `{ message }` with 400 (bad input), 429 (rate limit), 502 (LLM or booking API unavailable).

`GET /health` → `{ status: "ok" }`

### 4.2 Tools the model can call (and nothing else)
| Tool | Input (Zod) | Calls |
|---|---|---|
| `checkAvailability` | `branch`, `sector`, `date` | `GET /api/availability` |
| `proposeBooking` | `branch`, `sector`, `date`, `startTime`, `partySize`, `specialRequests?` | Nothing. It validates and returns a `BookingDraft` for read-back |

The actual booking is **not** a model tool (§5.3).

### 4.3 Required backend change (`resortos-booking-api`), needs approval
`POST /api/agent/reservations`, authenticated with a **service key** (`X-Agent-Key` header, env `AGENT_API_KEY`), never callable by browsers.
```ts
{ email, phone?, branch, sector, date, startTime, partySize, specialRequests?, clientIp }
```
* Same allocation, capacity and double-booking rules as `POST /api/reservations` (shared code path).
* Data model (see decision D1): `Reservation.userId` becomes optional, and new columns `guestEmail`, `guestPhone`, `source` (`APP` | `AGENT`) are added.
* Audit: `CREATE_RESERVATION` with `metadata.source = "AGENT"` and the guest's IP.
* Admin log and export show "Booked via assistant" with the guest's email/phone.
* `notify()` sends the confirmation to the guest email (still a log stub in v1).

## 5. Safety & Guardrails

### 5.1 Topic guard (booking-only)
Three layers, so one failure doesn't open the gate:
1. **Classifier step** before the agent: a short structured-output call `{ onTopic: boolean }`. Off-topic → fixed refusal. The agent never runs.
2. **System prompt** restricting the agent to booking help, with the rules from `technical_decisions.md` §2.
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
| 3 assistant bookings / hour / IP | agent |
| Max 2 upcoming assistant bookings per email | backend |
| Party size and window rules | backend (existing) |
| Unverified email risk | See decision D2 |

### 5.5 Privacy
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
    config.ts       # env: LLM_*, BOOKING_API_URL, AGENT_API_KEY, rate limits
    guard.ts        # topic classifier
    agent.ts        # Mastra agent + system prompt
    tools/          # checkAvailability, proposeBooking
    bookingApi.ts   # HTTP client for resortos-booking-api
    server.ts       # POST /chat, GET /health
  tests/
  README.md
```

## 8. Testing Plan
* **Deterministic tests (no real model):** a fake model scripts the tool calls. Covers the topic refusal, availability lookups, draft → confirm → book, booking without confirmation (must never happen), backend errors surfaced as readable replies, rate limits, input validation.
* **Backend HTTP mocked** in agent tests. The new backend endpoint gets its own tests in `resortos-booking-api` (Playwright + Jest, keeping 100% coverage).
* **Eval set (real model, run manually once the endpoint is provided):** ~40 prompts: on-topic questions, off-topic requests, injection attempts, ambiguous dates ("next Friday"), party > 50, past dates. Pass = correct refusal/answer and no unconfirmed booking.
* Test runner: decided in D6.

## 9. Open Decisions (owner)
| # | Decision | Recommendation |
|---|---|---|
| D1 | How guest bookings are stored | Guest fields on `Reservation` (`userId` optional, `guestEmail`, `guestPhone`, `source`). No passwordless user accounts, which keeps sign-up/login untouched. Trade-off: these bookings won't appear in "My bookings" if the guest registers later |
| D2 | Unverified email: confirm immediately, or hold until the guest clicks an emailed link? | **Confirm immediately** with the §5.4 limits (email delivery is still a stub). Revisit when real email is added |
| D3 | Do "booking questions" include the menu and allergy questions? | **No for v1:** sittings, rooms, availability and booking only. Dietary notes are still accepted as special requests |
| D4 | Where guests chat | Agent repo is an HTTP service only. A small chat screen is added to `resortos-booking-web` as a separate, later task. Mastra's dev studio is used for testing until then |
| D5 | Framework | Revisit: with the provider adapter (§12) in place, the simplest option is **no framework** (our own ~40-line tool loop over `LlmProvider`). Mastra stays the choice if you want its dev studio, memory and workflows |
| D6 | Test runner for this repo | **Vitest** (native ESM + TypeScript, which suits ESM-only Mastra without extra transforms). Alternative: Jest + `@swc/jest` like the API, if you prefer one tool everywhere |
| D7 | LLM endpoint | **Cent AI** (cent-ai.dev, owner's provider). See §11. Owner supplies the key and model ID; nothing is hard-coded |

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

**Considerations:** Cent AI is a third-party reseller in front of the model vendors, so prompts pass through it. The design keeps guest email/phone out of prompts (§5.5).

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

