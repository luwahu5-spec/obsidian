---
date: 2026-07-08
updated: 2026-07-10
source: Claude Code (Confluence PRD read via browser)
project: core-web / core-web-api
tags: [planning, prd, ai-assistant, chat, copilot-studio, rag, architecture]
---

# PRD "AI Assistant in CORE Web" — analysis & executable plan

> **Takeaway:** "frontend-only" is contradicted by the PRD itself — Web NFR #7 ("long term conversation storage shall be managed by backend services"), NFR #5 (no API keys/prompts in the client), NFR #6 (persistence configurable), and AI NFR #9 (audit logging of all prompts/responses) are all backend obligations. Industry practice for every embedded web AI assistant (Intercom Fin, Notion AI, GitHub Copilot Chat, MS Copilot) is the **backend proxy pattern**: the browser never talks to the AI provider; a thin backend owns keys, sessions, history, audit, rate limiting, and streams tokens through. The AI *model/RAG* being someone else's job doesn't remove the backend — it defines where the backend hands off. Appendix reveals the AI side is **Microsoft Copilot Studio**, which mandates server-side token minting (Direct Line) anyway.

## 1. The architecture (what "everyone else does")

```
Browser (Blazor UI components)
   │  SignalR circuit (Blazor Server — UI state lives server-side already)
CORE Web (Blazor Server)
   │  calls CORE API only — never the AI provider directly (Web NFR #1, #5)
CORE API — "AI Gateway" (the backend work product says doesn't exist)
   │  • mints short-lived AI-service tokens (Copilot Studio Direct Line requires this)
   │  • conversation + message persistence (Web NFR #7)
   │  • audit logging: prompt, response, metadata, userId, timestamp (AI NFR #9)
   │  • rate limiting / abuse control per user
   │  • streams response tokens back (SSE / relayed via SignalR)
AI Service (Copilot Studio / RAG — trained & governed by others)   ← the part we DON'T build
```

Why the browser can never call the AI directly: key custody (NFR #5), auditability (AI NFR #9), per-user authorization mapping (CORE identity → AI session), regional routing, and abuse control. Blazor Server softens this (component code already runs server-side), but the PRD's own Web NFR #1 requires "defined APIs" — the gateway belongs in CORE API, not in the web app's memory.

**Is the proxy pattern really the standard? (asked 2026-07-08)** Yes — universally: (1) every provider's docs mandate no keys client-side, and Copilot Studio's Direct Line auth is *designed* around backend token minting; (2) it became a product category ("AI gateway": Cloudflare AI Gateway, Azure APIM GenAI policies, Kong, LiteLLM); (3) every benchmark product (Intercom Fin, Notion AI, GitHub Copilot Chat) routes browser → own API → model — verifiable in DevTools; (4) the sole exception (real-time voice via WebRTC) still mints ephemeral backend tokens first. It's the BFF pattern applied to AI — the same "BFF routing layer" term the CORE Refactor §22 requirements already commit to.

## 1.5 "Won't routing everything through the backend be slow?" (asked 2026-07-08)

No — the proxy hop is **<1% of total time**. Budget: browser→web→API hops = 5–20 ms total; AI retrieval + first token = 500 ms–2 s; full answer generation = 5–20 s. The LLM is the bottleneck by four orders of magnitude; removing the backend saves nothing perceptible.

**Streaming is what makes it feel fast:** the LLM produces tokens word-by-word (20–100/sec); the gateway RELAYS each chunk the instant it arrives (never buffers the full answer) while copying the stream into audit/persistence as it passes — so persistence costs zero extra wait. The user sees the answer typing itself after ~1 s. Perceived latency = time-to-FIRST-token, which is why the PRD's "complete response ≤3 s" SLO must be renegotiated to a first-token SLO (AI-0.4).

Mechanics in our stack: browser↔web is the already-open SignalR circuit (no per-message handshake; `StateHasChanged` per chunk batch, throttled ~30-60 ms); web↔API is one streamed HTTP response per user message (`IAsyncEnumerable`/SSE, `HttpCompletionOption.ResponseHeadersRead` — standard .NET 8); API↔Copilot Studio is Direct Line over WebSocket. Every commercial assistant (ChatGPT included) relays every token through its own gateway exactly this way.

## 1.6 Plan B — if product insists on skipping CORE API (added 2026-07-08; context: lightweight model trained on the knowledge repo, self-hosted, CORE Web calls it)

**Saving grace:** CORE Web is Blazor Server — its "frontend" code runs server-side, so calling the model host directly from CORE Web keeps keys out of the browser (NFR5 ✓). The gateway responsibilities RELOCATE into CORE Web's server side rather than disappearing:
`AiAssistantService` (direct streamed calls, key in server config) + `IConversationStore` abstraction (`CircuitMemoryStore` now / `ApiConversationStore` later) + NLog audit target (interim).

**Losses to get signed off IN WRITING:**
1. No history/resume — CORE Web has no DB by design; chat dies on refresh/circuit drop. Product must explicitly accept "refresh loses the conversation".
2. Audit (AI NFR9, Must-Have) degrades to text logs — not queryable compliance storage.
3. Web NFR6/7 (backend-managed persistence) simply violated — same PRD, same author, author must reconcile.
4. Rate limiting in-memory per instance only.
Alternative keeping history without CORE API: the model-host service stores conversations itself — but then someone still built a backend, just in the AI team's yard. **"No backend" never removes the work; it relocates it (into CORE Web or the AI host) or cuts the features that need it.**

**Do NOW so nothing is wasted either way:** build UI against `IAiAssistantClient` + `IConversationStore` interfaces (swap DI when CORE API arrives — StubUserMiningMethodScopeService playbook); demand an **OpenAI-compatible streaming API** from the model host (vLLM/Ollama/TGI standard); settle server-to-server auth (API key/mTLS per environment/region) early; keep the time-to-first-token SLO renegotiation — small self-hosted models can be SLOWER per token under load, so capacity testing joins the checklist.

## 1.7 The "just use Microsoft Copilot" option, specifically (added 2026-07-08)

**Disambiguation:** M365 Copilot (Office assistant, per-employee license) is NOT embeddable for external customers — wrong product. The PRD appendix means **Copilot Studio** (low-code agent grounded on approved knowledge). Azure AI Foundry/OpenAI = the separate "train a lightweight model" track. ⚠️ The PRD appendix (Copilot Studio) and the "we'll train and host a lightweight model" message are **competing tracks — reconcile before building**; the integration differs substantially.

**What Copilot Studio delivers by configuration (honest credit):** RAG pipeline, SME-managed knowledge governance (AI FR4), citations (FR5), clarification (FR8), escalation topics (FR10), greeting (FR12), multilingual, prompts-don't-train-models (NFR7), moderation/prompt-injection hardening, admin analytics. Most of the AI requirements table — owned by the AI/product side.

**Two integration modes:**
- *Canned web-chat widget* ("frontend-only" in practice): fails CORE on (1) design system NFR4 — limited widget theming; (2) **identity — widget SSO expects Entra ID; CORE customers are in Keycloak** → effectively unauthenticated → breaks per-user audit (NFR9) + open token endpoint on Hexagon's message bill; (3) no user-facing history.
- *Direct Line API + our custom chat UI* (realistic): our Blazor panel speaks Direct Line. **Securing Direct Line REQUIRES a server-side secret→token exchange per Microsoft's own integration design** — the backend token endpoint is mandated by Microsoft, not by us (can live in CORE Web server-side per Plan B).

**Verify before committing:** Direct Line streaming support (historically full-message + typing indicator only → the 2-3s SLO renegotiation becomes MORE urgent); Keycloak↔Entra resolution (cleanest: our server piece authenticates via Keycloak and attributes users when minting tokens); Power Platform environment regions + Dataverse vs CORE regional deployments; message-based billing modeled against expected adoption; history/resume still needs OUR `IConversationStore` (Copilot transcripts are admin analytics, not a user API).

**Bottom line:** "just use Copilot" deletes the AI-team work (model/RAG/governance) — it deletes NONE of the web-side plan (custom UI, UX, sanitization, localization) and mandates a server token endpoint by Microsoft's design, plus our storage if history matters.

## 1.8 Entitlement — DECIDED (2026-07-08): Administrator role only
- Launcher in `MainLayout` behind `AuthorizeView Roles="Administrator"`; gateway endpoints behind `[Authorize(Roles = Roles.Administrator)]` — UI gating AND endpoint gating, always both. Administrator is the only role in CORE, so site/drill-plan users are excluded automatically.
- Shrinks rate limiting, Copilot message billing, storage volume, rollout risk; natural pilot cohort.
- ⚠️ Contradicts the PRD's own personas (Mine Engineers, Site Supervisors) and success criteria (self-service reducing Support load) — confirm this is the PILOT scope with widening planned, not the end state; if end state, product must reconcile their success criteria.
- Implement as ONE swappable rule (named policy e.g. `CanUseAiAssistant` → currently Administrator), referenced by launcher + gateway — widening later is a one-line change, not a hunt through components.

## 1.9 API key custody (asked 2026-07-08) — same lifecycle as our DB connection strings
- **At rest:** AWS Secrets Manager, one secret per environment (existing `Environments.Secrets` pattern — DB strings and IdentityServer signing keys already live there). Repo contains only the secret NAME.
- **Retrieval:** Beanstalk instance IAM role grants `GetSecretValue` on that one secret (no key-to-fetch-the-key problem); loaded at startup, in-memory only, periodic refresh so rotation needs no redeploy.
- **Use:** attached as an Authorization header on the gateway→AI server-to-server hop ONLY. Browser sends its Keycloak bearer token to the gateway; the AI key never exists browser-side.
- **Leak paths to close:** NLog must redact Authorization headers; never relay raw provider error bodies to the browser (map to FR6 friendly errors); key in header never query string.
- **Copilot Studio bonus:** the stored Direct Line secret is only used to mint short-lived, conversation-scoped per-user tokens — the browser-adjacent credential is disposable and expiring by design (this exchange IS the mandated backend piece).
- **Rotation/blast radius:** support two valid keys during a rotation grace window; scope the key to converse-only if the host supports it; per-user rate limiting at the gateway caps what any compromised session can burn.

**Gateway pseudo code (asked 2026-07-08) — the two credentials never share a file:**
```csharp
// core-web: AiAssistantRepository : BaseApiRepository — attaches USER's Keycloak token via TokenStore
//   (same as MiningMethodRepository); AI key does not exist in this project.
// core-web-api: AssistantController [Authorize(Bearer, Roles=Administrator)]
//   → user token CONSUMED here: userId = User.FindFirst(sub) — verified, not client-claimed
//   → _store.AppendMessage(...) = persistence + audit; ownership check (conversation.UserId == userId)
//   → _aiClient.Ask(...) — user token NOT forwarded; provider never learns Keycloak exists.
// core-web-api: AiProviderClient — the ONLY class with the AI key: fetched once from Secrets Manager
//   via IAM role at startup, lives in the typed HttpClient's default headers, server memory only.
// Copilot Studio variant: only AiProviderClient internals change (secret → mint short-lived
//   Direct Line token per conversation); controller and web layers untouched.
```
Mental model: the gateway is a border crossing where identities are exchanged — the user's token proves WHO asks and is spent at the border (becomes a userId on DB rows); the AI key proves Hexagon may ask the model and exists only on the far side. Browser compromise yields no AI key; AI-host log compromise yields no user credentials.

## 1.10 AI team statement (2026-07-08): "conversations will be saved through web portal for audit logging"
Ambiguous — two readings; disambiguating question sent/pending: *"who writes the record, where does it physically live, can Support/Legal query it by CORE user and date?"*
- **Reading A (likely): CORE saves them** — AI service answers and forgets; capture happens at our gateway → web → CORE API → SQL (= the AiConversation/AiMessage design; audit + history are the same rows). This is the AI team formally assigning persistence/audit to our side — "frontend only" is then contradicted by the AI team's own design, not just the PRD NFR7.
- **Reading B: their admin portal** (Copilot Studio transcripts in Dataverse) — admin analytics, NOT: a compliance store with our retention rules; accessible under CORE's permission model; user-facing history/resume; or per-user attributed unless identity crosses via our gateway anyway.
- Either way: somebody persists conversations → the "no storage, purely frontend" version of this project no longer exists; only the fence-side of the database is open.

## 2. Session, memory, history — the user's exact question

Three different lifetimes, three different owners — this is the framework to present:

| Layer                                                                                        | Lifetime           | Owner                                                                                                                                                       | Notes                                                                                                                                     |
| -------------------------------------------------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Active conversation context** (FR6: "maintain context throughout the active user session") | minutes–hours      | AI service conversation ID (Copilot Studio Direct Line conversations are stateful server-side, with expiry) + gateway maps CORE userId → conversationId     | Frontend holds only the rendered messages                                                                                                 |
| **UI state across page navigation** (Web FR2: open/close without leaving page)               | one Blazor circuit | scoped `ChatStateService` in CORE Web (survives navigation, DIES on circuit drop/refresh)                                                                   | Blazor-Server-specific trap: a refresh or reconnect banner kills in-memory chat — this alone makes persistence non-optional for decent UX |
| **History & resume** ("check the history chat and pick up where it left")                    | months             | **backend DB** — `AiConversation` (id, userId, title, createdAt, lastMessageAt) + `AiMessage` (conversationId, role, content, sources, feedback, timestamp) | Also the audit-log source (one write path serves both, with retention policy per compliance)                                              |

Resume = load messages from DB into the panel + either (a) reattach to a still-live AI conversation ID, or (b) start a fresh AI conversation seeded by replaying/summarizing stored history. (b) is the robust one — provider conversation state always expires eventually.

### 2.1 Session control decisions (from 2026-07-10 walkthrough) — three clocks, one decision each
1. **Blazor circuit**: scoped `ChatStateService`; on circuit re-establishment rehydrate the active conversation from storage (or product signs off on loss). **Multi-tab**: two tabs = two circuits = two scoped states — simplest defensible rule: each tab is its own conversation, shared history list.
2. **Keycloak session**: AI client wraps every call in the existing `TokenStore` + `RunOrRefresh` pattern (as `SiteDetails.razor.cs` does) — token refresh must be invisible mid-chat; if refresh truly fails, show "session expired" as a chat-level message, never a hard redirect that eats the user's draft.
3. **AI conversation**: reset button = abandon provider conversation ID, request new (AI FR7); on provider-side expiry ("conversation not found") transparently start a new AI conversation seeded from stored history instead of erroring; define an inactivity policy (panel opened after N idle hours → fresh conversation, yesterday's chat visible above as history).

### 2.2 Storage — exactly where and exactly how (from 2026-07-10 discussion)

**Where: the existing regional SQL Server DB via EF Core** — new `AiConversation` + `AiMessage` tables in `ApplicationDbContext`. Reasons: every pattern exists (migration via MigrationManager, repository + UnitOfWork, TestDatabaseFixture); retention enforced by the existing **purger-Lambda pattern** (LambdaPurger/MwdDataPurger precedents); **regional data residency falls out free** (each regional deployment has its own DB); volume trivial next to MWD sensor data; encryption at rest inherited from RDS.
- Alternatives (documented escape hatches, not v1): DynamoDB (partition userId/sort timestamp, TTL) if volume or serverless gateway demands; S3 for cold transcript archive; Dataverse (Copilot) = AI-team telemetry, never our persistence.
- **Anti-answers to veto in meetings:** browser localStorage — mining sites share terminals; localStorage survives logout → shift worker B reads worker A's chats; also fails audit/multi-device. "The AI provider stores it" — provider conversation state is an expiring context cache, not a durable per-user archive with a query API.

**How — DECIDED (2026-07-10): document model — ONE row per conversation, the whole transcript as a structured JSON column** (user's call: no per-message table/index; role + timestamp inside each element):
- Single table `AiConversation`: Id (GUID), UserId (Keycloak sub — **verified from the bearer token, never client-claimed**), Title (derived from first message), CreatedAt, LastMessageAt, State, DeletedByUser, **Messages (JSON array)**, **RowVersion (concurrency token)**.
- Message element shape: `{ "id": "m2", "role": "assistant", "ts": "2026-07-10T09:14:09Z", "content": "<raw markdown>", "sources": [...], "feedback": null }` — per-message `id` kept NOT for indexing but because FR10 feedback (and any future regenerate) must point at a specific message; positional references break on insert/trim. Store RAW markdown; render/sanitize at display time.
- **EF Core 8 maps this natively**: `OwnsMany(c => c.Messages, b => b.ToJson())` → strongly-typed `List<AiChatMessage>` persisted as the JSON column; one-row fetch, zero joins; the array IS the `{role, content}` wire format — replay-for-context is a property access.
- **Guard 1 — concurrent appends** (the design's main risk: read-array/append/last-writer-wins can silently eat a message, e.g. multi-tab): `RowVersion` optimistic concurrency — EF throws on conflict, gateway retries the append; the "each tab = own conversation" rule makes collisions rare anyway.
- **Guard 2 — blob rewrite growth** (every append rewrites the whole transcript): cap messages/conversation (~200) → panel starts a fresh conversation with the old in history; wanted anyway for model context-window trimming.
- Accepted prices: content-level audit search = `OPENJSON`/full-text scan not indexed lookup (fine at pilot volume; per-user/date audit uses normal columns); history-list query must SELECT metadata columns only, never the blob.
- **Ownership check on every access**: `conversation.UserId == token.UserId` → user B can't read/continue user A's chat by guessing IDs.
- **One row, two lifecycles**: user "delete/clear" sets `DeletedByUser` (hidden instantly); physical row survives until the audit retention window expires and the purger removes it — FR7 and AI NFR9 served by one write path. Retention window = the compliance knob Security/Legal set.
- **Replay = memory**: the Messages array both renders the panel AND feeds the model its context — LLM "memory" IS the replayed transcript; our row is the durable memory, the provider session only a cache. Long conversations: model receives summary-of-older + recent tail; the row keeps everything.

### 2.3 Requirement-by-requirement implementation notes (from 2026-07-10 walkthrough)
- FR1 launcher: `MainLayout`, behind the `CanUseAiAssistant` policy (§1.8). FR2: scoped service, not component state.
- FR3 input: Enter submits / Shift+Enter newline / disabled while in-flight / IME composition safety (pt/es/fr locales).
- FR4 ordering: key messages by server-assigned IDs + persisted timestamps, never render/arrival order (restore depends on it).
- FR6 errors: map every failure class (gateway down, AI timeout, rate-limited, **auth expired — chat-level message, no redirect**). Never render provider error bodies verbatim (leak + NFR5).
- FR7 "clear the VISIBLE history": three distinct operations — clear panel/new conversation (AI FR7), delete stored history (user-facing), audit rows (survive both). Label buttons honestly; get product to name which exist.
- FR8: JS interop auto-scroll with "user scrolled up → stop + jump-to-latest"; `Virtualize` for long conversations.
- FR9–11 (N2H): feedback thumbs need a STORED message ID — even the nice-to-haves presume persistence.
- NFR3: every AI call async + `CancellationToken` wired to stop/panel-close/circuit-disposal (orphaned streams leak).
- NFR8: health check gates the launcher; all AI calls wrapped — an unhandled exception in Blazor Server = the error banner on the WHOLE page, the most visible possible violation of "without affecting the remainder of CORE Web".

## 3. Execution plan

**Phase 0 — Contracts & decisions (do before ANY UI)**
- AI-0.1: Confirm the AI provider interface (Copilot Studio Direct Line vs custom RAG API). Everything downstream shapes around this.
- AI-0.2: Define the gateway API contract: `POST /assistant/conversations`, `POST /assistant/conversations/{id}/messages` (streamed response), `GET /assistant/conversations?userId` (history list), `GET /assistant/conversations/{id}` (messages), `DELETE .../{id}`, `POST .../messages/{id}/feedback`.
- AI-0.3: Decide streaming transport: gateway → web via SSE or chunked; web → browser rides the existing SignalR circuit (natural in Blazor Server — `StateHasChanged` per token batch).
- AI-0.4: Renegotiate the latency SLO: P50 ≤ 2s / P95 ≤ 3s for a COMPLETE RAG answer is unrealistic; industry SLO is **time-to-first-token** (streaming makes 10s answers feel instant). Get this redefined now, not at UAT.

**Phase 1 — MVP chat (backend gateway + frontend panel)**
- AI-1.1 (API): gateway endpoints, token custody, per-user auth, audit logging table + writes.
- AI-1.2 (Web): entry point (persistent launcher in `MainLayout` — visible on every page, Web FR1) + slide-over panel that overlays without navigation (FR2). Scoped `ChatStateService`.
- AI-1.3 (Web): chat UI — input (Enter + button, FR3), chronological messages (FR4), typing/loading indicator (FR5), friendly errors + retry (FR6-web), reset conversation (AI FR7), auto-scroll + responsive (FR8). CORE design system + all 7 resx locales.
- AI-1.4 (Web): **markdown rendering with strict sanitization** — AI responses will contain markdown; rendering raw HTML from an LLM is an XSS vector. Whitelist-based renderer, no raw HTML pass-through.
- AI-1.5: source-attribution UI (citations under responses — AI FR5; collapsible, per NFR "without impacting readability").

**Phase 2 — Persistence & resume (the "not just frontend" proof)**
- AI-2.1 (API): conversation/message persistence + history endpoints + retention policy + the NFR6 persistence config flag.
- AI-2.2 (Web): history list in the panel, resume, delete/clear (FR7-web: "clear visible history" vs delete stored — clarify with product which they mean).
- AI-2.3: circuit-resilience — on reconnect/refresh, panel restores the active conversation from the backend (this is where Blazor Server users feel the difference).

**Phase 3 — Polish & N2H**
- AI-3.1: suggested prompts (web FR9), feedback thumbs (FR10) wired to the feedback endpoint, response metadata display (FR11).
- AI-3.2: multilingual verification per locale; graceful-degradation feature flag (health check → launcher hides/disables with a friendly notice; assistant failure must NEVER trigger the Blazor error banner — wrap all AI calls, no unhandled circuit exceptions).
- AI-3.3: introduction message (AI FR12) & closing recognition (FR13) — these are AI-side behaviors; frontend just renders. Confirm ownership.

## 4. Difficulties & risks to raise

1. **The latency SLO vs RAG physics** (see AI-0.4) — the #1 thing to renegotiate. Related: "responses exceeding 5s = failures" would mark most honest RAG answers failed without streaming semantics.
2. **Blazor Server circuit fragility** — in-memory chat dies on refresh/disconnect; persistence (backend) is required even for "session-level" continuity, not just long-term history.
3. **XSS via AI markdown** — sanitization is a security requirement, not styling.
4. **Auth/identity mapping** — gateway must map CORE identity → AI session; anonymous relaying breaks the audit requirement. Also decide whether the assistant is entitled per site/role or global.
5. **Regional deployments + multilingual** — the AI service must exist in each deployment region (PRD assumption; verify per region before committing dates); 7 locales of UI strings; the AI's answer language ≠ UI language necessarily.
6. **"Clear visible history" (web FR7) vs stored history** — UI clear ≠ audit-log delete (audit must survive; NFR9). Make this explicit to avoid a privacy misunderstanding later.
7. **Feature flagging for rollout** — assistant should be per-site/per-environment flaggable for staged rollout (fits existing config patterns; also satisfies graceful degradation NFR).
8. **Who owns the system prompt/config** — "prompts shall not be exposed in client" implies prompts live server-side with the AI team; the gateway must not need them. Verify the Copilot Studio boundary.

## 5. What is genuinely frontend (the customer-facing checklist)

Launcher placement & unread/attention state; slide-over panel that never navigates; input UX (Enter submits, Shift+Enter newline, disabled while sending, char limit); message bubbles user/assistant with timestamps; streaming text rendering; typing indicator; sanitized markdown incl. code blocks & lists; source citations (collapsible chips/footnotes); error states (retry, offline, degraded); reset + history list + resume UI; suggested prompt chips; feedback thumbs; auto-scroll with "jump to latest" when user scrolled up; focus management & ARIA (chat log as live region, focus trap in panel, Esc closes); responsive down to tablet widths; dark-on-brand CORE design tokens; localization ×7; bUnit tests for every state (empty, loading, streaming, error, degraded, restored).

## Related
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — same "backend provides, frontend displays" principle applies to AI responses
- [[2026-06-12 When the Blazor Error Banner Appears]] — why the assistant must never throw into the circuit
