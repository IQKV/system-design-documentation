# Roadmap

## v0.1 — Demo Release (completed)

Goal: all four components running on a public demo host with enough UI to evaluate the platform.

**Services**

- [x] IAM — registration, login, JWT RS256, tenant lifecycle, RBAC, email verification, password reset, brute-force lockout, token revocation, JWKS endpoint
- [x] IAM — platform rollout mode config (`platform.rolloutMode: MULTI_TENANT | SINGLE_TENANT`); single-tenant provisions default tenant at startup via bootstrap strategy
- [x] API Gateway — JWT validation, tenant resolution, routing
- [x] Billing — Stripe integration, webhook handling, lifecycle events, email notifications, plan catalog, multi-mode support
- [x] UI — React + Mantine, deployed as static build behind the Gateway

**UI screens required for demo**

- [x] Sign up / login / password reset
- [x] Dashboard (workspace overview, welcome message, member counts)
- [x] Team management (list members, view pending invitations)
- [x] Profile & Security (update profile, change password)
- [x] Invitation flow (send, list, revoke, accept)

**Infrastructure**

- [x] Helm chart per service with env-specific value files (local / sit / uat / prd)
- [x] Docker Compose for local dev (PostgreSQL, RabbitMQ, MailHog)
- [x] Drone CI/CD pipelines per service (verify → publish → deploy → promote)
- [x] Demo environment deployed and publicly accessible: [www.iqkv.site](https://www.iqkv.site)

---

## v0.2 — Administration, Self-Service & Observability (completed)

Goal: Platform administrators have full management tooling; tenants have billing self-service; platform has centralized audit trail.

**Platform Administration (`PLATFORM_ADMIN`)**

- [x] User management (list, view detail, edit, force-set password)
- [x] Organization management (list, view detail with Overview / Members / Billing / Subscriptions / Refunds tabs, edit metadata)
- [x] Invitation management (cross-tenant list, propose, revoke)
- [x] Plan catalog management (create, edit, deactivate plans)
- [x] Subscription management — global read-only list + detail view
- [x] Refunds — global refund list and detail views
- [x] Announcements — create, edit, publish, delete with multi-lingual translation support; async fan-out to all users
- [x] Audit service — centralized event-driven audit trail (`foundation-audit-service`) with `foundation-audit-spi`, JSONB storage, `PLATFORM_ADMIN`-restricted search API
- [x] Audit context propagation — Gateway injects `X-Audit-IP` / `X-Audit-UA`; domain services enrich outbound events via shared `MessagingService`
- [x] Global audit log view in Platform Admin UI
- [x] In-app notifications with real-time WebSocket push; notification bell UI

**Tenant Self-Service**

- [x] Billing portal integration (Stripe Customer Portal session)
- [x] Subscription management (view active subscription, plan catalog, billing info)
- [x] Refunds list in tenant billing UI
- [x] Tenant/workspace settings (organization metadata editing)
- [x] In-app notifications with real-time WebSocket push; notification bell UI

**IAM Enhancements**

- [x] Token exchange (`POST /auth/exchange`) — tenant switching without re-authentication
- [x] Avatar uploads — two-phase presigned S3/MinIO flow; old avatars auto-deleted
- [x] Site-wide announcements — multi-lingual, async fan-out in batches of 1000

**Localization & DX**

- [x] Runtime configuration injection (override `VITE_*` without rebuild)
- [x] Architecture enforcement (FSD boundary tests)
- [x] SaaS landing kit (`foundation-ui-saas-landing-kit`) — Astro + React + Tailwind CSS + shadcn/ui; static pages, auth-aware nav, React islands

---

## v0.3 — Plan Feature Access Control & Internationalization (In Progress)

Goal: Implement fine-grained plan-based feature access control and add internationalization support.

**Plan Feature Access Control**

- [x] YAML-based plan configuration as single source of truth
- [x] In-memory PlanFeatureRegistry for zero hot-path database calls
- [x] JWT plan_code claim propagation
- [x] Gateway X-Plan-Code header sanitization and propagation
- [x] PlanCatalogCache with 10-minute refresh cycle
- [x] RequiresPlanFeatureFilterFactory for route-level enforcement
- [x] maxUsers quota enforcement (invitation acceptance, signup)
- [x] User entitlements endpoint (`GET /api/v1/billing/entitlements/me`)
- [x] Internal plans endpoint (`GET /api/v1/billing/internal/plans`)
- [x] PlanFeatureGuard and PlanFeatureNotAvailableException
- [x] Subscription event handling with planCode propagation

**Internationalization**

- [x] Bulgarian (bg-BG) i18n translations added
- [x] Locale seed data for multi-lingual support
- [x] Announcements with multi-lingual translation support

**CMS Service**

- [x] Static page management with draft/published status
- [x] Multi-language support with en-US fallback
- [x] Hierarchical content structure with parent-child pages
- [x] SEO-friendly metadata (title, description, Open Graph tags, canonical URLs)
- [x] Tenant isolation with schema-per-tenant PostgreSQL architecture
- [x] Event-driven content lifecycle publishing to RabbitMQ
- [x] Public read-only API for published pages
- [x] Platform admin CRUD API for content management
- [x] Observability with Prometheus metrics and health checks

**Per-Seat Pricing**

- [x] `PricingModel` enum (`FLAT` / `PER_SEAT`) — new type in billing-service `plan` package
- [x] `StripeProductSchema` — optional `pricingModel` field; `effectivePricingModel()` defaults to `FLAT` (backward compatible)
- [x] `Plan` entity — `pricingModel` field; Liquibase migration adds `pricing_model VARCHAR(16) NOT NULL DEFAULT 'FLAT'` to `plan_catalog`
- [x] `PlanMapper` — `pricing_model` in all SELECT / INSERT / UPDATE statements
- [x] `BillingSeedRunner` — persists `pricingModel` from YAML at startup
- [x] `PlanFeatureRegistry` — `pricingModelForPlan()` O(1) lookup; `PlanCatalogEntry` bundles features + pricingModel; `allEntries()` for internal API
- [x] `SubscriptionService` — `resolveEffectiveQuantity` (FLAT → always 1; PER_SEAT → caller value ≥ 1) + `validateSeatCount` (throws `SeatLimitExceededException` HTTP 422 when seats > maxUsers)
- [x] `SubscriptionService.adjustSeats` — dedicated seat-change operation with proration; rejects non-PER_SEAT plans
- [x] `PATCH /api/v1/billing/subscriptions/{tenantKey}/{subscriptionId}/seats` — REST endpoint; 204 No Content; TENANT_OWNER or ADMIN authority
- [x] `SubscriptionEvent` — `seatCount` field (nullable Long); old publisher overloads deprecated; downstream consumers receive seatCount for PER_SEAT plans
- [x] Internal and public plans API — `pricingModel` exposed in both response DTOs
- [x] `foundation-iam-service` — `PlanFeatures` updated with `pricingModel` component, `isPerSeat()` helper, `has()` method; `PlanFeatureGuard` delegates to `features.has()`
- [x] `foundation-cms-service` and `foundation-microservice-project-layout` — `PlanFeatures` updated with `pricingModel` component and `isPerSeat()` helper
- [x] `foundation-ui-app` — `PricingModel` union type; `pricingModel` on Plan and PlanFeatures interfaces; plan card renders `/ seat / {period}` label for PER_SEAT plans

**Other Enhancements**

- [x] Spring Boot 4.1 upgrade across all services
- [x] Common Spring Web exception handlers
- [x] Tenant user stats endpoints (platform admin, owner/admin dashboard)
- [x] Advanced analytics plan feature gate
- [x] Audit of refund and subscription lifecycle events
- [x] Improved Stripe integration (plan codes, product retrieval/updates)
- [x] Personal workspace always accessible
- [x] Avatar presigned URL public endpoint rewrite
- [x] Skip tenant extraction for locales, announcements, /api-docs, /actuator, and WebSocket paths
- [x] Tenant App: plan-based feature access control integration, billing entitlements API, personal workspace handling, dark sidebar layout, demo credentials hint, E2E test refactoring and data-testid attributes
- [x] Platform Admin: read-only plan catalog UI, member signup trend chart, unified forms with Mantine Form + Zod, enterprise theme, dashboard widgets (subscription breakdown, audit feed, org health), E2E test refactoring and data-testid attributes, manage-platform-authority feature
- [x] SaaS Landing Kit: plan selector component, fetch plans from API, theme alignment with platform-admin, link to user documentation
- [x] Trial period support: added `trialPeriodDays` to plan catalog, Stripe checkout uses plan's trial, `isInTrial` and `trialDaysLeft` in subscription responses, UI shows trial badges and status

---

## Post-v0.3 — Scaling & Advanced Features

Items deferred until the core platform is fully manageable:

- Platform Admin UI — system health dashboard, background job monitoring
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Tenant App — additional locales (RU, IT; infrastructure already in place)
- SSO / SAML adapter (extension, not core)
- Rate limiting (per-tenant and per-user, at Gateway)
- Tenant resolution by subdomain
- Usage-based metered billing (`METERED` pricing model; per-seat flat pricing is complete)
- IAM per-seat enforcement — enforce purchased `seatCount` (not just plan `maxUsers`) at invite-accept and signup; requires `activeSeatCount` on tenant + `SubscriptionEventConsumer` update in IAM
- Multi-region support
- Managed hosting offering
