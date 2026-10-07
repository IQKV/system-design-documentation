# Proposal: AI Chat Integration (Spring AI 2.0 + Ollama)

**Status:** Implemented (v0.5)  
**Date:** 2026-10  
**Primary targets:** `foundation-ai-chat-service`, `foundation-gateway-service`, `foundation-ui-app`, `foundation-ui-platform-admin`  
**Implementation record:** [18-implemented-ai-chat-integration.md](18-implemented-ai-chat-integration.md)  
**Depends on:** [Addon system](17-proposal-addon-system-for-feature-inclusion-exclusion.md)

---

## Implementation Checklist

### New Microservice — `foundation-ai-chat-service`

- [x] Bootstrap from `foundation-microservice-project-layout` / CMS flat bounded-context pattern (`chat/`)
- [x] Spring AI 2.0 + Ollama `ChatClient` integration
- [x] Liquibase: `chat_sessions`, `chat_messages` (FK cascade delete)
- [x] MyBatis mappers for sessions and messages
- [x] JWT OAuth2 resource server (IAM JWKS)
- [x] `AiChatProperties` / env: system prompt, max input chars, max output tokens, temperature
- [x] User REST: `POST /chat`, session list/history/delete under `/api/v1/aichat`
- [x] Admin REST: `GET /admin/sessions` (`PLATFORM_ADMIN`)
- [x] Reachability probes: `/ping`, `/tenant/ping`, `/admin/ping`
- [x] `NonTransientAiException` → HTTP 502 `ProblemDetail`
- [x] ArchUnit + integration tests; OpenAPI / SpringDoc
- [x] Helm chart + Drone CI pipeline; demo compose wiring

### Gateway — `foundation-gateway-service`

- [x] Route `aichat-api`: `/api/v1/aichat/**` → AI Chat Service
- [x] `response-timeout: 180000` (180s) for LLM latency
- [x] Aggregate `/api/v1/aichat/api-docs` into Swagger UI
- [x] Public paths: `/api/v1/aichat/ping`, `/api/v1/aichat/api-docs`

### Tenant App — `foundation-ui-app`

- [x] Addon `platform-ai-chat` under `src/addons/`
- [x] Nav item + route `/addons/platform-ai-chat`
- [x] Session list + chat window UI; API client via shared HTTP client

### Platform Admin — `foundation-ui-platform-admin`

- [x] Addon `platform-ai-chat-sessions` under `src/addons/`
- [x] `PLATFORM_ADMIN`-gated nav + route `/admin/addons/ai-chat-sessions`
- [x] Read-only global session list + message inspection (no mutations)

### Documentation

- [x] Service `docs/api` + `docs/deployment`
- [x] Platform README + system-design docs (architecture, capabilities, vision, this pair)

---

## 1. Executive Summary

Add a sixth production microservice that demonstrates **Spring AI** on the same platform contracts as IAM, Billing, Audit, and CMS: database-per-service, JWKS JWT validation, Gateway routing, RFC 7807 errors, Liquibase, Helm, and Drone CI.

Ship **first-party UI addons** (not core SPA features) so chat can be included or excluded via the addon system without changing Tenant App / Platform Admin core.

**Scope for v0.5:** synchronous chat against Ollama, session persistence, configurable prompt controls, and admin read-only oversight. Streaming, RAG, tools/agents, tenant-schema isolation, and plan-feature gating are explicitly deferred.

---

## 2. Goals

- Provide an LLM-backed conversational API that tenants can use from the Tenant App.
- Keep the **downstream trust model unchanged** — IAM issues RS256 JWTs; AI Chat is an OAuth2 resource server validating IAM JWKS only.
- Persist conversation history (sessions + messages) for UX continuity and operator oversight.
- Allow ops to tune system prompt, input limits, output token budget, and temperature **without rebuild**.
- Accommodate slow inference with a dedicated Gateway route timeout (target: up to ~180s).
- Give `PLATFORM_ADMIN` **read-only** visibility into all sessions (who chats, which model, history inspection).
- Package UIs as addons (`platform-ai-chat`, `platform-ai-chat-sessions`) using the v0.4/v0.5 addon infrastructure.

## 3. Non-Goals

- Streaming / SSE token responses
- RAG, vector stores, embeddings, or tool/agent calling
- Replacing Ollama with cloud providers in v0.5 (architecture should not preclude a later `ChatClient` swap)
- Tenant-schema isolation for chat tables (`t_{tenantKey}`) — v0.5 uses **system schema**, user-scoped rows
- Plan entitlement / `RequiresPlanFeature` gating for chat access
- Admin mutations (delete/ban sessions from admin UI)
- Publishing RabbitMQ domain events per chat turn (Audit auto-capture deferred)
- Changing IAM, Billing, or Audit domain contracts

---

## 4. Current State (pre-v0.5)

| Area                        | State                                                  |
| --------------------------- | ------------------------------------------------------ |
| Core services               | IAM, Gateway, Billing, Audit, CMS                      |
| UI addon infrastructure     | Implemented (doc 17); no concrete product addons yet   |
| LLM / Spring AI             | Not present in the monorepo                            |
| Long-running Gateway routes | Default timeouts unsuitable for multi-minute inference |

---

## 5. Proposed Architecture

```
Tenant App (addon platform-ai-chat)
Platform Admin (addon platform-ai-chat-sessions)
        │
        ▼
   API Gateway  ── /api/v1/aichat/**  (timeout 180s)
        │
        ▼
 foundation-ai-chat-service
   ├── JWT (JWKS from IAM)
   ├── ChatService → Spring AI ChatClient → Ollama
   └── PostgreSQL (chat_sessions, chat_messages)
```

### 5.1 Service packaging

Follow the **CMS flat bounded-context** layout under `com.iqkv.foundation.aichatservice.chat` (domain, service, mappers, REST, DTOs) rather than inventing a new layering style.

### 5.2 Data model

Own database `foundation_aichat`. Store chat data in the **system schema** (not schema-per-tenant):

| Table           | Purpose                                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------------- |
| `chat_sessions` | One row per conversation: `id`, `user_id`, `title`, `model`, timestamps                                             |
| `chat_messages` | Turns: `id`, `session_id` FK **ON DELETE CASCADE**, `role` (`USER`\|`ASSISTANT`\|`SYSTEM`), `content`, `created_at` |

**Rationale for system schema:** simpler first delivery; users are already global in IAM; admin oversight is cross-tenant by nature. Tenant isolation can be revisited if compliance requires schema-per-tenant chat history.

### 5.3 Prompt engineering controls

Bind `iqkv.ai.*` (overridable via env):

| Property / env                             | Default  | Role                                |
| ------------------------------------------ | -------- | ----------------------------------- |
| System prompt / `AI_SYSTEM_PROMPT`         | built-in | Injected on every `ChatClient` call |
| Max input chars / `AI_MAX_INPUT_CHARS`     | 4000     | Truncation / validation             |
| Max output tokens / `AI_MAX_OUTPUT_TOKENS` | 1024     | `numPredict`                        |
| Temperature / `AI_TEMPERATURE`             | 0.7      | Creativity                          |

Default Ollama model: `llama3.1:8b` (configurable with the client).

### 5.4 Error handling

Map non-transient Spring AI / Ollama failures to **HTTP 502** RFC 7807 `ProblemDetail` so clients can distinguish validation (4xx) from LLM backend outages.

---

## 6. API Design (proposed)

Base path: `/api/v1/aichat`

| Method   | Path                                  | Auth             | Description                                      |
| -------- | ------------------------------------- | ---------------- | ------------------------------------------------ |
| `GET`    | `/ping`                               | public           | Reachability                                     |
| `GET`    | `/tenant/ping`                        | JWT + tenant     | Tenant resolution check                          |
| `GET`    | `/admin/ping`                         | `PLATFORM_ADMIN` | Admin auth check                                 |
| `POST`   | `/chat`                               | JWT              | Send message; omit `sessionId` to create session |
| `GET`    | `/chat/sessions`                      | JWT              | Caller's sessions (paginated)                    |
| `GET`    | `/chat/sessions/{sessionId}/messages` | JWT              | Owner-only history                               |
| `DELETE` | `/chat/sessions/{sessionId}`          | JWT              | Cascade-delete messages                          |
| `GET`    | `/admin/sessions`                     | `PLATFORM_ADMIN` | All sessions, cross-user                         |

Session ownership: non-owners receive **404** (no existence leak across users).

---

## 7. Gateway Changes

| Change                       | Why                                   |
| ---------------------------- | ------------------------------------- |
| Route `aichat-api` → AI Chat | Path-based parity with other services |
| `response-timeout: 180000`   | Inference can take 60–180s            |
| Aggregate `api-docs`         | Unified Swagger                       |
| Public `/ping` + `/api-docs` | Smoke tests and OpenAPI without JWT   |

Header sanitization and JWT context propagation remain unchanged — AI Chat consumes the same `X-User-*` / `X-Tenant-ID` contract as CMS/Billing.

---

## 8. UI Addons

### 8.1 Tenant App — `platform-ai-chat`

- Manifest id `platform-ai-chat`
- Register workspace nav → `/addons/platform-ai-chat`
- UI: session sidebar + chat window (send, continue, delete)
- Enabled via `VITE_ENABLED_UI_ADDONS`

### 8.2 Platform Admin — `platform-ai-chat-sessions`

- Manifest id `platform-ai-chat-sessions`
- Restrict to `PLATFORM_ADMIN` in manifest permissions
- Nav → `/admin/addons/ai-chat-sessions`
- **Read-only** list + message modal; show model badge and timestamps
- No delete/edit from admin in v0.5

---

## 9. Deployment

- Dedicated Helm chart `foundation-ai-chat-service` (SIT/UAT/PRD values)
- Prerequisites: PostgreSQL, reachable Ollama endpoint (in-cluster or external URL)
- No SMTP secrets required
- Wire into monorepo `compose.demo.yaml` / demo scripts alongside Gateway

---

## 10. Security & Tenancy Notes

- Resource server only — no new token issuance path
- Admin endpoints require `PLATFORM_ADMIN`; user endpoints enforce session ownership
- System-schema storage means chat history is **not** isolated by PostgreSQL tenant schema; application-level ownership still applies for user APIs
- Do not expose Ollama directly to the public internet; only the Gateway → AI Chat path is public-facing

---

## 11. Success Criteria

- [x] Authenticated user can start/continue/delete chats via Tenant App addon
- [x] Sessions persist across reloads
- [x] Prompt/limits tunable via env without rebuild
- [x] Gateway does not time out under typical Ollama latency (≤180s)
- [x] Platform Admin can list all sessions and inspect messages read-only
- [x] Service meets platform quality gates (tests, OpenAPI, Helm, CI)

---

## 12. Follow-ups (out of v0.5)

- Streaming responses
- RAG / tools / agents
- Tenant-schema chat isolation
- Plan-feature gating
- Chat turn audit events
- Admin mutations and retention / purge policies
