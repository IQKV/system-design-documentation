# Business Proposal

## Problem

Building B2B SaaS requires solving infrastructure before building product: user authentication, multi-tenant data isolation, subscription billing, API security, audit logging, and tenant provisioning. Teams typically spend 4–6 months building these foundational systems before they can focus on their actual product features.

Existing solutions either provide only UI scaffolding (leaving infrastructure unsolved) or create vendor lock-in through managed services.

---

## What We Built

A complete SaaS infrastructure platform consisting of six microservices, three production React/Astro applications, and one documentation site.

### IAM Service (`foundation-iam-service`)

- Self-service registration and tenant creation; signup-by-invitation (expiring tokens)
- JWT RS256 authentication — 15-min access tokens, 7-day refresh tokens; JWKS endpoint for distributed validation
- Magic link authentication with configurable TTL and rate limiting
- OAuth2 / OIDC federation — Google, GitHub, Microsoft, and tenant-scoped custom OIDC providers; IAM brokers external identities into the same internal JWT contract
- Token exchange for tenant switching; JTI denylist + global signout timestamp for revocation
- Password reset with rate limiting (3 requests / 15 min); brute-force lockout (5 attempts / 15 min)
- Multi-organization membership — one user, multiple tenants, independent authorities per tenant
- Role-based access control: `TENANT_OWNER`, `ADMIN`, `MEMBER`; platform authority: `PLATFORM_ADMIN`
- Email invitations with configurable default authority; new users created on accept (email pre-verified)
- Avatar uploads via two-phase presigned S3/MinIO flow; old avatars auto-deleted
- In-app notifications — persisted to DB + real-time WebSocket push (STOMP/SockJS)
- Site-wide announcements with multi-lingual support; async fan-out to all users in batches
- Platform admin APIs: user CRUD, tenant CRUD, cross-tenant invitations, force-set password, ban/unban/unlock users, linked-identity listing, forced unmerge
- User ban/unban: platform admin can ban/unban globally, tenant owner can ban/unban within tenant; banned users automatically logged out
- Member authority edit: tenant owner can update member authorities (TENANT_OWNER/MEMBER); guardrails on last owner
- Transfer ownership: tenant owner can transfer ownership to another active member; old owner becomes MEMBER
- Tenant SSO management: tenant owners can configure a custom OIDC provider; client secrets encrypted with AES-256-GCM
- PostgreSQL schema-per-tenant with Liquibase migrations; `ROLLOUT_MODE` for B2B/B2C switch
- Plan feature enforcement: `PlanFeatureGuard` annotation, `maxUsers` quota checks, `plan_code` stamped into JWT
- Custom metrics: auth outcomes, user lifecycle, tenant provisioning, security events

### API Gateway (`foundation-gateway-service`)

- Spring Cloud Gateway (WebFlux) — single entry point for all platform services
- RS256 JWT validation via IAM JWKS; configurable public paths bypass auth
- Header sanitization: strips `X-User-*`, `X-Tenant-ID`, `X-Audit-*`, `X-Plan-Code` before JWT processing — prevents identity and plan spoofing
- Context propagation: user, authorities, tenant, plan code, correlation ID forwarded as typed headers
- Audit context: captures client IP and User-Agent as `X-Audit-IP` / `X-Audit-UA`
- Per-route, per-tenant request metrics; Grafana dashboard included
- Aggregated Swagger UI; security response headers on every response
- Platform mode guard: polls IAM rollout mode; blocks traffic with 503 on mismatch
- Plan feature enforcement at route level via declarative filters

### Billing Service (`foundation-billing-service`)

- `PaymentGatewayPort` hexagonal abstraction — Stripe and Lemon Squeezy adapters implemented; swap gateways without business logic changes
- Auto-provisions customer (Stripe/Lemon Squeezy) on `tenant.created` event (RabbitMQ)
- Per-tenant `billing_settings`: billing email, tax ID/VAT, Customer Portal session
- **Plan catalog** defined in YAML and synchronized with active gateway at startup; supports `FLAT` and `PER_SEAT` pricing models (`METERED` usage-based pricing is deferred)
- Subscription checkout and management (tenant owner); trial period support
- **Per-seat pricing** — dedicated `PATCH .../seats` endpoint for mid-cycle seat adjustments with proration; seat-cap validation against `maxUsers`
- Refunds API — initiate and list refunds per tenant; platform admin refund overview
- Idempotent webhook ingestion (Stripe and Lemon Squeezy); publishes lifecycle events to the platform event bus
- Grafana dashboard with business KPIs: revenue, active subscriptions, webhook health
- ShedLock-protected scheduled jobs for trial-ending and payment-overdue notifications
- Custom metrics: MRR/ARR, payments, subscriptions, webhooks, seat adjustments

### Audit Service (`foundation-audit-service`)

- Passive observer — binds to the `iqkv.events` exchange; zero code changes in domain services for basic auditing
- Transforms domain events (`UserEvent`, `TenantEvent`, etc.) into a unified `AuditRecord` format
- Enriches records with client IP and User-Agent propagated from the Gateway
- Dedicated PostgreSQL database — high-volume logging isolated from business transactions
- SPI pattern (`foundation-audit-spi`) — plug in Elasticsearch or custom SIEM backends without touching core
- Secured admin search API — paginated, filterable by user, tenant, action, severity, and date range; restricted to `PLATFORM_ADMIN`
- Custom metrics: event consumption, persistence duration, search latency

### CMS Service (`foundation-cms-service`)

- Content management microservice for static page management
- Multi-language support with en-US fallback for internationalization
- Hierarchical content structure with parent-child page relationships
- SEO-friendly metadata (title, description, Open Graph tags, canonical URLs)
- Tenant isolation via schema-per-tenant PostgreSQL architecture
- Publishing status management (draft/published) for content workflows
- Event-driven architecture with RabbitMQ event publishing for content lifecycle events
- Public read-only API for fetching published pages
- Platform admin CRUD API for managing content

### AI Chat Service (`foundation-ai-chat-service`)

- LLM-backed conversational API via Spring AI 2.0 and Ollama (default model `llama3.1:8b`)
- Chat session and message persistence in PostgreSQL (system schema, user-scoped)
- JWT resource server validating IAM JWKS — same trust model as Billing and CMS
- Configurable prompt engineering via env: system prompt, max input chars, max output tokens, temperature
- User APIs: send message, list sessions, message history, delete session (`/api/v1/aichat`)
- Admin API: cross-user session list for `PLATFORM_ADMIN` oversight
- Gateway route with 180s response timeout for LLM inference latency
- First concrete consumers of the UI addon system (Tenant App chat + Platform Admin oversight)

### Tenant App (`foundation-ui-app`)

- React 19 + TypeScript + Mantine UI SPA for workspace members (Feature-Sliced Design architecture)
- Sign-in with tenant discovery, OAuth2/OIDC social login, enterprise SSO entry, sign-up with provisioning poll; forgot/reset password; email verification
- Accept invitations (`/invite/:token`) — new and existing users
- Dashboard, team member list, send/revoke invitations (`TENANT_OWNER`), ban/unban members (`TENANT_OWNER`)
- My Account — profile, password, organizations and roles; avatar upload; connected account linking / unlinking
- Billing — portal access (Stripe or Lemon Squeezy), active subscription (with trial status), plan catalog (with trial badges and per-seat pricing labels), billing info, refunds
- Tenant settings — organization metadata editing
- Security settings — tenant OIDC / SSO provider configuration for TENANT_OWNER
- In-app notifications with WebSocket support; notification bell UI and notification center
- Plan-based access control: `EntitlementsProvider`, `FeatureGate`, `useHasFeature`, `useQuota` hooks
- Env-driven UI addons — including `platform-ai-chat` conversational UI
- Silent token refresh, 30-minute inactivity sign-out, light/dark theme, Lingui i18n

### Platform Admin (`foundation-ui-platform-admin`)

- React 19 + TypeScript + Mantine UI SPA for `PLATFORM_ADMIN` operators (Feature-Sliced Design architecture)
- Dashboard — count cards for users, organizations, active subscriptions
- Users — paginated list, detail, edit profile, set password, ban/unban/unlock users, platform-authority management, OIDC identity remediation
- Organizations — overview, members, billing settings, subscriptions, refunds tabs
- Invitations — propose, edit, revoke across all tenants
- Subscriptions (global list + detail; cancel / pause / reactivate / quantity update); refunds list and detail
- Plan catalog — read-only list (plans are config-driven via YAML + deployment); announcements with multi-lingual translation support
- Audit logs — global audit log view across all tenants
- In-app notifications with WebSocket support; operator account and password
- Addon `platform-ai-chat-sessions` — read-only global AI chat session oversight
- Runtime config via `public/config.js` without rebuild

### SaaS Landing Kit (`foundation-ui-saas-landing-kit`)

- Astro + React + Tailwind CSS + shadcn/ui static site for marketing
- Home, Features, Pricing, About pages with responsive layout
- Auth-aware navigation and auth state management (Zustand)
- Plan selector with per-seat pricing support
- React islands for partial hydration

### Documentation Website (`foundation-docs-website`)

- VitePress-based documentation site with user guides and platform overview

---

## Key Features

### Hybrid Tenancy Model

Single codebase supports two deployment modes:

- **Multi-tenant**: Each signup creates a new organization with its own PostgreSQL schema
- **Single-tenant**: All users join a pre-configured default organization

Mode is controlled by `ROLLOUT_MODE` configuration — no code changes required. Switch between models at deploy time.

### Data Isolation

- Each tenant gets its own PostgreSQL schema (`t_tenantkey`)
- Automatic schema switching based on JWT claims via MyBatis interceptor
- System data (users, tenants) in public schema
- Cross-tenant queries prevented by design — wrong `search_path` returns no rows

### Security

- RS256 JWT tokens with JWKS endpoint for distributed validation
- Token revocation via JTI denylist and global signout timestamp
- Brute-force protection (5 attempts, 15-minute lockout)
- Rate-limited password reset (3 requests per 15 minutes)
- Gateway strips all spoofable headers (including `X-Plan-Code`) before JWT processing
- Avatar uploads via presigned S3 URLs — no binary data through application tier
- WebSocket authentication via JWT in STOMP CONNECT frames

### Async Processing

- RabbitMQ topic exchange (`iqkv.events`) for all platform events
- ShedLock for distributed job coordination across service instances
- Automatic cleanup of expired tokens, invitations, and lockout records
- Dead-letter exchange (`iqkv.dlx`) for failed message handling

### Plan & Entitlements System

- YAML-defined plan catalog synchronized with the active payment gateway (Stripe or Lemon Squeezy)
- `FLAT` and `PER_SEAT` pricing models with trial support (`METERED` deferred)
- `EntitlementsProvider` and `FeatureGate` in frontends
- `PlanFeatureGuard` annotation in backends
- `maxUsers` quota and feature flag enforcement
- `plan_code` stamped into JWT and propagated to all services

---

## Current Status

**v0.5 complete — AI Chat integration (Spring AI 2.0 + Ollama):**

- Sixth microservice: `foundation-ai-chat-service` — LLM chat API, session persistence, prompt controls, admin oversight API
- Gateway route `/api/v1/aichat/**` with 180s timeout; OpenAPI aggregation
- Tenant App addon `platform-ai-chat`; Platform Admin addon `platform-ai-chat-sessions` (read-only)
- Builds on completed v0.4 (identity federation, enterprise SSO, multi-gateway billing) and the UI addon system

**Also complete through v0.4:**

- Complete user authentication and authorization: JWT RS256, magic link, token exchange, RBAC, email verification, password reset, brute-force lockout, member ban/unban, ownership transfer (IAM Service)
- Multi-tenant and single-tenant data isolation with PostgreSQL schema-per-tenant (IAM and CMS)
- **Multi-gateway subscription billing**: Stripe and Lemon Squeezy, flat-rate and per-seat pricing, trial periods, seat-cap validation, mid-cycle seat adjustment, refunds, Customer Portal (Billing Service)
- Centralized audit trail with passive event consumption, JSONB storage, and SPI-based extensibility (Audit Service)
- Reactive API gateway: JWT validation, header sanitization, plan code propagation, audit context headers, per-tenant metrics (Gateway Service)
- Identity federation: OAuth2/OIDC social login, tenant-scoped enterprise SSO, account linking / unlinking, admin unmerge remediation (IAM + tenant/admin UIs)
- Content management system: static pages, multi-language support, hierarchical content, SEO metadata, tenant isolation (CMS Service)
- Tenant-facing React SPA: auth flows, social login, enterprise SSO, connected accounts, billing self-service, in-app notifications, plan-based feature access, AI Chat addon (foundation-ui-app)
- Platform admin React SPA: user/org/subscription/refund/announcement/audit log management, platform-authority tab, OIDC identities tab, AI Chat session oversight addon (foundation-ui-platform-admin)
- SaaS marketing landing kit with auth integration and plan selector (foundation-ui-saas-landing-kit)
- VitePress documentation website (foundation-docs-website)
- Kubernetes deployment with Helm charts across SIT / UAT / PRD environments; HPA configured
- Email notifications: verification, password reset, invitations, all billing lifecycle events (9 notification types)
- In-app notifications with real-time WebSocket delivery (STOMP/SockJS) and async streaming fan-out for announcements
- Async tenant provisioning with failure handling, stuck-tenant reaper, and owner-triggered retry
- Full CI/CD via Drone CI: 10 pipelines per Java service, 4 per frontend, 2 per library, 3 for infrastructure
- Prometheus + Grafana dashboards per service with business KPI tracking
- Loki + Promtail log aggregation in demo stack
- Structured JSON logging with correlation IDs propagated across all services
- Service template (`foundation-microservice-project-layout`) for adding new microservices

**Post-v0.5 items (platform hardening & AI follow-ups):**

- Platform Admin UI — subscription lifecycle mutations (change plan, apply discount); cancel / pause / reactivate / quantity update already shipped
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts)
- Additional locales (RU, IT; infrastructure already in place)
- OIDC audit history and automated test hardening
- AI Chat — streaming, RAG/tools, tenant-schema isolation, plan-feature gating

**Platform Numbers (October 2026):**

- 6 core microservices: IAM, Gateway, Billing, Audit, CMS, AI Chat
- 3 frontend applications: Tenant App, Platform Admin, SaaS Landing Kit
- 1 documentation website
- 5 PostgreSQL databases (IAM, Billing, Audit, CMS, AI Chat) — schema-per-tenant in IAM and CMS; AI Chat uses system schema
- 100+ REST endpoints across all services
- 20+ domain event types on RabbitMQ
- 60+ UI routes across Tenant App and Platform Admin (plus addon routes)
- 2 pricing models (`FLAT` and `PER_SEAT`) × 2 billing periods (MONTHLY/ANNUAL) + optional trial period per plan
- 2 payment gateways: Stripe and Lemon Squeezy (config-selectable)
- 2 deployment modes: MULTI_TENANT (SaaS) and SINGLE_TENANT (managed); zero-migration switch between them

---

## Target Users

Engineering teams (3–15 developers) building B2B SaaS applications who:

- Need multi-tenant architecture with real data isolation from day one
- Have Kubernetes deployment experience or want to acquire it
- Want to avoid vendor lock-in while getting production-ready infrastructure
- Prefer open-source solutions they can modify and extend

Works for both multi-customer SaaS platforms and single-tenant enterprise deployments.

---

## Business Model

**Open Core**: Core services are Apache 2.0 licensed and freely available.

**Extension Model**: The RabbitMQ event bus allows paid or community extensions to subscribe to platform events without modifying core services.

**Potential Revenue Streams:**

- Premium extensions (SAML SSO, advanced analytics, compliance tools, usage metering, deeper AI/RAG addons)
- Managed hosting for teams without Kubernetes expertise
- Enterprise support and custom development services (via iqkv.com)

---

## Technical Specifications

### Technology Stack

- **Backend**: Java 25, Spring Boot 4.1, Spring AI 2.0 (Ollama), MyBatis 3.x, PostgreSQL 17, Liquibase, RabbitMQ, MinIO, Redis, ShedLock
- **Frontend**: React 19, TypeScript 6, Mantine UI 9, TanStack Router + Query, Vite 8 + SWC, Zustand, Lingui 6, Zod, Vitest, Playwright, Astro, Tailwind CSS, shadcn/ui, VitePress, env-driven UI addons
- **Infrastructure**: Kubernetes, Helm, Docker, Nginx, Drone CI, Nexus, SonarQube, Ollama (LLM inference)
- **Security**: JJWT 0.13 (RS256), Spring Security OAuth2 Resource Server / Client, BCrypt strength 12
- **Monitoring**: Micrometer, Prometheus, Grafana, Loki, Promtail, structured JSON logging

### Scalability

- Stateless services for horizontal scaling
- Database read replicas and connection pooling (PgBouncer)
- Schema-per-tenant allows individual tenant migration to dedicated databases
- Event-driven architecture for async processing; dead-letter queue for resilience
- Production HPA: `minReplicas: 2`, `maxReplicas: 10`

### Deployment Options

- Kubernetes cluster with Helm charts (any cloud or on-premise) across SIT/UAT/PRD environments
- Local development with Docker Compose (full demo stack available)
- Environment-specific configurations (local, SIT, UAT, PRD)
- Infrastructure-as-code with version control; no vendor lock-in

---

## What's Next (Post-v0.5)

- Platform Admin UI — change plan and apply discount from the admin subscriptions surface
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Additional locales (RU, IT; infrastructure already in place)
- OIDC audit history and automated test hardening
- AI Chat — streaming responses, RAG / tools / agents, tenant-schema chat isolation, plan-feature gating

**Later milestones:**

- Platform Admin UI — system health dashboard, background job monitoring
- Platform Admin UI — impersonation
- SSO / SAML adapter
- Usage-based metered billing (`METERED` pricing model)
- Rate limiting (per-tenant and per-user, at Gateway)
- Tenant resolution by subdomain
- Multi-region support
- Managed hosting offering
