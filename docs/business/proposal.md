# Business Proposal

## Problem

Building B2B SaaS requires solving infrastructure before building product: user authentication, multi-tenant data isolation, subscription billing, API security, audit logging, and tenant provisioning. Teams typically spend 4–6 months building these foundational systems before they can focus on their actual product features.

Existing solutions either provide only UI scaffolding (leaving infrastructure unsolved) or create vendor lock-in through managed services.

---

## What We Built

A complete SaaS infrastructure platform consisting of four microservices and two production React SPAs.

### IAM Service (`foundation-iam-service`)

- User registration with email verification; signup-by-invitation (72h expiring tokens)
- JWT RS256 authentication — 15-min access tokens, 7-day refresh tokens; JWKS endpoint for distributed validation
- Token exchange for tenant switching; JTI denylist + global signout timestamp for revocation
- Password reset with rate limiting (3 requests / 15 min); brute-force lockout (5 attempts / 15 min)
- Multi-organization membership — one user, multiple tenants, independent authorities per tenant
- Role-based access control: `TENANT_OWNER`, `ADMIN`, `MEMBER`; platform authority: `PLATFORM_ADMIN`
- Email invitations with configurable default authority; new users created on accept (email pre-verified)
- Avatar uploads via two-phase presigned S3/MinIO flow; old avatars auto-deleted
- In-app notifications — persisted to DB + real-time WebSocket push (STOMP/SockJS)
- Site-wide announcements with multi-lingual support; async fan-out to all users in batches
- Platform admin APIs: user CRUD, tenant CRUD, cross-tenant invitations, force-set password, ban/unban/unlock users
- User ban/unban: platform admin can ban/unban globally, tenant owner can ban/unban within tenant; banned users automatically logged out and receive email
- Member authority edit: tenant owner can update member authorities (TENANT_OWNER/MEMBER); cannot remove last TENANT_OWNER from tenant; cannot remove your own TENANT_OWNER authority if you are the last owner
- Transfer ownership: tenant owner can transfer ownership to another active member; old owner becomes MEMBER
- PostgreSQL schema-per-tenant with Liquibase migrations; `ROLLOUT_MODE` for B2B/B2C switch

### API Gateway (`foundation-gateway-service`)

- Spring Cloud Gateway (WebFlux) — single entry point for all platform services
- RS256 JWT validation via IAM JWKS; configurable public paths bypass auth
- Header sanitization: strips `X-User-*`, `X-Tenant-ID`, `X-Audit-*` before JWT processing — prevents identity spoofing
- Context propagation: user, authorities, tenant, correlation ID forwarded as typed headers
- Audit context: captures client IP and User-Agent as `X-Audit-IP` / `X-Audit-UA`
- Per-route, per-tenant request metrics; Grafana dashboard included
- Aggregated Swagger UI; security response headers on every response
- Platform mode guard: polls IAM rollout mode; blocks traffic with 503 on mismatch

### Billing Service (`foundation-billing-service`)

- `PaymentGatewayPort` hexagonal abstraction — Stripe adapter implemented; swap gateways without business logic changes
- Auto-provisions Stripe customer on `tenant.provisioned` event (RabbitMQ)
- Per-tenant `billing_settings`: billing email, tax ID/VAT, Stripe Customer Portal session
- Plan catalog CRUD (platform admin); subscription checkout and management (tenant owner)
- Refunds API — initiate and list refunds per tenant; platform admin refund overview
- Idempotent Stripe webhook ingestion; publishes lifecycle events to the platform event bus
- Grafana dashboard with business KPIs: revenue, active subscriptions, webhook health
- ShedLock-protected scheduled jobs for trial-ending and payment-overdue notifications

### Audit Service (`foundation-audit-service`)

- Passive observer — binds to the `iqkv.events` exchange; zero code changes in domain services for basic auditing
- Transforms domain events (`UserEvent`, `TenantEvent`, etc.) into a unified `AuditRecord` format
- Enriches records with client IP and User-Agent propagated from the Gateway
- Dedicated PostgreSQL database — high-volume logging isolated from business transactions
- SPI pattern (`foundation-audit-spi`) — plug in Elasticsearch or custom SIEM backends without touching core
- Secured admin search API — paginated, filterable by user, tenant, and action; restricted to `PLATFORM_ADMIN`

### Tenant UI (`foundation-ui-app`)

- React 19 + TypeScript + Mantine UI SPA for workspace members
- Sign-in with tenant discovery; sign-up with provisioning poll; forgot/reset password; email verification
- Accept invitations (`/invite/:token`) — new and existing users
- Dashboard, team member list, send/revoke invitations (`TENANT_OWNER`), ban/unban members (`TENANT_OWNER`)
- My Account — profile, password, organizations and roles; avatar upload
- Billing — portal access, active subscription, plan catalog, billing info, refunds
- Tenant settings — organization metadata editing
- In-app notifications with WebSocket support; notification bell UI
- Silent token refresh, 30-minute inactivity sign-out, light/dark theme, Lingui i18n

### Platform Admin UI (`foundation-ui-platform-admin`)

- React 19 + TypeScript + Mantine UI SPA for `PLATFORM_ADMIN` operators
- Dashboard — count cards for users, organizations, active subscriptions
- Users — paginated list, detail, edit profile, set password, ban/unban/unlock users
- Organizations — overview, members, billing settings, subscriptions, refunds tabs
- Invitations — propose, edit, revoke across all tenants; plan catalog CRUD
- Subscriptions (read-only global list + detail); refunds list and detail
- Announcements — create, edit, publish, delete with multi-lingual translation support
- Audit logs — global audit log view across all tenants
- In-app notifications with WebSocket support; operator account and password

### Deployment & Operations

- Kubernetes Helm charts per service with env-specific value files
- Docker Compose for local development (PostgreSQL, RabbitMQ, MailHog, MinIO per service)
- Drone CI/CD pipelines: verify → publish artifacts → publish image → deploy → promote
- Database migrations with Liquibase (system schema + per-tenant schema)
- Async tenant provisioning via RabbitMQ; ShedLock-guarded reaper for stuck tenants
- Prometheus + Grafana dashboards per service; structured JSON logging; correlation ID tracing

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
- Gateway strips all spoofable headers before JWT processing
- Avatar uploads via presigned S3 URLs — no binary data through application tier

### Async Processing

- RabbitMQ topic exchange (`iqkv.events`) for all platform events
- ShedLock for distributed job coordination across service instances
- Automatic cleanup of expired tokens, invitations, and lockout records
- Dead-letter exchange (`iqkv.dlx`) for failed message handling

---

## Current Status

**Implemented and Working:**

- Complete user authentication and authorization (IAM Service)
- Multi-tenant data isolation with PostgreSQL schemas
- Stripe subscription billing with refunds and Customer Portal (Billing Service)
- Centralized audit logging with passive event consumption (Audit Service)
- Reactive API gateway with JWT validation and audit context propagation (Gateway Service)
- Tenant-facing React SPA with billing, notifications, and team management (foundation-ui-app)
- Platform admin React SPA with full operator tooling including audit logs and announcements (foundation-ui-platform-admin)
- Kubernetes deployment automation with Helm charts
- Email notifications (verification, password reset, invitations, billing events)
- In-app notifications with real-time WebSocket delivery
- Async tenant provisioning with failure handling and retry

**Operational Features:**

- Health checks and readiness probes on all services
- Prometheus metrics collection with Grafana dashboards (per-service + business KPIs)
- Structured JSON logging with correlation IDs
- Database connection pooling and migration management
- Configuration management via Helm values

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

- **Backend**: Java 25, Spring Boot 4.0, MyBatis 3.x, PostgreSQL 17
- **Frontend**: React 19, TypeScript, Mantine UI 9, TanStack Router + Query, Vite + SWC
- **Infrastructure**: Kubernetes, Helm, Docker, RabbitMQ, MinIO (S3-compatible)
- **Security**: JJWT 0.13 (RS256), Spring Security OAuth2 Resource Server, BCrypt
- **Monitoring**: Micrometer, Prometheus, Grafana, Loki, structured JSON logging

### Scalability

- Stateless services for horizontal scaling
- Database read replicas and connection pooling
- Schema-per-tenant allows individual tenant migration to dedicated databases
- Event-driven architecture for async processing; dead-letter queue for resilience

### Deployment Options

- Kubernetes cluster with Helm charts (any cloud or on-premise)
- Local development with Docker Compose
- Environment-specific configurations (local, staging, production)
- Infrastructure-as-code with version control; no vendor lock-in

---

## What's Not Included

Current limitations and out-of-scope features:

- SSO / SAML integration (planned as extension; Jackson-compatible)
- Usage-based billing and metering
- Multi-region deployment
- Managed hosting service
- Advanced analytics and reporting (MRR/ARR dashboard planned)
- Member role editing beyond invitation default (planned)
- Platform actions: impersonation (planned; unlock/ban/unban implemented)
