# Architecture

## Tenancy & Isolation

The platform uses a **Hybrid Tenancy Model** that supports both public SaaS (Multi-Tenant) and internal/enterprise (Single-Tenant) deployments using the same codebase.

### Platform Tenant (Common to All Modes)

A predefined **Platform Tenant** (tenant_key: `platform`, schema: `t_platform`) is always present in both deployment modes:

- **Single-Tenant Mode**: Acts as the default single source of truth tenant (all users operate within this tenant).
- **Multi-Tenant Mode**: Acts as an internal, hidden tenant for platform operations, accessible to users with `PLATFORM_ADMIN` authority.
- **All Modes**: Every user is automatically added as a `MEMBER` of the Platform Tenant, regardless of how they join the platform (signup or invitation).

### Deployment Archetypes

- **Multi-Tenant (Default):** Every registration creates a new organization with a unique 8-character NanoID key using alphabet `[a-z0-9]`.
- **Single-Tenant:** Tenancy is hidden. All users are automatically joined to the Platform Tenant (used as default tenant).

### Isolation Strategy

Isolation is handled via a **PostgreSQL Schema-Per-Tenant Model**:

1. **System Schema (`public`):** Contains platform-wide data (users, tenants, memberships, token denylist)
2. **Tenant Schemas (`t_{tenantKey}`):** Contains tenant-specific business data with complete isolation (including `t_platform` for the Platform Tenant)
3. **MyBatis Interceptor:** Automatically switches PostgreSQL `search_path` based on tenant context
4. **Liquibase Migrations:** Separate changesets for system vs tenant schemas

For details on the hybrid architecture, NanoID resolution, and bootstrapping, see the [Tenancy Deep Dive](./tenancy.md).

---

## Overview

```
  Client ──────────────────▶ API Gateway (Spring Cloud Gateway)
                                  │
                    ┌─────────────┴──────────────┐
                    ▼                            ▼
                   IAM                        Billing                     Audit                     CMS
              (Spring Boot)              (Spring Boot)               (Spring Boot)               (Spring Boot)
                    │                            │                           │                           │
                    ▼                            ▼                           ▼                           ▼
             PostgreSQL (iam)            PostgreSQL (billing)        PostgreSQL (audit)        PostgreSQL (cms)
            (Schema-per-tenant)          (Stripe integration)        (Centralized logs)       (Schema-per-tenant)

                    └─────────────┬──────────────┴───────────────┬───────────────┘
                                  ▼
                              RabbitMQ
                        (Async tenant provisioning,
                         audit events & lifecycle,
                         content events)

  UI (React + Mantine) ──▶ API Gateway (all requests proxied)
                          (JWT validation & context propagation)
```

---

## Services

### IAM Service

**Core Identity & Access Management with Multi-Tenant Support**

**Authentication & Authorization:**

- User signup with email verification (secure token-based)
- Magic link authentication: passwordless sign-in via time-limited token (initiate → email → exchange for JWT pair)
- JWT RS256 authentication: access tokens (15 min) + refresh tokens (7 days); `plan_code` claim stamped from tenant's active plan
- Password reset via signed email tokens (1h TTL), rate-limited (3 requests per 15min window)
- Brute-force protection: account lockout after 5 failed attempts for 15 minutes; platform admins can unlock
- Token revocation: JTI denylist + global signout timestamp with automatic cleanup
- JWKS endpoint (`/.well-known/jwks.json`) for distributed token validation
- Token exchange: `POST /auth/exchange` — workspace switching without re-authentication

**Tenant & Organization Management:**

- Tenant lifecycle: create (via `POST /tenants` after signup), suspend, delete, retry provisioning
- Async tenant provisioning via RabbitMQ with ShedLock-guarded reaper for stuck tenants
- Multi-tenant membership: one user can belong to multiple organizations
- RBAC with authorities: `TENANT_OWNER`, `ADMIN`, `PLATFORM_ADMIN`, `MEMBER`
- Cross-tenant user context switching and tenant discovery
- Member management: ban/unban, authority editing (TENANT_OWNER ↔ ADMIN ↔ MEMBER), ownership transfer

**Invitation System:**

- Email invitations with 72h expiring tokens, default authority `MEMBER`
- New users created on accept (email pre-verified)
- Existing users verified by password
- ShedLock-guarded reaper for stale invitation cleanup

**Multi-Mode Support:**

- Pluggable bootstrap strategies for different rollout modes
- Single-tenant: auto-provision default tenant at startup
- Multi-tenant: per-signup tenant creation
- Platform mode consistency validation across services

**Additional Capabilities:**

- Avatar uploads: two-phase presigned S3/MinIO flow; old avatars auto-deleted
- In-app notifications: `UserNotification` records + real-time WebSocket push (STOMP/SockJS) to `/user/{userId}/queue/notifications`
- Site-wide announcements: multi-lingual; async fan-out in batches of 1000; WebSocket broadcast
- Plan feature enforcement: `PlanFeatureGuard` annotation; `plan_code` JWT claim cached from billing; `maxUsers` quota checked at invite/signup
- Bulgarian (bg-BG) i18n: locale seed data; per-user BCP 47 locale stored in `users.locale`

**Events Published:** `tenant.created`, `tenant.provisioned`, `tenant.suspended`, `tenant.deleted`, `tenant.provisioning_failed`, `user.created`, `user.updated`, `user.invited`, `user.removed`, `user.deleted`, `announcement.publish`, `notification.iam.email`

**Tech Stack:** Java 25, Spring Boot 4.1, MyBatis 3.x, PostgreSQL 17, Liquibase, RabbitMQ, JJWT 0.13 (RS256), ShedLock 7.x, Spring WebSocket/STOMP, MinIO S3, Thymeleaf (email templates), Micrometer + Prometheus

---

### API Gateway Service

**Reactive Gateway with Security & Context Propagation**

**Core Capabilities:**

- Spring Cloud Gateway with WebFlux (reactive, non-blocking)
- JWT validation against IAM JWKS endpoint with authority extraction
- Multi-mode tenant resolution: JWT claims (multi-tenant) vs auto-injection (single-tenant)
- Platform mode guard: validates rollout mode consistency with IAM service every 60s; returns 503 on mismatch
- Header sanitization: prevents client spoofing of `X-User-*`, `X-Tenant-ID`, `X-Audit-*`, and `X-Plan-Code` headers

**Filter Chain (ordered):**

| Order           | Filter                         | Responsibility                                                                            |
| --------------- | ------------------------------ | ----------------------------------------------------------------------------------------- |
| `-201`          | `MonitoringFilter`             | Request rate, latency, status, and `tenant_id` metrics per route                          |
| `-200`          | `CorrelationIdFilter`          | Generate or propagate `X-Correlation-ID`; store in MDC                                    |
| `-190`          | `HeaderSanitizationFilter`     | Strip all spoofable headers from incoming client requests                                 |
| `-180`          | `AuditContextFilter`           | Extract client IP and User-Agent; forward as `X-Audit-IP`, `X-Audit-UA`, `X-Audit-Source` |
| Spring Security | JWT validation                 | RS256 signature via JWKS endpoint from IAM                                                |
| `-100`          | `JwtContextPropagationFilter`  | JWT claims → downstream headers including `X-Plan-Code`                                   |
| `-50`           | `TenantContextFilter`          | MULTI_TENANT: tenant from JWT; SINGLE_TENANT: inject default key if absent                |
| `MIN+1`         | `ResponseTransformationFilter` | Security response headers; echo `X-Correlation-ID`                                        |

**Context Propagation (downstream headers):**

`X-User-ID` · `X-Username` · `X-User-Email` · `X-User-Authorities` · `X-Tenant-ID` · `X-Plan-Code` · `X-Correlation-ID` · `X-Audit-IP` · `X-Audit-UA` · `X-Audit-Source`

**Plan Feature Enforcement:**

- `PlanCatalogCache` — reactive WebClient cache refreshed every 10 min from billing's internal plans endpoint
- `RequiresPlanFeatureFilterFactory` — declarative route-level plan enforcement via Spring Cloud Gateway YAML

**Routing & Security:**

- Path-based routing to IAM, Billing, CMS, and Audit services
- Configurable public paths (JWKS, auth, magic-link, webhooks, health checks, Swagger UI, billing internal)
- Global CORS configuration with configurable origins and methods
- Response security headers: `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy`

**Observability:**

- Prometheus metrics and health checks on separate management port 8081
- Swagger UI aggregation from downstream services at `/swagger-ui.html`
- Grafana dashboard for per-route and per-tenant traffic

**Events Published:** `api.request.metered` (planned)

**Tech Stack:** Java 25, Spring Boot 4.1, Spring Cloud Gateway, Spring Security OAuth2 Resource Server, WebFlux, Micrometer + Prometheus

---

### Billing Service

**Multi-Gateway Billing with Hexagonal Architecture**

**Core Responsibilities:**

- `PaymentGatewayPort` hexagonal abstraction — all business logic is gateway-agnostic; the active adapter is selected at configuration time via `iqkv.payment.gateway.type`
- Automatic customer provisioning per tenant via `tenant.created` events (adapter-agnostic)
- Tenant-to-customer mapping and billing metadata management
- Idempotent webhook processing with per-gateway signature verification (Stripe and Lemon Squeezy)
- Platform-wide lifecycle event publishing via RabbitMQ
- Async email notification publishing for billing events
- Pre-provisioned plan catalog with eligibility validation
- Multi-mode support: tenant-scoped (multi-tenant) vs user-scoped (single-tenant)
- **Per-seat pricing**: `PricingModel.PER_SEAT` plans route checkout with `quantity = seatCount`; seat-cap enforced against `maxUsers`; dedicated seat-adjustment endpoint with proration; `seatCount` propagated on subscription events

**Gateway Adapters:**

- **Stripe** (`@ConditionalOnGateway(STRIPE)`) — customer provisioning, webhook processing via `StripeWebhookRestResource` at `/api/v1/billing/webhooks/stripe`; product/price sync at startup via `BillingSeedRunner`
- **Lemon Squeezy** (`@ConditionalOnGateway(LEMON_SQUEEZY)`) — `LemonSqueezyGatewayAdapter` calls the JSON:API at `https://api.lemonsqueezy.com/v1/` via Spring `RestClient`; webhook processing via `LemonSqueezyWebhookRestResource` at `/api/v1/billing/webhooks/lemon-squeezy`; products/variants are dashboard-managed, `syncProduct` performs read-only variant verification only

**Gateway selection:**

```yaml
iqkv:
  payment:
    gateway:
      type: ${PAYMENT_GATEWAY_TYPE:STRIPE} # STRIPE | LEMON_SQUEEZY
```

**Stripe Integration (active):**

- Customer provisioning on tenant creation with metadata sync
- Webhook processing: `subscription.created`, `subscription.updated`, `subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`
- Signature verification and idempotency via `webhook_log` table
- Subscription state caching for fast reads without Stripe API calls
- Tax ID/VAT/GST sync to Stripe for compliant B2B invoices

**Plan Catalog:**

- Config-driven subscription plans (YAML `application-{env}.yml`); `BillingSeedRunner` syncs to Stripe at startup
- `PlanFeatureRegistry` in-memory O(1) lookups; typed quotas (`maxUsers`, `maxProjects`) + open feature map
- `FLAT` and `PER_SEAT` pricing models; `maxUsers` doubles as seat ceiling for PER_SEAT plans
- `trialPeriodDays` per plan; trial status (`isInTrial`, `trialDaysLeft`) in subscription responses
- Plan eligibility policy validates scope matches rollout mode
- Internal plans endpoint (`GET /internal/plans`) for service-to-service use; no auth required on internal network
- Feature-based entitlement evaluation for authorization decisions

**Multi-Mode Architecture:**

- **Multi-Tenant Mode**: Subscriptions scoped per tenant; `billing_settings` table used
- **Single-Tenant Mode**: Subscriptions scoped per user; `user_billing_settings` table used
- Subject resolution strategy pattern for mode-aware subscription handling
- Plan catalog filtered by scope based on active rollout mode

**Email Notifications:**

- Publishes async email notification events to RabbitMQ
- 9 notification types: subscription activated/updated/cancelled, trial ending, payment overdue, invoice paid, payment failed, billing updated, account suspended
- Email resolution: `billingEmail` from settings (multi-tenant) or `userBillingSettings` (single-tenant)
- Scheduled jobs for proactive notifications (trial ending, payment overdue)

**Events Published:** `subscription.created`, `subscription.updated`, `subscription.cancelled`, `invoice.created`, `invoice.finalized`, `invoice.paid`, `invoice.updated`, `payment.failed`, `refund.created`, `notification.billing.email`

**Tech Stack:** Java 25, Spring Boot 4.1, MyBatis 3.x, PostgreSQL 17, Stripe Java SDK, Spring RestClient (Lemon Squeezy), RabbitMQ, ShedLock 7.x, Thymeleaf (email templates)

---

### Audit Service

**Centralized Event-Driven Auditing Service**

**Core Responsibilities:**

- Acts as a platform-wide observer, consuming events from RabbitMQ to maintain a complete audit trail
- Normalizes disparate domain events (IAM, Billing, etc.) into a consistent `AuditRecord` format
- Provides a centralized search API for platform administrators to review activity across all tenants
- Implements a provider-friendly design, supporting multiple backends (PostgreSQL by default)
- Captures technical context (IP address, User-Agent) propagated from the API Gateway

**Implementation Details:**

- **Passive Observation**: Binds to `iqkv.events` exchange with wildcard routing keys to capture relevant business events without modifying domain logic
- **Context Propagation**: Leverages `foundation-audit-spi` to extract `X-Audit-*` headers injected by the Gateway
- **Persistence**: Uses MyBatis with PostgreSQL JSONB support for storing dynamic event details and metadata
- **Security**: REST endpoints are secured and restricted to users with `PLATFORM_ADMIN` authority

**Events Consumed:** `user.#`, `tenant.#`, `subscription.#`, `invoice.#`, `audit.#`

**Tech Stack:** Java 25, Spring Boot 4.1, MyBatis 3.x, PostgreSQL 17, RabbitMQ, Liquibase

### CMS Service

**Content Management Service**

**Core Capabilities:**

- Static page management with publishing status (draft/published)
- Multi-language support with en-US fallback
- Hierarchical content structure with parent/child page relationships
- SEO-friendly metadata (title, description, Open Graph tags, canonical URLs)
- Tenant isolation with schema-per-tenant PostgreSQL architecture
- Event-driven content lifecycle publishing (`cms.page.created`, `cms.page.updated`, `cms.page.deleted`)
- Public read-only API for fetching published pages
- Platform admin CRUD API for managing content

**Key Patterns:**

- `PageService` handles CRUD operations with publishing status management
- `PageTranslation` entity for multi-language content with fallback logic
- Tenant schema routing via `MyBatisSchemaInterceptor`
- Event publishing to RabbitMQ for content changes

**Events Published:** `cms.page.created`, `cms.page.updated`, `cms.page.deleted`

**Tech Stack:** Java 25, Spring Boot 4.x, MyBatis 3.x, PostgreSQL 17, Liquibase, RabbitMQ, Micrometer

### Billing Settings

Each tenant (multi-tenant mode) or user (single-tenant mode) has billing settings — the single source of truth for Stripe customer metadata:

**Multi-Tenant Mode (`billing_settings`):**

```sql
billing_settings
├── id                    UUID PK
├── tenant_key            VARCHAR(255) UNIQUE FK → tenant
├── external_customer_id  VARCHAR(255) UNIQUE    -- gateway customer ID (cus_xxx for Stripe)
├── billing_email         VARCHAR(255)           -- finance contact, no system access required
├── company_name          VARCHAR(255)
├── billing_address       JSONB                  -- street, city, country, postal_code
├── tax_id                VARCHAR(100)           -- VAT/GST number for B2B compliance
├── tax_id_type           VARCHAR(50)            -- Stripe enum: eu_vat, gb_vat, au_abn, etc.
├── currency              VARCHAR(3)             -- ISO 4217, default USD
├── profile_owner_id      UUID                   -- Soft ref to IAM users.id (nullable)
├── gateway_type          VARCHAR(32)            -- STRIPE | LEMON_SQUEEZY (v0.4)
├── created_at            TIMESTAMP
└── updated_at            TIMESTAMP
```

**Single-Tenant Mode (`user_billing_settings`):**

```sql
user_billing_settings
├── id                    UUID PK
├── user_id               UUID UNIQUE FK → user
├── external_customer_id  VARCHAR(255) UNIQUE    -- gateway customer ID
├── billing_email         VARCHAR(255)
├── company_name          VARCHAR(255)
├── billing_address       JSONB
├── tax_id                VARCHAR(100)
├── tax_id_type           VARCHAR(50)
├── currency              VARCHAR(3)
├── created_at            TIMESTAMP
└── updated_at            TIMESTAMP
```

**Subscription Model:**

```sql
subscriptions
├── id                        UUID PK
├── tenant_key                VARCHAR(255)
├── external_subscription_id  VARCHAR(255) UNIQUE    -- gateway subscription ID (sub_xxx for Stripe)
├── external_customer_id      VARCHAR(255)           -- gateway customer ID
├── external_order_id         VARCHAR(255)           -- LS order ID for refunds; null for Stripe (v0.4)
├── status                    VARCHAR(50)            -- active | past_due | canceled | unpaid | trialing
├── plan_id                   VARCHAR(255)           -- gateway price/variant ID
├── quantity                  BIGINT                 -- seat count for PER_SEAT plans; 1 for FLAT plans
├── current_period_start      TIMESTAMP
├── current_period_end        TIMESTAMP
├── cancel_at_period_end      BOOLEAN
├── canceled_at               TIMESTAMP
├── subject_type              VARCHAR(50)            -- TENANT | USER
├── subject_key               VARCHAR(255)           -- tenantKey or userId
├── gateway_type              VARCHAR(32)            -- STRIPE | LEMON_SQUEEZY (v0.4)
├── created_at                TIMESTAMP
└── updated_at                TIMESTAMP
```

**Plan Catalog:**

```sql
plan_catalog
├── id                UUID PK
├── plan_code         VARCHAR(100) UNIQUE
├── display_name      VARCHAR(255)
├── billing_period    VARCHAR(50)            -- MONTHLY | ANNUAL
├── price_minor       INTEGER                -- flat total OR per-seat unit price (see pricing_model)
├── currency          VARCHAR(3)
├── feature_set       JSONB                  -- Feature flags and limits (maxUsers doubles as seat ceiling)
├── scope             VARCHAR(50)            -- TENANT | USER
├── active            BOOLEAN
├── pricing_model     VARCHAR(16) NOT NULL   -- FLAT | PER_SEAT  (DEFAULT 'FLAT'; all legacy rows auto-migrated)
├── external_price_id VARCHAR(255)           -- Stripe price ID or LS variant ID (gateway-type-dependent)
├── gateway_type      VARCHAR(32)            -- STRIPE | LEMON_SQUEEZY (v0.4)
├── created_at        TIMESTAMP
└── updated_at        TIMESTAMP
```

**Key Design Decisions:**

- Auto-created on `tenant.created` event with registration defaults
- Decoupled from IAM users for billing independence (soft reference only)
- VAT/GST details flow directly into Stripe invoices via metadata sync
- Supports both tenant-scoped and user-scoped billing models
- Local subscription cache eliminates gateway API calls for reads
- Webhook idempotency via `webhook_log` table prevents duplicate processing
- Subject-aware event publishing for consistent entitlement evaluation
- `gateway_type` columns on `billing_settings`, `subscriptions`, and `plan_catalog` record which adapter owns each record — enables future cross-gateway migrations and observability
- `external_price_id` is gateway-type-dependent: Stripe Price ID (`price_…`) or Lemon Squeezy Variant ID (integer string); `external_order_id` on `subscriptions` stores the LS Order ID required for order-level refunds

---

## UI Applications

The platform ships three production-ready frontends, all communicating exclusively through the API Gateway.

### Tenant App (`foundation-ui-app`)

**Tech Stack:** React 19, TypeScript 6, Vite 8 (SWC), Mantine UI 9, TanStack Router + Query, Zustand, Lingui 6 (i18n), Axios, Vitest + Playwright, OxLint/OxFmt

**Architecture:** Feature-Sliced Design (`app → processes → pages → widgets → features → shared`); automated boundary tests via `pnpm test:arch`

**Key features:** sign-in with tenant discovery, sign-up with provisioning polling, password reset, email verification, invitation acceptance, team management (invite/ban/unban/role-edit/transfer-ownership), billing self-service (Stripe portal, subscription view with trial status, plan catalog with per-seat labels, refunds), in-app notifications with real-time WebSocket push, plan-based `FeatureGate` component and `EntitlementsProvider`, organization settings, light/dark theme, Lingui i18n (English + Bulgarian).

### Platform Admin (`foundation-ui-platform-admin`)

**Tech Stack:** React 19, TypeScript, Vite + SWC, Mantine UI 9, mantine-datatable, TanStack Router + Query, Zustand, Lingui, Zod + Mantine Form, Vitest + Playwright, OxLint/OxFmt

**Key features:** PLATFORM_ADMIN-only access; global user/organization/invitation/subscription/plan/announcement/audit/refund/notification management; ban/unban/unlock users; member authority management; dashboard with count widgets, subscription breakdown, signup trend chart, audit feed, org health cards; enterprise dark theme; runtime `public/config.js` override.

### SaaS Landing Kit (`foundation-ui-saas-landing-kit`)

**Tech Stack:** Astro, React, Tailwind CSS, shadcn/ui, Zustand, TypeScript

**Key features:** Static marketing pages (Home, Features, Pricing, About); auth-aware navigation; plan selector fetched from billing API with per-seat label support; React islands for partial hydration; aligned theme tokens.

### Documentation Website (`foundation-docs-website`)

VitePress-based site with user guides, platform overview, and quick-start instructions.

**Tenancy Adaptation (all apps):**

- Platform mode detected via IAM `/actuator/info` at runtime
- Multi-tenant: shows organization switcher, create-org flow, org-scoped management
- Single-tenant: hides tenancy concepts; workspace focuses on features
- Runtime `public/config.js` overrides `VITE_*` build-time variables without rebuilding

---

## Data Layer

### PostgreSQL — Database-Per-Service Pattern

Each service owns its own PostgreSQL database with complete data isolation. No shared databases or cross-service table access — inter-service communication flows through APIs or the event bus.

| Service | Database             | Contents                                                                                    |
| ------- | -------------------- | ------------------------------------------------------------------------------------------- |
| IAM     | `foundation_iam`     | Users, tenants, memberships, authorities, invitations, tokens, notifications, announcements |
| Billing | `foundation_billing` | Stripe customer refs, subscription cache, webhook logs, plans                               |
| Audit   | `foundation_audit`   | Centralized audit logs, technical context, activity records                                 |
| CMS     | `foundation_cms`     | Pages, page translations, hierarchical content (schema-per-tenant)                          |

### Schema-Per-Tenant Architecture (IAM Database)

Within `foundation_iam`, each tenant gets a dedicated PostgreSQL schema with automatic routing:

```
foundation_iam/
├── public/                    # Platform registry
│   ├── users                  # Global user accounts
│   ├── tenants               # Tenant registry with status
│   ├── tenant_memberships    # User-tenant relationships
│   ├── token_denylist        # Revoked JWT tokens
│   ├── failed_login_attempts # Brute-force tracking
│   └── shedlock             # Distributed job coordination
├── t_abc12345/              # Tenant-specific schemas
│   ├── tenant_member_authorities  # Per-tenant RBAC
│   ├── invitations               # Tenant invitations
│   └── [business entities]       # Tenant-scoped data
├── t_def67890/
└── t_ghi13579/
```

**Schema Switching Implementation:**

```java
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare")})
public class MyBatisSchemaInterceptor implements Interceptor {
    public Object intercept(Invocation invocation) throws Throwable {
        Connection connection = (Connection) invocation.getArgs()[0];
        String tenantKey = TenantContext.getCurrentTenant();
        String schema = "t_" + tenantKey;

        try (PreparedStatement stmt = connection.prepareStatement(
                "SET search_path TO " + schema + ", public")) {
            stmt.execute();
        }
        return invocation.proceed();
    }
}
```

**Migration Strategy:**

- **System Migrations:** `db/changelog/system/db.changelog-master.xml`
- **Tenant Migrations:** `db/changelog/tenant/master.xml`
- **Liquibase Runner:** Automatic schema creation and migration per tenant
- **Cross-Tenant Queries:** Not possible in normal application flow (security by design)

### RabbitMQ — Event-Driven Architecture

Asynchronous communication and tenant provisioning via RabbitMQ with durable queues and dead letter handling:

| Exchange      | Routing Key                  | Consumer          | Purpose                                                            |
| ------------- | ---------------------------- | ----------------- | ------------------------------------------------------------------ |
| `iqkv.events` | `tenant.created`             | Billing Service   | Create Stripe customer, init billing settings                      |
| `iqkv.events` | `tenant.provisioned`         | Billing Service   | Send subscription activated notification                           |
| `iqkv.events` | `tenant.provisioning.failed` | Monitoring/Alerts | Handle provisioning failures                                       |
| `iqkv.events` | `tenant.suspended`           | Billing Service   | Send account suspended notification                                |
| `iqkv.events` | `tenant.deleted`             | —                 | Future: cancel Stripe customer                                     |
| `iqkv.events` | `user.created`               | —                 | Downstream extensions                                              |
| `iqkv.events` | `user.updated`               | —                 | Downstream extensions                                              |
| `iqkv.events` | `user.invited`               | —                 | Invitation notifications                                           |
| `iqkv.events` | `user.removed`               | Billing Service   | Clear `profileOwnerId` on tenant billing settings                  |
| `iqkv.events` | `user.deleted`               | Billing Service   | Clear `profileOwnerId` across all billing settings                 |
| `iqkv.events` | `subscription.created`       | IAM Service       | Cache `planCode` on tenant; stamp into JWT                         |
| `iqkv.events` | `subscription.updated`       | IAM Service       | Update cached `planCode` on tenant                                 |
| `iqkv.events` | `subscription.cancelled`     | IAM Service       | Suspend tenant on cancellation / payment failure                   |
| `iqkv.events` | `invoice.created`            | Audit Service     | Audit logging                                                      |
| `iqkv.events` | `invoice.finalized`          | Audit Service     | Audit logging                                                      |
| `iqkv.events` | `invoice.paid`               | Audit Service     | Payment success; audit logging                                     |
| `iqkv.events` | `invoice.updated`            | Audit Service     | Audit logging                                                      |
| `iqkv.events` | `payment.failed`             | Audit Service     | Payment failure handling; audit logging                            |
| `iqkv.events` | `refund.created`             | Audit Service     | Refund audit logging                                               |
| `iqkv.events` | `cms.page.created`           | —                 | Content lifecycle extensions                                       |
| `iqkv.events` | `cms.page.updated`           | —                 | Content lifecycle extensions                                       |
| `iqkv.events` | `cms.page.deleted`           | —                 | Content lifecycle extensions                                       |
| `iqkv.events` | `announcement.publish`       | IAM Service       | Fan-out trigger: batch notification creation + WebSocket broadcast |
| `iqkv.events` | `notification.iam.email`     | IAM Service       | Send email + persist in-app notification + push WebSocket          |
| `iqkv.events` | `notification.billing.email` | Billing Service   | Async email delivery for billing events                            |
| `iqkv.events` | `audit.*`                    | Audit Service     | Direct audit events published by domain services                   |
| `iqkv.events` | `user.#`, `tenant.#`         | Audit Service     | Business events consumed for passive auditing                      |

**Event Processing Patterns:**

- **Idempotent Consumers:** All event handlers are idempotent and safe to retry
- **Dead Letter Queues:** Failed events are routed to DLQ for manual investigation
- **ShedLock Coordination:** Prevents duplicate processing in clustered deployments
- **Outbox Pattern:** Ensures reliable event publishing with transactional guarantees

---

## Tenant Provisioning Flow

### Multi-Tenant Mode (Default)

```
1. POST /api/v1/iam/auth/signup
   ├── Create user account (email verification required)
   ├── Generate 8-char NanoID tenant key
   └── Create tenant record (status: PROVISIONING)
       │
2. Publish tenant.provisioning.requested → RabbitMQ
       │
3. Return HTTP 201 { tenantKey, status: "PROVISIONING" }
       │
4. Async Processing:
   ├── TenantProvisioningConsumer
   │   ├── Create PostgreSQL schema t_{tenantKey}
   │   ├── Run Liquibase tenant migrations
   │   ├── Update tenant status: ACTIVE
   │   └── Publish tenant.provisioned event
   │
   └── Billing Consumer (on tenant.provisioned)
       ├── Create Stripe customer
       ├── Store customer ID in billing_settings
       └── Initialize default billing configuration

5. Client polls GET /api/v1/iam/auth/signup/status/{tenantKey} until ACTIVE
```

### Single-Tenant Mode (Bootstrap)

When `iqkv.platform.rollout-mode: SINGLE_TENANT`, IAM runs the same provisioning flow at application startup:

```
1. Application Startup (ApplicationReadyEvent)
       │
2. SingleTenantBootstrapStrategy.bootstrap()
   ├── Check if default tenant exists
   ├── If not: create default tenant (status: PROVISIONING)
   └── Run same async provisioning flow
       │
3. Default tenant becomes ACTIVE
       │
4. All user signups join the default tenant with MEMBER authority
   (TENANT_OWNER authority reserved for initial admin)
```

**Failure Handling:**

- **Exponential Backoff:** Workers retry with exponential backoff on failure
- **Stuck Tenant Reaper:** ShedLock-guarded job cleans up tenants stuck in `PROVISIONING` (configurable timeout: 10 minutes)
- **Manual Retry:** Owners can trigger `POST /tenants/{tenantKey}/retry-provisioning` for `PROVISIONING_FAILED` tenants
- **Monitoring:** Prometheus metrics track provisioning success/failure rates

---

## Security Architecture

### JWT-Based Authentication

**Token Structure:**

- **Algorithm:** RS256 (asymmetric signing)
- **Access Token:** 15-minute expiry with user context and authorities
- **Refresh Token:** 7-day expiry for token rotation
- **Claims:** `userId`, `username`, `email`, `tenant_id`, `authorities`, `email_verified`, `plan_code`

**Token Lifecycle:**

```
1. Login → Generate token pair (access + refresh)
2. API requests → Validate access token via JWKS
3. Token expiry → Use refresh token to get new pair
4. Logout → Add JTI to denylist
5. Global logout → Update user.last_global_signout_at
```

**Validation Chain:**

1. **Signature Verification:** Against IAM JWKS endpoint
2. **Expiry Check:** Token not expired
3. **Denylist Check:** JTI not in revocation list
4. **Global Signout Check:** Token issued after last global signout
5. **Tenant Context:** Validate tenant membership and authorities

### Multi-Tenant Security

**Tenant Isolation:**

- **Database Level:** PostgreSQL schema isolation with `search_path` switching
- **Application Level:** Tenant context validation on every request
- **API Level:** Gateway strips and re-injects tenant headers to prevent spoofing

**Authorization Model:**

```
User → TenantMembership → Authorities (per tenant)
     ↓
   TENANT_OWNER: Full tenant management
   ADMIN: User management, invitations
   MEMBER: Basic access
```

**Cross-Tenant Protection:**

- **Context Validation:** Every database operation validates tenant context
- **Header Sanitization:** Gateway prevents client-supplied tenant headers
- **Schema Isolation:** Database-level isolation prevents cross-tenant queries

---

## Infrastructure as Code

### Helm Chart Architecture

Each service has a dedicated Helm chart with environment-specific configurations:

```
helm-charts/IQKV/
├── foundation-iam-service/
│   ├── Chart.yaml
│   ├── values.yaml                    # Default values
│   ├── values-sit.yaml               # System integration testing
│   ├── values-uat.yaml               # User acceptance testing
│   └── values-prd.yaml               # Production
├── foundation-gateway-service/
├── foundation-billing-service/
├── foundation-audit-service/
├── foundation-cms-service/
├── foundation-ui-app/
├── foundation-ui-platform-admin/
├── foundation-ui-saas-landing-kit/
└── foundation-infra/                  # Shared infrastructure (PostgreSQL, RabbitMQ, Redis, MinIO)
```

### Deployment Strategy

**Service Independence:**

- Each service deploys independently with its own release cycle
- Shared infrastructure (PostgreSQL, RabbitMQ) managed separately
- Configuration via Helm values and Kubernetes secrets

**Example Deployment:**

```bash
# Deploy IAM service to production
helm upgrade --install foundation-iam-service ./foundation-iam-service \
  --values ./values.yaml \
  --values ./values-prd.yaml \
  --set secrets.database.password=$PG_PASSWORD \
  --set secrets.rabbitmq.password=$RMQ_PASSWORD \
  --set secrets.jwt.privateKey=$JWT_PRIVATE_KEY \
  --namespace iqkv-prd \
  --atomic --wait --timeout=10m
```

**Configuration Management:**

- **Secrets:** Kubernetes secrets for sensitive data (passwords, keys)
- **ConfigMaps:** Non-sensitive configuration (URLs, timeouts, feature flags)
- **Environment Variables:** Runtime configuration injection
- **Helm Values:** Environment-specific overrides

### CI/CD Pipeline Integration

**Pipeline Stages (Java services — 10 pipelines):**

1. **VerifyCode:** `mvn clean verify` → SonarQube quality gate → PMD → SpotBugs
2. **PublishArtifacts:** `mvn deploy` (SNAPSHOT on branches; release JAR on tags); GitHub Release via `release-it`
3. **PublishDockerImage:** Multi-stage Docker build; tag strategy: branch name (`wip`), stripped feature name, or semver tag
4. **DeployWorkInProgress:** `helm upgrade --install --atomic` to `iqkv-sit-env` on `wip` push
5. **RollbackWorkInProgress / PromoteFeatureDeployment / RollbackFeatureDeployment:** Feature branch SIT lifecycle
6. **PromoteDeployment / RollbackDeployment:** UAT/PRD deployment and rollback on semver tags
7. **ReleasePackage:** Strip SNAPSHOT, create git tag, bump pom + package.json to next SNAPSHOT, update CHANGELOG

**Frontend (4 pipelines):** `VerifyCode` (formatter + lint + test:coverage + SonarQube + build) → `PublishArtifacts` → `DeployWorkInProgress` (pnpm build + Nginx Docker image + Helm) → `RollbackWorkInProgress`

**Library modules (2 pipelines):** `VerifyCode` + `PublishArtifacts` only

**Infrastructure (3 pipelines):** `Info` (helm template dry-run) → `PromoteInfrastructure` (Helm deploy + kubectl wait + DB connectivity checks) → `RollbackInfrastructure` (helm uninstall + force-delete PVCs)

**Pipeline Configuration:**

```yaml
# Drone CI pipeline example — VerifyCode stage
kind: pipeline
name: VerifyCode
type: docker
trigger:
  event: [push, tag]
  ref:
    include: [refs/heads/dev, refs/heads/feature/*, refs/tags/*]
steps:
  - name: code-coverage-gate
    image: cicdtools/pipeline-runner
    commands:
      - mvn clean verify -Dstyle.color=always
  - name: static-analysis-gate
    depends_on: [code-coverage-gate]
    commands:
      - mvn sonar:sonar -Dsonar.qualitygate.wait=true
      - pmd check --minimum-priority High -d src -R ruleset.xml
      - mvn spotbugs:check
```

---

## Observability & Operations

### Monitoring Stack

**Metrics Collection:**

- **Micrometer + Prometheus:** JVM, HTTP, and custom business metrics on all services; scrape at `/actuator/prometheus`
- **Grafana Dashboards:** Pre-provisioned dashboards per service — JVM, IAM (auth/security/lifecycle), Gateway (per-tenant traffic), Billing (MRR/churn/webhooks), Audit (event volume)
- **Log Aggregation:** Loki + Promtail (demo stack) for centralized log querying alongside metrics
- **Alerting:** Prometheus AlertManager integration (Slack/email)

**Key Metrics:**

- **Authentication:** Login success/failure rates, token validation latency
- **Tenant Provisioning:** Provisioning success rate, time to active, stuck tenant count
- **API Gateway:** Request throughput, response times, error rates by service
- **Database:** Connection pool usage, query performance, schema count

**Logging Strategy:**

- **Structured JSON Logs:** Logstash encoder for consistent log format
- **Correlation IDs:** Request tracing across service boundaries
- **Log Aggregation:** Centralized logging with ELK stack or similar
- **Log Levels:** Configurable per service and package

### Health Checks & Readiness

**Spring Boot Actuator:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
```

**Health Check Hierarchy:**

- **Liveness:** Service is running and not deadlocked
- **Readiness:** Service can handle traffic (database connected, dependencies available)
- **Custom Checks:** Platform mode consistency, tenant provisioning capacity

**Operational Endpoints:**

- `GET /actuator/health` - Kubernetes liveness/readiness probes
- `GET /actuator/info` - Service version, build info, platform mode
- `GET /actuator/metrics` - Application metrics
- `GET /actuator/prometheus` - Prometheus scrape endpoint

---

## Extension Model & Future Architecture

### Event-Driven Extensions

Core services publish to a versioned RabbitMQ exchange, enabling extensions without core code modifications:

```
Core Services (IAM, Gateway, Billing, Audit, CMS)
    ↓ (publishes events)
Platform Exchange (iqkv.events)
    ↓ (routes to)
├── Core Workers (tenant provisioning, billing sync, audit logging)
└── Extensions (SAML SSO, analytics, custom SIEM backends, etc.)
```

**Extension Patterns:**

- **Event Subscribers:** React to platform events without modifying core services
- **API Extensions:** Additional REST endpoints via separate services
- **UI Extensions:** Micro-frontend architecture for additional features
- **Webhook Extensions:** External system integrations via webhook consumers

**Versioning Strategy:**

- **Event Schema Versioning:** Backward-compatible event schema evolution
- **API Versioning:** Semantic versioning for REST APIs
- **Database Migrations:** Forward-only Liquibase migrations
- **Service Contracts:** OpenAPI specifications for service interfaces

### Scalability Considerations

**Horizontal Scaling:**

- **Stateless Services:** All services are stateless and horizontally scalable
- **Database Scaling:** Read replicas, connection pooling, query optimization
- **Message Queue Scaling:** RabbitMQ clustering for high availability
- **Caching Strategy:** Redis for session storage and frequently accessed data

**Performance Optimization:**

- **Connection Pooling:** HikariCP for database connections
- **Async Processing:** Non-blocking I/O with WebFlux where appropriate
- **Batch Processing:** Bulk operations for data-intensive tasks
- **CDN Integration:** Static asset delivery via CDN

The event schema serves as the public API contract, allowing core internals to evolve freely while maintaining extension compatibility.
