# Capabilities

Status key: ✅ implemented · 🚧 partial · 📋 planned

Repositories: `foundation-iam-service`, `foundation-gateway-service`, `foundation-billing-service`, `foundation-audit-service`, `foundation-audit-spi`, `foundation-audit-model`, `foundation-ui-app` (tenant), `foundation-ui-platform-admin` (operator), `foundation-microservice-project-layout` (service template).

---

## IAM

Identity, access, and tenant lifecycle. All auth flows pass through this service.

| Capability                | Notes                                                                                                                                                                                                                                                                | Status |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Signup                    | Email/password with self-service tenant creation; email verification required before access; supports both multi-tenant (new tenant per signup) and single-tenant (join default tenant) modes                                                                        | ✅     |
| Signup status             | `GET /auth/signup/status/{tenantKey}` — poll tenant provisioning until `ACTIVE` after signup                                                                                                                                                                         | ✅     |
| Authentication            | JWT RS256 access token (15 min) + refresh token (7 day); tokens carry user context and tenant membership; JJWT library with custom claims                                                                                                                            | ✅     |
| Token exchange            | `POST /auth/exchange` — exchange a Bearer access token for a new tenant-scoped token pair; used by multi-tenant UI for workspace switching without re-authentication                                                                                                 | ✅     |
| Platform admin auth       | `POST /auth/admin/signin` and `/auth/admin/refresh` — platform-scoped token pair (`tenant_id` null); operator self-service at `/auth/admin/me`                                                                                                                       | ✅     |
| Account recovery          | Password reset via signed email token (1h TTL), rate-limited (3 requests per 15min window); Thymeleaf email templates                                                                                                                                                | ✅     |
| Brute-force lockout       | Failed login tracking per email; temporary account lock after 5 attempts for 15 minutes; automatic cleanup of expired lockout records                                                                                                                                | ✅     |
| Token revocation          | JTI denylist (single session) + global signout timestamp; both access and refresh tokens validated against denylist; automatic cleanup of expired denylist entries                                                                                                   | ✅     |
| JWKS endpoint             | `/.well-known/jwks.json` — gateway and downstream services validate RS256 tokens locally; public key rotation support                                                                                                                                                | ✅     |
| Organizations             | Create, update, suspend, delete; async provisioning via RabbitMQ with ShedLock-guarded reaper for stuck tenants; automatic retry mechanism for failed provisioning                                                                                                   | ✅     |
| RBAC                      | Tenant authorities: `TENANT_OWNER`, `ADMIN`, `MEMBER`; platform authority: `PLATFORM_ADMIN`; per-tenant membership with independent authorities; `@PreAuthorize` on endpoints                                                                                        | ✅     |
| Platform admin APIs       | Paginated admin CRUD for users and tenants; cross-tenant invitations; member authority management; force-set password                                                                                                                                                | ✅     |
| OAuth2 / OIDC federation  | Social login for Google, GitHub, Microsoft, plus tenant-scoped custom OIDC providers; IAM brokers external identity into the same internal RS256 JWT contract                                                                                                        | ✅     |
| Account linking           | Existing users can link / unlink external providers; external identities stored in `user_identities`; last-credential guard on unlink                                                                                                                                | ✅     |
| Tenant SSO                | TENANT_OWNER can configure a per-tenant OIDC provider in `tenant_oidc_providers`; client secrets encrypted with AES-256-GCM; provider available as `oidc:{tenantKey}`                                                                                                | ✅     |
| OIDC admin remediation    | `PLATFORM_ADMIN` endpoints to list linked identities and force-unmerge an external identity from a user account                                                                                                                                                      | ✅     |
| Invitations               | Email invite with 72h expiring token; `authority` defaults to `MEMBER` (also `ADMIN`); new users created on accept (email pre-verified); existing users verified by password; ShedLock-guarded reaper                                                                | ✅     |
| Multi-org                 | One user can belong to multiple organizations with different authorities; tenant discovery by credentials; cross-tenant user context switching                                                                                                                       | ✅     |
| Rollout mode              | `MULTI_TENANT` (default) or `SINGLE_TENANT` — configured via `platform.rollout-mode`; single-tenant provisions one default tenant at startup; mode consistency enforced across services                                                                              | ✅     |
| Events                    | Publishes `tenant.created`, `tenant.provisioned`, `tenant.suspended`, `user.invited`, `user.removed` via RabbitMQ; Billing consumes provisioning events                                                                                                              | ✅     |
| Schema isolation          | PostgreSQL schema-per-tenant with `t_` prefix; MyBatis interceptor for automatic schema switching; Liquibase migrations per tenant; identical model in both single and multi-tenant modes                                                                            | ✅     |
| Email notifications       | Thymeleaf-rendered transactional emails (verification, password reset, invitations) via SMTP; MailHog for local dev                                                                                                                                                  | ✅     |
| Email verification        | Secure token-based email verification; resend capability with rate limiting; verification status tracking                                                                                                                                                            | ✅     |
| Token validation          | `POST /auth/validate` — introspection for gateway; validates signature, expiry, denylist, and global signout; returns user context                                                                                                                                   | ✅     |
| Tenant discovery          | `POST /users/tenants` — credential-gated discovery of tenant memberships (used by tenant UI sign-in)                                                                                                                                                                 | ✅     |
| Scheduled jobs            | ShedLock-protected background jobs: token denylist cleanup, invitation expiry, stuck tenant reaper, email verification cleanup                                                                                                                                       | ✅     |
| Bootstrap strategies      | Pluggable tenant bootstrap for different rollout modes; default tenant resolution and creation; startup-time tenant provisioning for single-tenant mode                                                                                                              | ✅     |
| Avatar uploads            | Two-phase presigned S3/MinIO flow: initiate returns presigned PUT URL, client uploads directly, confirm persists URL; old avatars auto-deleted on replacement or account deletion                                                                                    | ✅     |
| In-app notifications      | `UserNotification` records persisted to DB; real-time push via WebSocket (STOMP/SockJS) to `/user/{userId}/queue/notifications`; unread count badge; bulk mark-as-read; delete                                                                                       | ✅     |
| Announcements             | Platform admins create multi-lingual announcements; publish triggers async fan-out in batches of 1000; broadcasts to all connected clients via `/topic/announcements`; public locale-filtered read API                                                               | ✅     |
| Observability             | Prometheus metrics, structured JSON logging with correlation IDs, health checks, actuator endpoints; `platform.rollout-mode` on `/actuator/info`                                                                                                                     | ✅     |
| User ban/unban            | Platform admin can ban/unban globally; tenant owner can ban/unban within tenant; banned users automatically logged out, receive email, cannot log in; optional reason and expiration; REST APIs at /users/{userId}/ban and /tenants/{tenantKey}/members/{userId}/ban | ✅     |
| Member authority edit     | Update member authorities (TENANT_OWNER/MEMBER); cannot remove last TENANT_OWNER from tenant; cannot remove your own TENANT_OWNER authority if you are the last owner; REST API at /tenants/{tenantKey}/members/{userId}/authorities                                 | ✅     |
| Transfer ownership        | Transfer tenant ownership to another member; old owner becomes MEMBER; REST API at /tenants/{tenantKey}/members/{userId}/transfer-ownership                                                                                                                          | ✅     |
| Unlock user               | Platform admin can unlock user by resetting failed login attempts; REST API at /admin/users/{userId}/unlock                                                                                                                                                          | ✅     |
| Plan code caching         | Active plan code cached on tenant; stamped into JWT as `plan_code` claim; updated via subscription lifecycle events                                                                                                                                                  | ✅     |
| Tenant user stats         | Endpoints: `GET /admin/tenants/{tenantKey}/stats` (platform admin), `GET /tenants/{tenantKey}/stats` (owner/admin); gated behind `advanced_analytics` plan feature                                                                                                   | ✅     |
| Max users quota           | Quota check on invitation acceptance and single-tenant signup; throws `PlanMemberQuotaException` → 402 Payment Required; 0 = unlimited                                                                                                                               | ✅     |
| Plan feature guard        | `PlanFeatureGuard` annotation for checking plan features; `PlanFeatureNotAvailableException` with HTTP 402 mapping                                                                                                                                                   | ✅     |
| Personal workspace        | Personal workspace always accessible, even if tenant is suspended                                                                                                                                                                                                    | ✅     |
| Avatar URL rewrite        | Presigned URLs rewritten to public endpoint for external clients to avoid mixed-content issues                                                                                                                                                                       | ✅     |
| Common exception handlers | Shared Spring Web exception handlers for consistent error responses across services                                                                                                                                                                                  | ✅     |
| Bulgarian i18n            | Bulgarian (bg-BG) translations added; locale seed data; announcements support multi-lingual content                                                                                                                                                                  | ✅     |
| Magic link authentication | Passwordless sign-in: `POST /auth/magic-link/initiate`, `/resend`, `/exchange`; configurable TTL; rate-limited resend; always 204 on initiate/resend (prevents enumeration)                                                                                          | ✅     |
| Create tenant endpoint    | `POST /tenants` — authenticated endpoint to create a new tenant after signup; calling user becomes `TENANT_OWNER`; returns `tenantKey`                                                                                                                               | ✅     |
| User memberships list     | `GET /users/me/memberships` — lists current user's active tenant memberships for org-switcher UIs; no `X-Tenant-ID` required                                                                                                                                         | ✅     |

---

## API Gateway

Entry point for all client traffic. No request reaches IAM or Billing without passing through here.

| Capability                | Notes                                                                                                                                           | Status |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Routing                   | Path-based routing to IAM, Billing, and Audit; Spring Cloud Gateway with WebFlux; Stripe webhooks on public path                                | ✅     |
| JWT validation            | Validates RS256 tokens against IAM JWKS endpoint; extracts authorities from JWT claims                                                          | ✅     |
| Header sanitization       | Strips client-supplied `X-User-*`, `X-Tenant-ID`, `X-Organization-ID`, and `X-Audit-*` before JWT processing — prevents identity spoofing       | ✅     |
| Context propagation       | Extracts user context from JWT and forwards: `X-User-ID`, `X-Username`, `X-User-Email`, `X-User-Authorities`, `X-Tenant-ID`, `X-Correlation-ID` | ✅     |
| Audit context             | `AuditContextFilter` (order -180): extracts client IP and User-Agent; forwards as `X-Audit-IP`, `X-Audit-UA`, `X-Audit-Source`                  | ✅     |
| Monitoring filter         | `MonitoringFilter` (order -201): records request rate, latency, status, and `tenant_id` per route; Grafana dashboard included                   | ✅     |
| Tenant context filter     | `SINGLE_TENANT`: auto-injects `default-tenant-key` when absent; `MULTI_TENANT`: tenant from JWT only                                            | ✅     |
| Public paths              | Configurable public endpoints (JWKS, auth, webhooks, health, Swagger UI); enforced by `SecurityConfig` + `GatewayProperties`                    | ✅     |
| Platform mode guard       | Polls IAM `/actuator/info` every 60s; blocks traffic with 503 on rollout-mode mismatch; fail-open if IAM unreachable                            | ✅     |
| Response security         | `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy`; echoes correlation ID                                       | ✅     |
| CORS                      | Global CORS configuration with configurable origins, methods, and headers                                                                       | ✅     |
| Request logging           | Structured logs with correlation ID filter for request tracing                                                                                  | ✅     |
| Observability             | Prometheus metrics, health checks, and actuator endpoints on separate management port                                                           | ✅     |
| Swagger aggregation       | Aggregates API documentation from downstream services (IAM, Billing) in unified Swagger UI                                                      | ✅     |
| Metering events           | Publishes `api.request.metered` per request                                                                                                     | 📋     |
| Plan code header          | Extracts `plan_code` from JWT and propagates as `X-Plan-Code`; sanitizes client-supplied `X-Plan-Code` to prevent spoofing                      | ✅     |
| Plan catalog cache        | Reactive cache (`PlanCatalogCache`) that refreshes from billing service every 10 minutes; stores active plans with full details                 | ✅     |
| Plan feature filter       | `RequiresPlanFeatureFilterFactory` for declarative route-level plan feature enforcement in Spring Cloud Gateway routes                          | ✅     |
| Public plans endpoint     | Exposes billing service internal plans endpoint on public path                                                                                  | ✅     |
| WebSocket X-Tenant-ID     | Allows `X-Tenant-ID` header passthrough on WebSocket paths                                                                                      | ✅     |
| Magic link authentication | Passwordless sign-in: `POST /auth/magic-link/initiate`, `/resend`, `/exchange`; configurable TTL and rate limiting                              | ✅     |
| Create tenant endpoint    | `POST /tenants` — authenticated endpoint for creating new tenant after signup (owner is caller)                                                 | ✅     |

---

## Audit

Centralized, event-driven activity logging. Passive observation of platform events.

| Capability           | Notes                                                                                                                                                              | Status |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| Event consumption    | RabbitMQ topic listeners for `user.#`, `tenant.#`, `billing.#`, etc.; automatic normalization into `AuditRecord`                                                   | ✅     |
| Context propagation  | Technical context (IP, User-Agent) captured at Gateway and propagated via headers; `foundation-audit-spi` provides `AuditContextHolder` and `AuditContextEnricher` | ✅     |
| Search API           | Secured REST endpoints for paginated audit log search; filtered by tenant; restricted to `PLATFORM_ADMIN`                                                          | ✅     |
| Extensible backends  | `AuditProvider` SPI for pluggable storage (PostgreSQL implemented; Elasticsearch planned)                                                                          | ✅     |
| Metadata support     | JSONB storage for dynamic event details; preserves full domain event payload for deep inspection                                                                   | ✅     |
| Enrichment           | Domain services automatically decorate outbound events with audit context via shared `MessagingService`                                                            | ✅     |
| Severity filter      | Audit log search and count endpoints support `severity` filter parameter; `ActivitySeverity` enum                                                                  | ✅     |
| AuditRecord JavaBean | Converted AuditRecord to JavaBean for MyBatis type handling; wired JsonbTypeHandler programmatically                                                               | ✅     |
| Consumer setup fix   | Fixed RabbitMQ consumer setup: message converter, DLQ name, conditional guards, tenantKey mapping                                                                  | ✅     |
| JWT claim alignment  | Aligned JWT claim names across services                                                                                                                            | ✅     |

---

## Billing

Payment gateway abstraction with plan catalog and subscription state. The active gateway is selected via `iqkv.payment.gateway.type` — Stripe and Lemon Squeezy are both supported behind the same gateway-agnostic `PaymentGatewayPort`, with adapters wired conditionally via `@ConditionalOnGateway`.

| Capability                    | Notes                                                                                                                                                                                                                                    | Status |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Payment gateway port          | `PaymentGatewayPort` hexagonal abstraction — 11 gateway-agnostic methods; active adapter selected via `iqkv.payment.gateway.type`                                                                                                        | ✅     |
| `@ConditionalOnGateway`       | Meta-annotation + `GatewayCondition` for clean `STRIPE` / `LEMON_SQUEEZY` bean wiring; prevents both adapters being active simultaneously                                                                                                | ✅     |
| Stripe adapter                | `@ConditionalOnGateway(STRIPE)` — customer provisioning on `tenant.created`; webhook at `/api/v1/billing/webhooks/stripe`; product/price sync at startup                                                                                 | ✅     |
| Lemon Squeezy adapter         | `@ConditionalOnGateway(LEMON_SQUEEZY)` — Spring `RestClient` against `https://api.lemonsqueezy.com/v1/`; webhook at `/api/v1/billing/webhooks/lemon-squeezy`; `syncProduct` is read-only variant verification (dashboard-managed)        | ✅     |
| Plan catalog                  | Admin-managed plans (`planCode`, pricing, `featureSet`, `scope` TENANT/USER, `trialPeriodDays`); tenant read APIs; soft-delete via `active=false`                                                                                        | ✅     |
| Plan catalog config path      | `iqkv.billing.plan-catalog.products` — renamed from `iqkv.billing.stripe.schema.products`; `externalVariantId` field added for LS variant IDs pre-configured in YAML                                                                     | ✅     |
| Plan eligibility              | Validates plan `scope` matches rollout mode and subject type at subscription creation                                                                                                                                                    | ✅     |
| Billing settings              | Per-tenant `billing_settings`: gateway customer ID, billing email, company info, tax ID, billing address; `gateway_type` column records owning adapter                                                                                   | ✅     |
| Customer Portal               | `POST /billing/settings/{tenantKey}/portal` — Stripe: Customer Portal session; LS: customer portal session via `/v1/customer-portal-sessions`                                                                                            | ✅     |
| Refunds                       | `POST /billing/payments/{tenantKey}/refund` — initiate refund; `GET /billing/payments/{tenantKey}/refunds` — list; admin overview; LS refunds target Order ID stored in `subscriptions.external_order_id`                                | ✅     |
| Platform admin APIs           | Admin CRUD for billing settings, subscriptions, plans, and refunds under `/admin/*`                                                                                                                                                      | ✅     |
| User billing                  | `user_billing_settings` for single-tenant mode; auto-created gateway customer per user on first access                                                                                                                                   | ✅     |
| Subscription cache            | Local cache of gateway subscription state; updated via webhooks for fast reads without gateway API calls                                                                                                                                 | ✅     |
| Subject resolution            | `SubscriptionSubjectResolver` — TENANT-scoped (multi-tenant) vs USER-scoped (single-tenant)                                                                                                                                              | ✅     |
| Entitlement evaluation        | `EntitlementEvaluator` — active subscription + plan `featureSet` for authorization decisions                                                                                                                                             | ✅     |
| Multi-mode support            | Supports both multi-tenant (tenant-scoped) and single-tenant (user-scoped) billing models                                                                                                                                                | ✅     |
| Webhook processing            | Idempotent webhook handling; normalized `GatewayWebhookEvent` sealed hierarchy; `gateway_type` written to subscription on processing                                                                                                     | ✅     |
| LS webhook event mapping      | LS `meta.event_name` values mapped to normalized event types: `subscription_created/updated/cancelled/paused` → `GatewaySubscriptionEvent`; `subscription_payment_success/failed` → `GatewayInvoiceEvent` / `GatewayPaymentFailureEvent` | ✅     |
| REST API                      | Settings, subscriptions (`/{tenantKey}` and `/me`), plan catalog; JWT + tenant isolation                                                                                                                                                 | ✅     |
| Lifecycle events              | Publishes `subscription.created`, `subscription.cancelled`, `invoice.paid`, `payment.failed`, `refund.created` via RabbitMQ                                                                                                              | ✅     |
| Email notifications           | Billing notification types published to RabbitMQ (subscription, trial, invoice, payment events) — consumed by email worker                                                                                                               | ✅     |
| Scheduled jobs                | ShedLock-protected trial-ending (9 AM UTC) and payment-overdue (10 AM UTC) notification jobs                                                                                                                                             | ✅     |
| Tax compliance                | Tax ID/VAT/GST storage and Stripe metadata sync for B2B invoicing                                                                                                                                                                        | ✅     |
| Webhook idempotency           | `webhook_log` table tracks processed events; prevents duplicate processing                                                                                                                                                               | ✅     |
| Email resolution              | Multi-tenant: `billing_settings.billingEmail`; single-tenant: `user_billing_settings.billingEmail` with fallback chain                                                                                                                   | ✅     |
| Observability                 | Prometheus metrics (revenue, subscriptions, webhook health), structured JSON logging, health checks; Grafana dashboard included                                                                                                          | ✅     |
| YAML plan config              | Plan features defined in YAML config (application-prd.yaml, etc.); single source of truth                                                                                                                                                | ✅     |
| PlanFeatureRegistry           | In-memory registry for O(1) plan feature lookups; loads from YAML config                                                                                                                                                                 | ✅     |
| Typed plan features           | Plan features split into typed quotas (`maxUsers`, `maxProjects`) and open feature map (`Map<String, PlanFeature>`); no hot-path DB calls                                                                                                | ✅     |
| Internal plans API            | `GET /api/v1/billing/internal/plans` — public internal endpoint (no auth) returns all active plans with full details                                                                                                                     | ✅     |
| User entitlements API         | `GET /api/v1/billing/entitlements/me` — returns user/tenant entitlements with active plan features and status                                                                                                                            | ✅     |
| Gateway product sync          | Stripe: retrieve/update products at startup; LS: read-only variant verification — products/variants are dashboard-managed in LS                                                                                                          | ✅     |
| Plan description              | Plan description added to plan catalog; includes display name, pricing, description, maxUsers, maxProjects, features                                                                                                                     | ✅     |
| Plan catalog read-only        | Plan catalog made read-only; removed admin CRUD endpoints                                                                                                                                                                                | ✅     |
| Subscription updated event    | Publishes `subscription.updated` event with planCode                                                                                                                                                                                     | ✅     |
| Subject context fix           | Preserves subject context on `subscription.cancelled` webhook events                                                                                                                                                                     | ✅     |
| No active sub entitlement     | `GET /api/v1/billing/entitlements/me` always returns 200 OK with free plan entitlements when no active subscription (instead of 404)                                                                                                     | ✅     |
| Subscription trial fields     | Subscription responses include `isInTrial` (boolean) and `trialDaysLeft` (number) for better UX                                                                                                                                          | ✅     |
| Per-seat pricing              | `PricingModel` enum (`FLAT` / `PER_SEAT`) on plan; `pricing_model` column in `plan_catalog` (Liquibase migration, `DEFAULT 'FLAT'`); `effectivePricingModel()` on `ProductSchema` — backward-compatible default                          | ✅     |
| Seat checkout routing         | `SubscriptionService.resolveEffectiveQuantity` — `FLAT` plans always use `quantity=1`; `PER_SEAT` plans forward caller-supplied seat count; applied to both `createCheckoutSession` and `createCheckoutSessionForSubject`                | ✅     |
| Seat cap validation           | `validateSeatCount` throws `SeatLimitExceededException` (HTTP 422) when `requestedSeats > maxUsers` (and `maxUsers > 0`); `maxUsers=0` means unlimited                                                                                   | ✅     |
| Seat adjustment API           | `PATCH /api/v1/billing/subscriptions/{tenantKey}/{subscriptionId}/seats` — mid-cycle seat changes with configurable proration (`create_prorations` default); rejects non-`PER_SEAT` plans; authority: `TENANT_OWNER` or `ADMIN`          | ✅     |
| seatCount in events           | `SubscriptionEvent` carries `seatCount` (nullable `Long`); `null` for `FLAT` plans; `publishSubscriptionCreated` / `publishSubscriptionUpdated` overloads with explicit `seatCount`; old overloads deprecated, delegate with `null`      | ✅     |
| pricingModel in plan APIs     | `GET /api/v1/billing/internal/plans` and `/public` both expose `pricingModel`; `PlanFeatureRegistry.pricingModelForPlan()` O(1) lookup; `PlanCatalogEntry` bundles `features + pricingModel`                                             | ✅     |
| gateway_type columns          | `gateway_type VARCHAR(32)` added to `billing_settings`, `subscriptions`, and `plan_catalog`; records which adapter owns each row; populated by adapters at creation time                                                                 | ✅     |
| external_order_id on subs     | `external_order_id VARCHAR(255)` on `subscriptions`; stores LS Order ID from `subscription_payment_success` webhook for order-level refund requests; null for Stripe                                                                     | ✅     |
| gatewayType in webhook events | `gatewayType()` method on `GatewayWebhookEvent` sealed interface; populated by each adapter; written to `subscriptions.gateway_type` during webhook processing                                                                           | ✅     |

---

## CMS

Content management service for static pages, multi-language support, and hierarchical content.

| Capability           | Notes                                                                                                  | Status |
| -------------------- | ------------------------------------------------------------------------------------------------------ | ------ |
| Page management      | Create, edit, delete, publish pages with draft/published status                                        | ✅     |
| Multi-language       | Per-locale page translations with en-US fallback                                                       | ✅     |
| Hierarchical content | Parent-child page relationships with SEO-friendly slugs                                                | ✅     |
| SEO metadata         | Title, description, Open Graph tags, canonical URLs                                                    | ✅     |
| Tenant isolation     | Schema-per-tenant PostgreSQL architecture with MyBatis schema interceptor                              | ✅     |
| Event publishing     | Publishes `cms.page.created`, `cms.page.updated`, `cms.page.deleted` events to RabbitMQ topic exchange | ✅     |
| Public API           | Read-only public endpoints for fetching published pages with locale fallback                           | ✅     |
| Admin API            | Platform admin CRUD endpoints for managing content with tenant isolation                               | ✅     |
| Observability        | Prometheus metrics, health checks, and actuator endpoints on separate management port                  | ✅     |

---

## UI — Tenant app (`foundation-ui-app`)

React 19 + Mantine SPA for workspace members. All requests go through the API Gateway with tenant-scoped JWTs and `X-Tenant-ID`. Static build (Nginx or CDN).

| Screen / flow               | Notes                                                                                                                                                                                                | Status |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Sign-in                     | Credentials → tenant discovery (`POST /users/tenants`); multi-tenant picker; single-tenant direct sign-in                                                                                            | ✅     |
| OAuth2 / OIDC sign-in       | Google / GitHub / Microsoft social login plus enterprise SSO entry via `oidc:{tenantKey}`; callback handled at `/auth/callback`                                                                      | ✅     |
| Sign-up                     | Self-service registration + tenant creation; polls provisioning until `ACTIVE`                                                                                                                       | ✅     |
| Password reset              | Forgot-password email + token-based reset (`/forgot-password`, `/reset-password`)                                                                                                                    | ✅     |
| Email verification          | Token-based verification from email link (`/verify-email`)                                                                                                                                           | ✅     |
| Accept invitation           | Public `/invite/:token` — new and existing users                                                                                                                                                     | ✅     |
| Dashboard                   | Workspace name, welcome, team member count                                                                                                                                                           | ✅     |
| Team — members              | Searchable member list, ban/unban members (`TENANT_OWNER`)                                                                                                                                           | ✅     |
| Team — invitations          | Send, list, revoke pending invitations (`TENANT_OWNER`)                                                                                                                                              | ✅     |
| My account                  | Profile view/edit, change password, organizations and roles; avatar upload; connected accounts (link / unlink OAuth2 providers)                                                                      | ✅     |
| Billing self-service        | Billing portal access (Stripe or Lemon Squeezy), active subscription view (with trial status and days left), plan catalog (with trial badge and per-seat pricing labels), billing info, refunds list | ✅     |
| Tenant settings             | Organization metadata editing                                                                                                                                                                        | ✅     |
| Tenant SSO configuration    | TENANT_OWNER security panel for issuer URI, client ID / secret, scopes, enabled flag; write-only client secret                                                                                       | ✅     |
| Notifications               | In-app notification list, unread badge, mark-as-read, delete; real-time WebSocket push (STOMP/SockJS)                                                                                                | ✅     |
| Session security            | Access token in memory; refresh + tenant key in `sessionStorage`; silent refresh; 30min inactivity timeout                                                                                           | ✅     |
| i18n & theme                | Lingui (English and Bulgarian catalogs); locale cookie; light/dark theme; unified dark sidebar layout                                                                                                | ✅     |
| Member role editing         | Change member authorities beyond invitation default; transfer ownership (`TENANT_OWNER`)                                                                                                             | ✅     |
| Plan-based feature access   | Integrates with billing entitlements API for plan-based feature visibility and access control                                                                                                        | ✅     |
| Personal workspace handling | Hides personal workspace from org settings and nav; handles personal workspaces in entitlements & billing pages                                                                                      | ✅     |
| Demo credentials hint       | Shows demo credentials hint when VITE_DEMO_MODE is enabled                                                                                                                                           | ✅     |
| Billing setup form          | Shows setup form instead of error when billing settings not found                                                                                                                                    | ✅     |
| E2E testing                 | Restructured E2E test suite for scalability; added data-testid attributes to all key components; centralized test utilities                                                                          | ✅     |

---

## UI — Platform admin (`foundation-ui-platform-admin`)

Separate operator SPA (`PLATFORM_ADMIN` only). Platform-scoped JWT (`tenant_id` null). Deployed internally or behind restricted ingress.

| Screen / flow                     | Notes                                                                                                                                     | Status |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Admin sign-in                     | `POST /auth/admin/signin`; refresh via `/auth/admin/refresh`; `/unauthorized` without authority                                           | ✅     |
| Dashboard                         | Parallel count cards: users, organizations, active subscriptions                                                                          | ✅     |
| Users                             | Paginated list; detail with Overview, Organizations, Platform Authority, and OIDC Identities tabs; edit profile; set password             | ✅     |
| Organizations                     | Paginated list with status filter; detail with Overview, Members, Billing, Subscriptions, Refunds tabs                                    | ✅     |
| Invitations                       | Cross-tenant list; propose, edit, revoke                                                                                                  | ✅     |
| Subscriptions                     | Global list with search, status filter, sorting; detail view; cancel / pause / reactivate and quantity update                             | ✅     |
| Refunds                           | Global refund list and detail views                                                                                                       | ✅     |
| Plan catalog                      | Read-only list; plans are config-driven (YAML + deployment); no create/edit/deactivate UI                                                 | ✅     |
| Announcements                     | Create, edit, publish, delete announcements with multi-lingual translation support                                                        | ✅     |
| Audit logs                        | Global audit log view across all tenants; filterable by user, tenant, action                                                              | ✅     |
| Notifications                     | In-app notification list, unread badge, mark-as-read; real-time WebSocket push                                                            | ✅     |
| Operator account                  | View/edit operator profile; change password                                                                                               | ✅     |
| OIDC admin remediation            | User-detail OIDC identities tab; forced unmerge flow for linked external identities                                                       | ✅     |
| Session security                  | Access token in memory; refresh in `sessionStorage`; silent refresh; inactivity sign-out                                                  | ✅     |
| i18n                              | Lingui with English and Bulgarian catalogs; locale switcher UI                                                                            | ✅     |
| Platform actions                  | Ban/unban/unlock (done), impersonation                                                                                                    | 🚧     |
| System administration             | Health dashboard, background job monitoring                                                                                               | 📋     |
| Advanced metrics                  | MRR/ARR, growth charts on dashboard                                                                                                       | 📋     |
| Enterprise theme                  | Enterprise theme tokens, compact layout, unified dark sidebar with logo zone, surface depth, Paper/Card radius md, users row actions menu | ✅     |
| Read-only plan catalog            | Removed create/edit/deactivate UI; plan catalog is now read-only                                                                          | ✅     |
| Member signup trend chart         | Added member signup trend chart to dashboard                                                                                              | ✅     |
| Unified forms                     | Unify form handling, error handling, elements using Mantine Form + Zod                                                                    | ✅     |
| Dashboard widgets                 | Dashboard with subscription breakdown, audit feed, and org health cards; server-side severity and status filters                          | ✅     |
| Manage-platform-authority feature | Added manage-platform-authority feature                                                                                                   | ✅     |
| E2E testing                       | Reorganized E2E tests to FSD-aligned structure; added data-testid attributes to all key components; updated test selectors                | ✅     |

---

## UI — SaaS Landing Kit (`foundation-ui-saas-landing-kit`)

Astro + React + Tailwind CSS + DaisyUI + shadcn/ui landing page kit.

| Feature                 | Notes                                                                                                                                  | Status |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Plan selector component | Plan selector with feature comparison; fetches plans from API                                                                          | ✅     |
| Theme alignment         | Aligns look and feel with platform-admin theme; Inter/JetBrains Mono fonts, HSL slateGray/indigoAccent colors, structured border radii | ✅     |
| Documentation link      | Adds link to user documentation website                                                                                                | ✅     |
| Auth-aware navigation   | Top nav bar shows Login/Sign Up when unauthenticated; user menu when authenticated                                                     | ✅     |
| React islands           | Partial hydration for interactive components                                                                                           | ✅     |

---

## Infrastructure

| Capability           | Notes                                                                                                                                                                                   | Status |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Kubernetes           | Deployments with HPA, pod anti-affinity, network policies                                                                                                                               | 🚧     |
| Helm                 | Dedicated chart per service; env-specific value files (local/sit/uat/prd); PgBouncer sidecar for connection pooling                                                                     | ✅     |
| Docker Compose       | Per-service `compose.yaml` (infra only) + `compose.container.yaml` (full stack); demo `compose.demo.yaml` starts entire platform with Nginx, all services, and full observability stack | ✅     |
| Database-per-service | Each service owns its own PostgreSQL database; no cross-service table access                                                                                                            | ✅     |
| Schema-per-tenant    | PostgreSQL schema isolation per tenant within IAM and CMS databases; identical model in both single and multi-tenant modes                                                              | ✅     |
| Async provisioning   | RabbitMQ event-driven; ShedLock-guarded reaper for tenants stuck in `PROVISIONING`                                                                                                      | ✅     |
| Service template     | `foundation-microservice-project-layout` — Spring Boot 4.1, MyBatis, Liquibase, RabbitMQ, JWT, plan feature cache, quality gates pre-wired                                              | ✅     |
| Object storage       | MinIO S3-compatible; avatar uploads (IAM); presigned PUT URLs with expiry; per-tenant bucket prefixes; local dev via compose                                                            | ✅     |
| Cache / state store  | Redis available in demo stack (`foundation-redis :6379`); used for caching and distributed state                                                                                        | ✅     |
| Database admin UI    | DbGate in demo stack: unified web admin for PostgreSQL (×3), Redis, RabbitMQ, and MinIO                                                                                                 | ✅     |
| Secrets management   | K8s Secrets injected at deploy time via CI pipeline — never committed to source                                                                                                         | ✅     |
| TLS                  | cert-manager integration via Helm ingress values                                                                                                                                        | 🚧     |
| Observability        | Prometheus (Micrometer), Grafana dashboards (JVM + per-service), Loki + Promtail log aggregation, structured JSON logs, correlation ID propagated across all services                   | ✅     |
| CI/CD                | Drone: 10 pipelines per Java service, 4 per frontend, 2 per library module, 3 for infrastructure; SonarQube quality gate + PMD + SpotBugs on every build                                | ✅     |

---

## Planned (Post-v0.4)

### Current Open Items

| Capability                              | Notes                                                                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Platform Admin — subscription mutations | Change plan and apply discount from admin UI; cancel / pause / reactivate / quantity update are already implemented       |
| Platform Admin — advanced metrics       | MRR/ARR, growth charts, trends on dashboard                                                                               |
| Per-seat IAM enforcement                | Enforce purchased `seatCount` (not just plan `maxUsers`) at invite-accept and signup; `activeSeatCount` column on tenants |
| Additional locales                      | RU, IT (infrastructure already in place)                                                                                  |
| OIDC admin audit history                | Admin history / audit endpoints for identity-link and unmerge operations                                                  |
| OIDC token hardening                    | JWK-backed `id_token` validation beyond current nonce / claim checks                                                      |
| OIDC focused tests                      | Automated coverage for provisioning, state JWT, Redis PKCE store, and callback flows                                      |

### Later Milestones

| Capability                     | Notes                                                                                                                                   |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| SSO / SAML                     | Extension — subscribes to the event bus, not core                                                                                       |
| Rate limiting                  | Per-tenant and per-user (gateway)                                                                                                       |
| Tenant resolution by subdomain | Resolve tenant from request subdomain                                                                                                   |
| Usage-based billing metering   | Metered billing (`METERED` pricing model) on top of active gateway — per-seat flat pricing is implemented; metered aggregation deferred |
| Platform operator actions      | Impersonation in `foundation-ui-platform-admin`                                                                                         |
| System health dashboard        | Background job monitoring in `foundation-ui-platform-admin`                                                                             |
| Multi-region                   | Cross-region deployment support                                                                                                         |
| Managed hosting                | Hosted version of the platform                                                                                                          |
