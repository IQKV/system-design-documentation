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

## Post-v0.2 — Scaling & Advanced Features

Items deferred until the core platform is fully manageable:

- Platform Admin UI — ban/unban (done), unlock, user impersonation (admin support tool)
- Platform Admin UI — subscription lifecycle mutations (change plan, cancel, reactivate, apply discount)
- Platform Admin UI — system health dashboard, background job monitoring
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Tenant App — member ban/unban (done), member role editing (promote to Admin, transfer Ownership)
- Tenant App — additional locales (RU, IT; infrastructure already in place)
- SSO / SAML adapter (extension, not core)
- Rate limiting (per-tenant and per-user, at Gateway)
- Tenant resolution by subdomain
- Usage-based billing metering
- Multi-region support
- Managed hosting offering
