# Roadmap

## v0.1 — Demo Release (completed)

Goal: all four components running on a public demo host with enough UI to evaluate the platform.

**Services**

- [x] IAM — registration, login, JWT RS256, tenant lifecycle, RBAC, email verification, password reset, brute-force lockout, token revocation, JWKS endpoint
- [x] IAM — platform rollout mode config (`platform.rolloutMode: MULTI_TENANT | SINGLE_TENANT`); single-tenant provisions default tenant at startup via bootstrap strategy
- [x] API Gateway — JWT validation, tenant resolution, routing
- [x] Billing — Stripe integration, webhook handling (5 event types), lifecycle events, email notifications, plan catalog, multi-mode support
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

## v0.2 — Administration & Self-Service (current focus)

Goal: Provide platform administrators with management tools and tenants with billing self-service.

**Platform Administration (`PLATFORM_ADMIN`)**

- [x] User management (list, view detail, edit, force-set password)
- [x] Organization management (list, view detail, edit metadata)
- [x] Invitation management (cross-tenant list, propose, revoke)
- [x] Plan catalog management (create, edit, deactivate plans)
- [x] Audit service — centralized event-driven audit trail with `foundation-audit-spi`, JSONB storage, `PLATFORM_ADMIN`-restricted search API
- [x] Audit context propagation — Gateway injects `X-Audit-IP` / `X-Audit-UA`; domain services enrich outbound events via shared `MessagingService`
- [ ] Platform actions (ban/unban, unlock)
- [ ] Subscription lifecycle management (change plan, cancel, apply discounts)
- [ ] System health & background job monitoring

**Tenant Self-Service**

- [ ] Billing portal integration (Stripe Customer Portal)
- [ ] Subscription management (view details, change plan, cancel)
- [ ] Tenant/workspace settings (rename, delete workspace)
- [ ] Member role editing (promote to Admin, transfer Ownership)

**Localization & DX**

- [x] Runtime configuration injection (override `VITE_*` without rebuild)
- [x] Architecture enforcement (FSD boundary tests)
- [x] SaaS landing kit (`foundation-ui-saas-landing-kit`) — Astro + React + Tailwind CSS + shadcn/ui; static pages, auth-aware nav, React islands
- [ ] Additional locales compiled and available (RU, IT)
- [ ] Global audit log (UI surface in Platform Admin)

---

## Post-v0.2 — Scaling & Advanced Features

Items deferred until the core platform is fully manageable:

- Platform Admin UI — ban/unban, unlock, user impersonation (admin support tool)
- Platform Admin UI — subscription lifecycle mutations (change plan, cancel, reactivate, apply discount)
- Platform Admin UI — system health dashboard, background job monitoring, global audit log
- Platform Admin UI — advanced dashboard metrics (MRR/ARR, growth charts, trends)
- Tenant App — billing self-service (Stripe Customer Portal integration)
- Tenant App — workspace settings (rename, delete) and member role editing
- Tenant App — additional locales (RU, IT; infrastructure already in place)
- SSO / SAML adapter (extension, not core)
- Rate limiting (per-tenant and per-user, at Gateway)
- Tenant resolution by subdomain
- Usage-based billing metering
- Multi-region support
- Managed hosting offering
