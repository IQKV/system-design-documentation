# Business Proposal

## Problem

Building B2B SaaS requires solving infrastructure before building product: user authentication, multi-tenant data isolation, subscription billing, API security, audit logging, and tenant provisioning. Teams typically spend 4–6 months building these foundational systems before they can focus on their actual product features.

Existing solutions either provide only UI scaffolding (leaving infrastructure unsolved) or create vendor lock-in through managed services.

---

## What We Built

A complete SaaS infrastructure platform consisting of five microservices, three production React/Astro applications, and one documentation site.

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
- Plan catalog defined in YAML and synchronized with active gateway at startup; supports `FLAT` and `PER_SEAT` pricing models
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

- YAML-defined plan catalog synchronized with Stripe
- `FLAT` and `PER_SEAT` pricing models with trial support
- `EntitlementsProvider` and `FeatureGate` in frontends
- `PlanFeatureGuard` annotation in backends
- `maxUsers` quota and feature flag enforcement
- `plan_code` stamped into JWT and propagated to all services

---

## Current Status

**v0.4 complete — identity federation, enterprise SSO, and multi-gateway billing:**

- Complete user authentication and authorization: JWT RS256, magic link, token exchange, RBAC, email verification, password reset, brute-force lockout, member ban/unban, ownership transfer (IAM Service)
- Multi-tenant and single-tenant data isolation with PostgreSQL schema-per-tenant (IAM and CMS)
- **Multi-gateway subscription billing**: Stripe and Lemon Squeezy support, flat-rate and per-seat pricing, trial periods, seat-cap validation, mid-cycle seat adjustment, refunds, Customer Portal (Billing Service)
- Centralized audit trail with passive event consumption, JSONB storage, and SPI-based extensibility (Audit Service)
- Reactive API gateway: JWT validation, header sanitization, plan code propagation, audit context headers, per-tenant metrics (Gateway Service)
- Identity federation: OAuth2/OIDC social login, tenant-scoped enterprise SSO, account linking / unlinking, admin unmerge remediation (IAM + tenant/admin UIs)
- Content management system: static pages, multi-language support, hierarchical content, SEO metadata, tenant isolation (CMS Service)
- Tenant-facing React SPA: auth flows, social login, enterprise SSO, connected accounts, billing self-service, in-app notifications, plan-based feature access (foundation-ui-app)
- Platform admin React SPA: user/org/subscription/refund/announcement/audit log management, platform-authority tab, OIDC identities tab with forced unmerge, enterprise theme, dashboard widgets, member signup trend chart, read-only plan catalog (foundation-ui-platform-admin)
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

Core v0.4 workstreams complete:

- OAuth2/OIDC social login in IAM with Google, GitHub, and Microsoft providers
- Tenant-scoped enterprise SSO with custom OIDC provider configuration
- Account linking / unlinking plus platform-admin linked-identity remediation
- Tenant-app OAuth callback handling, connected accounts UI, and enterprise SSO entry point
- Platform-admin OIDC identities tab and force-unmerge flow
- `LemonSqueezyGatewayAdapter` implementing the existing `PaymentGatewayPort` — all 11 gateway-agnostic methods
- `@ConditionalOnGateway` meta-annotation for clean `STRIPE` / `LEMON_SQUEEZY` bean wiring
- Gateway-neutral plan catalog configuration (`iqkv.billing.plan-catalog`); `externalVariantId` field for pre-configured LS variant IDs
- `LemonSqueezyWebhookRestResource` with HMAC-SHA256 signature verification (`X-Signature` header)
- Normalized webhook event mapping: LS event names → existing `GatewayWebhookEvent` sealed hierarchy
- `gateway_type` column on `billing_settings`, `subscriptions`, and `plan_catalog` for observability and future migrations
- `external_order_id` on `subscriptions` to support LS order-level refunds
- `BillingSeedRunner` hardening: LS adapter performs read-only variant verification instead of programmatic product creation

This release should be understood primarily as the identity-federation and enterprise-SSO release; Lemon Squeezy support is the parallel billing simplification track that completed at the same time.

**Post-v0.4 items (platform hardening):**

- Platform Admin UI — subscription lifecycle mutations (change plan, cancel, reactivate, apply discount)
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts)
- Per-seat IAM enforcement — enforce purchased `seatCount` at invite-accept and signup
- Additional locales (RU, IT; infrastructure already in place)

**Platform Numbers (June 2026):**

- 5 core microservices: IAM, Gateway, Billing, Audit, CMS
- 3 frontend applications: Tenant App, Platform Admin, SaaS Landing Kit
- 1 documentation website
- 4 PostgreSQL databases (IAM, Billing, Audit, CMS) — schema-per-tenant in IAM and CMS
- 100+ REST endpoints across all services
- 20+ domain event types on RabbitMQ
- 60+ UI routes across Tenant App and Platform Admin
- 2 pricing models (`FLAT` and `PER_SEAT`) × 2 billing periods (MONTHLY/ANNUAL) + optional trial period per plan
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

- Premium extensions (SAML/SSO, advanced analytics, compliance tools, usage metering)
- Managed hosting for teams without Kubernetes expertise
- Enterprise support and custom development services (via iqkv.com)

---

## Technical Specifications

### Technology Stack

- **Backend**: Java 25, Spring Boot 4.1, MyBatis 3.x, PostgreSQL 17, Liquibase, RabbitMQ, MinIO, Redis, ShedLock
- **Frontend**: React 19, TypeScript 6, Mantine UI 9, TanStack Router + Query, Vite 8 + SWC, Zustand, Lingui 6, Zod, Vitest, Playwright, Astro, Tailwind CSS, shadcn/ui, VitePress
- **Infrastructure**: Kubernetes, Helm, Docker, Nginx, Drone CI, Nexus, SonarQube
- **Security**: JJWT 0.13 (RS256), Spring Security OAuth2 Resource Server, BCrypt strength 12
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

## What's Next (Post-v0.4 Hardening)

- Platform Admin UI — change plan and apply discount from the admin subscriptions surface
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Per-seat IAM enforcement — enforce purchased `seatCount` against `activeSeatCount` on tenant
- Additional locales (RU, IT; infrastructure already in place)
- OIDC audit history and automated test hardening

**Later milestones:**

- Platform Admin UI — system health dashboard, background job monitoring
- Platform Admin UI — impersonation
- SSO / SAML adapter
- Rate limiting (per-tenant and per-user, at Gateway)
- Tenant resolution by subdomain
- Usage-based metered billing (`METERED` pricing model; per-seat flat pricing is complete)
- Multi-region support
- Managed hosting offering
