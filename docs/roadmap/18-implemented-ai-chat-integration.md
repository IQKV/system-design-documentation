# AI Chat Integration — Implementation Summary

**Status:** Implemented (v0.5)  
**Proposal ref:** [18-proposal-ai-chat-integration.md](18-proposal-ai-chat-integration.md)  
**Primary targets:** `foundation-ai-chat-service`, `foundation-gateway-service`, `foundation-ui-app`, `foundation-ui-platform-admin`  
**Related:** [Addon system](17-implemented-addon-system-for-feature-inclusion-exclusion.md)

---

## 1. Overview

v0.5 adds a sixth production microservice — `foundation-ai-chat-service` — that exposes an LLM-backed conversational API via **Spring AI 2.0** and **Ollama**, plus first-party UI addons for tenant chat and platform-admin session oversight.

The service follows the same platform contracts as IAM, Billing, Audit, and CMS: JWKS JWT validation, gateway routing, RFC 7807 errors, Liquibase migrations, Helm + Drone CI, and OpenAPI aggregation.

---

## 2. Delivered Outcome

### 2.1 Backend — `foundation-ai-chat-service`

| Capability             | Detail                                                                                                                      |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Spring AI 2.0 + Ollama | `ChatClient` with per-request `OllamaChatOptions` (model, temperature, `numPredict`); default model `llama3.1:8b`           |
| Session persistence    | `chat_sessions` + `chat_messages` in PostgreSQL **system schema** (not tenant-scoped); FK cascade delete on messages        |
| JWT resource server    | Validates IAM RS256 tokens via JWKS; tenant context resolved per request from headers/claims                                |
| Prompt controls        | `iqkv.ai.*` / env: `AI_SYSTEM_PROMPT`, `AI_MAX_INPUT_CHARS` (4000), `AI_MAX_OUTPUT_TOKENS` (1024), `AI_TEMPERATURE` (0.7)   |
| User APIs              | `POST /chat`, `GET /chat/sessions`, `GET /chat/sessions/{id}/messages`, `DELETE /chat/sessions/{id}` under `/api/v1/aichat` |
| Admin APIs             | `GET /admin/sessions` — cross-user session list (`PLATFORM_ADMIN` only)                                                     |
| Reachability probes    | Public `/ping`; tenant `/tenant/ping`; admin `/admin/ping`                                                                  |
| Error mapping          | `NonTransientAiException` → HTTP 502 `ProblemDetail` with Ollama error detail                                               |
| Layout                 | Flat bounded-context package `chat/` (CMS-style): domain, service, MyBatis mappers, REST resources, DTOs                    |

### 2.2 Gateway — `foundation-gateway-service`

| Change              | Detail                                           |
| ------------------- | ------------------------------------------------ |
| Route `aichat-api`  | Path `/api/v1/aichat/**` → AI Chat Service       |
| Response timeout    | `180000` ms (180s) — LLM inference latency       |
| OpenAPI aggregation | `/api/v1/aichat/api-docs` included in Swagger UI |
| Public paths        | `/api/v1/aichat/ping`, `/api/v1/aichat/api-docs` |

### 2.3 Tenant App addon — `platform-ai-chat`

| Item       | Detail                                                                     |
| ---------- | -------------------------------------------------------------------------- |
| Codebase   | `foundation-ui-app/src/addons/platform-ai-chat/`                           |
| Enablement | Addon loader / `VITE_ENABLED_UI_ADDONS` (addon system)                     |
| Nav        | Workspace nav item → `/addons/platform-ai-chat`                            |
| UI         | Session list + chat window; send message, continue session, delete session |
| API client | Calls `/api/v1/aichat/**` via shared HTTP client                           |

### 2.4 Platform Admin addon — `platform-ai-chat-sessions`

| Item      | Detail                                                                              |
| --------- | ----------------------------------------------------------------------------------- |
| Codebase  | `foundation-ui-platform-admin/src/addons/platform-ai-chat-sessions/`                |
| Authority | `PLATFORM_ADMIN` only (manifest `permissions.roles`)                                |
| Nav       | `/admin/addons/ai-chat-sessions`                                                    |
| UI        | Read-only global session list; inspect messages / model metadata — **no mutations** |
| API       | `GET /api/v1/aichat/admin/sessions` (+ message history as needed for inspection)    |

---

## 3. Data Model

Sessions and messages live in the AI Chat service database (`foundation_aichat`), **system schema** — users are global identifiers from IAM JWTs; rows are not partitioned into `t_{tenantKey}` schemas.

```sql
chat_sessions
├── id          UUID PK
├── user_id     UUID NOT NULL          -- from JWT user_id claim
├── title       VARCHAR(500)           -- first ~60 chars of first user message
├── model       VARCHAR(100) NOT NULL  -- e.g. llama3.1:8b
├── created_at  TIMESTAMPTZ NOT NULL
└── updated_at  TIMESTAMPTZ NOT NULL

chat_messages
├── id          UUID PK
├── session_id  UUID FK → chat_sessions(id) ON DELETE CASCADE
├── role        VARCHAR(20) NOT NULL   -- USER | ASSISTANT | SYSTEM
├── content     TEXT NOT NULL
└── created_at  TIMESTAMPTZ NOT NULL
```

---

## 4. API Surface (summary)

Base path: `/api/v1/aichat`

| Method   | Path                                  | Auth             | Notes                                                 |
| -------- | ------------------------------------- | ---------------- | ----------------------------------------------------- |
| `GET`    | `/ping`                               | public           | Reachability                                          |
| `GET`    | `/tenant/ping`                        | JWT + tenant     | Tenant resolution check                               |
| `GET`    | `/admin/ping`                         | `PLATFORM_ADMIN` | Admin auth check                                      |
| `POST`   | `/chat`                               | JWT              | Send message; omit `sessionId` to start a new session |
| `GET`    | `/chat/sessions`                      | JWT              | Caller's sessions (paginated)                         |
| `GET`    | `/chat/sessions/{sessionId}/messages` | JWT              | Owner-only history                                    |
| `DELETE` | `/chat/sessions/{sessionId}`          | JWT              | Cascade-deletes messages                              |
| `GET`    | `/admin/sessions`                     | `PLATFORM_ADMIN` | All sessions, cross-user                              |

Full request/response shapes: service `docs/api/README.md`.

---

## 5. Architecture Notes

- **No RabbitMQ domain events** in v0.5 — chat is request/response against Ollama; Audit does not auto-consume chat turns unless extended later.
- **Trust model unchanged** — IAM still issues RS256 JWTs; AI Chat is an OAuth2 resource server like Billing/CMS.
- **Long timeout only on AI Chat route** — other gateway routes keep default timeouts.
- **Addon packaging** — first concrete consumers of the UI addon system (see doc 17); features ship behind env-driven enablement without core SPA changes.

---

## 6. Deployment & Ops

- Dedicated Helm chart `foundation-ai-chat-service` (SIT / UAT / PRD values)
- Requires PostgreSQL + reachable Ollama endpoint
- Prompt and generation limits tunable via env vars without rebuild
- Demo stack: Gateway routes `/api/v1/aichat/**`; Ollama runs alongside the service in compose where configured

---

## 7. Out of Scope (intentionally deferred)

- Streaming / SSE token responses
- RAG, vector stores, tool/agent calling
- Tenant-schema isolation for chat history
- Plan-feature gating of AI Chat (`FeatureGate` / `RequiresPlanFeature`)
- Admin mutation of sessions (delete/ban from admin UI)
- Audit event emission for every chat turn
