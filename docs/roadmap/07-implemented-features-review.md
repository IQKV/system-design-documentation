# Implemented Features Review — v0.2 Completion

> Snapshot of features implemented after May 2026 review, completing the v0.2 milestone: Administration, Self-Service & Observability.
> Sources: Service READMEs, UI app documentation, and platform capabilities audit.

---

## New Service: Audit Service

### Foundation Audit Service (`foundation-audit-service`)

**Tech stack:** Java 25 / Spring Boot 4.0 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · Micrometer + Prometheus

**Purpose:** Centralized microservice for platform-wide event consumption, transformation, and storage. Acts as the single source of truth for all activity logs across the platform.

#### Core Capabilities

| Feature                        | Detail                                                                                                                                                |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Passive observation            | Binds to existing `iqkv.events` exchange; zero code changes required in domain services for basic auditing                                            |
| Event transformation           | Maps domain-specific data (`UserEvent`, `TenantEvent`, etc.) into generic `AuditRecord` format with enriched technical context                        |
| Technical context enrichment   | Captures client IP addresses and User-Agents propagated from Gateway via `X-Audit-IP` / `X-Audit-UA` headers                                          |
| Storage isolation              | Dedicated PostgreSQL database; high-volume audit logging doesn't impact business-critical transactions                                                |
| Extensible architecture (SPI)  | Provider-friendly design via `foundation-audit-spi`; easy plug-in of alternative backends (Elasticsearch, custom SIEMs)                               |
| High-sensitivity tracking      | Consumes standard `AuditEvent` messages published by services for critical actions that don't trigger typical business events                         |
| Admin search API               | Secured, paginated, filterable API for `PLATFORM_ADMIN` to review audit trails across all tenants                                                     |
| JSONB metadata storage         | Dynamic event metadata stored as JSONB; custom MyBatis `TypeHandler` for flexible schema-less data                                                    |

#### API Endpoints

| Method | Path    | Auth             | Description                                                        |
| ------ | ------- | ---------------- | ------------------------------------------------------------------ |
| `GET`  | `/`     | `PLATFORM_ADMIN` | Search audit logs (paginated, filterable by user, tenant, action)  |
| `GET`  | `/{id}` | `PLATFORM_ADMIN` | Get detailed audit record including dynamic metadata               |

Base path: `/api/v1/audits`

#### Observability

- **Custom Metrics:**
  - `audit.event.consumption` — Rate of events consumed by type and source service
  - `audit.persistence.duration` — Latency of audit record storage operations
  - `audit.search.latency` — Performance of administrative log queries
  - `audit.storage.usage` — Volume of audit records persisted

---

## Backend Service Enhancements

### IAM Service — New Features

#### Token Exchange

| Feature         | Detail                                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| Endpoint        | `POST /auth/exchange`                                                                                                 |
| Purpose         | Tenant switching without re-authentication; user provides current JWT + target `tenantKey`                            |
| Authorization   | User must have active membership in target tenant                                                                     |
| Response        | New RS256 access + refresh token pair scoped to target tenant                                                         |
| Use case        | Multi-tenant users switching between workspaces without re-entering credentials                                       |

#### Avatar Uploads

| Feature                 | Detail                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Storage backend         | S3-compatible (AWS S3 or MinIO)                                                                                                       |
| Upload flow             | Two-phase: (1) Request presigned URL via `POST /users/me/avatar/upload-url`; (2) Client uploads directly to S3; (3) Confirm via `POST /users/me/avatar/confirm` |
| Auto-cleanup            | Old avatars automatically deleted when new avatar is uploaded                                                                         |
| Tenant isolation        | Avatar keys include tenant context; per-tenant S3 bucket prefixes                                                                     |
| Supported formats       | JPEG, PNG, WebP                                                                                                                       |
| Size limits             | Configurable via `iqkv.avatar.max-size-bytes` (default: 5MB)                                                                          |

#### Announcements

| Feature                  | Detail                                                                                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| Multi-lingual support    | Announcement content stored with translations (title + body per locale)                                                               |
| Async fan-out            | Batch processing (1000 users per batch) to avoid blocking; uses RabbitMQ for distribution                                             |
| Delivery tracking        | Tracks delivery status per user; supports retry on failure                                                                            |
| Admin API                | `POST /admin/announcements` — create; `GET /admin/announcements` — list; `PUT /admin/announcements/{id}` — edit; `DELETE /admin/announcements/{id}` — delete |
| Notification integration | Announcements automatically create in-app notifications for all active users                                                           |

#### In-App Notifications

| Feature                 | Detail                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Real-time push          | WebSocket integration; notifications pushed to connected clients immediately                                                          |
| Notification types      | System announcements, billing alerts, subscription changes, payment failures, trial expiry warnings                                   |
| User API                | `GET /users/me/notifications` — list (paginated); `PATCH /users/me/notifications/{id}/read` — mark as read; `DELETE /users/me/notifications/{id}` — dismiss |
| Unread count            | `GET /users/me/notifications/unread-count` — for notification bell badge                                                              |
| Persistence             | Stored in PostgreSQL; per-tenant schema isolation                                                                                     |

### Gateway Service — Audit Context Propagation

| Feature                     | Detail                                                                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `AuditContextFilter`        | New filter (order `-45`) extracts client IP and User-Agent from incoming requests                                                     |
| Downstream headers          | Injects `X-Audit-IP` and `X-Audit-UA` for consumption by domain services                                                              |
| Header stripping            | Strips `X-Audit-*` headers from incoming requests to prevent spoofing                                                                 |
| Service integration         | Domain services use shared `MessagingService` to enrich outbound events with audit context                                            |
| Monitoring filter           | New `MonitoringFilter` tracks per-tenant request metrics; integrates with Grafana dashboard                                           |

### Billing Service — Refunds & Customer Portal

#### Refunds API

| Endpoint                                  | Auth             | Description                                   |
| ----------------------------------------- | ---------------- | --------------------------------------------- |
| `GET /refunds/{tenantKey}`                | `TENANT_OWNER`   | List refunds for tenant                       |
| `GET /refunds/me`                         | Any authenticated | List refunds for current subject              |
| `GET /admin/refunds`                      | `PLATFORM_ADMIN` | Global paginated refund list                  |
| `GET /admin/refunds/count`                | `PLATFORM_ADMIN` | Total refund count                            |
| `GET /admin/refunds/{id}`                 | `PLATFORM_ADMIN` | Get refund by ID                              |
| `POST /admin/refunds`                     | `PLATFORM_ADMIN` | Issue refund (full or partial)                |

Refund fields: `id`, `subscriptionId`, `amountMinor`, `currency`, `reason`, `status` (PENDING|SUCCEEDED|FAILED), `stripeRefundId`, `createdAt`, `processedAt`

#### Stripe Customer Portal

| Feature                 | Detail                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Endpoint                | `POST /billing/{tenantKey}/portal-session`                                                                                            |
| Authorization           | `TENANT_OWNER` only                                                                                                                   |
| Purpose                 | Generate Stripe Customer Portal session URL for self-service billing management                                                       |
| Capabilities            | Update payment method, view invoices, download receipts, cancel subscription (if allowed by plan)                                     |
| Return URL              | Configurable via `iqkv.billing.portal-return-url`                                                                                     |

#### Grafana Dashboard Integration

- Pre-built Grafana dashboard for billing metrics
- Tracks: MRR, subscription churn, payment success/failure rates, refund volume
- Per-tenant and platform-wide views
- Prometheus metrics exported via `/actuator/prometheus`

---

## Frontend Application Enhancements

### Tenant App (`foundation-ui-app`) — New Features

#### Billing Self-Service

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Active subscription view    | ✅ Done  | Display current plan, billing period, next renewal date, amount                                                       |
| Plan catalog                | ✅ Done  | Browse available plans with feature comparison                                                                        |
| Stripe Customer Portal      | ✅ Done  | One-click redirect to Stripe-hosted billing portal for payment method updates, invoice history, subscription changes |
| Refunds list                | ✅ Done  | View refund history with status, amount, reason                                                                       |
| Billing info                | ✅ Done  | View and edit billing email, company name, tax ID, billing address                                                    |

#### Tenant Settings

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Organization metadata       | ✅ Done  | Edit tenant name, display name                                                                                        |
| Workspace configuration     | ✅ Done  | Basic tenant-level settings management                                                                                |

#### In-App Notifications

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Notification bell           | ✅ Done  | Header icon with unread count badge                                                                                   |
| Notification dropdown       | ✅ Done  | Recent notifications with mark-as-read and dismiss actions                                                            |
| Real-time WebSocket push    | ✅ Done  | Notifications appear instantly without page refresh                                                                   |
| Notification center         | ✅ Done  | Full-page view with pagination, filtering by read/unread status                                                       |

#### Routes Added

`/billing` · `/billing/subscription` · `/billing/refunds` · `/settings` · `/notifications`

---

### Platform Admin (`foundation-ui-platform-admin`) — New Features

#### Audit Logs

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Global audit log view       | ✅ Done  | Paginated, sortable, filterable audit trail across all tenants                                                        |
| Search filters              | ✅ Done  | Filter by user, tenant, action type, date range                                                                       |
| Audit detail view           | ✅ Done  | Full audit record with JSONB metadata expansion                                                                       |
| Export capability           | 📋 Planned | CSV/JSON export for compliance reporting                                                                              |

#### Announcements Management

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Create announcement         | ✅ Done  | Multi-lingual editor (title + body per locale)                                                                        |
| Announcement list           | ✅ Done  | Paginated list with status (draft, published, archived)                                                               |
| Edit announcement           | ✅ Done  | Update content, translations, publish/unpublish                                                                       |
| Delete announcement         | ✅ Done  | Soft-delete with confirmation                                                                                         |
| Delivery tracking           | ✅ Done  | View delivery status per announcement (total sent, delivered, failed)                                                 |

#### Refunds Management

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Global refund list          | ✅ Done  | Paginated, sortable, filterable by tenant, status, date range                                                         |
| Refund detail view          | ✅ Done  | Full refund record with subscription context, Stripe refund ID                                                        |
| Issue refund                | ✅ Done  | Create full or partial refund with reason; integrates with Stripe API                                                 |

#### Organization Detail Enhancements

| Tab             | Status   | Detail                                                                                                                |
| --------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Overview        | ✅ Done  | Tenant metadata, status, creation date, member count                                                                  |
| Members         | ✅ Done  | Paginated member list with roles                                                                                      |
| Billing         | ✅ Done  | Billing settings, Stripe customer ID, billing email, tax info                                                         |
| Subscriptions   | ✅ Done  | Active and historical subscriptions for tenant                                                                        |
| Refunds         | ✅ Done  | Refund history for tenant                                                                                             |

#### In-App Notifications

| Feature                     | Status   | Detail                                                                                                                |
| --------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| Notification bell           | ✅ Done  | Header icon with unread count badge                                                                                   |
| Notification dropdown       | ✅ Done  | Recent notifications with mark-as-read and dismiss actions                                                            |
| Real-time WebSocket push    | ✅ Done  | Notifications appear instantly without page refresh                                                                   |

#### Routes Added

`/admin/audit-logs` · `/admin/audit-logs/:id` · `/admin/announcements` · `/admin/announcements/new` · `/admin/announcements/:id` · `/admin/refunds` · `/admin/refunds/:id` · `/admin/organizations/:tenantKey/subscriptions` · `/admin/organizations/:tenantKey/refunds`

---

### SaaS Landing Kit (`foundation-ui-saas-landing-kit`)

**Tech stack:** Astro · React · Tailwind CSS · shadcn/ui · Zustand · TypeScript

**Status:** ✅ Shipped

| Feature                     | Detail                                                                                                                |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Static pages                | Home, Features, Pricing, About                                                                                        |
| Responsive layout           | Mobile-first; `BaseLayout.astro` base template                                                                        |
| Auth-aware navigation       | `TopNav.tsx` shows Login/Sign Up when unauthenticated; user menu with avatar + logout when authenticated              |
| Auth state                  | Zustand store with `localStorage` persistence                                                                         |
| React islands               | Partial hydration for interactive components                                                                          |
| Code quality                | OxLint, OxFmt, Husky pre-commit hooks, commitlint                                                                     |

---

## Cross-Cutting Platform Enhancements

### Messaging (RabbitMQ) — New Events

| Event                           | Publisher | Consumer(s)                                    |
| ------------------------------- | --------- | ---------------------------------------------- |
| `announcement.created`          | IAM       | IAM (async fan-out to users)                   |
| `announcement.published`        | IAM       | IAM (notification creation)                    |
| `notification.created`          | IAM       | IAM (WebSocket push)                           |
| `refund.issued`                 | Billing   | IAM (notification), Audit (logging)            |
| `refund.succeeded`              | Billing   | IAM (notification), Audit (logging)            |
| `refund.failed`                 | Billing   | IAM (notification), Audit (logging)            |
| `audit.event` (generic)         | All       | Audit (centralized logging)                    |

### Observability Enhancements

| Component                       | Enhancement                                                                                                           |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Gateway monitoring filter       | Per-tenant request metrics (request count, latency, error rate); Prometheus export                                    |
| Grafana dashboards              | Pre-built dashboards for Gateway (per-tenant traffic), Billing (MRR, churn, refunds), Audit (event volume)           |
| Audit service metrics           | Event consumption rate, persistence latency, search performance, storage usage                                        |
| Structured logging              | All services emit JSON logs with correlation ID, tenant context, user context                                         |

### Security Enhancements

| Feature                         | Detail                                                                                                                |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Audit context propagation       | Gateway injects `X-Audit-IP` / `X-Audit-UA`; services enrich events with client context                               |
| Header sanitization             | Gateway strips `X-Audit-*` headers from incoming requests to prevent audit context spoofing                           |
| Avatar upload security          | Presigned URLs with expiry; S3 bucket policies enforce tenant isolation                                               |
| WebSocket authentication        | JWT-based WebSocket handshake; per-tenant channel isolation                                                           |

---

## Infrastructure & DevOps

### Docker Compose Enhancements

| Enhancement                     | Detail                                                                                                                |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Audit service integration       | Added `foundation-audit-service` to `compose.demo.yaml` with dedicated PostgreSQL database                            |
| MinIO for avatar storage        | S3-compatible object storage for local development; pre-configured buckets                                            |
| Grafana + Prometheus            | Pre-configured Grafana instance with platform dashboards; Prometheus scrapes all service `/actuator/prometheus`       |
| WebSocket support               | Nginx reverse proxy configuration for WebSocket upgrade headers                                                       |

### Helm Chart Updates

| Chart                           | Enhancement                                                                                                           |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `foundation-audit-service`      | New Helm chart with PostgreSQL dependency, RabbitMQ connection, health probes                                         |
| IAM chart                       | Added S3/MinIO configuration for avatar uploads; WebSocket ingress annotations                                        |
| Gateway chart                   | Added audit context filter configuration; monitoring filter toggle                                                    |
| Billing chart                   | Added Stripe Customer Portal configuration; refund API feature flag                                                   |

---

## Documentation Updates

### New Documentation

| Document                        | Purpose                                                                                                               |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `local-development.md`          | Comprehensive guide for Docker-based local development (per-service and full demo stack modes)                        |
| `audit-service-architecture.md` | Deep dive into audit service design, SPI pattern, storage backends                                                    |
| `websocket-integration.md`      | WebSocket setup, authentication, channel isolation, client integration                                                |

### Updated Documentation

| Document                        | Updates                                                                                                               |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `capabilities.md`               | Added audit service, IAM enhancements (token exchange, avatars, announcements, notifications), billing refunds        |
| `compliance.md`                 | Added audit trail section, expanded regulatory alignment, object storage security                                     |
| `proposal.md`                   | Expanded "What We Built" from 3 to 6 components; updated current status                                               |
| `vision.md`                     | Marked v0.2 as completed; updated Post-v0.2 roadmap                                                                   |
| Service READMEs                 | All service READMEs updated with new endpoints, features, configuration options                                       |

---

## Summary: v0.2 Completion

The v0.2 milestone delivered a **fully manageable platform** with comprehensive administration tooling, tenant self-service capabilities, and centralized observability:

### Key Achievements

1. **New Audit Service** — Centralized, event-driven audit trail with SPI-based extensibility
2. **Platform Administration** — Complete CRUD for users, organizations, plans, subscriptions, refunds, announcements, audit logs
3. **Tenant Self-Service** — Billing portal, subscription management, refunds, workspace settings, notifications
4. **IAM Enhancements** — Token exchange, avatar uploads, announcements, in-app notifications with WebSocket push
5. **Observability** — Grafana dashboards, per-tenant metrics, audit event tracking
6. **SaaS Landing Kit** — Production-ready landing page template with auth integration

### Platform Maturity

- **Services:** 4 core microservices (IAM, Gateway, Billing, Audit) + 3 UI applications
- **API Endpoints:** 100+ REST endpoints across all services
- **Event Types:** 20+ domain events on RabbitMQ event bus
- **UI Routes:** 50+ routes across Tenant App and Platform Admin
- **Documentation:** 15+ comprehensive docs covering architecture, compliance, development, deployment

The platform is now **production-ready for B2B SaaS** with multi-tenant or single-tenant deployment modes, comprehensive administration, and self-service capabilities.

---

## What's Next: Post-v0.2

See [Roadmap](vision.md) for deferred features including:
- Advanced admin capabilities (ban/unban, impersonation, subscription mutations)
- Enhanced metrics (MRR/ARR, growth charts)
- SSO/SAML integration
- Rate limiting
- Subdomain-based tenant resolution
- Usage-based billing metering
