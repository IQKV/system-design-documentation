# Implemented Features Review — June 2026 (Final)

> Comprehensive snapshot of everything built and shipped across the platform as of June 2026, including Lemon Squeezy support and OAuth2/OIDC identity federation.
> Sources: All service READMEs, source code exploration, and prior roadmap review documents.
> Supersedes document 13 (June 2026 review) as the authoritative current state.

---

## Platform Overview

**Services:** 5 core microservices + 3 frontend applications + 2 shared libraries
**Tech stack:** Java 25 / Spring Boot 4.1 · React 19 · TypeScript 6 · PostgreSQL 17 · RabbitMQ · MinIO · Redis
**Live demo:** [iqkv.site](https://iqkv.site) · Tenant app: [app.iqkv.site](https://app.iqkv.site) · Admin: [admin.iqkv.site](https://admin.iqkv.site) · API docs: [api.iqkv.site/swagger-ui.html](https://api.iqkv.site/swagger-ui.html)

---

## Backend Services

### IAM Service (`foundation-iam-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · JJWT 0.13 RS256 · ShedLock 7.x · Spring WebSocket/STOMP · MinIO S3 · Thymeleaf · Micrometer + Prometheus · Redis

#### Authentication & Token Management

| Feature                    | Detail                                                                                                                      |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Self-service signup        | Creates user, adds as `MEMBER` to platform tenant; tenant creation separate via dedicated endpoint                          |
| Signup status polling      | `GET /auth/signup/status/{tenantKey}` polls until tenant `ACTIVE`                                                           |
| Tenant-scoped sign-in      | `POST /auth/signin` + `X-Tenant-ID`; RS256 access (15 min) + refresh (7 day) token pair                                     |
| Platform admin sign-in     | Separate `POST /auth/admin/signin`; platform-scoped token (`tenant_id = null`)                                              |
| Token exchange             | `POST /auth/exchange` — workspace switching without re-authentication                                                       |
| Magic link authentication  | `POST /auth/magic-link/initiate` + `/resend` + `/exchange` — passwordless sign-in with configurable TTL and rate limiting   |
| Token refresh              | `POST /auth/refresh` and `POST /auth/admin/refresh` — rotate both tokens                                                    |
| Single-session sign-out    | `POST /auth/signout` — adds JTI to denylist                                                                                 |
| Global sign-out            | `POST /auth/signout-all` — sets `last_global_signout_at`; invalidates all sessions                                          |
| Token validation           | `POST /auth/validate` — gateway introspection; checks denylist + global signout                                             |
| Tenant discovery           | `POST /users/tenants` — credential-gated (no JWT); returns all active memberships                                           |
| JWKS endpoint              | `GET /.well-known/jwks.json` — public RSA key for downstream JWT validation                                                 |
| RS256 JWT issuance         | Access tokens carry: `sub`, `userId`, `username`, `email`, `tenant_id`, `authorities`, `email_verified`, `jti`, `plan_code` |
| Two-layer token revocation | JTI denylist (per-signout) + `last_global_signout_at` (global signout); both checked in `JwtAuthenticationFilter`           |

#### OAuth2/OIDC Identity Federation

| Feature                              | Detail                                                                                                       |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Social providers                     | Google, GitHub, and Microsoft supported via Spring Security OAuth2 Client                                    |
| Tenant-scoped enterprise SSO         | Custom OIDC provider configuration per tenant stored in database                                             |
| Browser redirect flow                | `GET /api/v1/iam/auth/oauth2/authorize` + callback endpoint for browser-based sign-in                        |
| SPA code exchange                    | `POST /api/v1/iam/auth/oauth2/exchange` for SPA-based auth code exchange                                     |
| PKCE with Redis-backed state storage | OAuth2 PKCE flow with state stored in Redis for security                                                     |
| Account linking/unlinking            | Link additional providers to existing accounts; unlink with safeguards against removing last sign-in method  |
| User identity management             | `user_identities` table for linking external identities to internal users                                    |
| Tenant OIDC provider configuration   | `tenant_oidc_providers` table with encrypted client secrets (AES-256-GCM)                                    |
| User-facing endpoints                | `/providers`, `/identities`, `/link/{provider}`, `/link/{provider}/authorize-url`, `/link/{provider}` delete |
| Admin remediation endpoints          | `/admin/oidc/users/{userId}/identities` and forced unlink                                                    |
| Verified email auto-linking          | External identities with verified email auto-linked to existing internal users with same email               |
| GitHub verified email requirement    | GitHub identities must expose a verified email to be used for sign-in                                        |

#### Multi-Tenancy

| Feature                     | Detail                                                                                                         |
| --------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Multi-tenant mode           | Each signup creates a new tenant; user gets `TENANT_OWNER` authority                                           |
| Single-tenant mode          | Signup joins pre-provisioned default tenant with `MEMBER` authority; no new tenant creation                    |
| Schema-per-tenant isolation | PostgreSQL `t_{tenantKey}` schemas; `MyBatisSchemaInterceptor` sets `search_path` per request                  |
| Tenant provisioning         | Async via RabbitMQ; `TenantProvisioningConsumer` runs Liquibase, sets `ACTIVE`, publishes `tenant.provisioned` |
| Tenant lifecycle            | Transitions: `PROVISIONING → ACTIVE ↔ SUSPENDED → DELETED`; `PROVISIONING_FAILED` with retry                   |
| Stuck tenant reaper         | ShedLock-guarded every 5 min; marks stuck `PROVISIONING` tenants as `PROVISIONING_FAILED`                      |
| Platform tenant             | Fixed `platform` key/schema; every user is auto-added as `MEMBER` in all modes                                 |
| Rollout mode validation     | `ROLLOUT_MODE` validated at startup; published via `/actuator/info` for cross-service consistency              |
| Single→multi migration      | Zero-migration architectural symmetry — same schema-per-tenant model in both modes                             |

#### User Management (Self-Service)

| Endpoint                                | Description                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| `GET/PATCH /users/me`                   | View and update own profile                                                     |
| `POST /users/me/password`               | Change own password; invalidates all sessions                                   |
| `DELETE /users/me`                      | Remove own membership from current tenant                                       |
| `POST /users/me/avatar`                 | Initiate two-phase S3 presigned upload                                          |
| `POST /users/me/avatar/confirm`         | Confirm avatar upload after S3 PUT; old avatar auto-deleted                     |
| `DELETE /users/me/avatar`               | Delete current user's avatar                                                    |
| `GET /users/me/memberships`             | List current user's tenant memberships (for org/tenant picker UIs)              |
| `POST /users/email/verify`              | Token-based email verification                                                  |
| `POST /users/email/resend-verification` | Rate-limited resend (3/hour)                                                    |
| `POST /users/password/forgot`           | Initiate password reset (rate-limited, 3/15 min); enumeration-safe (always 200) |
| `POST /users/password/reset`            | Complete password reset; 1h TTL token; invalidates all sessions                 |

#### In-App Notifications

| Feature               | Detail                                                                                                                                             |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Persistent storage    | `user_notifications` table; locale captured at creation time                                                                                       |
| Real-time push        | WebSocket STOMP/SockJS at `/api/v1/iam/ws`; personal queue `/user/{userId}/queue/notifications`                                                    |
| Notification REST API | `GET/PATCH/DELETE /users/notifications`; unread count at `/users/notifications/unread/count`; single-item `PATCH/DELETE /users/notifications/{id}` |
| Bulk mark-as-read     | `PATCH /users/notifications` with `{"isRead": true}` body                                                                                          |
| Notification types    | System announcements, billing alerts, subscription changes, payment failures, trial expiry warnings                                                |
| i18n support          | User locale (BCP 47) stored on user; `MessageSource` translates event type at persistence time                                                     |

#### Site-Wide Announcements

| Feature                 | Detail                                                                                                         |
| ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| Multi-lingual editor    | `en-US` mandatory + optional translations per locale                                                           |
| Status lifecycle        | `DRAFT → PENDING → PUBLISHING → PUBLISHED / FAILED`; published announcements are immutable                     |
| Async streaming fan-out | MyBatis cursor with `fetchSize=1000`; batches of 1000 users; each batch commits independently (`REQUIRES_NEW`) |
| WebSocket broadcast     | `AnnouncementBroadcastResponse` pushed to `/topic/announcements` after fan-out completes                       |
| Admin API               | `POST/GET/PUT/DELETE /admin/announcements` + `POST /admin/announcements/{id}/publish`                          |
| Public read API         | `GET /announcements?locale=…` — returns `PUBLISHED` announcements for requested locale                         |

#### Tenant Owner Member Management

| Feature                 | Detail                                                                                                          |
| ----------------------- | --------------------------------------------------------------------------------------------------------------- |
| List/count members      | `GET /tenants/{key}/members` + `/count`                                                                         |
| Get/replace authorities | `GET/PUT /tenants/{key}/members/{userId}/authorities`                                                           |
| Remove member           | `DELETE /tenants/{key}/members/{userId}`                                                                        |
| Ban/unban member        | `POST /tenants/{key}/members/{userId}/ban` + `/unban`; banned user sessions invalidated                         |
| Transfer ownership      | `POST /tenants/{key}/members/{userId}/transfer-ownership`; old owner becomes `MEMBER`; guardrails on last owner |

#### Platform Admin APIs (IAM)

| Endpoint group                                              | Description                             |
| ----------------------------------------------------------- | --------------------------------------- |
| `GET/POST/PUT/PATCH/DELETE /admin/users`                    | Paginated user CRUD; force-set password |
| `POST /admin/users/{id}/ban` + `/unban`                     | Global user ban/unban                   |
| `POST /admin/users/{id}/unlock`                             | Reset failed login attempts             |
| `GET /admin/users/{id}/authorities` + `PUT`                 | Platform authority management           |
| `GET /admin/users/{id}/memberships`                         | View user's tenant memberships          |
| `GET/PUT/PATCH/DELETE /admin/tenants/{key}`                 | Tenant CRUD                             |
| `GET /admin/tenants/{key}/members` + `/count`               | Member listing                          |
| `GET/PUT /admin/tenants/{key}/members/{userId}/authorities` | Member authority management             |
| `GET/POST/DELETE /admin/invitations`                        | Cross-tenant invitation management      |
| `GET/POST/PUT/DELETE /admin/announcements` + `/publish`     | Announcement lifecycle management       |
| `GET /admin/users/count` + `GET /admin/tenants/count`       | Count endpoints for dashboard widgets   |

#### Security & Brute-Force Protection

| Feature               | Detail                                                                                         |
| --------------------- | ---------------------------------------------------------------------------------------------- |
| Brute-force lockout   | Failed login tracking per email; 5-attempt threshold; 15-min lockout; automatic record cleanup |
| Password complexity   | Lowercase + uppercase + digit + special char; 8–128 chars                                      |
| BCrypt strength 12    | Password hashing with strength 12                                                              |
| Account lockout reset | Platform admin can unlock via `POST /admin/users/{id}/unlock`                                  |

#### Plan Feature Enforcement (IAM)

| Feature                   | Detail                                                                                                   |
| ------------------------- | -------------------------------------------------------------------------------------------------------- |
| `plan_code` JWT claim     | Active plan code stamped into access tokens; updated via subscription lifecycle events                   |
| `PlanCatalogCache`        | Non-reactive `RestTemplate`-based cache; refreshes from billing service every 10 min                     |
| `PlanFeatureGuard`        | Annotation for checking plan features on endpoints; throws `PlanFeatureNotAvailableException` → HTTP 402 |
| `maxUsers` quota          | Checked at invitation acceptance and single-tenant signup; `PlanMemberQuotaException` → HTTP 402         |
| `advanced_analytics` gate | `GET /admin/tenants/{key}/stats` and `GET /tenants/{key}/stats` gated behind plan feature                |
| Personal workspace access | Personal workspace always accessible even if tenant is suspended                                         |

#### Observability & Infrastructure (IAM)

| Feature                 | Detail                                                                                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Custom metrics          | `iam.auth.outcome` (by reason + tenant), `iam.auth.duration`, `iam.security.event`, `iam.user.lifecycle`, `iam.tenant.provisioning`, `iam.messaging.publish` |
| Grafana dashboards      | JVM dashboard + IAM business/security metrics dashboard                                                                                                      |
| Structured JSON logging | MDC with correlation ID and tenant context                                                                                                                   |
| Health probes           | `/actuator/health` with `PlatformModeHealthIndicator`                                                                                                        |
| ShedLock jobs           | Token denylist cleanup, invitation expiry, stuck tenant reaper, email verification cleanup                                                                   |

---

### Gateway Service (`foundation-gateway-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · Spring Cloud Gateway (WebFlux) · Spring Security OAuth2 Resource Server · Micrometer + Prometheus

#### Filter Chain

| Order           | Filter                         | Responsibility                                                                                          |
| --------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `-201`          | `MonitoringFilter`             | Record request metrics (rate, duration, status, tenant) per route                                       |
| `-200`          | `CorrelationIdFilter`          | Generate or propagate `X-Correlation-ID`; store in MDC                                                  |
| `-190`          | `HeaderSanitizationFilter`     | Strip `X-User-*`, `X-Tenant-ID`, `X-Organization-ID`, `X-Audit-*`, `X-Plan-Code` from incoming requests |
| `-180`          | `AuditContextFilter`           | Extract client IP and User-Agent; forward as `X-Audit-IP`, `X-Audit-UA`, `X-Audit-Source`               |
| Spring Security | JWT validation                 | RS256 signature via JWKS from IAM                                                                       |
| `-100`          | `JwtContextPropagationFilter`  | JWT claims → downstream headers including `X-Plan-Code`                                                 |
| `-50`           | `TenantContextFilter`          | MULTI_TENANT: tenant from JWT; SINGLE_TENANT: inject default tenant key if absent                       |
| `MIN+1`         | `ResponseTransformationFilter` | Security response headers; echo `X-Correlation-ID` to client                                            |

#### Downstream Headers

| Header               | Source                        |
| -------------------- | ----------------------------- |
| `X-User-ID`          | JWT `userId` claim            |
| `X-Username`         | JWT `username` claim          |
| `X-User-Email`       | JWT `email` claim             |
| `X-User-Authorities` | JWT `authorities` claim       |
| `X-Tenant-ID`        | JWT `tenant_id` claim         |
| `X-Plan-Code`        | JWT `plan_code` claim         |
| `X-Correlation-ID`   | Generated / propagated        |
| `X-Audit-IP`         | Client IP address             |
| `X-Audit-UA`         | Client User-Agent             |
| `X-Audit-Source`     | Configured gateway identifier |

#### Routes

| Route              | Predicate                               | Auth                                               |
| ------------------ | --------------------------------------- | -------------------------------------------------- |
| `iam-jwks`         | `/.well-known/**` → IAM                 | Public                                             |
| `iam-api`          | `/api/v1/iam/**` → IAM                  | Public/protected (via `iqkv.gateway.public-paths`) |
| `billing-webhooks` | `/api/v1/billing/webhooks/**` → Billing | Public (Stripe/Lemon Squeezy signature)            |
| `billing-api`      | `/api/v1/billing/**` → Billing          | Protected (JWT required)                           |
| `billing-internal` | `/api/v1/billing/internal/**` → Billing | Public (internal network)                          |

#### Plan Feature Enforcement (Gateway)

| Feature                            | Detail                                                                                      |
| ---------------------------------- | ------------------------------------------------------------------------------------------- |
| `PlanCatalogCache`                 | Reactive WebClient-based cache; refreshes from billing service every 10 min                 |
| `RequiresPlanFeatureFilterFactory` | Route-level declarative plan feature enforcement in Spring Cloud Gateway YAML               |
| Plan code header                   | Extracts `plan_code` from JWT; propagates as `X-Plan-Code`; sanitizes client-supplied value |

#### Platform Mode Guard

`PlatformModeGuardFilter` polls IAM `/actuator/info` every 60s for `platform.rollout-mode`. Mismatch → sets `ReadinessState.REFUSING_TRAFFIC`, returns 503. IAM unreachable → fail-open.

#### Observability (Gateway)

Prometheus metrics at `/actuator/prometheus`. Aggregated Swagger UI at `/swagger-ui.html` proxies downstream `/api-docs`. Grafana dashboard for Gateway health and per-tenant traffic.

---

### Billing Service (`foundation-billing-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · Stripe Java SDK · Lemon Squeezy API · ShedLock 7.x · Micrometer + Prometheus

#### Multi-Gateway Support

| Feature                          | Detail                                                                                                                        |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Runtime gateway selection        | `GatewayType` enum (STRIPE, LEMON_SQUEEZY); selected via `iqkv.payment.gateway.type` config                                   |
| Conditional bean wiring          | `@ConditionalOnGateway` annotation for gateway-specific beans                                                                 |
| Gateway-neutral port abstraction | `PaymentGatewayPort` interface; both Stripe and Lemon Squeezy implement it                                                    |
| Plan catalog neutrality          | Plan catalog config moved from `iqkv.billing.stripe.schema.products` to `iqkv.billing.plan-catalog.products`                  |
| Gateway-aware data model         | `billing_settings.gateway_type`, `subscriptions.gateway_type`, `subscriptions.external_order_id`, `plan_catalog.gateway_type` |
| Normalized webhook events        | Both Stripe and Lemon Squeezy webhooks normalized to internal `GatewayWebhookEvent` subtypes                                  |
| Stripe & Lemon Squeezy adapters  | Separate `StripeGatewayAdapter` and `LemonSqueezyGatewayAdapter` implementations                                              |
| Lemon Squeezy REST client config | `LemonSqueezyRestClientConfig` for REST client                                                                                |
| Lemon Squeezy webhook endpoint   | `POST /webhooks/lemon-squeezy` public endpoint with HMAC-SHA256 verification                                                  |
| Stripe webhook remains active    | `POST /webhooks/stripe` still available for existing deployments                                                              |
| Product sync behavior            | Stripe: creates products/prices; Lemon Squeezy: verifies variant exists (read-only)                                           |
| Portal session support           | Both gateways support customer portal session creation                                                                        |

#### Plan Catalog

Plans are defined in YAML (`application-{env}.yml`) and synchronized with payment gateway at startup by `BillingSeedRunner`. No REST API for plan mutations — all catalog changes go through configuration and deployment.

| Endpoint                      | Auth                    | Description                                          |
| ----------------------------- | ----------------------- | ---------------------------------------------------- |
| `GET /plans`                  | Any authenticated       | List all active plans                                |
| `GET /plans/{planCode}`       | Any authenticated       | Get plan by code                                     |
| `GET /admin/plans`            | `PLATFORM_ADMIN`        | List all plans (including inactive)                  |
| `GET /admin/plans/{planCode}` | `PLATFORM_ADMIN`        | Get plan by code                                     |
| `GET /internal/plans`         | None (internal network) | Full plan feature catalog for service-to-service use |
| `GET /internal/plans/public`  | None (internal network) | Full plan catalog for public pricing pages           |

Plan fields: `planCode`, `displayName`, `description`, `billingPeriod` (MONTHLY/ANNUAL), `priceMinor` (cents), `currency`, `entitlement`, `scope` (TENANT/USER), `active`, `pricingModel` (FLAT/PER_SEAT), `trialPeriodDays`, `gatewayType`, `externalVariantId`

`PlanFeatureRegistry` serves an in-memory map loaded at startup for O(1) entitlement evaluation. `PlanEntitlement` has typed quotas (`maxUsers`, `maxProjects`) and an open `Map<String, PlanFeature>` keyed by feature code.

#### Pricing Models

| Model            | Stripe line item       | Lemon Squeezy variant                     | `priceMinor` meaning              |
| ---------------- | ---------------------- | ----------------------------------------- | --------------------------------- |
| `FLAT` (default) | `quantity = 1` always  | Variant with quantity 1                   | Total price per billing period    |
| `PER_SEAT`       | `quantity = seatCount` | Variant with quantity 1, adjusted via API | Price per seat per billing period |

`maxUsers` doubles as the seat ceiling for `PER_SEAT` plans (0 = unlimited). Checkout routing: `resolveEffectiveQuantity` returns 1 for FLAT plans, caller-supplied value for PER_SEAT. `validateSeatCount` throws `SeatLimitExceededException` → HTTP 422 when `requestedSeats > maxUsers`.

#### Subscriptions

| Endpoint                                                  | Auth              | Description                                          |
| --------------------------------------------------------- | ----------------- | ---------------------------------------------------- |
| `GET /subscriptions/{tenantKey}/active`                   | `TENANT_OWNER`    | Active subscription for tenant                       |
| `GET /subscriptions/{tenantKey}`                          | `TENANT_OWNER`    | All subscriptions for tenant                         |
| `POST /subscriptions/{tenantKey}/checkout`                | `TENANT_OWNER`    | Create checkout session (Stripe/Lemon Squeezy)       |
| `POST /subscriptions/{tenantKey}/{subscriptionId}`        | `TENANT_OWNER`    | Update existing subscription                         |
| `PATCH /subscriptions/{tenantKey}/{subscriptionId}/seats` | `TENANT_OWNER`    | Adjust seat count (PER_SEAT plans only); returns 204 |
| `GET /subscriptions/me/active`                            | Any authenticated | Active subscription for current subject              |
| `GET /subscriptions/me`                                   | Any authenticated | All subscriptions for current subject                |
| `GET /admin/subscriptions`                                | `PLATFORM_ADMIN`  | Global paginated list                                |
| `GET /admin/subscriptions/{id}`                           | `PLATFORM_ADMIN`  | Detail view                                          |
| `PATCH /admin/subscriptions/{id}`                         | `PLATFORM_ADMIN`  | Partial update                                       |
| `POST /admin/subscriptions/{id}/cancel`                   | `PLATFORM_ADMIN`  | Cancel via payment gateway                           |
| `POST /admin/subscriptions/{id}/pause`                    | `PLATFORM_ADMIN`  | Pause via payment gateway                            |
| `POST /admin/subscriptions/{id}/reactivate`               | `PLATFORM_ADMIN`  | Reactivate via payment gateway                       |
| `DELETE /admin/subscriptions/{id}`                        | `PLATFORM_ADMIN`  | Delete subscription                                  |

Subscription response includes `isInTrial` (boolean) and `trialDaysLeft` (number).

Subject resolution: MULTI_TENANT → `TENANT/tenantKey`; SINGLE_TENANT → `USER/userId`.

#### Entitlements

| Endpoint               | Auth              | Description                                                                                                                         |
| ---------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `GET /entitlements/me` | Any authenticated | Active plan + subscription status + typed features for current subject. Always returns 200 (free plan when no active subscription). |

Response includes `planCode`, `status`, `isInTrial`, `trialDaysLeft`, `currentPeriodEnd`, and full `PlanEntitlement` record.

#### Billing Settings

| Endpoint                                                          | Auth             | Description                       |
| ----------------------------------------------------------------- | ---------------- | --------------------------------- |
| `GET/POST/PATCH /settings/{tenantKey}`                            | `TENANT_OWNER`   | Tenant billing settings CRUD      |
| `POST /settings/{tenantKey}/portal`                               | `TENANT_OWNER`   | Create customer portal session    |
| `GET/POST/PUT/PATCH/DELETE /admin/tenants/{key}/billing-settings` | `PLATFORM_ADMIN` | Admin billing settings management |

Billing settings fields: `externalCustomerId`, `billingEmail`, `companyName`, `billingAddress` (JSONB), `taxId`, `taxIdType`, `currency`, `gatewayType`.

#### Payments & Refunds

| Endpoint                            | Auth             | Description                  |
| ----------------------------------- | ---------------- | ---------------------------- |
| `POST /payments/{tenantKey}/refund` | `TENANT_OWNER`   | Create refund                |
| `GET /payments/{tenantKey}/refunds` | `TENANT_OWNER`   | List refunds for tenant      |
| `GET /admin/refunds`                | `PLATFORM_ADMIN` | Global paginated refund list |
| `GET /admin/refunds/{id}`           | `PLATFORM_ADMIN` | Get refund by ID             |

#### Webhook Processing

| Gateway       | Endpoint                       | Verification Method            |
| ------------- | ------------------------------ | ------------------------------ |
| Stripe        | `POST /webhooks/stripe`        | Stripe signature verification  |
| Lemon Squeezy | `POST /webhooks/lemon-squeezy` | HMAC-SHA256 with `X-Signature` |

Handled events (both gateways normalized): subscription.created/updated/deleted, invoice.created/finalized/paid/updated, payment.failed, charge.refunded.

#### Event-Driven Integrations

- Consumes `tenant.created/provisioned` → creates payment gateway customer, stores `externalCustomerId`
- Consumes `tenant.suspended` → sends `ACCOUNT_SUSPENDED` notification
- Consumes `user.removed/deleted` → clears `profileOwnerId` on billing settings
- Publishes `subscription.created`, `subscription.updated`, `subscription.cancelled`, `invoice.created`, `invoice.finalized`, `invoice.paid`, `invoice.updated`, `payment.failed`, `refund.created`

#### Scheduled Jobs

- Daily 9AM UTC: trial-ending notifications (2–3 days before expiry), guarded by ShedLock
- Daily 10AM UTC: payment-overdue notifications for `past_due` subscriptions, guarded by ShedLock

#### Observability (Billing)

Custom metrics: `billing_revenue_total`, `billing_payments_total`, `billing_subscriptions_active_count`, `billing_subscriptions_total`, `billing_seat_adjustments_total`, `billing_webhooks_total`, `billing_webhooks_processing_duration_seconds`. Grafana dashboard included.

---

### Audit Service (`foundation-audit-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · Micrometer + Prometheus

#### Core Capabilities

| Feature                      | Detail                                                                                                                                                                      |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Passive observation          | Binds to `iqkv.events` with wildcard routing keys (`user.#`, `tenant.#`, `subscription.#`, `invoice.#`, `audit.#`); zero code changes in domain services for basic auditing |
| Event normalization          | Maps `UserEvent`, `TenantEvent`, etc. into unified `AuditRecord` format                                                                                                     |
| Technical context enrichment | Captures client IP and User-Agent from `X-Audit-IP` / `X-Audit-UA` headers                                                                                                  |
| JSONB metadata storage       | Full domain event payload preserved; custom MyBatis `TypeHandler` for type-safe handling                                                                                    |
| Dedicated database           | High-volume audit logging isolated from business-critical transactions                                                                                                      |
| SPI pattern                  | `AuditProvider` interface; `PostgresAuditProvider` is the default; plug in Elasticsearch or custom SIEMs without touching core                                              |
| `AuditRecord` JavaBean       | Converted for MyBatis type handling; `JsonbTypeHandler` wired programmatically                                                                                              |
| Severity filter              | Search and count endpoints support `severity` filter; `ActivitySeverity` enum                                                                                               |
| JWT claim alignment          | JWT claim names aligned across all consuming services                                                                                                                       |
| RabbitMQ consumer fix        | Message converter, DLQ name, conditional guards, `tenantKey` mapping all properly wired                                                                                     |

#### Admin Search API

| Endpoint                  | Auth             | Description                                                                             |
| ------------------------- | ---------------- | --------------------------------------------------------------------------------------- |
| `GET /api/v1/audits`      | `PLATFORM_ADMIN` | Search audit logs (paginated, filterable by user, tenant, action, severity, date range) |
| `GET /api/v1/audits/{id}` | `PLATFORM_ADMIN` | Get detailed audit record with JSONB metadata                                           |

#### Observability (Audit)

Custom metrics: `audit.event.consumption` (by type and source), `audit.persistence.duration`, `audit.search.latency`, `audit.storage.usage`.

---

### CMS Service (`foundation-cms-service`)

**Tech stack:** Java 25 / Spring Boot 4.1 · MyBatis 3.x · PostgreSQL 17 · Liquibase · RabbitMQ · Micrometer + Prometheus

#### Core Capabilities

| Feature                | Detail                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------- |
| Static page management | Create/edit/publish pages with draft/published status and page templates                     |
| Multi-language support | `PageTranslation` entity per locale; `en-US` fallback if no translation for requested locale |
| Hierarchical content   | Parent/child page relationships with SEO-friendly slugs                                      |
| SEO metadata           | Per-page title, description, Open Graph tags, canonical URLs                                 |
| Tenant isolation       | Schema-per-tenant via `MyBatisSchemaInterceptor`                                             |
| Event publishing       | `cms.page.created`, `cms.page.updated`, `cms.page.deleted` on `iqkv.events` exchange         |
| Plan feature awareness | `PlanEntitlement` record includes `pricingModel` and `isPerSeat()` helper                    |

#### API

| Endpoint                                                  | Auth                 | Description                         |
| --------------------------------------------------------- | -------------------- | ----------------------------------- |
| `GET /api/v1/cms/pages`                                   | None + `X-Tenant-ID` | List all published pages for locale |
| `GET /api/v1/cms/pages/{slug}`                            | None + `X-Tenant-ID` | Get published page by slug          |
| `GET/POST/PUT/DELETE /api/v1/cms/admin/{tenantKey}/pages` | `PLATFORM_ADMIN`     | Admin CRUD for pages                |

Custom metrics: `cms_pages_total`, `cms_pages_published_total`, `cms_events_published_total`.

---

## Shared Libraries

### `foundation-tenancy`

**Tech stack:** Java 25 / Spring Boot 4.1 · Liquibase · MyBatis

#### Core Capabilities

| Feature                    | Detail                                                                        |
| -------------------------- | ----------------------------------------------------------------------------- |
| `TenantContext`            | Thread-local tenant context holder                                            |
| `MyBatisSchemaInterceptor` | MyBatis interceptor that sets `search_path` to `t_{tenantKey}` for each query |
| `TenantLiquibaseRunner`    | Runs Liquibase migrations for tenant schemas                                  |
| `TenancyAutoConfiguration` | Spring Boot auto-configuration for tenancy infrastructure                     |

### `foundation-entitlement-plan-resolver-mvc`

**Tech stack:** Java 25 / Spring Boot 4.1 · Spring Web

#### Core Capabilities

| Feature                         | Detail                                                          |
| ------------------------------- | --------------------------------------------------------------- |
| `PlanResolver`                  | Interface for resolving current plan for a subject              |
| `PlanFeatureGuard`              | Annotation for plan feature checking in Spring MVC controllers  |
| `PlanEntitlement`               | Record representing a plan's entitlements                       |
| `PlanFeature`                   | Record representing a single plan feature                       |
| `PlanResolverAutoConfiguration` | Spring Boot auto-configuration for plan resolver infrastructure |

---

## Frontend Applications

### Tenant App (`foundation-ui-app`)

**Tech stack:** React 19 · TypeScript 6 · Vite 8 (SWC) · Mantine UI 9 · mantine-datatable · TanStack Router (file-based) · TanStack Query · Zustand · React Hook Form + Zod · Lingui 6 · Axios · Vitest + Playwright · OxLint / OxFmt

**Architecture:** Feature-Sliced Design (`app → processes → pages → widgets → features → shared`); boundary tests via `pnpm test:arch`.

#### Implemented Features

| Area                      | Status         | Detail                                                                                                       |
| ------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------ |
| Sign-in                   | Done           | Two-step: credentials → tenant discovery; multi-tenant users pick workspace; single-tenant signs in directly |
| Sign-in with OAuth2/OIDC  | Done           | Social login buttons for Google, GitHub, Microsoft; tenant SSO entry point; callback page for token exchange |
| Sign-up                   | Done           | Self-service registration + tenant creation; polls provisioning status until `ACTIVE`                        |
| Forgot / reset password   | Done           | Email flow + token-based reset                                                                               |
| Email verification        | Done           | Token-based verification page (`/verify-email?token=…`)                                                      |
| Accept invitation         | Done           | Works for new and existing users                                                                             |
| Create organization       | Done           | `/create-organization` route for authenticated users                                                         |
| Dashboard                 | Done (basic)   | Workspace name, welcome message, team member count                                                           |
| Team — member list        | Done           | Searchable member list                                                                                       |
| Team — invitations        | Done           | Send, list, revoke (TENANT_OWNER only)                                                                       |
| Team — ban/unban          | Done           | Ban/unban members (TENANT_OWNER only)                                                                        |
| Team — role editing       | Done           | Change member role (TENANT_OWNER only)                                                                       |
| Transfer ownership        | Done           | Transfer ownership to another member (TENANT_OWNER only)                                                     |
| Profile & change password | Done           | View/edit name, change password, organizations and roles                                                     |
| Connected accounts        | Done           | Link/unlink external identity providers; view connected accounts                                             |
| Tenant SSO configuration  | Done           | Tenant owner can configure custom OIDC provider for enterprise SSO                                           |
| Billing self-service      | Done           | Portal access, subscription view, plan catalog, billing info, refunds                                        |
| Plan-based access control | Done           | `EntitlementsProvider`, `FeatureGate`, `useHasFeature`, `useQuota` hooks                                     |
| Tenant settings           | Done           | Organization metadata editing                                                                                |
| In-app notifications      | Done           | Notification bell, dropdown, WebSocket real-time push, notification center                                   |
| Session security          | Done           | Access token in memory; refresh token + tenant key in `sessionStorage`; 30-min inactivity sign-out           |
| Light/dark theme          | Done           | Persisted via Zustand                                                                                        |
| i18n                      | Done (en + bg) | Lingui 6 PO catalogs; locale cookie; `Accept-Language` on API requests; Bulgarian (bg-BG) catalog included   |

#### Billing & Entitlements Architecture

- `EntitlementsProvider` wraps subtrees that need plan data
- `hasFeature(code)` — boolean check against open feature map
- `getQuota(field)` — typed quota value (0 = unlimited)
- `FeatureGate` component — conditional render with optional `showUpgradePrompt`
- Default fallbacks: personal workspace (maxUsers: 1), free tenant (maxUsers: 1, maxProjects: 1)

#### Routes

`/sign-in` · `/signup` · `/forgot-password` · `/reset-password` · `/verify-email` · `/invite/:token` · `/create-organization` · `/` (dashboard) · `/team` · `/billing` · `/account/settings/general` · `/account/settings/security` · `/account/settings/organization` · `/account/settings/notifications` · `/unauthorized` · `/404` · `/500`

---

### Platform Admin (`foundation-ui-platform-admin`)

**Tech stack:** React 19 · TypeScript · Vite + SWC · Mantine UI 9 · mantine-datatable · TanStack Router · TanStack Query · Zustand · Lingui · Zod + Mantine Form · Vitest + Playwright · OxLint / OxFmt

**Architecture:** FSD-style layers; `pnpm test:arch` enforces boundaries.

#### Implemented Features

| Area                        | Status         | Detail                                                                                  |
| --------------------------- | -------------- | --------------------------------------------------------------------------------------- |
| Sign-in & session           | Done           | `PLATFORM_ADMIN` credentials; access token in memory; refresh token in `sessionStorage` |
| Dashboard                   | Done           | Parallel count cards: total users, organizations, active subscriptions                  |
| User list & detail          | Done           | List + edit/set password; Overview + Organizations + Identities tabs; ban/unban/unlock  |
| Organization list & detail  | Done           | List + Overview, Members, Billing, Subscriptions, Refunds tabs; edit metadata           |
| Member authority management | Done           | Set TENANT_OWNER/ADMIN/MEMBER per tenant                                                |
| Invitations                 | Done           | List with filters; propose, edit, revoke                                                |
| Subscriptions               | Done           | Read-only global list + detail view                                                     |
| Plan catalog                | Done           | Read-only list; plans are config-driven via YAML + deployment                           |
| Announcements               | Done           | Create, edit, publish, delete with multi-lingual translation support                    |
| Audit Logs                  | Done           | Global audit log view; filter by user, tenant, action, severity, date range             |
| Notifications               | Done           | In-app + WebSocket real-time push; notification bell                                    |
| Refunds                     | Done           | Global refund list + detail view                                                        |
| Operator account            | Done           | View/edit profile; change password                                                      |
| i18n                        | Done (en + bg) | Lingui; English and Bulgarian catalogs; locale switcher UI                              |
| Runtime config              | Done           | Override `VITE_*` via `public/config.js` without rebuild                                |
| User identity management    | Done           | View linked identities for users; force-unlink external identities                      |
| Platform actions            | Partial        | Ban/unban/unlock done; impersonation planned                                            |

#### Not Yet Implemented

Platform actions (impersonation), subscription lifecycle mutations from admin UI, system health/background jobs monitoring, advanced dashboard metrics (MRR/ARR, growth charts), multi-tab user/org detail views (auth history, notes, activity).

#### Routes

`/sign-in` · `/unauthorized` · `/admin` · `/admin/users` · `/admin/users/:userId` · `/admin/organizations` · `/admin/organizations/:tenantKey` · `/admin/organizations/:tenantKey/members` · `/admin/organizations/:tenantKey/billing` · `/admin/organizations/:tenantKey/subscriptions` · `/admin/organizations/:tenantKey/refunds` · `/admin/invitations` · `/admin/subscriptions` · `/admin/subscriptions/:id` · `/admin/plans` · `/admin/plans/:planCode` · `/admin/announcements` · `/admin/audit-logs` · `/admin/notifications` · `/admin/refunds` · `/admin/refunds/:refundId` · `/admin/account`

---

### SaaS Landing Kit (`foundation-ui-saas-landing-kit`)

**Tech stack:** Astro · React · Tailwind CSS · shadcn/ui · Zustand · TypeScript

| Feature               | Detail                                                                                |
| --------------------- | ------------------------------------------------------------------------------------- |
| Static pages          | Home, Features, Pricing, About                                                        |
| Responsive layout     | Mobile-first; `BaseLayout.astro` base template                                        |
| Auth-aware navigation | Login/Sign Up when unauthenticated; user menu with avatar + logout when authenticated |
| Auth state            | Zustand store with `localStorage` persistence                                         |
| Plan selector         | Fetches plans from billing API; per-seat pricing label support                        |
| React islands         | Partial hydration for interactive components                                          |
| Code quality          | OxLint, OxFmt, Husky pre-commit hooks, commitlint                                     |
| Docs link             | Link to user documentation site                                                       |

---

### Documentation Website (`foundation-docs-website`)

VitePress-based documentation website with user guides, platform overview, and quick start instructions. CI pipeline: `VerifyCode` + `PublishArtifacts` only (no deployment pipeline).

---

## Cross-Cutting Platform Capabilities

### Security

| Capability                   | Implementation                                                                                          |
| ---------------------------- | ------------------------------------------------------------------------------------------------------- |
| RS256 JWT                    | JJWT 0.13; RSA private key in IAM; public key distributed via JWKS                                      |
| Two-layer token revocation   | JTI denylist (per-signout) + `last_global_signout_at` (global signout)                                  |
| Identity spoofing prevention | Gateway strips all `X-User-*`, `X-Tenant-ID`, `X-Audit-*`, `X-Plan-Code` headers before JWT propagation |
| Plan spoofing prevention     | Gateway strips client-supplied `X-Plan-Code`; only gateway sets it from validated JWT                   |
| Brute-force protection       | Per-email failed attempt tracking; 5-attempt threshold; 15-min lockout                                  |
| Password policy              | Lowercase + uppercase + digit + special char; 8–128 chars; BCrypt strength 12                           |
| Security response headers    | `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy` on all responses     |
| Audit context propagation    | Gateway injects `X-Audit-IP` / `X-Audit-UA`; services enrich outbound events                            |
| WebSocket authentication     | JWT in STOMP `CONNECT` frame headers; per-user channel isolation                                        |
| S3 upload security           | Presigned URLs with expiry; tenant-scoped bucket prefixes                                               |
| OAuth2/OIDC security         | PKCE flow; Redis-backed state; signed state JWT; AES-256-GCM encrypted tenant OIDC client secrets       |

### Messaging (RabbitMQ)

Exchange: `iqkv.events` (topic). Dead-letter exchange: `iqkv.dlx`. All queues: 24h TTL + dead-letter exchange.

| Event                                    | Publisher | Consumer(s)                               |
| ---------------------------------------- | --------- | ----------------------------------------- |
| `tenant.created`                         | IAM       | Billing (create payment gateway customer) |
| `tenant.provisioned`                     | IAM       | Billing (send activation notification)    |
| `tenant.provisioning_failed`             | IAM       | —                                         |
| `tenant.updated`                         | IAM       | —                                         |
| `tenant.suspended`                       | IAM       | Billing (send suspension notification)    |
| `tenant.deleted`                         | IAM       | —                                         |
| `user.created`                           | IAM       | —                                         |
| `user.updated`                           | IAM       | —                                         |
| `user.invited`                           | IAM       | —                                         |
| `user.removed`                           | IAM       | Billing (clear `profileOwnerId`)          |
| `user.deleted`                           | IAM       | Billing (clear `profileOwnerId`)          |
| `subscription.created`                   | Billing   | IAM (cache `planCode`)                    |
| `subscription.updated`                   | Billing   | IAM (cache `planCode`)                    |
| `subscription.cancelled`                 | Billing   | IAM (suspend tenant)                      |
| `invoice.created/finalized/paid/updated` | Billing   | Audit (logging)                           |
| `payment.failed`                         | Billing   | Audit (logging)                           |
| `refund.created`                         | Billing   | Audit (logging)                           |
| `cms.page.created/updated/deleted`       | CMS       | —                                         |
| `announcement.publish`                   | IAM       | IAM (fan-out consumer)                    |
| `notification.iam.email`                 | IAM       | IAM (email + in-app notification)         |
| `notification.billing.email`             | Billing   | Billing (email worker)                    |
| `audit.*`, `user.#`, `tenant.#`, etc.    | All       | Audit (centralized logging)               |

### Observability Stack

| Component               | Detail                                                                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Prometheus              | Micrometer on all services; scrape at `/actuator/prometheus`                                                                                     |
| Grafana                 | Pre-provisioned dashboards: JVM, IAM (auth/security/lifecycle), Gateway (per-tenant traffic), Billing (MRR/churn/webhooks), Audit (event volume) |
| Loki + Promtail         | Log aggregation and shipping (demo stack)                                                                                                        |
| Structured JSON logging | Logstash Logback Encoder on all services; MDC with correlation ID and tenant context                                                             |
| Correlation ID          | Generated at Gateway; propagated through all downstream services and logs via `X-Correlation-ID`                                                 |
| Health probes           | `/actuator/health` with liveness + readiness; separate management port 8081                                                                      |
| Swagger UI              | Individual service Swagger at `/swagger-ui.html`; Gateway aggregates all at `api.iqkv.local/swagger-ui.html`                                     |

### Local Development

| Mode                        | Command                                                  | Use case                                            |
| --------------------------- | -------------------------------------------------------- | --------------------------------------------------- |
| Per-service (IDE)           | `docker compose up -d` → run from IDE                    | Contributing to a single service                    |
| Per-service (containerized) | `docker compose -f compose.container.yaml up -d --build` | Service without IDE                                 |
| Full demo stack             | `./demo.sh` or `.\demo.ps1`                              | End-to-end evaluation, E2E tests, stakeholder demos |

Demo stack includes: Nginx reverse proxy, all 5 backend services (IAM, Gateway, Billing, Audit, CMS), 4 PostgreSQL instances, RabbitMQ, Redis, MailHog, MinIO, Prometheus, Grafana, Loki, Promtail.

Infrastructure admin tools in demo stack: DbGate (unified DB/Redis/RabbitMQ/MinIO admin), MinIO console.

---

## CI/CD Pipelines (Drone CI)

### Java Services (10 pipelines each)

| Pipeline                    | Trigger                                   | Action                                                                               |
| --------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------ |
| `VerifyCode`                | push/tag on dev/feature/\*/hotfix/\*/tags | `mvn clean verify` → SonarQube → PMD → SpotBugs                                      |
| `PublishArtifacts`          | push to dev/prerelease/\*/tags            | `mvn deploy` SNAPSHOT to Nexus; release JAR on tags; GitHub Release via `release-it` |
| `PublishDockerImage`        | push to wip/feature/\*/tags               | Package JAR; build Docker image; push to private registry                            |
| `DeployWorkInProgress`      | push to wip                               | `helm upgrade --install --atomic` to `iqkv-sit-env`                                  |
| `RollbackWorkInProgress`    | rollback→sit on wip                       | `helm uninstall` from SIT                                                            |
| `PromoteFeatureDeployment`  | promote→sit on feature/\*                 | Helm deploy to SIT with feature image                                                |
| `RollbackFeatureDeployment` | rollback→sit on feature/\*                | `helm uninstall` from SIT                                                            |
| `PromoteDeployment`         | promote→uat/prd on tags                   | Helm deploy to UAT or PRD using semver tag                                           |
| `RollbackDeployment`        | rollback→uat/prd on tags                  | `helm uninstall` from UAT/PRD                                                        |
| `ReleasePackage`            | promote→release on dev/\*.x               | Strip SNAPSHOT, git tag, GitHub release, bump to next SNAPSHOT                       |

Static analysis: SonarQube `5.6.0.6792` + PMD (High priority, custom ruleset) + SpotBugs `4.9.8.3`. SonarQube quality gate required to pass (`sonar.qualitygate.wait=true`). JaCoCo 80% coverage gate.

### Frontend Apps (4 pipelines each)

`VerifyCode` (formatter + lint + test:coverage + SonarQube + build) → `PublishArtifacts` (pnpm publish to Nexus NPM) → `DeployWorkInProgress` (pnpm build + Nginx Docker image + Helm to SIT) → `RollbackWorkInProgress`.

### Library Modules (2 pipelines each)

`VerifyCode` + `PublishArtifacts` only. No Docker image, no deployment.

### Infrastructure (3 pipelines)

`Info` (dry-run `helm template`) → `PromoteInfrastructure` (Helm deploy with `kubectl wait` and psql connectivity checks) → `RollbackInfrastructure` (deep cleanup: `helm uninstall` + force-delete PVCs).

### Docker Build Strategy

Multi-stage builds: Maven compiles in `eclipse-temurin:25-jdk-alpine`, runtime uses `eclipse-temurin:25-jre-alpine` with non-root `appuser` and layered JAR extraction for optimal cache reuse.

Image tag strategy: `wip` → branch name, `feature/*` → stripped name (prefix removed), tags → semver tag.

Dependency caching: host-mounted volumes for Maven cache (`/app/.m2`) and pnpm store (`/app/.pnpm-store`).

---

## Infrastructure as Code

### Helm Charts

Each service has a dedicated Helm chart with environment-specific value files:

| File              | Environment                |
| ----------------- | -------------------------- |
| `values.yaml`     | Default values             |
| `values-sit.yaml` | System integration testing |
| `values-uat.yaml` | User acceptance testing    |
| `values-prd.yaml` | Production                 |

Production HPA: `minReplicas: 2`, `maxReplicas: 10`, `targetCPUUtilizationPercentage: 70`.

Services are stateless and scale independently. Session state (token denylist, global signout timestamp) stored in the database — any replica can handle any request.

PgBouncer included in Helm chart for PostgreSQL connection pooling at scale.

### Kubernetes Namespaces

| Environment | Namespace      |
| ----------- | -------------- |
| SIT         | `iqkv-sit-env` |
| UAT         | `iqkv-uat-env` |
| PRD         | `iqkv-prd-env` |

---

## Service Template

`foundation-microservice-project-layout` serves as the canonical template for adding new microservices. Includes:

- Standard project structure (DDD layers, `infrastructure/`, `shared/`)
- `PlanEntitlement` record with `pricingModel` and `isPerSeat()` helper
- `PlanCatalogCache` (non-reactive, RestTemplate-based) for plan feature lookups
- `PlanFeatureGuard` annotation and `PlanFeatureNotAvailableException` with HTTP 402 mapping
- Drone CI pipeline template (10 pipelines)
- Helm chart structure
- Issue templates, labels, Dependabot, GitHub Actions workflows
- Checkstyle, JaCoCo, ArchUnit, Husky git hooks

---

## Summary: Platform Maturity as of June 2026

### Services

| Service | Status           | API endpoints          |
| ------- | ---------------- | ---------------------- |
| IAM     | Production-ready | 60+ endpoints          |
| Gateway | Production-ready | Routing + filters only |
| Billing | Production-ready | 40+ endpoints          |
| Audit   | Production-ready | 2 admin endpoints      |
| CMS     | Production-ready | 7 endpoints            |

### Platform Numbers

- **100+ REST endpoints** across all services
- **20+ domain event types** on the RabbitMQ event bus
- **60+ UI routes** across Tenant App and Platform Admin
- **4 PostgreSQL databases** (IAM, Billing, Audit, CMS) with schema-per-tenant in IAM and CMS
- **2 pricing models** (`FLAT` and `PER_SEAT`) × 2 billing periods (MONTHLY/ANNUAL) + optional `trialPeriodDays` per plan
- **2 deployment modes**: MULTI_TENANT (SaaS) and SINGLE_TENANT (managed); zero-migration migration path between them
- **2 payment gateways**: Stripe and Lemon Squeezy (runtime configurable)
- **3 social login providers**: Google, GitHub, Microsoft (OAuth2/OIDC)
- **Enterprise SSO**: Tenant-scoped custom OIDC providers

### What's Next (Post-v0.4)

Per `vision.md` deferred items:

- Platform Admin UI — system health dashboard, background job monitoring
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Tenant App — additional locales (RU, IT; infrastructure already in place)
- Per-seat IAM enforcement — enforce purchased `seatCount` against `activeSeatCount` on tenant (not just plan `maxUsers`)
- SSO / SAML adapter
- Rate limiting (per-tenant and per-user, at Gateway)
- Tenant resolution by subdomain
- Usage-based metered billing (`METERED` pricing model; per-seat flat pricing is complete)
- Multi-region support
- Managed hosting offering
